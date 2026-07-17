# 08-数据库SQL

> 课时：60 min | 难度：★★★☆☆

## 学习目标

1. 掌握CRUD核心语法与WHERE/HAVING条件过滤
2. 理解四种JOIN类型及区别，能根据需求选择正确JOIN
3. 掌握子查询、聚合函数与GROUP BY组合使用
4. 能够在测试场景中运用SQL进行数据准备、验证、清理

## 核心概念

### CRUD 语法

| 操作 | 关键字 | 语法模板 |
|------|--------|---------|
| 增（Create） | INSERT | `INSERT INTO table (col1, col2) VALUES (val1, val2)` |
| 查（Read） | SELECT | `SELECT col1, col2 FROM table WHERE condition` |
| 改（Update） | UPDATE | `UPDATE table SET col1=val1 WHERE condition` |
| 删（Delete） | DELETE | `DELETE FROM table WHERE condition` |

**完整示例（基于电商订单表）：**

```sql
-- 创建测试订单表
CREATE TABLE orders (
    order_id        INT PRIMARY KEY AUTO_INCREMENT,
    user_id         INT NOT NULL,
    product_id      INT NOT NULL,
    order_no        VARCHAR(32) UNIQUE NOT NULL,
    amount          DECIMAL(10,2) NOT NULL,
    status          TINYINT DEFAULT 0 COMMENT '0-待支付 1-已支付 2-已发货 3-已完成 4-已退款',
    create_time     DATETIME DEFAULT CURRENT_TIMESTAMP,
    pay_time        DATETIME,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status)
);

-- 插入测试数据
INSERT INTO orders (user_id, product_id, order_no, amount, status) VALUES
(1001, 2001, 'ORD202401150001', 299.00, 1),
(1001, 2003, 'ORD202401150002', 599.00, 0),
(1002, 2001, 'ORD202401150003', 299.00, 3),
(1003, 2005, 'ORD202401150004', 1299.00, 2),
(1002, 2002, 'ORD202401150005', 499.00, 4);

-- 查询所有已支付订单
SELECT order_no, amount, pay_time 
FROM orders 
WHERE status = 1;

-- 更新订单状态为已支付
UPDATE orders 
SET status = 1, pay_time = NOW() 
WHERE order_no = 'ORD202401150002';

-- 删除测试数据（谨慎操作，需加WHERE）
DELETE FROM orders WHERE order_no = 'ORD202401150005';
```

### WHERE vs HAVING

| 对比维度 | WHERE | HAVING |
|---------|-------|--------|
| 过滤对象 | 过滤行（原始数据） | 过滤分组（聚合结果） |
| 执行顺序 | 在分组前过滤 | 在分组后过滤 |
| 能否用聚合函数 | 不能 | 能 |
| 语法位置 | GROUP BY 之前 | GROUP BY 之后 |

```sql
-- WHERE：过滤单行数据（消费超过500的订单）
SELECT user_id, amount FROM orders WHERE amount > 500;

-- HAVING：过滤分组结果（总消费超过1000的用户）
SELECT user_id, SUM(amount) AS total 
FROM orders 
GROUP BY user_id 
HAVING total > 1000;
```

### JOIN 类型

```
INNER JOIN（内连接）：仅返回两表匹配行
  ┌─────┐     ┌─────┐
  │ A ●●●│●●● B │     ← 仅交集
  └─────┘     └─────┘

LEFT JOIN（左连接）：左表全部 + 右表匹配行（无匹配填NULL）
  ┌────────┐  ┌─────┐
  │ A ●●●●●│●●● B │  ← A全保留
  └────────┘  └─────┘

RIGHT JOIN（右连接）：右表全部 + 左表匹配行（无匹配填NULL）
  ┌─────┐  ┌────────┐
  │ A ●●●│●●●●●● B │  ← B全保留
  └─────┘  └────────┘

FULL OUTER JOIN（全外连接）：两表全部行（无匹配填NULL）
  ┌────────┐ ┌────────┐
  │ A ●●●●●│●●●●●● B │ ← 全部保留
  └────────┘ └────────┘
```

**语法示例：**

```sql
-- 关联查询：订单 + 用户信息
-- INNER JOIN：只返回有对应用户的订单
SELECT o.order_no, u.username, o.amount
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id;

-- LEFT JOIN：所有订单，包括已删除用户的订单
SELECT o.order_no, u.username, o.amount
FROM orders o
LEFT JOIN users u ON o.user_id = u.user_id;

-- WHERE 实现排除（找出没有用户的订单）
SELECT o.order_no, o.user_id
FROM orders o
LEFT JOIN users u ON o.user_id = u.user_id
WHERE u.user_id IS NULL;
```

** JOIN 速查表：**

| JOIN 类型 | 返回结果 | 典型场景 |
|----------|---------|---------|
| INNER JOIN | 交集 | 查询有效关联数据 |
| LEFT JOIN | 左表全部 + 右表匹配 | 主表数据完整保留 |
| RIGHT JOIN | 右表全部 + 左表匹配 | 以右表为主（少用，通常改用LEFT实现） |
| FULL JOIN | 并集 | 两表差异对比（MySQL不支持，用UNION模拟） |

