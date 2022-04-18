# 信息收集的思路及工具
参考链接：
- 整体思路：
  - 子域名 -> 全端口 -> http、https -> 获取资产 -> 指纹识别
  - 小程序、公众号、APP
- [信息收集之“骚”姿势 @鱼七](https://zone.huoxian.cn/d/618)
- [做好信息收集是一次渗透测试的良好开端 @Paper-Pen](https://github.com/Paper-Pen/GatherInfo)

# 综合利用工具
- [水泽-信息收集自动化工具](https://github.com/0x727/ShuiZe_0x727)
- [SRC子域名资产监控](https://github.com/LangziFun/LangSrcCurise)
- [ARL(Asset Reconnaissance Lighthouse)资产侦察灯塔系统](https://github.com/TophantTechnology/ARL)
- [Goby](https://gobies.org)
- [Xray](https://github.com/chaitin/xray)
- [Nuclei](https://github.com/projectdiscovery/nuclei)

 
# 公司名资产收集
- [天眼查](https://www.tianyancha.com/)
- [小蓝本](https://www.xiaolanben.com/)
- [爱企查](https://aiqicha.baidu.com/)
- [企查查](https://www.qcc.com/weblogin?back=%2F)
- [鹰图](https://user.skyeye.qianxin.com/user/sign-in?next=https://hunter.qianxin.com/)
- [360威胁情报中心](https://ti.360.net/#/homepage)



# 子域名收集
- [phpinfo - 在线](https://phpinfo.me/domain/)
- [dnsgrep - 在线](https://www.dnsgrep.cn/subdomain)
- [bufferover - 在线](https://dns.bufferover.run/dns?q=baidu.com)
- [OneForAll](https://github.com/shmilylty/OneForAll)



# CDN判断
- [站长之家多地ping](http://ping.chinaz.com/)
- [ipip](http://tools.ipip.net/ping.php)



# IP反查域名（旁站查询）
- [360 ip反查](http://tools.ipip.net/ping.php)
- [dnsgrep ip反查](https://www.dnsgrep.cn/ip)
- [bugscaner ip反查](http://dns.bugscaner.com/203.107.33.157.html)



# 指纹识别
- 浏览器插件: Wappalyzer
- [潮汐 - 在线指纹识别](http://finger.tidesec.net/)
- [bugscaner - 在线指纹识别](http://whatweb.bugscaner.com/look/)
- [EHole - 红队重点攻击系统指纹探测工具](https://github.com/EdgeSecurityTeam/EHole)
- [云悉 - 在线指纹识别](https://www.yunsee.cn/)
- [what web - 在线指纹识别](https://www.whatweb.net/)


# js接口
- JSFinder: https://github.com/Threezh1/JSFinder
- LinkFinder: https://github.com/GerbenJavado/LinkFinder
- Packer-Fuzzer: https://github.com/rtcatc/Packer-Fuzzer (webpack)
- 搜索关键接口
  - config/api
  - method:"get"
  - http.get("
  - method:"post"
  - http.post("
  - $.ajax
  - service.httppost
  - service.httpget
  - path
  - api
  - 存在xxx.js.map文件时
    - 搜索 axios 关键字，查找路由


# APP
- [小蓝本](https://www.xiaolanben.com/pc)
- [七麦](https://www.qimai.cn)
- [AppStore](https://www.apple.com/app-store)



# WAF识别
- [WhatWaf](https://github.com/Ekultek/WhatWaf)
- [wafw00f](https://github.com/EnableSecurity/wafw00f)



# 敏感信息
## 网盘引擎
- [超能搜](https://www.chaonengsou.com)

## Googlehack语法
- 后台地址
  - site:xxx.com intitle:管理|后台|登陆|管理员|系统|内部
  - site:xxx.com inurl:login|admin|system|guanli|denglu|manage|admin_login|auth|dev
- 敏感文件
  - site:xxx.com (filetype:doc OR filetype:ppt OR filetype:pps OR filetype:xls OR filetype:docx OR filetype:pptx OR filetype:ppsx OR filetype:xlsx OR filetype:odt OR --filetype:ods OR filetype:odg OR filetype:odp OR filetype:pdf OR filetype:wpd OR filetype:svg OR filetype:svgz OR filetype:indd OR filetype:rdp OR filetype:sql OR filetype:xml OR filetype:db OR filetype:mdb OR filetype:sqlite OR filetype:log OR filetype:conf)
- 测试环境
  - site:xxx.com inurl:test|ceshi
  - site:xxx.com intitle:测试
- 邮箱
  - site:xxx.com (intitle:"Outlook Web App" OR intitle:"邮件" OR inurl:"email" OR inurl:"webmail")
- 其他
  - site:xxx.com inurl:api|uid=|id=|userid=|token|session
  - site:xxx.com intitle:index.of "server at"

## Github
  - @xxx.com password/secret/credentials/token/config/pass/login/ftp/ssh/pwd
  - @xxx.com security_credentials/connetionstring/JDBC/ssh2_auth_password/send_keys
