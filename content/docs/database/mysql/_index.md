---
title: MySQL 数据库
sidebar:
  exclude: true
---

`utf8mb3`：阉割过的`utf8`字符集，只使用 1 ～ 3 个字节表示字符。`utf8mb4`：正宗的`utf8`字符集，使用 1 ～ 4 个字节表示字符。有一点需要大家十分的注意，在`MySQL`中`utf8`是`utf8mb3`的别名，所以之后在`MySQL`中提到`utf8`就意味着使用 1~3 个字节来表示一个字符，如果大家有使用 4 字节编码一个字符的情况，比如存储一些 emoji 表情什么的，那请使用`utf8mb4`。