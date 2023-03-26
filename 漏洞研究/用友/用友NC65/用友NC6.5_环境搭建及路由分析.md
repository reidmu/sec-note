# 用友NC6.5 - 环境搭建/调试环境/路由分析
# 0x01 环境搭建
🔗 [用友nc6.5详细安装过程](https://blog.csdn.net/weixin_38766356/article/details/103983787)

## 1.1 服务器与数据库
本次测试环境使用:

Windows Server 2012

SQL Server 2008 R2

数据库内容按照 🔗 [用友nc6.5详细安装过程](https://blog.csdn.net/weixin_38766356/article/details/103983787) 配置即可

## 1.2 安装并配置nc6.5服务端
创建好数据库以后，解压并运行安装包 `NC6.5` 文件夹下的 `setup.bat`
### 1.2.1 服务器信息配置
服务器选项可以配置JDK版本（需要与windows环境变量中的一致），还可以配置调试参数（点击读取应用服务器，在虚拟机参数后加入如下内容，再点击保存）

虚拟机参数(方便后续debug)

```
-server -Xmx768m -XX:PermSize=128m -XX:MaxPermSize=512m -Djava.awt.headless=true -Dfile.encoding=GBK -Xdebug -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5555
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219880698-912d548b-c111-438b-8a16-cb322039e270.png" /></div>

### 1.2.2 数据源
注：先点击读取，然后点击添加，需要填的信息全部填上，填不了的点击确定后会自动补齐

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219880728-7346626a-179d-4dcc-b1d4-4eb48591f119.png" /></div>

### 1.2.3 文件服务器
文件服务器不用管，安全数据源也是先读取，在保存即可。
### 1.2.4 部署
最后点击部署EJB，等待部署完成基本上就好了。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205448511-b72e45bc-dd68-4225-a84b-15a6192d1112.png" /></div>

### 1.2.5 启动服务端
在 `home` 目录下找到 `startup.bat` ，双击打开，有时你会发现它一闪而过，然后它就退出了，可以在命令行执行 `startup.bat`，有什么错误就知道了，多数是jdk目录的问题，jdk目录不能带有中文和空格。

如下启动完成：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205448595-9546c06f-9f6e-4dbb-b35f-7f953d558f2e.png" /></div>

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205448605-19fe2259-c29e-4997-9050-ff5bee7a62f1.png" /></div>

可以在浏览器访问了，下载UClient客户端：

<div align=center><img width="941" alt="image" src="https://user-images.githubusercontent.com/84888757/205448656-4fc86b01-3b40-41fa-83c0-9029655a437f.png" /></div>


## 1.3 安装客户端
下载客户端

<div align=center><img width="741" alt="image" src="https://user-images.githubusercontent.com/84888757/219880799-dd0d6e36-dedb-421e-b2f1-d6b5591fd1bb.png" /></div>

应用设置->编辑模式，添加JVM参数方便后续调试客户端->以超级管理员运行

这里标记的JVM参数和IDEA里设置的一致。

![image](https://user-images.githubusercontent.com/84888757/205449063-42b9d805-19bf-4ff1-8efe-08a0b52a9583.png)


注意安装目录，在如下路径生成对应app安装文件夹：
C:\Users\Administrator\AppData\Local\uclient\apps\7c57023b-a568-3854-bcde-ccc7f4c0a46f

![image](https://user-images.githubusercontent.com/84888757/205449139-4e4a0ef8-329e-4868-8137-55d521965940.png)

用IDEA打开该目录，导入其中的jar包：

![image](https://user-images.githubusercontent.com/84888757/205449153-6ed6817a-52f2-4150-9003-9f4a0a3e5f85.png)

打开客户端，随便输入账号密码点击登录后，查看 `app.log` 可以看到有 `login` 还有 `serialize` 等关键字：

![image](https://user-images.githubusercontent.com/84888757/205449185-50da9f45-fc27-45c7-9967-489d377b7303.png)

**登录系统**

如果要登录系统进行后续安装操作参考这个[链接](https://www.360docs.net/doc/5114992311-9.html)

然后我们使用 `super/空` 登录，这里设置新密码的时候，要求至少是8位，并且不能太简单。我们可以设置密码：

`super/nc65@2022`

<div align=center><img width="641" alt="image" src="https://user-images.githubusercontent.com/84888757/205449300-92a7efc4-fd2f-43f1-a35c-30819daaf8f6.png" /></div>

进入系统管理界面，新增帐套：系统管理->新增。


<div align=center><img width="641" alt="image" src="https://user-images.githubusercontent.com/84888757/205449358-a2f82528-86f6-4ca6-b17b-b3bbde147e0f.png" /></div>


输入帐套信息，帐套编码可以是3位，也可以是4位编码的。再新建一个管理员，输入失效日期和密码。见下图：

新管理员：`1/nc65@1111`

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205449445-2756e478-1a98-444f-813d-be8da45d83eb.png" /></div>

点击保存，弹出产品建库界面，选择待安装的产品，然后点击下一步。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205449454-943f47ba-1e0c-4e15-9907-90e289272e88.png" /></div>

开始根据选择的产品创建数据库。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205449466-d81c5bab-06c3-4826-a6bc-673675705147.png" /></div>

安装完毕后，提示重启中间件，重启后，可以使用管理员登陆系统。

（说明：如果您的帐套编号想修改，例如有三位的001想修改成0001，只需把现有的001删除，然后再新建一个0001，但是0001的系统名称一定要和001保持一致。新建0001后，直接保存即可，无须重启中间件。）

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205449487-f6ebf3d0-c556-4bcb-b3dd-18ea6f37fa25.png" /></div>

在 `home` 目录下找到 `startup.bat` ,双击打开：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205448595-9546c06f-9f6e-4dbb-b35f-7f953d558f2e.png" /></div>

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205449554-d64a5c22-96de-4a3f-bb1e-d3158e229c56.png" /></div>

## 1.4 IE浏览器直接访问
- “你的安全设置已阻止自签名应用程序运行”问题解决：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219883461-9afe6571-6815-46fc-b93e-92a8caa09df4.png" /></div>


- 访问系统管理员页面，是如下链接：
  - http://192.168.50.13:8888/admin.jsp

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219883478-5df50bfe-5be7-4f38-a6c2-0e6c23e748f7.png" /></div>

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219883494-f8e8cb5e-adfd-487f-95a9-4f9285fd782a.png" /></div>

# 0x02 调试环境
## 2.1 客户端调试-IDEA设置
IDEA设置如下：

记得对应服务端或者客户端启动后，idea才能远程调试，不然会显示拒绝连接。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205449682-a1589da3-17dd-48d0-ab29-9fef68a646d5.png" /></div>

启动客户端以后成功：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205449697-114ee4a4-9502-459f-a43e-ad7ef7f1b26e.png" /></div>

<div align=center><img src="https://user-images.githubusercontent.com/84888757/205449707-9ddc8b55-cdb8-4837-82c4-02ec55a08801.png" /></div>

## 2.2 服务端调试-IDEA设置

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219880940-89c1a2d2-c5c2-4faf-acbd-eeee1a15f531.png" /></div>

刚才在安装用友NC时，已经把idea里面的设置粘贴在虚拟机参数的后面：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219881009-55b2559c-a627-4d90-892c-7d8522d4e474.png" /></div>

点击调试以后：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219881037-07d73cff-6230-4442-b92b-7c03a107f242.png" /></div>

服务器端显示5555端口正在监听咯：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219881057-10866d97-f125-4da7-bbe3-ac04eef09b1d.png" /></div>

启动环境后测试一下是否可以进行正常调试，在 `webapps/nc_web/WEB-INF/web.xml` 中可以看到所有的请求都会经过 `LoggerFilter` 过滤器，所以我们就找这个过滤器对应的类 `LoggerServletFilter` 打个断点进行测试。

访问 http://192.168.50.13:8888/admin.jsp， 成功断点：

<div align=center><img src="https://user-images.githubusercontent.com/84888757/219881113-f61c4c63-0dc9-4033-8cef-fb8e213bc032.png" /></div>

# 0x03 路由分析
注意，这里进行的路由分析是针对服务端代码的（搜索路由都在服务端代码中搜索），我暂且取名为 `NC65_Server_home`。

客户端的代码会在jndi注入漏洞中分析。

用友的路由处理逻辑在 `web.xml` 中，`/servlet` 和 `/service` 开头的都路由到 `nc.bs.framework.server.InvokerServlet`中。

<div align=center><img width="641" alt="image" src="https://user-images.githubusercontent.com/84888757/205449804-1aa21dea-945c-4cfd-b72e-c2943e136889.png" /></div>


<div align=center><img width="741" alt="image" src="https://user-images.githubusercontent.com/84888757/205449808-416b0dd0-ce5b-4f39-be43-ca5e4bf8c7e4.png" /></div>

`nc.bs.framework.server.InvokerServlet` 的 `doAction` 处理路由逻辑如下：

`nc.bs.framework.server.InvokerServlet#doAction`

```
private void doAction(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
  String pathInfo = request.getPathInfo();
  log.debug("Before Invoke: " + pathInfo);
  long requestTime = System.currentTimeMillis();

  try {
     pathInfo = pathInfo.trim();
     String moduleName = null;
     String serviceName = null;
     int beginIndex;
     if(pathInfo.startsWith("/~")) {
        moduleName = pathInfo.substring(2);
        beginIndex = moduleName.indexOf("/");
        if(beginIndex >= 0) {
           serviceName = moduleName.substring(beginIndex);
           if(beginIndex > 0) {
              moduleName = moduleName.substring(0, beginIndex);
           } else {
              moduleName = null;
           }
        } else {
           moduleName = null;
           serviceName = pathInfo;
        }
     } else {
        serviceName = pathInfo;
     }
     
    String method;
    obj = this.getServiceObject(moduleName, serviceName);
    
    if(obj instanceof Servlet) {
        Logger.init(obj.getClass());

        try {
           if(obj instanceof GenericServlet) {
              ((GenericServlet)obj).init();
           }

           this.preRemoteProcess();
           ((Servlet)obj).service(request, response);
           this.postRemoteProcess();
           ...
     } else if(obj instanceof IHttpServletAdaptor) {
        IHttpServletAdaptor msg = (IHttpServletAdaptor)obj;
        this.preRemoteProcess();
        msg.doAction(request, response);
        this.postRemoteProcess();
```

获得 `pathinfo` 后，截取`/~`后的字符串，用 `/` 分割为 `moduleName` 和 `serviceName`，然后根据 `getServiceObject(moduleName, serviceName)` 去找 `Servlet` 类进行调用，相当于实现任意 `Servlet` 调用。

核心逻辑是截取出 `moduleName` 和 `serviceName` ，然后反射调用对应的 `Servlet` 。 `moduleName` 可以从 `/home/modules` 中查找。如果所调用的 `Servlet` 位于 `/home/lib` 的某个 `jar` 文件中，那么 `moduleName` 可以是 `modules` 中的任意一个。

🌰 比如http://1.1.1.1:8089/servlet/~ic/MonitorServlet
- `ic`是 `moduleName` ，是在 `/home/modules` 目录下的一个目录名。
- `MonitorServlet` 是 `serviceName`，是要被调用的 `Servlet` 类名，由于 `MonitorServlet.class` 在 `/home/lib` 目录，因此其他 `module` 也可以调用，比如：
  - `/servlet/~gl/nc.bs.framework.mx.monitor.MonitorServlet`
  - `/servlet/~sc/nc.bs.framework.mx.monitor.MonitorServlet`
- 这里有几种写法都能路由到同一 `servlet`，可以用来绕 `waf` 。
  - `/servlet/~ic/nc.bs.framework.mx.monitor.MonitorServlet`
  - `/servlet/~ic/MonitorServlet`
  - `/servlet/monitorservlet`
- `service` 和 `servlet` 均由 `NCInvokerServlet` 处理，因此用友NC的绝大部分漏洞都存在两种触发方式，比如 `monitorservlet` 。以下url均可触发。
  - `/servlet/monitorservlet`
  - `/service/monitorservlet`


🌰 比如http://1.1.1.1:8089/servlet/~uapss/com.yonyou.ante.servlet.FileReceiveServlet ，是调用的 `uapss` 模块下的 `FileReceiveServlet` 类。
- `/home/modules`目录下的 `jar` 包中的 `Servlet` ，需要使用对应的 `moduleName` ，一般也可以使用较为通用的 `ic` ，或者不指定 `moduleName` ，以 `FileReceiveServlet` 为例，就是 `uapss` 模块下的 `jar` 包中的 `Servlet` ，以下 `url` 均可触发。
  - `/servlet/~uapss/com.yonyou.ante.servlet.FileReceiveServlet`
  - `/servlet/~ic/com.yonyou.ante.servlet.FileReceiveServlet`
  - `/servlet/FileReceiveServlet`

来两张图看看实际路径：

  - 两下 `shift` 键搜索 `FileReceiveServlet` ，可以找到 `class` 文件。

![image](https://user-images.githubusercontent.com/84888757/205450064-117ddd34-6fcf-4ef6-8b4f-083917ed32dc.png)

![image](https://user-images.githubusercontent.com/84888757/205450066-071a0eea-8de2-4db0-8fad-df3b28f4214d.png)


  - `command+shift+F` 搜索 `FileReceiveServlet`

![image](https://user-images.githubusercontent.com/84888757/205450088-e9c2201c-f1b8-49b8-b8c4-f6856fb96782.png)


## accessProtected
要访问的路由是否需要 `token` 验证，看的是 `.upm` 文件中的 `accessProtected` 字段，如果为 `true` ，将会进行 `token` 验证。

<div align=center><img width="732" alt="image" src="https://user-images.githubusercontent.com/84888757/205450494-dc561d56-7e90-4d03-84d6-7a1746feaed1.png" /></div>


🌰：这里可以看到 `DeleteServlet` 的 `accessProtected` 设置为 `false`，不会进行 `token` 验证，可以直接访问到，通过类名访问对应路由。

![image](https://user-images.githubusercontent.com/84888757/205450138-6656fc1d-a643-4392-b120-69abb3f348e0.png)

这里用一下[工具yonyouNCTools.jar](https://github.com/Ghost2097221/YongyouNC-Unserialize-Tools/releases/download/YongyouNC-Unserialize-Tools/yonyouNCTools.jar)探测本次搭建的环境是否存在历史漏洞接口：

![image](https://user-images.githubusercontent.com/84888757/205450177-c0169492-0653-41d7-98f2-a8c97b50283a.png)

# 0x04 历史漏洞
```
# 反序列化漏洞
http://192.168.50.13:8888/servlet/mxservlet
http://192.168.50.13:8888/service/~uapss/nc.search.file.parser.FileParserServlet
http://192.168.50.13:8888/servlet/~uapss/com.yonyou.ante.servlet.FileReceiveServlet
http://192.168.50.13:8888/servlet/~aert/com.ufida.zior.console.ActionHandlerServlet
http://192.168.50.13:8888/servlet/~ic/uap.framework.rc.controller.ResourceManagerServlet
http://192.168.50.13:8888/servlet/~baseapp/nc.document.pub.fileSystem.servlet.DeleteServlet
http://192.168.50.13:8888/servlet/~baseapp/nc.document.pub.fileSystem.servlet.DownloadServlet
http://192.168.50.13:8888/servlet/~baseapp/nc.document.pub.fileSystem.servlet.UploadServlet
http://192.168.50.13:8888/service/~xbrl/XbrlPersistenceServlet

# JNDI 注入漏洞
http://192.168.50.13:8888/ServiceDispatcherServlet
 -> 参考：https://drea1v1.github.io/2020/06/17/%E7%94%A8%E5%8F%8Bnc%E8%BF%9C%E7%A8%8B%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/

# 目录遍历
http://192.168.50.13:8888/NCFindWeb?service=IPreAlertConfigService&filename=

# Beanshell RCE
http://192.168.50.13:8888/servlet/~ic/bsh.servlet.BshServlet
```

# 0x05 参考链接
- [ax1sX/yongyou_NC_Audit](https://github.com/ax1sX/SecurityList/blob/main/Yongyou/yongyou_NC_Audit.md)
- [用友nc6.5详细安装过程](https://blog.csdn.net/weixin_38766356/article/details/103983787)
- [工具 YongyouNC-Unserialize-Tools](https://github.com/Ghost2097221/YongyouNC-Unserialize-Tools/releases/tag/YongyouNC-Unserialize-Tools)
