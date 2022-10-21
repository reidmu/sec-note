# Fastjson 利用参考
- [Fastjson姿势技巧集合](https://github.com/safe6Sec/Fastjson)
- [fastjson不出网回显利用（fastjson-c3p0）](https://github.com/depycode/fastjson-c3p0)


# 历史漏洞
- Fastjson <= 1.2.24 反序列化 -> RCE
  - JdbcRowSetImpl
  - c3p0#JndiRefForwardingDataSource
  - shiro#JndiObjectFactory
  - shiro#JndiRealmFactory
  - bcel
- Fastjson <= 1.2.47 RCE

- 解决不出网利用
  - TemplatesImpl 利用条件苛刻
  - c3p0#WrapperConnectionPoolDataSource  fastjson <1.2.47

- Fastjson 1.2.36 - 1.2.62 正则表达式拒绝服务漏洞
- Fastjson1.2.5 <= 1.2.59 需要开启AutoType
- Fastjson1.2.5 <= 1.2.60 无需开启 autoType
- Fastjson1.2.5 <= 1.2.61 
- Fastjson <= 1.2.62
- Fastjson <= 1.2.66
- Fastjson <= 1.2.67
- Fastjson <= 1.2.68
