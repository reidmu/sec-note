# Jackson
## Jackson 概念
Jackson是一款基于Java平台的开源数据处理框架，可以方便的实现 Json 对象和 Java 对象的相互转换，很多Java应用都使用Jackson框架进行Json对象处理。

## Jackson 安全问题
Jackson的漏洞主要集中在`jackson-databind`中，当启用`Global default typing`，类似于FastJson的`autoType`，会存在各种各样的反序列化绕过类，而官方更新的防护措施一般都是将新出现的恶意类加入黑名单。

## Jackson 历史漏洞
- [CVE-2017-7525 Jackson-databind 反序列化漏洞](https://mp.weixin.qq.com/s/PPVbQVvlDLY8dRGLqLt2LQ)
  - FasterXML Jackson-databind < 2.6.7.1
  - FasterXML Jackson-databind < 2.7.9.1
  - FasterXML Jackson-databind < 2.8.9   
- [CVE-2019-12384 SSRF -> RCE](https://blog.csdn.net/xuandao_ahfengren/article/details/106967892)
  - FasterXML Jackson-databind 2.0.0 ~2.9.9.1
- [CVE-2020-36188 JNDI注入](https://zhuanlan.zhihu.com/p/344568487)
  - FasterXML Jackson-databind 2.x < 2.9.10.8
