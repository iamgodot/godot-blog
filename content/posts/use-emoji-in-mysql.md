---
title: Use emoji in MySQL
date: 2020-11-21T17:22:18+08:00
categories:
  - Code
tags:
  - mysql
draft: false
---

最近碰到一个服务器处理请求报错，但是本身的代码逻辑没有问题，排查后发现原来是参数中包含了 emoji，导致向 MySQL 插入数据时失败了。

解决起来倒是不麻烦，因为业务上不要求做相关的支持，所以捕捉异常之后返回错误码也就好了。但是好奇之下，做了个 MySQL 插入 emoji 的实验，发现里面还是有不少门道的。

# What's emoji 🧐

以前的粗浅理解就是由 unicode 支持的表情字符，这次认真查了下，找到了一篇讲解很详尽的文章：[Everything you need to know about emoji](https://www.smashingmagazine.com/2016/11/character-sets-encoding-emoji/)

根据里面的介绍，关于 emoji 比较官方的解释：

> Emoji are pictographs (pictorial symbols) that are typically presented in a colorful form and used inline in text. They represent things such as faces, weather, vehicles and buildings, food and drink, animals and plants, or icons that represent emotions, feelings, or activities.

而 emoji 是怎么来的呢：

> Emoji are "picture characters" originally associated with cellular telephone usage in Japan, but now popular worldwide. The word emoji comes from the Japanese 絵 (e ≅ picture) + 文字 (moji ≅ written character).

最关键的是，emoji 需要 4 个字节来表示，所以如果 MySQL 使用的是普通 utf8 字符集的话，是不足以表达一个 emoji 的信息的，所以自然会插入失败。

# Charset in MySQL

从 MySQL 的官方解释看，utf8mb4 支持最多每个字符四字节的编码方式，而对于超出 BMP（简单来说就是绝大部分的文字和符号字符，参考[维基百科](https://en.wikipedia.org/wiki/Plane_(Unicode))）范围的字符（比如 emojis），utf8mb3（也就是 utf8）是不够用的，所以要使用 utf8mb4 才可以。

![](https://static.iamgodot.com/content/images/2242299f3a78a9008c42e9162d4c31ee.png)

另外在 MySQL 中，charset 是有多个层级的设置的，所以如果需求明确的话，可以只对必要的 column/table 使用 utf8mb4，毕竟新的字符集也是要占用更多空间的。

如果想一步到位也是可以的，需要注意的是，MySQL 对字符集的设置大概分成下面几层，从上到下粒度也从大到小，而每一级如果没有显式设定的话会默认使用上一级的配置：

- character_set_server
- character_set_database
- table charset
- column charset

前两者可以使用 `show variables like "character%";` 查看，而后面两个则从数据表的创建语句中体现 `show create table t_name;`.

# More charset in MySQL

按理说完成上面的步骤之后，字段就可以完美支持 emoji 的插入了。但是打开一个客户端尝试插入 emoji 发现仍然会失败：

![](https://static.iamgodot.com/content/images/4ba81444f24f7e49aecb7cbf9f8605f1.png)

原因是还有其他的 charset 没有设置正确，通过查看变量可以确认：

![](https://static.iamgodot.com/content/images/f052fdc1a96804c876dc089077cdd657.png)

character_set_client 和 character_set_connection 这两项使用的还是 utf8，它们控制的是客户端在交互和传输过程中使用的字符集，更新之后插入成功，但是查询发现结果并没有展示正常的 emoji，而是表示乱码的问号：

![](https://static.iamgodot.com/content/images/66ff09d89c59a36755278c296462ae09.png)

这是因为还有一项 charset 没有设置正确，即 character_set_results，这是服务器在回传响应时使用的字符集，所以虽然成功储存了数据，但是展示出来却面目全非。直接更新设置，再次查询便一切正常了：

![](https://static.iamgodot.com/content/images/9bca21f21f1dc99235efa27a1e51e1d3.png)

# Django settings

在服务器开发中，通常与数据库交互的客户端都是服务器应用，所以也应当确保在代码或者框架中配置的 charset 是正确的。

在 Django 中是通过 DATABASES OPTIONS 来指定连接数据库使用的字符集的（针对 MySQL），示例如下：

```python
DATABASES = {
    "default": {
        # ...
        "OPTIONS": {
            "charset": "utf8mb4",
            # ...
        }
    }
}
```

# End

另外在查找资料的过程中还发现有其他的实现思路，比如如果不想改动数据库的话，可以在应用层进行转换，核心思想就是维护一个 emoji 和普通字符串的对应关系，然后把转换结果存储入数据表，这样的好处就是不破坏已有数据库的状态，同时对于应用层的转换逻辑也可以有更加灵活的控制。

当然，最好的做法还是在一开始对需求本身做好分析，如果有确定的存储要求，比如用户名或者评论应当支持 emoji，那么在建表时就直接使用类似 utf8mb4 的新式字符集，一劳永逸。

