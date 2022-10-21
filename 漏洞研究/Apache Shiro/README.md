# Apache Shiro

# 0x00 shiro简介
Shiro是一个Java平台的开源权限框架，用于认证和访问授权。

# 0x01 历史漏洞
- CVE-2016-4437 Shiro 550 反序列化漏洞
- CVE-2016-6802 身份认证绕过
- [CVE-2019-12422 Shiro 721 反序列化漏洞](https://github.com/inspiringz/Shiro-721)
- CVE-2020-1957 身份认证绕过
- CVE-2020-11989 身份认证绕过
- CVE-2020-13933 身份认证绕过
- CVE-2020-17510 身份认证绕过
- [CVE-2020-17523 身份认证绕过](https://github.com/jweny/shiro-cve-2020-17523)
- CVE-2021-41303 身份认证绕过

# 0x02 FOFA Dork
> header="rememberme=deleteMe" && country!="CN"

# 0x03 识别shiro
## 1、勾选 `Remember Me`，查看响应包
前台登录，注意需要勾选`Remember Me` ，抓包查看在返回包的 `Set-Cookie` 中存在 `rememberMe=deleteMe` 字段，存在shiro反序列化。

![image](https://user-images.githubusercontent.com/84888757/186797989-ec84736e-8343-424e-a840-d8c85496b5e3.png)

## 2、部分Shiro前端采用layer（web弹出层组件）
🌰 Sumap搜索语法：`title:"xx" && data:"layer"`

<div align=center><img src="https://user-images.githubusercontent.com/84888757/186798696-ddb8365c-d59c-4e29-b3be-5788e739c53c.png" /></div>

## 3、非默认配置的注意点
当java应用前台没有记住我功能时，响应包未发现shiro特征：

![image](https://user-images.githubusercontent.com/84888757/186798897-7f27a8ae-0b63-4512-a59f-2a5b475329ce.png)

在Cookie处添加`;rememberMe=1`
响应包发现shiro特征：`rememberMe=deleteMe;`

![image](https://user-images.githubusercontent.com/84888757/186798966-4d447957-6c80-443e-832b-d82029e179f7.png)

# 参考链接
- [攻防演练中的Shiro @notsec](https://mp.weixin.qq.com/s/BTqPYZkrrMiUtjodbXsrfw)
- [SummerSec /ShiroAttack2](https://github.com/SummerSec/ShiroAttack2)
  - shiro反序列化漏洞综合利用,包含（回显执行命令/注入内存马）修复原版中NoCC的问题
- [记一次从shiro-550到内网渗透的全过程](https://xz.aliyun.com/t/11201)
- [Shiro 历史漏洞分析](https://tttang.com/archive/1645/)