### 子查询（Subquery）

```sql
-- WHERE 子查询：查找消费超过平均值的商品
SELECT product_id, amount 
FROM orders 
WHERE amount > (SELECT AVG(amount) FROM orders);

-- FROM 子查询：订单统计后的二次分析
SELECT status_type, COUNT(*) AS cnt
FROM (
    SELECT CASE 
        WHEN status = 0 THEN '待支付'
        WHEN status = 1 THEN '已支付'
        WHEN status = 2 THEN '已发货'
        WHEN status = 3 THEN '已完成'
        WHEN status = 4 THEN '已退款'
    END AS status_type
    FROM orders
) t
GROUP BY status_type;

-- EXISTS 子查询：查找有订单的用户（效率优于 IN）
SELECT user_id, username 
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.user_id
);
```

### 聚合函数与 GROUP BY

| 函数 | 功能 | 示例结果 |
|------|------|---------|
| COUNT() | 行数统计 | 订单总数 |
| SUM() | 求和 | 总销售额 |
| AVG() | 平均值 | 客单价 |
| MAX() | 最大值 | 最高单笔 |
| MIN() | 最小值 | 最低单笔 |

```sql
-- 每个用户的订单情况
SELECT 
    user_id,
    COUNT(*)                                                    AS order_count,
    SUM(amount)                                                 AS total_amount,
    AVG(amount)                                                 AS avg_amount,
    MAX(amount)                                                 AS max_order,
    MIN(create_time)                                            AS first_order,
    MAX(create_time)                                            AS latest_order
FROM orders
WHERE status != 4           -- 排除退款
GROUP BY user_id
HAVING order_count >= 2     -- 仅显示复购用户
ORDER BY total_amount DESC;
```

---

## 动手实操

### 数据准备（测试前）

```sql
-- 场景：测试"满500减50"优惠券功能，需构造不同金额订单

-- 构造低于门槛的订单
INSERT INTO orders (user_id, product_id, order_no, amount, status) VALUES
(2001, 3001, 'TEST_COUPON_001', 299.00, 1),
(2001, 3002, 'TEST_COUPON_002', 499.00, 1);

-- 构造等于门槛的订单
INSERT INTO orders (user_id, product_id, order_no, amount, status) VALUES
(2001, 3003, 'TEST_COUPON_003', 500.00, 1);

-- 构造超过门槛的订单
INSERT INTO orders (user_id, product_id, order_no, amount, status) VALUES
(2001, 3004, 'TEST_COUPON_004', 501.00, 1),
(2001, 3005, 'TEST_COUPON_005', 1299.00, 1);

-- 验证数据已就绪
SELECT order_no, amount FROM orders WHERE order_no LIKE 'TEST_COUPON_%';
```

### 数据验证（测试中）

```sql
-- 验证1：订单金额合计是否正确（应收 = 商品总价 - 优惠）
SELECT 
    o.order_no,
    o.original_amount,
    o.discount_amount,
    o.payable_amount,
    (o.original_amount - o.discount_amount)   AS calc_payable,
    CASE 
        WHEN o.payable_amount = (o.original_amount - o.discount_amount) 
        THEN 'PASS' 
        ELSE 'FAIL' 
    END                                        AS check_result
FROM orders o
WHERE o.order_no LIKE 'TEST_COUPON_%';

-- 验证2：支付后状态变更是否正确
SELECT 
    order_no,
    status,
    pay_time,
    CASE 
        WHEN status = 1 AND pay_time IS NOT NULL THEN 'PASS'
        WHEN status = 0 AND pay_time IS NULL     THEN 'PASS'
        ELSE 'FAIL'
    END AS check_result
FROM orders
WHERE order_no LIKE 'TEST_COUPON_%';

-- 验证3：优惠券使用次数是否超限（每人限用3张）
SELECT 
    user_id,
    COUNT(*) AS used_count,
    CASE WHEN COUNT(*) <= 3 THEN 'PASS' ELSE 'FAIL' END AS check_result
FROM coupon_usage
WHERE user_id = 2001
  AND use_time >= '2024-01-01'
GROUP BY user_id;
```

### 数据清理（测试后）

```sql
-- 清理测试数据（推荐软删除或带条件精确删除）
DELETE FROM orders WHERE order_no LIKE 'TEST_%';
DELETE FROM coupon_usage WHERE coupon_code LIKE 'TEST_%';

-- 或使用事务保证一致性
START TRANSACTION;
    DELETE FROM order_items WHERE order_id IN (
        SELECT order_id FROM orders WHERE order_no LIKE 'TEST_%'
    );
    DELETE FROM orders WHERE order_no LIKE 'TEST_%';
COMMIT;

-- 验证清理完成
SELECT COUNT(*) AS remaining_test_data FROM orders WHERE order_no LIKE 'TEST_%';
-- 预期结果：0
```

