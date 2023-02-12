# AK/SK 思路
# 0x01 概念
AK/SK用于在通过对象存储服务API（一个提供存储能力的web服务接口）访问存储数据时，生成鉴权信息进行安全认证。
AccessKey包括AccessKeyID和AccessKeySecret两部分，AccessKeyID用于标识用户，AccessKeySecret用于验证用户的密钥，主要用于程序方式调用云服务API。

🌰 使用AK/SK访问OSS
对象存储服务（Object Storage Service，OSS），OSS是一种云存储服务，适合存放任意类型的文件。

常见对象存储服务OSS厂商
- 腾讯云
- 七牛云
- 阿里云
- 百度bos
- AWS S3
- 又拍云
- Azure blob
- ......


# 0x02 AK/SK 获取思路
## 1、APK文件反编译
这是一道CTF题，思路：给出APK文件 -> 反编译 -> 翻文件搜索关键词查找敏感信息泄露

- [apktool工具](https://bitbucket.org/iBotPeaches/apktool/downloads/)：获取资源文件，可以提取出图片文件和布局文件等进行使用查看。
  - 包括源代码、图片、XML配置、语言资源等

- 查看配置文件：
  - 🌰 string.xml
    - strings.xml是一个字符串资源文件，所有的界面字符串应该在这个文件中指定，这样可以在一个位置管理所有界面字符串，让字符串的查找、更新和本地化变得更加容易，同时也可以节省空间资源。
    - string.xml一般在res\values路径下，既然是管理资源的文件，说到资源就会想到 -> 存储方式，可以在string.xml中尝试寻找对象存储服务（OSS）的AccessKey

![image](https://user-images.githubusercontent.com/84888757/161361936-e591e13c-6799-4176-af37-d8bdcc262094.png)

## 2、app.xxxx.js 中存在AccessKey

![image](https://user-images.githubusercontent.com/84888757/161361976-8ecd53f6-28d0-4194-a650-986b79080b8c.png)


## 3、Spring 泄露

- http://xxxx/actuator/env

🌰 OSS.accessKeyId

![image](https://user-images.githubusercontent.com/84888757/161363443-431ae3fb-ac86-4118-960f-2267d4d948f0.png)

🌰 vod.accessKeyId

![image](https://user-images.githubusercontent.com/84888757/161362607-ac55966f-cda0-4c8a-b14b-1c090566bb0c.png)

根据 [使用 MAT 查找 spring heapdump](https://landgrey.me/blog/16/)中的密码明文。
通过http://xxxx/actuator/heapdump 下载jvm heap信息，查找密码明文。

- [获取被星号脱敏的密码的明文](https://blog.csdn.net/weixin_45039616/article/details/106637978)

## 4、Github泄露
搜索目标信息关键词仓库，匹配accessKey等关键字。

## 5、后台
- ThinkCMF七牛

![image](https://user-images.githubusercontent.com/84888757/164454869-7a03c778-341b-49b2-8097-cad93f4051f2.png)

## 6、数据库
数据库弱口令 -> 翻config表找到AK/SK

![image](https://user-images.githubusercontent.com/84888757/171735217-ab8b4dd6-1960-4de0-8661-c2e9b18e5d38.png)

## 7、各种配置文件
文件包含/文件读取/各种其它漏洞 -> 读取配置文件
包括但不限于以下几类途径：
- jar包反编译 -> `.properties`文件
- apk反编译
- 小程序解包后获取的源码
- java的内存文件heapdump泄露

# 0x03 获取到 AK | SK 后如何连接或利用？
3.1 通过API接口
- 阿里云：https://api.aliyun.com/#/?product=Ecs
- 腾讯云：https://cloud.tencent.com/document/product/1340/52676

3.2 通过第三方管理工具
- [阿里云 - 图形化管理工具ossbrowser](https://help.aliyun.com/document_detail/92268.html)
- [阿里云 - 命令行ossutil工具下载链接](https://help.aliyun.com/document_detail/120075.htm)
  - 可用于在图形化工具查看完文件以后，找到重要数据的oss路径，再统计数目
  - `./ossutil ls oss://bucket_name/路径 -i ak -k sk -e https://oss-cn-hangzhou.aliyuncs.com`
- [七牛云 - 图形化工具 Kodo Browser](https://developer.qiniu.com/kodo/5972/kodo-browser)
- [行云管家 - 多种云存储服务管理](https://www.cloudbility.com/)
- [aksk_tool - AK资源管理工具](https://github.com/wyzxxz/aksk_tool)
- [cf - 云环境利用框架](https://github.com/teamssix/cf)

3.3 getshell
- [记一次实战阿里云主机泄露Access Key到getshell](https://cn-sec.com/archives/482553.html)

3.4 其它攻击思路
- [实战阿里云OSS云攻防 @RoBypass](https://mp.weixin.qq.com/s/MEodN2cUG55rOnBMz48LqA)
- [分享几个比较有意思的储存桶测试案例 @leishianquan](https://www.t00ls.com/thread-66044-1-1.html)
