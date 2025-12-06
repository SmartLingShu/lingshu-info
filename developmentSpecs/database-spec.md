# 数据库设计规范

> 适用于 MySQL 5.7/8.0，以性能、可扩展性为核心原则。

## 目录

- [1. 命名规范](#1-命名规范)
- [2. 表设计规范](#2-表设计规范)
- [3. 字段设计规范](#3-字段设计规范)
- [4. 索引设计规范](#4-索引设计规范)
- [5. SQL 编写规范](#5-sql-编写规范)
- [6. 性能优化规范](#6-性能优化规范)

---

## 1. 命名规范

### 1.1 表命名

```sql
-- 【强制】使用小写字母 + 下划线
-- 【强制】使用 t_ 前缀标识业务表
-- 【强制】使用单数形式

-- 正确示例
t_user              -- 用户表
t_reservation       -- 预约表
t_seat              -- 座位表
t_classroom         -- 教室表

-- 关联表命名：t_主表_从表_rel
t_user_role_rel     -- 用户角色关联表

-- 日志表命名：t_xxx_log
t_operation_log     -- 操作日志表
t_login_log         -- 登录日志表

-- 历史表命名：t_xxx_history
t_reservation_history  -- 预约历史表
```

### 1.2 字段命名

```sql
-- 【强制】使用小写字母 + 下划线
-- 【强制】禁止使用数据库关键字

-- 主键
id                  -- 自增主键或雪花ID

-- 外键
user_id             -- 用户ID
seat_id             -- 座位ID
classroom_id        -- 教室ID

-- 状态字段
status              -- 状态
is_deleted          -- 逻辑删除标识

-- 时间字段
created_time        -- 创建时间
updated_time        -- 更新时间
start_time          -- 开始时间
end_time            -- 结束时间

-- 人员字段
created_by          -- 创建人
updated_by          -- 更新人
```

### 1.3 索引命名

```sql
-- 主键索引：pk_表名
pk_user

-- 唯一索引：uk_表名_字段名
uk_user_phone
uk_user_email

-- 普通索引：idx_表名_字段名
idx_reservation_user_id
idx_reservation_status
idx_reservation_start_time

-- 联合索引：idx_表名_字段1_字段2
idx_reservation_user_id_status
idx_seat_classroom_id_status
```

---

## 2. 表设计规范

### 2.1 基础表结构

```sql
-- 【强制】每张表必须包含以下字段
CREATE TABLE t_user (
    id              BIGINT          NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    -- 业务字段...
    is_deleted      TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '逻辑删除:0-未删除,1-已删除',
    created_time    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_time    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    created_by      BIGINT          DEFAULT NULL COMMENT '创建人ID',
    updated_by      BIGINT          DEFAULT NULL COMMENT '更新人ID',
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户表';
```

### 2.2 表设计原则

```sql
-- 【强制】使用 InnoDB 存储引擎
-- 【强制】使用 utf8mb4 字符集（支持 emoji）
-- 【强制】使用 utf8mb4_unicode_ci 排序规则
-- 【强制】每个字段必须有注释
-- 【强制】单表字段数不超过 50 个
-- 【强制】单表数据量超过 500 万考虑分表
-- 【推荐】大字段（TEXT/BLOB）拆分到扩展表
```

### 2.3 示例：预约表设计

```sql
CREATE TABLE t_reservation (
    id              BIGINT          NOT NULL COMMENT '预约ID(雪花算法)',
    user_id         BIGINT          NOT NULL COMMENT '用户ID',
    seat_id         BIGINT          NOT NULL COMMENT '座位ID',
    classroom_id    BIGINT          NOT NULL COMMENT '教室ID',
    reservation_date DATE           NOT NULL COMMENT '预约日期',
    start_time      TIME            NOT NULL COMMENT '开始时间',
    end_time        TIME            NOT NULL COMMENT '结束时间',
    status          TINYINT         NOT NULL DEFAULT 0 COMMENT '状态:0-待签到,1-使用中,2-已完成,3-已取消,4-爽约',
    check_in_time   DATETIME        DEFAULT NULL COMMENT '签到时间',
    check_out_time  DATETIME        DEFAULT NULL COMMENT '签退时间',
    cancel_reason   VARCHAR(200)    DEFAULT NULL COMMENT '取消原因',
    remark          VARCHAR(500)    DEFAULT NULL COMMENT '备注',
    is_deleted      TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '逻辑删除',
    created_time    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_time    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    KEY idx_reservation_user_id (user_id),
    KEY idx_reservation_seat_id (seat_id),
    KEY idx_reservation_date_status (reservation_date, status),
    KEY idx_reservation_user_date (user_id, reservation_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='预约记录表';
```

---

## 3. 字段设计规范

### 3.1 数据类型选择

| 场景 | 推荐类型 | 说明 |
|------|----------|------|
| 主键 | BIGINT | 雪花算法或自增 |
| 状态/类型 | TINYINT | 0-127 足够 |
| 金额 | DECIMAL(10,2) | 避免浮点精度问题 |
| 短文本 | VARCHAR(n) | n 按实际需求设置 |
| 长文本 | TEXT | 超过 5000 字符 |
| 日期 | DATE | 仅日期 |
| 时间 | TIME | 仅时间 |
| 日期时间 | DATETIME | 日期+时间 |
| 布尔 | TINYINT(1) | 0/1 |
| IP地址 | INT UNSIGNED | 使用 INET_ATON 转换 |
| JSON | JSON | MySQL 5.7+ |

### 3.2 字段长度规范

```sql
-- 【强制】VARCHAR 长度按实际需求设置，不要随意设置过大
user_name       VARCHAR(50)     -- 用户名
nick_name       VARCHAR(50)     -- 昵称
phone           VARCHAR(20)     -- 手机号
email           VARCHAR(100)    -- 邮箱
password        VARCHAR(100)    -- 密码（加密后）
avatar          VARCHAR(255)    -- 头像URL
remark          VARCHAR(500)    -- 备注
description     VARCHAR(1000)   -- 描述
content         TEXT            -- 内容（长文本）
```

### 3.3 字段约束

```sql
-- 【强制】主键使用 NOT NULL
-- 【强制】外键字段使用 NOT NULL（除非业务允许为空）
-- 【强制】状态字段设置默认值
-- 【强制】时间字段设置默认值

status          TINYINT         NOT NULL DEFAULT 0
is_deleted      TINYINT(1)      NOT NULL DEFAULT 0
created_time    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP
```

---

## 4. 索引设计规范

### 4.1 索引原则

```sql
-- 【强制】主键索引：每张表必须有主键
-- 【强制】唯一索引：唯一业务字段必须建唯一索引
-- 【强制】外键索引：外键字段必须建索引
-- 【强制】查询索引：WHERE、ORDER BY、GROUP BY 字段建索引
-- 【强制】单表索引数量不超过 5 个
-- 【强制】联合索引字段数不超过 5 个
```

### 4.2 索引设计技巧

```sql
-- 【强制】遵循最左前缀原则
-- 联合索引 (a, b, c) 可用于：
-- WHERE a = ?
-- WHERE a = ? AND b = ?
-- WHERE a = ? AND b = ? AND c = ?
-- 不可用于：
-- WHERE b = ?
-- WHERE b = ? AND c = ?

-- 【强制】区分度高的字段放前面
-- 错误：idx_status_user_id（status 区分度低）
-- 正确：idx_user_id_status（user_id 区分度高）

-- 【强制】覆盖索引优化
-- 查询字段都在索引中，避免回表
SELECT user_id, status FROM t_reservation WHERE user_id = 1;
-- 索引：idx_user_id_status (user_id, status)

-- 【推荐】前缀索引（长字符串）
CREATE INDEX idx_email ON t_user(email(20));
```

### 4.3 索引示例

```sql
-- 用户表索引
CREATE TABLE t_user (
    id              BIGINT NOT NULL,
    user_name       VARCHAR(50) NOT NULL,
    phone           VARCHAR(20) NOT NULL,
    email           VARCHAR(100),
    status          TINYINT NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_user_phone (phone),
    UNIQUE KEY uk_user_email (email),
    KEY idx_user_status (status)
);

-- 预约表索引
CREATE TABLE t_reservation (
    id              BIGINT NOT NULL,
    user_id         BIGINT NOT NULL,
    seat_id         BIGINT NOT NULL,
    reservation_date DATE NOT NULL,
    status          TINYINT NOT NULL,
    PRIMARY KEY (id),
    KEY idx_reservation_user_date (user_id, reservation_date),
    KEY idx_reservation_seat_date (seat_id, reservation_date),
    KEY idx_reservation_date_status (reservation_date, status)
);
```

---

## 5. SQL 编写规范

### 5.1 查询规范

```sql
-- 【强制】禁止使用 SELECT *
-- 错误
SELECT * FROM t_user WHERE id = 1;
-- 正确
SELECT id, user_name, phone, status FROM t_user WHERE id = 1;

-- 【强制】禁止使用 SELECT ... FOR UPDATE 长时间锁表
-- 【强制】WHERE 条件必须使用索引字段
-- 【强制】避免在 WHERE 中对字段进行函数操作
-- 错误
SELECT * FROM t_reservation WHERE DATE(created_time) = '2024-01-01';
-- 正确
SELECT * FROM t_reservation 
WHERE created_time >= '2024-01-01 00:00:00' 
  AND created_time < '2024-01-02 00:00:00';

-- 【强制】避免隐式类型转换
-- 错误（phone 是 VARCHAR，传入数字会导致索引失效）
SELECT * FROM t_user WHERE phone = 13800138000;
-- 正确
SELECT * FROM t_user WHERE phone = '13800138000';
```

### 5.2 分页规范

```sql
-- 【强制】大数据量分页使用游标分页
-- 错误：深度分页性能差
SELECT * FROM t_reservation ORDER BY id LIMIT 1000000, 10;

-- 正确：游标分页
SELECT * FROM t_reservation WHERE id > 1000000 ORDER BY id LIMIT 10;

-- 正确：延迟关联
SELECT r.* FROM t_reservation r
INNER JOIN (
    SELECT id FROM t_reservation ORDER BY id LIMIT 1000000, 10
) t ON r.id = t.id;
```

### 5.3 更新规范

```sql
-- 【强制】UPDATE/DELETE 必须带 WHERE 条件
-- 【强制】UPDATE/DELETE 必须使用索引字段
-- 【强制】批量更新使用 CASE WHEN 或临时表

-- 批量更新状态
UPDATE t_reservation 
SET status = 4, updated_time = NOW()
WHERE reservation_date < CURDATE() 
  AND status = 0 
  AND is_deleted = 0;

-- 【强制】禁止一次性更新大量数据，分批处理
-- 每次更新 1000 条
UPDATE t_reservation 
SET status = 4 
WHERE id IN (
    SELECT id FROM (
        SELECT id FROM t_reservation 
        WHERE reservation_date < CURDATE() AND status = 0 
        LIMIT 1000
    ) t
);
```

### 5.4 插入规范

```sql
-- 【强制】批量插入使用 INSERT INTO ... VALUES
INSERT INTO t_reservation (id, user_id, seat_id, reservation_date, start_time, end_time, status)
VALUES 
    (1, 100, 200, '2024-01-01', '08:00:00', '10:00:00', 0),
    (2, 101, 201, '2024-01-01', '08:00:00', '10:00:00', 0),
    (3, 102, 202, '2024-01-01', '08:00:00', '10:00:00', 0);

-- 【强制】单次批量插入不超过 500 条
-- 【强制】使用 INSERT IGNORE 或 ON DUPLICATE KEY UPDATE 处理冲突
INSERT INTO t_user (id, user_name, phone) 
VALUES (1, 'zhangsan', '13800138000')
ON DUPLICATE KEY UPDATE user_name = VALUES(user_name);
```

---

## 6. 性能优化规范

### 6.1 查询优化

```sql
-- 【强制】使用 EXPLAIN 分析执行计划
EXPLAIN SELECT * FROM t_reservation WHERE user_id = 1;

-- 关注指标：
-- type: 至少达到 range 级别，最好是 ref
-- key: 使用的索引
-- rows: 扫描行数
-- Extra: 避免 Using filesort、Using temporary

-- 【强制】避免全表扫描
-- 【强制】避免索引失效场景：
-- 1. 对索引字段使用函数
-- 2. 对索引字段进行运算
-- 3. 使用 != 或 <>
-- 4. 使用 OR（改用 UNION）
-- 5. LIKE '%xxx'（改用全文索引）
-- 6. 隐式类型转换
```

### 6.2 JOIN 优化

```sql
-- 【强制】小表驱动大表
-- 【强制】JOIN 字段必须有索引
-- 【强制】避免超过 3 张表 JOIN
-- 【推荐】使用 EXISTS 代替 IN（大数据量）

-- 错误
SELECT * FROM t_user WHERE id IN (
    SELECT user_id FROM t_reservation WHERE status = 1
);

-- 正确
SELECT * FROM t_user u WHERE EXISTS (
    SELECT 1 FROM t_reservation r WHERE r.user_id = u.id AND r.status = 1
);
```

### 6.3 事务优化

```sql
-- 【强制】事务尽量短小
-- 【强制】避免大事务（超过 1 秒）
-- 【强制】事务内避免远程调用
-- 【强制】合理设置事务隔离级别

-- 默认使用 READ COMMITTED（减少锁冲突）
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### 6.4 锁优化

```sql
-- 【强制】避免锁表，使用行锁
-- 【强制】加锁顺序一致，避免死锁
-- 【强制】使用乐观锁处理并发更新

-- 乐观锁示例
UPDATE t_seat 
SET status = 1, version = version + 1 
WHERE id = 1 AND version = 1;

-- 【推荐】使用 Redis 分布式锁代替数据库锁
```

---

## 附录

### A. 常用 SQL 模板

```sql
-- 分页查询模板
SELECT id, user_name, phone, status, created_time
FROM t_user
WHERE is_deleted = 0
  AND status = #{status}
  AND user_name LIKE CONCAT('%', #{keyword}, '%')
ORDER BY created_time DESC
LIMIT #{offset}, #{pageSize};

-- 统计查询模板
SELECT 
    DATE(created_time) AS date,
    COUNT(*) AS total,
    SUM(CASE WHEN status = 1 THEN 1 ELSE 0 END) AS success_count
FROM t_reservation
WHERE created_time >= #{startDate}
  AND created_time < #{endDate}
  AND is_deleted = 0
GROUP BY DATE(created_time)
ORDER BY date;

-- 排行榜查询模板
SELECT 
    u.id,
    u.user_name,
    u.avatar,
    SUM(TIMESTAMPDIFF(MINUTE, r.check_in_time, r.check_out_time)) AS total_minutes
FROM t_user u
INNER JOIN t_reservation r ON u.id = r.user_id
WHERE r.status = 2
  AND r.reservation_date >= #{startDate}
  AND r.reservation_date <= #{endDate}
GROUP BY u.id
ORDER BY total_minutes DESC
LIMIT 10;
```

### B. 检查清单

- [ ] 表名、字段名是否符合规范
- [ ] 是否有主键
- [ ] 是否有必要的索引
- [ ] 是否有逻辑删除字段
- [ ] 是否有创建/更新时间字段
- [ ] 字段类型是否合适
- [ ] 字段长度是否合理
- [ ] 是否有字段注释
- [ ] SQL 是否使用索引
- [ ] 是否有慢查询风险
