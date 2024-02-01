---
title: Postgresql jsonb类型按key排序问题
date: 2023-06-08 10:54:00 +0800
categories: [PostgreSQL]
tags: [postgresql]
mermaid: true
---

今天在做一个JSON入库的时候碰到一个问题，就是JSON存入时的数据和取出时的数据不一致，大概是这个样子

```json
//原JSON
{
    "name": "张三",
    "age": 18,
    "sex": "男",
    "address":"重庆市渝北区"
}
```

入库后再次使用API取出，发现键值对顺序变了

```json
//变化后JSON
{
    "address":"重庆市渝北区",
    "age": 18,
    "name": "张三",
    "sex": "男"
}
```

是的，你没看错，JSON按key进行了升序排列，一开始以为是.NET的API会对其进行排序存储，后面将entity字段改为 ***String*** 去存储顺序依然会发生变化。

再次将数据库字段修改为 ***text*** ,顺序正常了。后面翻了Postgresql的 [官方文档](https://www.postgresql.org/docs/current/datatype-json.html) 发现原来是 **jsonb**字段的问题

> Because the `json` type stores an exact copy of the input text, it will preserve semantically-insignificant white space between tokens, as well as the order of keys within JSON objects. Also, if a JSON object within the value contains the same key more than once, all the key/value pairs are kept. (The processing functions consider the last value as the operative one.) By contrast, `jsonb` does not preserve white space, does not preserve the order of object keys, and does not keep duplicate object keys. If duplicate keys are specified in the input, only the last value is kept.

大意如下，json类型存储时会保持原样，jsonb会移除空白内容且不会保持JSON数据原有顺序。将数据库类型修改为 json类型，问题解决~~

***jsonb*** 数据格式会压缩数据，以二进制的方式存储，效率比json更高，但同时也会移除 ***json*** 本身的一些内容，用的时候看具体业务需求。