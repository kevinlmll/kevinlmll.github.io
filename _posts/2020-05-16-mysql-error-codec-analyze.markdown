---
layout:     post
title:      "MySQL一次数据拉取乱码的过程分析"
subtitle:   "\"哦，又见乱码。\""
date:       2020-05-16 12:00:00
author:     "kevinlmll"
header-img: "img/post-bg-2015.jpg"
tags:
    - mysql
---
#### 背景
周二（2019.11.19）时，业务上反馈，从xx的接口拉取数据会少量出现乱码，而且明确是一个emoji表情（🐠）有问题。

#### 过程
底层数据是用DB存储的，托管在公司的MySQL集群上。因此，第一时间想到的有两个点：
- 数据库表是否用"utf8mb4"作为字符集；
- 连接db里，charset是否指定为"utf8"；

此时，另一个同事贴了建表的DDL语句：
```sql
create table `xxxxx` (
   ...
) engine=innodb default charset=utf8mb4 comment='xxx'
```
![](/img/in-post/20200516_01_01.png)
明确了数据库表是用"utf8mb4"字符集。
同时，负责接口的同事也把连接db时的参数贴了出来，确认了连接时使用的字符集是"utf8"。
```
?charset=utf8&parseTime=true&loc=Local
```

考虑到以住的经验，**当数据库表用"utf8mb4"作为字符集时，连接的charset不能用"utf8"，要用"utf8mb4"或者系统默认（latin1）**。所以当时给了这个处理建议：**调整连接时使用的charset，改为"utf8mb4"**。然而同事在调整了对应的参数后，还是没有生效，新写入数据在读出时还是乱码，因此开启了一段关于MySQL字符集的探索之旅。

**剧透**：其实上面提到的，将连接时用的charset改为"utf8mb4"是没问题，真正没有生效的原因是同事使用了DB连接缓存池，调整charset是通过配置下发的方式，只有新建立的连接才能生效，旧的连接不会马上调整，因此整体服务还是用"utf8"读写数据。后来对接口进程进行重启后，新数据读写就不再有乱码了。

#### 踩坑
在处理过程中，我们是踩了两个坑的
- 数据库表是utf8mb4，连接用的是utf8，为什么写入的数据是乱码；
- 中间分析数据时，我们分别用python/golang拉取数据库里的原始数据，其中python能正常显示'🐠'，而golang却报错；

当时写入数据库的原始数据'🐠'，16进制是
```
\xed\xa0\xbd\xed\xb0\xa0，正常的编码应该是\xf0\x9f\x90\xa0
```
用python是可以正常输出
![](/img/in-post/20200516_01_02.png)

#### MySQL字符集探索
MySQL对utf8的支持有两类字符集:utf8和utf8mb4，两者的差异在于utf8的maxlen是**3**，而utf8mb4的maxlen是**4**，对于一些特殊的字符，例如emoji表情，需要用utf8mb4字符集才能存储。为了避免混乱，下面用**utf8mb3**代替utf8。

MySQL中字符集的转换受到3个系统变量控制[1]，如下：

| 系统变量 | 描述 |
| --- | --- |
| character_set_client | 服务器解码请求时使用的字符集 |
| character_set_connection | 服务器处理请求时会把请求字符串从*character_set_client* 转为 *character_set_connection* |
| character_set_results | 服务器向客户端返回数据时使用的字符集 |

另外要补充的点是，通过*set names encoding*的方式，是将以上3者都统一进行设置。
```sql
set names utf8  // 统一将character_set_{client,connection, result}设置为utf8
```

这几个系统变量在客户端请求服务端时的作用体现如下[1]：
![](/img/in-post/20200516_01_03.png)

利用上图这个流程，我们分析一下前文提到的乱码是怎么产生的。

| 原数据 | utf8(mb3) hex | utf8mb4 hex |
| --- | --- | --- |
| 🐠 | \xed\xa0\xbd\xed\xb0\xa0 | \xf0\x9f\x90\xa0 |

1. 操作系统默认的字符集一般是utf8（注意，这个是通用的utf8，不是MySQL里面的utf8），这时传输的数据是"**\xf0\x9f\x90\xa0**"；
2. 初始化连接DB时指定charset为utf8(utf8mb3，实际)，由于charset_client和charset_connection一致，所以这一步可以不转换；
3. 数据库表创建时指定的charset是utf8mb4，而charset_connection是utf8mb3，两者有差异，所以MySQL会对"**\xf0\x9f\x90\xa0**"做一次转换，变成"**\xed\xa0\xbd\xed\xb0\xa0**"写入数据库表；
4. 读取时，由于utf8mb4是utf8(mb3)的超集，因此，MySQL不会进行转换，客户端拿到的数据是"**\xed\xa0\xbd\xed\xb0\xa0**"；
5. 客户端无法识别"**\xed\xa0\xbd\xed\xb0\xa0**"，不是有效的utf8编码，展示变成乱码（如果用python的话，可以显示正常）；

那么，当我们把连接时的charset设置为utf8mb4时，也就解决了前面第3步里转换的bug，MySQL判断charset_connection与列的charset一致，不会做转换，数据正确。（问题：设置为latin1为什么也可以？）

#### python,golang对utf8的编解码处理
前面有提到，用python读取数据库里有问题的utf8数据时，是正常展示的，而且golang等其它方式则会报错，这是为什么呢？带着这个疑问，去翻了python 2.7版本和golang的源代码，如下：
python版，python放松了3字节utf8的限制，兼容（\xed\xa0\x80-\xed\xbf\xbf) 这段区间的值。
![](/img/in-post/20200516_01_04.png)
![](/img/in-post/20200516_01_05.png)
golang版，golang要求3字节的utf8,最低位的字节是在0x80～0xBF 之间
![](/img/in-post/20200516_01_06.png)
因此，对于「🐠」被转成3字节的utf8 "*\xed\xa0\xbd\xed\xb0\xa0*"，python能够完全兼容（代码里注释是可以uncomment if条件里的注释，使之非法），但是golang等其它则会报错。

#### 意料之外
上面的问题发现在公司的集群DB上，在自己机器的MySQL服务上试了下，结果出乎意料之外，只在设置utf8mb4数据才能正常写入。公司集群MySQL与自己机器上的MySQL差异，这个就无从追查了（T_T）。
自己机器上的MySQL
![](/img/in-post/20200516_01_07.png)
公司集群MySQL
![](/img/in-post/20200516_01_08.png)

#### 引用
[1] https://juejin.im/book/5bffcbc9f265da614b11b731 MySQL 是怎样运行的：从根儿上理解 MySQL--第4节

