# Atlassian Confluence
# 基本介绍
Confluence是一个专业的企业知识管理与协同软件，也可以用于构建企业wiki（多人协作的写作系统）。使用简单，但它强大的编辑和站点管理特征能够帮助团队成员之间共享信息、文档协作、集体讨论，信息推送。

# FOFA dork
> app="ATLASSIAN-Confluence"

# Confluence 历史漏洞
- OGNL表达式注入代码执行漏洞（CVE-2021-26084）
- Confluence路径穿越与命令执行漏洞（CVE-2019-3396）
- Atlassian Confluence 远程代码执行（CVE-2022-26134）

# Confluence 后利用思路
- 来自 https://github.com/pen4uin/pentest-note


- 修改数据库，实现用户登录
  - 修改用户登录口令
  - 修改Personal Access Tokens(by 3gstudent)
- 写文件
  - Windows: Confluence默认权限为network service，具有写权限
  - Linux： Confluence默认权限为confluence，没有写权限，可以尝试内存马

- 目录结构

![image](https://user-images.githubusercontent.com/84888757/172693115-6015845d-de84-4af6-804c-b825fde98e8a.png)


- 数据库配置文件 - `confluence.cfg.xml`

```
# Windows 默认路径
C:\Program Files\Atlassian\Application Data\Confluence\confluence.cfg.xml
# Ubuntu 默认路径
/var/atlassian/application-data/confluence/confluence.cfg.xml
```

![image](https://user-images.githubusercontent.com/84888757/172693620-4fcddc65-beb1-4a54-9c19-12e7b0655b0e.png)
