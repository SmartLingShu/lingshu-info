# 数据库设计规范

> 适用于 MySQL 5.7/8.0，以性能、可扩展性为核心原则，遵循阿里巴巴 Java 开发手册。

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
-- 【强制】使用小写字母 + 下划线（snake_case）
-- 【强制】禁止使用复数形式
-- 【强制】禁止使用 MySQL 保留字（如 order, user, desc 等）
-- 【禁止】不要使用 t_、tb_、tbl_ 等前缀（阿里规范明确禁止）

-- 正确示例
user                -- 用户表（如与保留字冲突可用 sys_user）
reservation         -- 预约表
seat                -- 座位表
classroom           -- 教室表

-- 与保留字冲突时的处理
sys_user            -- 用户表（user 是 MySQL 保留字）
sys_order           -- 订单表（order 是 MySQL 保留字）

-- 关联表命名：主表_从表
user_role           -- 用户角色关联表

-- 日志表命名：xxx_log
operation_log       -- 操作日志表
login_log           -- 登录日志表

-- 历史表/归档表命名：xxx_history / xxx_archive
reservation_history -- 预约历史表
```

### 1.2 字段命名

```sql
-- 【强制】使用小写字母 + 下划线（snake_case）
-- 【强制】禁止使用 MySQL 保留字
-- 【强制】布尔类型字段禁止使用 is_ 前缀（阿里规范）
--        原因：MyBatis 映射时 is_xxx 会去掉 is 前缀，导致映射失败

-- 主键
id                  -- 主键ID（雪花算法或自增）

-- 外键（关联字段）
user_id             -- 用户ID
seat_id             -- 座位ID
classroom_id        -- 教室ID

-- 状态字段
status              -- 状态（TINYINT）
deleted             -- 逻辑删除标识（0-未删除，1-已删除）
                    -- 【注意】不要用 is_deleted

-- 时间字段
create_time         -- 创建时间
update_time         -- 更新时间
start_time          -- 开始时间
end_time            -- 结束时间

-- 人员字段
creator             -- 创建人ID
updater             -- 更新人ID

-- 布尔类型字段命名（禁止 is_ 前缀）
deleted             -- 是否删除（不要用 is_deleted）
enabled             -- 是否启用（不要用 is_enabled）
visible             -- 是否可见（不要用 is_visible）
locked              -- 是否锁定（不要用 is_locked）
```

### 1.3 索引命名

```sql
-- 【强制】主键索引：pk_字段名（MySQL 自动命名为 PRIMARY）
-- 【强制】唯一索引：uk_字段名
-- 【强制】普通索引：idx_字段名
-- 【强制】联合索引：idx_字段1_字段2（按字段顺序）

-- 唯一索引
uk_phone            -- 手机号唯一索引
uk_email            -- 邮箱唯一索引
uk_open_id          -- 微信 OpenID 唯一索引

-- 普通索引
idx_user_id         -- 用户ID索引
idx_status          -- 状态索引
idx_create_time     -- 创建时间索引

