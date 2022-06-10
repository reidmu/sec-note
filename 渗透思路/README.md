## 🚩 未授权文件上传 -> 获取OAuth凭证 -> 获取敏感信息
- 网站页面看到新的图标和名称 -> 猜测是小程序、公众号或APP，最后关键字搜索找到是某小程序 -> 小程序反编译，得到API接口信息。

- 访问任意接口，从回显信息可知网站使用 `springframework.security.oauth2` 认证，一般情况下，不存在未授权访问漏洞，所以需要找到一个有效凭证进行越权测试。
  - 这里推荐两篇文章助于理解：
    - [理解OAuth 2.0](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
    - [登录认证(一) —— 几种登录方式简介](https://blog.csdn.net/qq_38895905/article/details/103338651)

```
Authorization
是一种http header ，有具体的格式要求 ，格式为 Authorizatio: type xxxx ，即类型空格内容。

type 分为两种，一种是 Basic，另外一种是 Bearer。

Basic
basic会在每次请求的时候将用户名与密码，使用base64编码后发送一次。此时服务端当发现是basic的时候就会对内容进行解码然后用：分隔用户名与密码，之后到数据库当中认证。因为我们每次请求都在https上，比较正规，不会被抓包。由于每次请求都会带着用户名和密码，也不用担心分布式部署到多个服务器时出现问题。缺点就是每次都要访问数据库，效率比较低。

Bearer
bearer后面接的是token，token就是一个令牌。比如在用户登录时用户名密码认证完成，服务端通过将用户信息和一段服务端知道的文本采用对称加密算法加密后发送给客户端，客户端在下一次请求时将token带着，服务器端就会对token进行解密。token不需要对数据库进行查询，依赖于加密算法不被攻破作认证。

```

![image](https://user-images.githubusercontent.com/84888757/171743755-0688e471-e3c7-4e0b-a1c9-3c9cadf6751a.png)

- 未授权文件上传 -> 获取凭证
  - 在小程序反编译得到的代码中搜索Bearer关键字，找到一处上传接口，通过代码审计发现该接口调用LT函数可获取一个凭证。
  - 猜测该文件上传接口为未认证情况下默认允许上传，但OAuth在这里强制校验 `Authorizaton` 认证字段，所以系统需在该上传功能被使用时生成一个token凭证。
  - 搜索LT函数名，找到可生成凭证的接口，构造数据包获得凭证信息 `xx-x-x-xx`。

- 从网站js获取大量后台路由信息 -> 访问任意后台路由，抓包，添加 `Authorizaton` 认证字段，即 `Authorizaton: Bearer xx-x-x-xx` ，获得大量敏感数据。


## 🚩 企业微信管理后台密钥泄露利用
🌰 [企业微信+腾讯IM密钥泄漏利用](https://xz.aliyun.com/t/11092)
- 任意文件上传 -> 配置文件获取企业微信的`id`和`appSecrest` -> 获取`access_token`

```
#腾讯企业微信企业id
qywx.corpid=xxxxxxxxxxx
#腾讯企业微信管理后台的应用密钥
qywxapplet.appSecret=xxxxxxxxxxxxxxxx
#腾讯企业微信管理后台绑定的小程序appid
qywxapplet.appid=xxxxxxxxxxx
#腾讯ocr appid,演示环境使用了腾讯的ocr接口，行方不使用腾讯ocr接口则不必配置这里。配置成"-"即可
ocr.tenc.appId=-
```
  - 企业微信调试工具：https://open.work.weixin.qq.com/wwopen/devtool/interface/combine
    - access_token 的有效期通过返回的 expires_in 来传达，正常情况下为 7200 秒（2 小时），有效期内重复获取返回相同结果，过期后获取会返回新的 access_token。

![image](https://user-images.githubusercontent.com/84888757/171748775-37ccf625-9755-4122-8ad3-b1464e1a749d.png)

  - 查看 access_token 权限（可查看是否属于目标资产）
    - 错误码查询工具：https://open.work.weixin.qq.com/devtool/query

![image](https://user-images.githubusercontent.com/84888757/171749749-07ca48f8-b943-4169-8b93-6ddc6771b09f.png)


## 🚩 打点 - SQL注入 + XSS -> getshell
- 整体利用思路
  - 目标站点[伪静态注入](https://mp.weixin.qq.com/s/2f7Lg6foWha5hMtrD-Ptbg)获取sql-shell，修改数据库日志文件位置 -> 旁站数据库交互点插入一句话木马 -> payload被记录到数据库日志文件中 -> getshell

- 目标站点存在伪静态注入
  - sqlmap获取到sql-shell(mysql)
    - 直接写入shell失败(into outfile/dump)，猜测函数被禁用
    - 目录遍历 -> 发现历史遗留木马 -> 得知该路径下有写入权限
    - sqlmap成功修改数据库日志路径，日志写入shell失败
      - 手工sql注入携带php一句话无法正常提交网页数据，返回报错页面
      - 当前存在伪静态注入页面的表单其它数据点携带php一句话，`<>` 会被转义（后期拿到shell查看日志发现的）
      
  - 查询到web管理后台账号密码，登录失败，原因不详

- 旁站搜索框存在一处XSS
  - 与目标站点使用同一数据库
  - 存在搜索框、且与数据库存在交互
  - XSS -> 没有过滤尖括号 `<>`
    - 意味着可以插入php一句话 -> 会记录到日志文件中 -> getshell


## 🚩 打点 - 任意文件读取->暴破.bash_history

- 🔗 [渗透中 /etc/passwd 文件的进一步利用思路](http://blog.leanote.com/post/snowming/98864b36fc08)
- 🔗 [实例：中国电信某业务可文件遍历(泄漏数据库配置，外网redis可连接)](https://wy.zone.ci/bug_detail.php?wybug_id=wooyun-2015-091444)
1. 任意文件读取/etc/passwd
2. 提取passwd第一列，即root等一系列用户名

![image](https://user-images.githubusercontent.com/84888757/173130643-8c18eaa0-faf0-4f45-bdf7-196ef2b51db6.png)

3. 读history：../../root/.bash_history
4. 暴破所有用户的.bash_history：../../../home/§root§/.bash_history


## 🚩 打点-文件下载/信息收集 mlocate.db
`locate` 是用来查找文件或目录的，它不搜索具体目录，而是搜索一个数据库`mlocate.db`

`mlocate.db` 含有本地所有文件信息。Linux系统自动创建这个数据库，并且每天自动更新一次。

`mlocate.db` 的路径：`/var/lib/mlocate/mlocate.db`

我们利用任意文件下载漏洞将`mlocate.db`文件下载下来，利用`locate`命令将数据输出成文件，这里面包含了全部的文件路径信息。

下载以后可以使用 `mlocate -d mlocate.db "nginx"` 或 `locate -d mlocate.db "nginx"` 命令读取nginx相关的目录

```
root@pentest:~# mlocate -d mlocate.db "nginx"
/opt/metasploit-framework/embedded/framework/modules/auxiliary/scanner/http/nginx_source_disclosure.rb
/opt/metasploit-framework/embedded/framework/modules/exploits/linux/http/nginx_chunked_size.rb
/opt/metasploit-framework/embedded/lib/ruby/gems/3.0.0/gems/puma-5.5.2/docs/nginx.md
```

`-d`命令用来指定`mlocate.d`b文件，指定从目标下载下来的db文件即可。
后面跟"xxxxx" 就是要搜索的内容，比如如上命令搜索了nginx，路径或文件中包含nginx的文件和路径都会列出来。
也可以使用`-r`参数制定正则表达式搜索，比如搜索服务器上所有的`.conf`后缀的文件：

```
root@pentest:~# locate -d mlocate.db -r ".*\.conf"
root@pentest:~# mlocate -d mlocate.db -r ".*\.conf"
/etc/adduser.conf
/etc/ca-certificates.conf
/etc/ca-certificates.conf.dpkg-old
/etc/debconf.conf
/etc/deluser.conf
```

实战中可以搜索`.env`、`.conf`一类的文件，甚至搜索源代码路径，遍历下载源代码进行审计。

## 🚩 打点 - 重复发包
因为存在一些无法破解的加密算法，无法用burp直接重复发包，所以使用脚本控制鼠标和键盘代替手工点击和输入。
- mouse.py
```
# -*- coding: utf-8 -*-
'''
无法重复发包时，通过脚本控制鼠标键盘遍历工号重复发包（代替手工）
'''
import pyautogui
import pyautogui as pg 
# import cv2
# from PIL import Image
import numpy as np
import os
import time

time.sleep(5)

# 获取页面各元素的坐标
print (pg. position())

with open( 'pass.txt', 'r' ) as f1:
    list1 = f1. readlines( )
    for line in list1:
        line = line.strip()
# pyautogui.doubleClick(x=490，y=333) # 用户名
        pyautogui.click(x=1753, y=398)  # 密码
        pg .write(line)
        pyautogui.click(x=1450, y=413)
```