### ShopTest 订单验证SQL 五例

```sql
-- 【示例1】查找支付成功但未发货的异常订单（超过24小时）
SELECT 
    order_no,
    user_id,
    amount,
    pay_time,
    TIMESTAMPDIFF(HOUR, pay_time, NOW()) AS hours_since_pay
FROM orders
WHERE status = 1
  AND pay_time < DATE_SUB(NOW(), INTERVAL 24 HOUR)
ORDER BY pay_time ASC;

-- 【示例2】商品销量TOP10（验证排序功能）
SELECT 
    p.product_id,
    p.product_name,
    COUNT(o.order_id)   AS sold_count,
    SUM(o.quantity)     AS total_quantity,
    SUM(o.amount)       AS total_revenue
FROM products p
LEFT JOIN orders o ON p.product_id = o.product_id 
    AND o.status IN (1, 2, 3)       -- 仅有效订单
GROUP BY p.product_id, p.product_name
ORDER BY sold_count DESC
LIMIT 10;

-- 【示例3】用户退款率检测（识别恶意退款）
SELECT 
    user_id,
    COUNT(*)                                                AS total_orders,
    SUM(CASE WHEN status = 4 THEN 1 ELSE 0 END)           AS refund_orders,
    ROUND(SUM(CASE WHEN status = 4 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) 
                                                            AS refund_rate
FROM orders
GROUP BY user_id
HAVING total_orders >= 5
   AND refund_rate > 50        -- 退款率超50%且订单>=5
ORDER BY refund_rate DESC;

-- 【示例4】订单流水号连续性校验（发现数据丢失）
SELECT 
    a.order_no  AS current_order,
    b.order_no  AS next_order,
    TIMESTAMPDIFF(MINUTE, a.create_time, b.create_time) AS gap_minutes
FROM orders a
INNER JOIN orders b ON a.order_id + 1 = b.order_id
WHERE TIMESTAMPDIFF(MINUTE, a.create_time, b.create_time) > 60
ORDER BY gap_minutes DESC;

-- 【示例5】每日销售报表（验证统计准确性）
SELECT 
    DATE(create_time)                               AS order_date,
    COUNT(*)                                        AS order_count,
    COUNT(DISTINCT user_id)                        AS unique_buyers,
    SUM(amount)                                     AS total_gmv,
    ROUND(AVG(amount), 2)                          AS avg_order_value,
    SUM(CASE WHEN status = 4 THEN amount ELSE 0 END) AS refund_amount,
    SUM(amount) - SUM(CASE WHEN status = 4 THEN amount ELSE 0 END) AS net_gmv
FROM orders
WHERE create_time >= '2024-01-01'
  AND create_time < '2024-02-01'
GROUP BY DATE(create_time)
ORDER BY order_date DESC;
```

---

## 常见坑

| 错误 | 说明 |
|------|------|
| DELETE 后未加 WHERE | 清空整表数据，灾难性操作 |
| HAVING 替代 WHERE 过滤单行 | `HAVING amount > 100` 语法在部分数据库报错，应使用 WHERE |
| JOIN 条件遗漏导致笛卡尔积 | 忘记写 ON 条件，结果集膨胀至两表乘积 |
| NULL 值比较 | `WHERE col = NULL` 永远不匹配，应为 `WHERE col IS NULL` |
| GROUP BY 遗漏非聚合列 | SELECT 中出现非聚合列必须包含在 GROUP BY 中 |
| IN 子查询数据量过大 | IN 子查询返回大量数据时性能极差，改用 EXISTS 或 JOIN |
| 浮点数精度 | DECIMAL 与 FLOAT 混用导致金额计算偏差，金额字段统一用 DECIMAL |
| 未考虑时区 | create_time 时区不一致导致统计偏差，统一使用 UTC 存储 |

---

## 自测清单

- [ ] 能否独立完成包含 JOIN + GROUP BY + HAVING 的复合查询
- [ ] 能否区分四种 JOIN 的返回结果差异
- [ ] 能否正确选择 WHERE 与 HAVING 的过滤场景
- [ ] 能否使用子查询替代 JOIN 实现相同查询结果
- [ ] 能否编写测试数据准备的完整SQL（含构造与验证）
- [ ] 能否编写测试数据清理的事务SQL
- [ ] 能否发现订单统计SQL中的浮点精度问题

---

## 延伸阅读

- [菜鸟教程 - SQL 教程](https://www.runoob.com/sql/sql-tutorial.html)
- [菜鸟教程 - MySQL 教程](https://www.runoob.com/mysql/mysql-tutorial.html)
- [SQLZOO 练习](https://sqlzoo.net/)
- [LeetCode 数据库题库](https://leetcode.cn/problemset/database/)
- [MySQL 官方文档 - JOIN 语法](https://dev.mysql.com/doc/refman/8.0/en/join.html)
