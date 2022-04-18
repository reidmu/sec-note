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

# 面试题
- Redis计划任务反弹shell失败原因？
  - 目标机器不出网
  - Redis向计划任务文件里写内容出现乱码而导致语法错误，这些乱码来自redis的缓存数据，无法避免，centos会忽略乱码去执行格式正确的任务计划，而ubuntu不会忽略这些乱码，导致命令执行失败。
  - 不是以root权限启动的redis
  - 容器化启动redis，没有计划任务文件
  - 因为默认 redis 写文件后是 644 权限，但是 Ubuntu 要求执行定时任务文件 `/var/spool/cron/crontabs/<username>` 权限必须是 600 才会执行，否则会报错 (root) INSECURE MODE (mode 0600 expected)，而 Centos 的定时任务文件 `/var/spool/cron/<username>` 权限 644 也可以执行
  - 由于系统的不同，crontrab 定时文件位置也不同
    - Centos 的定时任务文件在 `/var/spool/cron/<username>`
    - Ubuntu 的定时任务文件在 `/var/spool/cron/crontabs/<username>`