-- 联合索引（按最左前缀原则，区分度高的字段在前）
idx_user_id_status          -- 用户ID + 状态
idx_classroom_id_status     -- 教室ID + 状态
idx_reservation_date_status -- 预约日期 + 状态
```

---

## 2. 表设计规范

### 2.1 基础表结构

```sql
-- 【强制】每张表必须包含以下字段
CREATE TABLE sys_user (
    id              BIGINT          NOT NULL COMMENT '主键ID（雪花算法）',
    -- 业务字段...
    deleted         TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '逻辑删除:0-未删除,1-已删除',
    create_time     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_time     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    creator         BIGINT          DEFAULT NULL COMMENT '创建人ID',
    updater         BIGINT          DEFAULT NULL COMMENT '更新人ID',
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
CREATE TABLE reservation (
    id              BIGINT          NOT NULL COMMENT '预约ID（雪花算法）',
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
    deleted         TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '逻辑删除',
    create_time     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_time     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    KEY idx_user_id (user_id),
    KEY idx_seat_id (seat_id),
    KEY idx_reservation_date_status (reservation_date, status),
    KEY idx_user_id_reservation_date (user_id, reservation_date)
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
deleted         TINYINT(1)      NOT NULL DEFAULT 0
create_time     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP
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
SELECT user_id, status FROM reservation WHERE user_id = 1;
-- 索引：idx_user_id_status (user_id, status)

-- 【推荐】前缀索引（长字符串）
CREATE INDEX idx_email ON sys_user(email(20));
```

### 4.3 索引示例

```sql
-- 用户表索引
CREATE TABLE sys_user (
    id              BIGINT NOT NULL,
    user_name       VARCHAR(50) NOT NULL,
    phone           VARCHAR(20) NOT NULL,
    email           VARCHAR(100),
    status          TINYINT NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_phone (phone),
    UNIQUE KEY uk_email (email),
    KEY idx_status (status)
);

-- 预约表索引
CREATE TABLE reservation (
    id              BIGINT NOT NULL,
    user_id         BIGINT NOT NULL,
    seat_id         BIGINT NOT NULL,
    reservation_date DATE NOT NULL,
    status          TINYINT NOT NULL,
    PRIMARY KEY (id),
    KEY idx_user_id_reservation_date (user_id, reservation_date),
    KEY idx_seat_id_reservation_date (seat_id, reservation_date),
    KEY idx_reservation_date_status (reservation_date, status)
);
```

---

## 5. SQL 编写规范

### 5.1 查询规范

```sql
-- 【强制】禁止使用 SELECT *
-- 错误
SELECT * FROM sys_user WHERE id = 1;
-- 正确
SELECT id, user_name, phone, status FROM sys_user WHERE id = 1;

-- 【强制】禁止使用 SELECT ... FOR UPDATE 长时间锁表
-- 【强制】WHERE 条件必须使用索引字段
-- 【强制】避免在 WHERE 中对字段进行函数操作
-- 错误
SELECT * FROM reservation WHERE DATE(create_time) = '2024-01-01';
-- 正确
SELECT * FROM reservation 
WHERE create_time >= '2024-01-01 00:00:00' 
  AND create_time < '2024-01-02 00:00:00';

-- 【强制】避免隐式类型转换
-- 错误（phone 是 VARCHAR，传入数字会导致索引失效）
SELECT * FROM sys_user WHERE phone = 13800138000;
-- 正确
SELECT * FROM sys_user WHERE phone = '13800138000';
```

### 5.2 分页规范

```sql
-- 【强制】大数据量分页使用游标分页
-- 错误：深度分页性能差
SELECT * FROM reservation ORDER BY id LIMIT 1000000, 10;

-- 正确：游标分页（推荐）
SELECT * FROM reservation WHERE id > 1000000 ORDER BY id LIMIT 10;

-- 备选：延迟关联（仍有性能问题，尽量用游标分页）
-- 注意：此处使用了子查询，仅在无法使用游标分页时使用
SELECT r.* FROM reservation r
WHERE r.id IN (
    SELECT id FROM reservation ORDER BY id LIMIT 1000000, 10
);
```

### 5.3 更新规范

```sql
-- 【强制】UPDATE/DELETE 必须带 WHERE 条件
-- 【强制】UPDATE/DELETE 必须使用索引字段
-- 【强制】禁止一次性更新大量数据，分批处理

-- 批量更新状态（注意：应在应用层计算日期范围，避免使用 CURDATE()）
UPDATE reservation 
SET status = 4, update_time = NOW()
WHERE reservation_date < '2024-01-15'
  AND status = 0 
  AND deleted = 0
LIMIT 1000;

-- 【推荐】应用层分批处理伪代码
-- while (true) {
--     int count = reservationMapper.batchUpdateExpired(date, 1000);
--     if (count == 0) break;
-- }
```

### 5.4 插入规范

```sql
-- 【强制】批量插入使用 INSERT INTO ... VALUES
INSERT INTO reservation (id, user_id, seat_id, reservation_date, start_time, end_time, status)
VALUES 
    (1, 100, 200, '2024-01-01', '08:00:00', '10:00:00', 0),
    (2, 101, 201, '2024-01-01', '08:00:00', '10:00:00', 0),
    (3, 102, 202, '2024-01-01', '08:00:00', '10:00:00', 0);

-- 【强制】单次批量插入不超过 500 条
-- 【强制】使用 INSERT IGNORE 或 ON DUPLICATE KEY UPDATE 处理冲突
INSERT INTO sys_user (id, user_name, phone) 
VALUES (1, 'zhangsan', '13800138000')
ON DUPLICATE KEY UPDATE user_name = VALUES(user_name);
```

---

## 6. 性能优化规范

### 6.1 查询优化

```sql
-- 【强制】使用 EXPLAIN 分析执行计划
EXPLAIN SELECT * FROM reservation WHERE user_id = 1;

-- 关注指标：
-- type: 至少达到 range 级别，最好是 ref
-- key: 使用的索引
-- rows: 扫描行数
-- Extra: 避免 Using filesort、Using temporary

-- 【强制】避免全表扫描
-- 【强制】避免索引失效场景：
-- 1. 对索引字段使用函数（如 DATE()、YEAR()）
-- 2. 对索引字段进行运算
-- 3. 使用 != 或 <>
-- 4. LIKE '%xxx'（前缀模糊匹配）
-- 5. 隐式类型转换
```

### 6.2 关联查询处理

```sql
-- ⚠️ 【重要】禁止使用 JOIN，改用应用层组装
-- 参考后端规范 ORM 章节

-- 【禁止】不要这样写
SELECT u.*, r.* FROM sys_user u
INNER JOIN reservation r ON u.id = r.user_id
WHERE r.status = 1;

-- 【正确】分两次查询，应用层组装
-- 第一步：查询预约
SELECT user_id, seat_id, status FROM reservation WHERE status = 1;
-- 第二步：根据 user_id 批量查询用户
SELECT id, user_name, avatar FROM sys_user WHERE id IN (1, 2, 3);
-- 第三步：在 Java 代码中组装数据
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
UPDATE seat 
SET status = 1, version = version + 1 
WHERE id = 1 AND version = 1;

-- 【推荐】使用 Redis 分布式锁代替数据库锁
```

---

## 附录

### A. 常用 SQL 模板

```sql
-- 分页查询模板（简单单表查询）
SELECT id, user_name, phone, status, create_time
FROM sys_user
WHERE deleted = 0
  AND status = #{status}
ORDER BY create_time DESC
LIMIT #{offset}, #{pageSize};

-- 游标分页模板（推荐，性能更好）
SELECT id, user_name, phone, status, create_time
FROM sys_user
WHERE deleted = 0
  AND id > #{lastId}
ORDER BY id
LIMIT #{pageSize};

-- 批量 ID 查询模板
SELECT id, user_name, avatar
FROM sys_user
WHERE deleted = 0
  AND id IN (1, 2, 3, 4, 5);

-- 时间范围查询模板（避免使用 DATE() 函数）
SELECT id, user_id, seat_id, status
FROM reservation
WHERE deleted = 0
  AND create_time >= #{startTime}
  AND create_time < #{endTime};
```

### B. 禁止使用的 SQL 模式

```sql
-- ❌ 禁止：JOIN 多表关联
SELECT u.*, r.* FROM sys_user u
INNER JOIN reservation r ON u.id = r.user_id;

-- ❌ 禁止：UNION 联合查询
SELECT * FROM table_a UNION ALL SELECT * FROM table_b;

-- ❌ 禁止：复杂聚合（应使用 Redis 缓存预计算）
SELECT user_id, COUNT(*), SUM(duration) FROM reservation GROUP BY user_id;

-- ❌ 禁止：WHERE 中使用函数
SELECT * FROM reservation WHERE DATE(create_time) = '2024-01-01';

-- ❌ 禁止：子查询
SELECT * FROM sys_user WHERE id IN (SELECT user_id FROM reservation);
```

### C. 检查清单

- [ ] 表名是否符合规范（无 t_ 前缀）
- [ ] 字段名是否符合规范（无 is_ 前缀）
- [ ] 是否有主键
- [ ] 是否有必要的索引
- [ ] 是否有逻辑删除字段（deleted）
- [ ] 是否有创建/更新时间字段（create_time/update_time）
- [ ] 字段类型是否合适
- [ ] 字段长度是否合理
- [ ] 是否有字段注释
- [ ] SQL 是否使用索引
- [ ] 是否有 JOIN/UNION/子查询（禁止）
- [ ] 是否在 WHERE 中使用函数（禁止）
- [ ] 是否有慢查询风险
