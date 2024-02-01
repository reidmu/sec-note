# Cobalt Strike
- [Cobalt Strike @狼组安全](https://wiki.wgpsec.org/knowledge/intranet/Cobalt-Strike.html)
  - 无图，但很全

- [Cobalt Strike 学习笔记 @oatmeal](https://oatmeal.vip/tools/cobalt-strike/)


# Mimikatz
- [Mimikatz使用大全 ](https://www.cnblogs.com/-mo-/p/11890232.html)
- [了解和防御Mimikatz抓取密码的原理](https://cloud.tencent.com/developer/article/1838785)


# Sqlmap
- [Sqlmap使用手册](https://www.cnblogs.com/cainiao-chuanqi/p/15100351.html)

- [Sqlmap源码流程图](https://www.processon.com/view/5835511ce4b0620292bd7285)

- [Sqlmap源码浅析上 Sqlmap-1.3](https://ciyfly.github.io/2019/11/08/sqlmap%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90/)
  - 从最简单的参数-u分析各种Sqlmap函数
  - 分析Sqlmap如何判断各类sql注入

- [sqlmap源码浅析下](https://ciyfly.github.io/2021/06/11/sqlmap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%8B/)
  - 补充联合查询一个点
  - 以及sqlmap –os-shell -d api参数等分析

- [Sqlmap中的文件操作 @pen4uin](https://github.com/pen4uin/code-review-lab/tree/main/python/sqlmap)
  - 如何在pycharm中debug?
  - File system access的实现分析(以--file-write为例)

# BurpSuite
## BurpSuite配置TLS绕过ja3
- [by lu2ker](https://www.t00ls.com/viewthread.php?tid=70042)

- 旧版burp：`Project options` -> `TLS` -> `Use custom protocols and ciphers` ，修改下 `Cipher` ，这样通过自定义 `Cipher Suites` 可以简单快速的修改 `ja3` 特征。

不确定是不是通⽤的，原作者是遍历所有的 `TLS Ciphers` 并依次单个勾选，最终发现 `TLS_RSA_WITH_ AES_128_CBC_SHA` 以及附近相似的都可以。或许可以通过查看证书使⽤的签名算法来挑？

<div align=center><img width="631" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/588df562-16b3-4421-a045-08907cf1dc6e" /></div>

<div align=center><img width="1167" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/241f9b76-b0af-41b6-b4ad-38f39ac4cd49" /></div>


- 新版burp设置如下：

<div align=center><img width="1226" alt="image" src="https://github.com/reidmu/sec-note/assets/84888757/bc05466d-524e-4c82-8796-d6e7ffe3ba1e" /></div>


# 搜索引擎
- [各类搜索引擎语法](https://oatmeal.vip/tools/search-engine/)
