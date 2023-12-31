---
title: Use emoji in MySQL
date: 2020-11-21T17:22:18+08:00
categories:
  - Code
tags:
  - mysql
draft: true
---

最近碰到一个服务器报错，排查后发现是参数中包含了 emoji，导致数据库插入记录失败了。

虽然业务上不要求支持，但好奇之下，我还是基于 MySQL 做了个实验。

# What's emoji 🧐

关于 emoji 比较官方的解释：

> Emoji are pictographs (pictorial symbols) that are typically presented in a colorful form and used inline in text. They represent things such as faces, weather, vehicles and buildings, food and drink, animals and plants, or icons that represent emotions, feelings, or activities.

那么 emoji 是怎么来的呢：

> Emoji are "picture characters" originally associated with cellular telephone usage in Japan, but now popular worldwide. The word emoji comes from the Japanese 絵 (e ≅ picture) + 文字 (moji ≅ written character).

关于更详细的说明，可以阅读这篇文章：[Everything you need to know about emoji](https://www.smashingmagazine.com/2016/11/character-sets-encoding-emoji/)。

# Charset in MySQL

MySQL 中的 `utf8`（也就是 `utf8mb3`）最多只用 3 个字节编码，所以是无法支持 [BMP](https://en.wikipedia.org/wiki/Plane_(Unicode))（简单来说就是绝大部分的文字和符号）以外的字符的，emoji（需要 4 个字节表示）就是其中之一，此时就需要使用 `utf8mb4` 编码，如下图所示：

![](https://static.iamgodot.com/content/images/2242299f3a78a9008c42e9162d4c31ee.png)

Charset 有多个级别的设置，可以只对单表或者单列使用 `utf8mb4`，毕竟这种方式要占用更多的存储空间。

MySQL 对 Charset 的支持分为以下几种，从上到下的粒度依次减小，如果某一级没有显式设定的话会默认使用上一级的配置：

- `character_set_server`
- `character_set_database`
- `table character set`
- `column character set`

前两者可以在 `show variables like "character%";` 的结果中找到，后面两个（如果指定了的话）可以通过建表语句 `show create table t_name;` 查看。

# More charset in MySQL

按理说完成上述设置后字段就能够支持 emoji 了，但插入时却失败了：

![](https://static.iamgodot.com/content/images/4ba81444f24f7e49aecb7cbf9f8605f1.png)

原因是还有变量没设置正确：

![](https://static.iamgodot.com/content/images/f052fdc1a96804c876dc089077cdd657.png)

`character_set_client` 和 `character_set_connection` 的值还是 `utf8`，它们控制的分别是客户端在交互和传输过程中使用的字符集。更新之后插入成功，但查询结果却显示乱码：

![](https://static.iamgodot.com/content/images/66ff09d89c59a36755278c296462ae09.png)

这是因为 `character_set_results` 还没改过来，它表示服务器在回传响应时使用的字符集。再次更新后，便一切正常了：

![](https://static.iamgodot.com/content/images/9bca21f21f1dc99235efa27a1e51e1d3.png)

# Django settings

在服务器开发中，还要记得更改代码中的 Charset，比如 Django 里的配置：

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

---

除了更换字符集，也有应用层的解决方案。比如先维护 emoji 与普通字符串的映射，然后在存储前和查询后做转换。虽然麻烦些，但好处是不破坏现有数据库的状态，更加灵活地控制转换逻辑。

当然，如果能在一开始确认需求，比如为用户名或评论支持 emoji，然后建表时直接使用 `utf8mb4` 是最好的。
