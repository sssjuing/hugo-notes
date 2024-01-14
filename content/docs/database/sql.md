---
title: SQL教程
date: "2022-08-31"
---

## 聚合函数（Aggregate Functions）

```sql
SELECT
MAX(invoice_total) as max,
MIN(invoice_total) as min,
AVG(invoice_total) as avg,
SUM(invoice_total) as total_sales
FROM invoices;
```

```sql
SELECT
COUNT(payment_date) AS no_of_paid, -- 注意count不统计null值
COUNT(*) AS no_of_invoices -- 统计全部值
FROM invoices;
```

```sql
SELECT
    'Q1 2019' AS data_range,
    SUM(invoice_total) AS total_sales,
    SUM(payment_total) AS total_payments,
    SUM(invoice_total - payment_total) AS unpaid
FROM invoices
WHERE invoice_date BETWEEN '2019-01-01' AND '2019-03-31'
UNION
SELECT
    'Q2 2019' AS data_range,
    SUM(invoice_total) AS total_sales,
    SUM(payment_total) AS total_payments,
    SUM(invoice_total - payment_total) AS unpaid
FROM invoices
WHERE invoice_date BETWEEN '2019-04-01' AND '2019-06-30'
UNION
SELECT
    'Total' AS data_range,
    SUM(invoice_total) AS total_sales,
    SUM(payment_total) AS total_payments,
    SUM(invoice_total - payment_total) AS unpaid
FROM invoices
WHERE invoice_date BETWEEN '2019-01-01' AND '2019-06-30';
```

| data_range | total_sales | total_payments | unpaid |
| ---------- | ----------- | -------------- | ------ |
| Q1 2019    | 699.95      | 323            | 43     |
| Q2 2019    | 32          | 23             | 34     |
| Q3 2019    | 34          | 3              | 22     |
| Q4 2019    | 55          | 32             | 1      |
| Q1 2019    | 97          | 32             | 2      |
| Total      | 357         | 875            | 785    |

<br/>

## 分组查询

```sql
SELECT
    state, city
    SUM(invoice_total) AS total_sales
FROM invoices i
JOIN clients USING (client_id) -- 相当于 clients.client_id = invoices.client_id
WHERE invoice_date <= '2019-7-01'
GROUP BY state, city
ORDER BY total_sales DESC;
```

```sql
SELECT
    date,
    pm.name AS payment_method,
    SUM(amount) AS total_payments
FROM payments p
JOIN payment_methods pm
	ON p.payment_method = pm.payment_method_id
GROUP BY date, payment_method
ORDER BY date;
```

<br/>

## HAVING 语句

允许在 GROUP BY 语句后执行过滤

```sql
SELECT
    date,
    pm.name AS payment_method,
    SUM(amount) AS total_payments
FROM payments p
JOIN payment_methods pm
	ON p.payment_method = pm.payment_method_id
GROUP BY date, payment_method
HAVING total_payments > 500 AND no_of_invoices > 5
ORDER BY date;
```

```sql
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    SUM(oi.quantity * oi.unit_price) AS total_spent
FROM customers c
JOIN orders o USING (customer_id)
JOIN order_items oi USING (order_id)
WHERE city = 'Atlanta'
GROUP BY
    c.customer_id,
    c.first_name,
    c.last_name
HAVING total_spent > 100;
```

<br/>

## String Functions

```sql
SELECT LENGTH('MsSQL'); -- 5
SELECT UPPER('google'); -- GOOGLE
SELECT LOWER('GOOGLE'); -- google
SELECT TRIM('  banana   '); -- banana
SELECT LTRIM('  banana   '); -- banana
SELECT RTRIM('  banana   '); --   banana
SELECT LEFT('congratulations', 4); -- cong
SELECT RIGHT('congratulations', 4); -- ions
SELECT SPACE(6); -- 生成6个空格
SELECT REVERSE('abce'); -- dcba
SELECT REPEAT('*', 4); -- ****
SELECT CONCAT('abc', 'def', 'ghi'); -- abcdefghi 强制连接
SELECT CONCAT('abc', NULL, 'def', 'ghi'); -- NULL
SELECT CONCAT(14.3) -- 字符串的14.3
```

<br/>

## Mathematical Functions

ABS, CEILING, FLOOR, LOG, LOG10, POW, ROUND(四舍五入), TRUNCATE(截断), RAND(0-1 之间随机数)

```sql
SELECT ROUND(-1.24); -- -1
SELECT ROUND(-1.65); -- -2
SELECT ROUND(-1.23456, 2); -- -1.23
SELECT ROUND(2.2873, 2); -- 2.29

SELECT TRUNCATE(2.2878, 3); -- 2.287
SELECT TRUNCATE(122, -1); -- 120
```

<br/>

## DateTime Functions (过)

NOW, CURDATE, CURTIME, YEAR(NOW()), MONTH(NOW()), DAY(NOW()), HOUR(NOW()), DAYNAME('2020-01-31')(获取星期)

```sql
SELECT EXTRACT(YEAR_MONTH FROM '2020-01-31 05:06:07'); -- 202001

SELECT * FROM orders WHERE YEAR(order_date) >= YEAR(NOW());
```

<br/>

## IF 条件语句

```sql
SELECT order_id,
    IFNULL(shipper_id, 'Not Assigned') AS shipper
FROM orders;
```

```sql
SELECT order_id,
    COALESCE(shipper_id, comments, 'Not Assigned') AS shipper
    -- 依次查找shipper_id, comments, 找到就停止, 否则设置为 'Not Assigned'
FROM orders;
```

```sql
-- IF
  ## IF(condition, value_if_true, value_if_false)
SELECT
    order_id, order_date,
    IF(YEAR(order_date) = '2019', 'Active', 'Archived') AS category
FROM orders;
```

```sql
SELECT
    product_id, name,
    COUNT(*) as orders,
    IF(COUNT(*) > 1, 'More Than Once', 'Once') AS Frequency
FROM products
JOIN order_items USING(product_id)
GROUP BY product_id, name;
```

<br/>

## CASE 条件语句

```sql
SELECT
    order_id, order_date,
    CASE
        WHEN YEAR(order_date) = YEAR(NOW()) THEN 'this year'
        WHEN YEAR(order_date) = YEAR(NOW()) - 1 THEN 'last year'
        WHEN YEAR(order_date) = YEAR(NOW()) - 2 THEN 'two years ago'
        ELSE 'long long ago'
        END AS category
FROM orders;
```

```sql
SELECT
    CONCAT(first_name, ' ', last_name) AS customer,
    points,
    CASE
        WHEN points > 3000 THEN 'Gold'
        WHEN points >= 2000 THEN 'Silver'
        ELSE 'Bronze'
    END AS category
FROM customers;
```

<br/>

## Subquery 子查询

```sql
SELECT * FROM student
WHERE height > (SELECT AVG(height) FROM student);
```

### IN / NOT IN

```sql
SELECT * FROM products WHERE product_id NOT IN (SELECT product_id FROM order_items); -- 查询所有不在order_items表中的product
SELECT * FROM products LEFT JOIN order_items USING (product_id) WHERE invoice_id IS NULL; -- 与上面结论相同的JOIN写法
```

### ALL / ANY

```sql
SELECT * FROM invoices WHERE invoice_total > ALL(SELECT invoice_total FROM invoices WHERE client_id = 3);
-- 主句中的 invoice_total 大于子句中所有的
SELECT * FROM invoices WHERE invoice_total > (SELECT MAX(invoice_total) FROM invoices WHERE client_id = 3);
-- 与上句等价
```

此外，IN 和 ANY 也是等价的
