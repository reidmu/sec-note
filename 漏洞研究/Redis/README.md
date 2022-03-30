# 参考链接
[Redis漏洞利用 -- 全](https://mp.weixin.qq.com/s/ungjainqdPhA_dDOOMKn4Q)

[Redis RCE 文章 @浅蓝](https://paper.seebug.org/1169/)

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
  - python反序列化
  - php反序列化

- 主从复制RCE

- Lua RCE
