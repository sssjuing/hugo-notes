---
title: MySQL 数据库
---

---

`utf8mb3`：阉割过的`utf8`字符集，只使用 1 ～ 3 个字节表示字符。`utf8mb4`：正宗的`utf8`字符集，使用 1 ～ 4 个字节表示字符。有一点需要大家十分的注意，在`MySQL`中`utf8`是`utf8mb3`的别名，所以之后在`MySQL`中提到`utf8`就意味着使用 1~3 个字节来表示一个字符，如果大家有使用 4 字节编码一个字符的情况，比如存储一些 emoji 表情什么的，那请使用`utf8mb4`。

## 命令行创建 MySQL 数据库并赋权限

首先输入以下命令进入到容器内部命令行：

```bash
docker exec -it <container_name> /bin/bash
```

之后输入以下命令进入到 MySQL：

```bash
mysql -u root -p
```

以下是相关的 MySQL 命令：

```sql
-- 创建用户 mysql_username 只允许从本地连接到数据库
CREATE USER 'mysql_username'@'localhost' IDENTIFIED BY 'mysql_username_password';

-- 创建用户 mysql_username 允许从远程连接到数据库
CREATE USER 'mysql_username'@'%' IDENTIFIED BY 'mysql_username_password';

-- 查看全部用户
SELECT user, host FROM mysql.user;

-- 删除用户
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'mysql_username'@'localhost';
FLUSH PRIVILEGES;
DELETE FROM mysql.user WHERE User = 'mysql_username' AND Host = 'localhost';

-- 创建数据库 my_database
CREATE DATABASE my_database CHARACTER SET utf8mb4;
SHOW DATABASES;

-- 给 mysql_username 赋予数据库 my_database 的全部权限
GRANT ALL PRIVILEGES ON `my_database`.* TO `mysql_username`@`%`;
SHOW GRANTS FOR 'mysql_username'@'%';
```

最后，按 Ctrl + D 退出容器命令行
