---
title: 注入攻击
date: "2022-08-31"
sidebar:
  exclude: true
---

## 识别漏洞

如下请求如果结果相同，说明很可能存在 SQL 注入漏洞

```
http://www.victim.com/showproducts.php?category=bikes # 原始请求
http://www.victim.com/showproducts.php?category=bi'||'kes # Oracle 和 PostgreSQL
http://www.victim.com/showproducts.php?category=bi'+'kes # MS SQLServer
http://www.victim.com/showproducts.php?category=bi' 'kes # MySQL
```
