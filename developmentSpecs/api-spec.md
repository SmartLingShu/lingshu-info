# API 设计规范

> 适用于 RESTful API 设计，以一致性、易用性、性能为核心原则。

## 目录

- [1. URL 设计规范](#1-url-设计规范)
- [2. HTTP 方法规范](#2-http-方法规范)
- [3. 请求规范](#3-请求规范)
- [4. 响应规范](#4-响应规范)
- [5. 错误处理规范](#5-错误处理规范)
- [6. 版本控制规范](#6-版本控制规范)
- [7. 安全规范](#7-安全规范)
- [8. 文档规范](#8-文档规范)

---

## 1. URL 设计规范

### 1.1 基本原则

```
# 【强制】使用小写字母
# 【强制】使用连字符 - 分隔单词
# 【强制】使用名词复数形式表示资源集合
# 【强制】使用名词单数形式表示单个资源
# 【强制】URL 层级不超过 3 层

# 基础格式
/{version}/{resource}/{id}/{sub-resource}

# 示例
GET  /api/v1/users                    # 用户列表
GET  /api/v1/users/123                # 单个用户
GET  /api/v1/users/123/reservations   # 用户的预约列表
POST /api/v1/reservations             # 创建预约
```

### 1.2 资源命名

```
# 【强制】使用名词，不使用动词
# 错误
GET /api/v1/getUsers
POST /api/v1/createReservation

# 正确
GET /api/v1/users
POST /api/v1/reservations

# 【强制】特殊操作使用动词后缀
POST /api/v1/reservations/123/cancel     # 取消预约
POST /api/v1/reservations/123/check-in   # 签到
POST /api/v1/users/123/reset-password    # 重置密码
POST /api/v1/auth/login                  # 登录
POST /api/v1/auth/logout                 # 登出
```

### 1.3 查询参数

```
# 【强制】分页参数
GET /api/v1/reservations?pageNum=1&pageSize=10

# 【强制】排序参数
GET /api/v1/reservations?sortField=createdTime&sortOrder=desc

# 【强制】过滤参数
GET /api/v1/reservations?status=1&userId=123

# 【强制】时间范围参数
GET /api/v1/reservations?startTimeBegin=2024-01-01&startTimeEnd=2024-01-31

# 【强制】模糊搜索参数
GET /api/v1/users?keyword=zhang
```

---

## 2. HTTP 方法规范

### 2.1 方法语义

| 方法 | 语义 | 幂等性 | 安全性 | 示例 |
|------|------|--------|--------|------|
| GET | 获取资源 | 是 | 是 | 获取用户列表 |
| POST | 创建资源 | 否 | 否 | 创建预约 |
| PUT | 全量更新 | 是 | 否 | 更新用户信息 |
| PATCH | 部分更新 | 是 | 否 | 更新用户状态 |
| DELETE | 删除资源 | 是 | 否 | 删除预约 |

### 2.2 使用示例

```
# 获取资源
GET    /api/v1/reservations           # 获取预约列表
GET    /api/v1/reservations/123       # 获取单个预约

# 创建资源
POST   /api/v1/reservations           # 创建预约

# 更新资源
PUT    /api/v1/reservations/123       # 全量更新预约
PATCH  /api/v1/reservations/123       # 部分更新预约

# 删除资源
DELETE /api/v1/reservations/123       # 删除单个预约
DELETE /api/v1/reservations?ids=1,2,3 # 批量删除预约

# 特殊操作
POST   /api/v1/reservations/123/cancel    # 取消预约
POST   /api/v1/reservations/123/check-in  # 签到
```

---

## 3. 请求规范

### 3.1 请求头

```
# 【强制】Content-Type
Content-Type: application/json

# 【强制】认证信息
Authorization: Bearer {token}

# 【推荐】请求追踪
X-Request-Id: uuid
X-Trace-Id: trace-id

# 【推荐】客户端信息
X-Client-Version: 1.0.0
X-Client-Platform: web/mini-program
```

### 3.2 请求体格式

```json
// 创建预约请求
POST /api/v1/reservations
{
    "seatId": 123,
    "reservationDate": "2024-01-15",
    "startTime": "08:00:00",
    "endTime": "10:00:00",
    "remark": "备注信息"
}

// 更新用户请求
PUT /api/v1/users/123
{
    "nickName": "张三",
    "phone": "13800138000",
    "email": "zhangsan@example.com"
}

// 部分更新请求
PATCH /api/v1/users/123
{
    "status": 0
}

// 批量操作请求
DELETE /api/v1/reservations
{
    "ids": [1, 2, 3]
}
```

### 3.3 参数校验

```java
// 【强制】使用 JSR-303 注解校验
public class ReservationDTO {
    
    @NotNull(message = "座位ID不能为空")
    private Long seatId;
    
    @NotNull(message = "预约日期不能为空")
    @Future(message = "预约日期必须是将来日期")
    private LocalDate reservationDate;
    
    @NotNull(message = "开始时间不能为空")
    private LocalTime startTime;
    
    @NotNull(message = "结束时间不能为空")
    private LocalTime endTime;
    
    @Size(max = 500, message = "备注不能超过500字")
    private String remark;
}
```

---

## 4. 响应规范

### 4.1 统一响应格式

```json
{
    "code": "00000",
    "message": "操作成功",
    "data": { ... },
    "timestamp": 1704067200000
}
```

### 4.2 响应示例

```json
// 单个对象响应
{
    "code": "00000",
    "message": "操作成功",
    "data": {
        "id": 123,
        "userId": 456,
        "seatId": 789,
        "seatName": "A-01",
        "classroomName": "图书馆3楼自习室",
        "reservationDate": "2024-01-15",
        "startTime": "08:00:00",
        "endTime": "10:00:00",
        "status": 0,
        "statusName": "待签到",
        "createdTime": "2024-01-14 10:30:00"
    }
}

// 分页列表响应
{
    "code": "00000",
    "message": "操作成功",
    "data": {
        "total": 100,
        "records": [
            {
                "id": 123,
                "seatName": "A-01",
                "status": 0
            }
        ]
    }
}

// 创建成功响应
{
    "code": "00000",
    "message": "创建成功",
    "data": 123
}

// 无数据响应
{
    "code": "00000",
    "message": "操作成功",
    "data": null
}
```

### 4.3 HTTP 状态码

| 状态码 | 含义 | 使用场景 |
|--------|------|----------|
| 200 | OK | 请求成功 |
| 201 | Created | 资源创建成功 |
| 204 | No Content | 删除成功，无返回内容 |
| 400 | Bad Request | 请求参数错误 |
| 401 | Unauthorized | 未认证 |
| 403 | Forbidden | 无权限 |
| 404 | Not Found | 资源不存在 |
| 409 | Conflict | 资源冲突 |
| 429 | Too Many Requests | 请求过于频繁 |
| 500 | Internal Server Error | 服务器内部错误 |

---

## 5. 错误处理规范

### 5.1 错误码设计

```
# 错误码格式：AABBB
# AA: 模块代码
# BBB: 错误序号

# 00xxx - 通用错误
00000 - 操作成功
00001 - 系统异常
00002 - 参数错误
00003 - 未授权
00004 - 禁止访问
00005 - 资源不存在

# 01xxx - 用户模块
01001 - 用户不存在
01002 - 用户已禁用
01003 - 密码错误
01004 - 验证码错误
01005 - Token 已过期

# 02xxx - 预约模块
02001 - 座位不可用
02002 - 预约时间无效
02003 - 时间范围无效
02004 - 预约不存在
02005 - 预约无法取消
02006 - 已有预约冲突

# 03xxx - 座位模块
03001 - 座位不存在
03002 - 座位已被锁定
03003 - 教室已关闭
```

### 5.2 错误响应格式

```json
// 参数校验错误
{
    "code": "00002",
    "message": "参数错误",
    "data": {
        "errors": [
            {
                "field": "seatId",
                "message": "座位ID不能为空"
            },
            {
                "field": "startTime",
                "message": "开始时间必须是将来时间"
            }
        ]
    }
}

// 业务错误
{
    "code": "02001",
    "message": "座位不可用，该时段已被预约"
}

// 系统错误
{
    "code": "00001",
    "message": "系统异常，请稍后重试"
}
```

---

## 6. 版本控制规范

### 6.1 版本策略

```
# 【强制】URL 路径版本控制
/api/v1/users
/api/v2/users

# 版本升级原则：
# - 新增字段：不升级版本，保持向后兼容
# - 删除字段：升级版本
# - 修改字段类型：升级版本
# - 修改业务逻辑：升级版本
```

### 6.2 版本兼容

```java
// 【强制】新版本保持对旧版本的兼容期（至少 3 个月）
// 【强制】废弃 API 使用 @Deprecated 标注
// 【强制】响应头返回废弃警告

@Deprecated
@GetMapping("/api/v1/users")
public Result<List<UserVO>> listUsersV1() {
    // 设置废弃警告头
    response.setHeader("X-API-Deprecated", "true");
    response.setHeader("X-API-Sunset", "2024-06-01");
    // ...
}
```

---

## 7. 安全规范

### 7.1 认证授权

```
# 【强制】敏感接口必须认证
# 【强制】使用 JWT Token 认证
# 【强制】Token 有效期不超过 2 小时
# 【强制】支持 Token 刷新机制

# 认证流程
1. POST /api/v1/auth/login -> 返回 accessToken + refreshToken
2. 请求携带 Authorization: Bearer {accessToken}
3. accessToken 过期后使用 refreshToken 刷新
4. POST /api/v1/auth/refresh -> 返回新的 accessToken
```

### 7.2 接口安全

```
# 【强制】敏感数据脱敏
{
    "phone": "138****8000",
    "email": "z***@example.com",
    "idCard": "110***********1234"
}

# 【强制】防重复提交
# 使用幂等性 Token 或请求签名

# 【强制】请求频率限制
# 普通接口：100 次/分钟
# 敏感接口：10 次/分钟
# 登录接口：5 次/分钟

# 【强制】SQL 注入防护
# 使用参数化查询

# 【强制】XSS 防护
# 输出转义
```

### 7.3 数据安全

```
# 【强制】HTTPS 传输
# 【强制】敏感数据加密存储
# 【强制】日志脱敏
# 【强制】接口签名（对外开放接口）

# 签名算法示例
sign = MD5(appId + timestamp + nonce + secret + params)
```

---

## 8. 文档规范

### 8.1 Swagger 注解

```java
@Tag(name = "预约管理", description = "预约相关接口")
@RestController
@RequestMapping("/api/v1/reservations")
public class ReservationController {

    @Operation(summary = "分页查询预约记录", description = "支持按状态、时间范围筛选")
    @Parameters({
        @Parameter(name = "status", description = "状态：0-待签到，1-使用中，2-已完成"),
        @Parameter(name = "pageNum", description = "页码，默认1"),
        @Parameter(name = "pageSize", description = "每页条数，默认10")
    })
    @GetMapping
    public Result<PageResult<ReservationVO>> page(ReservationQuery query) {
        // ...
    }

    @Operation(summary = "创建预约")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "创建成功，返回预约ID"),
        @ApiResponse(responseCode = "400", description = "参数错误"),
        @ApiResponse(responseCode = "409", description = "座位已被预约")
    })
    @PostMapping
    public Result<Long> create(@Valid @RequestBody ReservationDTO dto) {
        // ...
    }
}
```

### 8.2 接口文档模板

```markdown
## 创建预约

### 基本信息

- **接口路径**: POST /api/v1/reservations
- **接口描述**: 创建座位预约
- **认证方式**: Bearer Token

### 请求参数

| 参数名 | 类型 | 必填 | 描述 |
|--------|------|------|------|
| seatId | Long | 是 | 座位ID |
| reservationDate | String | 是 | 预约日期，格式：yyyy-MM-dd |
| startTime | String | 是 | 开始时间，格式：HH:mm:ss |
| endTime | String | 是 | 结束时间，格式：HH:mm:ss |
| remark | String | 否 | 备注，最大500字 |

### 请求示例

```json
{
    "seatId": 123,
    "reservationDate": "2024-01-15",
    "startTime": "08:00:00",
    "endTime": "10:00:00",
    "remark": "准备考研复习"
}
```

### 响应参数

| 参数名 | 类型 | 描述 |
|--------|------|------|
| code | String | 响应码 |
| message | String | 响应消息 |
| data | Long | 预约ID |

### 响应示例

```json
{
    "code": "00000",
    "message": "创建成功",
    "data": 456
}
```

### 错误码

| 错误码 | 描述 |
|--------|------|
| 02001 | 座位不可用 |
| 02002 | 预约时间无效 |
| 02006 | 已有预约冲突 |
