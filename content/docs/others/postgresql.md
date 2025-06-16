---
title: PostgreSQL 数据库
date: "2025-06-16"
---

---

登录到数据库。如果是本地数据库，可以省略主机名和端口号。

```shell
psql -h <主机名或IP地址> -p <端口号> -U <用户名> -d <数据库名称>
```

快捷操作

```shell
-- 查看当前数据库中的所有表
\dt

-- 查看数据库列表
\l

-- 查看用户列表
\du

-- 切换到另一个数据库
\c other_database
```

向某个角色赋予某个数据库的全部权限

```shell
-- 创建用户
CREATE USER john WITH PASSWORD 'secret';

-- 创建数据库
CREATE DATABASE mydb;

-- 赋予用户对数据库的权限
GRANT ALL PRIVILEGES ON DATABASE mydb TO john;

-- 赋予用户对数据库中所有表的权限
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO john;

-- 赋予用户对数据库中所有表的默认权限
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO john;

-- 赋予用户对数据库中所有序列的权限
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO john;

-- 赋予用户对数据库中所有函数的权限
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO john;

-- 赋予用户对数据库中所有方案的权限
GRANT ALL PRIVILEGES ON SCHEMA public TO john;
```
