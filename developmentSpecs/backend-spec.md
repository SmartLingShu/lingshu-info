# 后端开发规范

> 适用于 Java 17 + Spring Boot 3.x 技术栈，遵循阿里巴巴 Java 开发手册。

## 目录

- [1. 命名规范](#1-命名规范)
- [2. 代码分层规范](#2-代码分层规范)
- [3. 异常处理规范](#3-异常处理规范)
- [4. 日志规范](#4-日志规范)
- [5. 注释规范](#5-注释规范)
- [6. 并发编程规范](#6-并发编程规范)
- [7. ORM 规范](#7-orm-规范)
- [8. 缓存规范](#8-缓存规范)
- [9. 安全规范](#9-安全规范)
- [10. 性能规范](#10-性能规范)

---

## 1. 命名规范

### 1.1 包命名

```
com.lingshu.{模块名}.{层名}

示例：
com.lingshu.reservation.controller
com.lingshu.reservation.service
com.lingshu.reservation.service.impl
com.lingshu.reservation.mapper
com.lingshu.reservation.model.entity
com.lingshu.reservation.model.dto
com.lingshu.reservation.model.vo
com.lingshu.reservation.model.query
com.lingshu.reservation.model.converter
```

### 1.2 类命名

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| Controller | XxxController | ReservationController |
| Service 接口 | XxxService | ReservationService |
| Service 实现 | XxxServiceImpl | ReservationServiceImpl |
| Mapper 接口 | XxxMapper | ReservationMapper |
| 实体类 | Xxx | Reservation |
| DTO | XxxDTO | ReservationDTO |
| VO | XxxVO | ReservationVO |
| Query | XxxQuery | ReservationQuery |
| Converter | XxxConverter | ReservationConverter |
| 枚举 | XxxEnum | ReservationStatusEnum |
| 常量类 | XxxConstant | ReservationConstant |
| 工具类 | XxxUtil / XxxUtils | DateUtil |
| 异常类 | XxxException | BusinessException |
| 配置类 | XxxConfig | RedisConfig |
| 切面类 | XxxAspect | LogAspect |
| 监听器 | XxxListener | OrderEventListener |
| 处理器 | XxxHandler | GlobalExceptionHandler |

### 1.3 方法命名

| 场景 | 命名规则 | 示例 |
|------|----------|------|
| 获取单个对象 | getXxx / findXxx | getById, findByUserId |
| 获取多个对象 | listXxx / findXxxList | listByStatus, findActiveList |
| 获取分页对象 | pageXxx | pageByCondition |
| 获取统计值 | countXxx | countByStatus |
| 插入 | saveXxx / insertXxx | saveReservation |
| 批量插入 | batchSaveXxx | batchSaveReservations |
| 更新 | updateXxx / modifyXxx | updateStatus |
| 删除 | removeXxx / deleteXxx | removeById |
| 判断存在 | existsXxx / hasXxx | existsByUserId |
| 校验 | validateXxx / checkXxx | validateTimeSlot |
| 转换 | convertXxx / toXxx | convertToVO |

### 1.4 变量命名

```java
// 【强制】使用 lowerCamelCase 风格
private String userName;
private Integer reservationCount;

// 【强制】布尔类型变量不要加 is 前缀（POJO 类除外）
private boolean active;      // 正确
private boolean isActive;    // 错误

// 【强制】常量使用 UPPER_SNAKE_CASE
public static final String DEFAULT_CHARSET = "UTF-8";
public static final int MAX_RETRY_COUNT = 3;

// 【强制】集合类型变量名使用复数或带有集合含义的后缀
private List<User> users;
private Map<Long, User> userMap;
private Set<String> userIdSet;
```

### 1.5 数据库字段映射

```java
// 数据库字段: user_name -> Java 属性: userName
// 数据库字段: create_time -> Java 属性: createTime
// 数据库字段: update_time -> Java 属性: updateTime
// 数据库字段: deleted -> Java 属性: deleted
// 【注意】数据库字段禁止使用 is_ 前缀，参考数据库规范
```

---

## 2. 代码分层规范

### 2.1 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Controller 层                          │
│  - 参数校验、权限校验                                        │
│  - 调用 Service，不包含业务逻辑                              │
│  - 返回统一响应格式                                          │
├─────────────────────────────────────────────────────────────┤
│                       Service 层                            │
│  - 业务逻辑处理                                              │
│  - 事务控制                                                  │
│  - 调用 Mapper 或其他 Service                                │
├─────────────────────────────────────────────────────────────┤
│                       Mapper 层                             │
│  - 数据库访问                                                │
│  - 不包含业务逻辑                                            │
├─────────────────────────────────────────────────────────────┤
│                       Model 层                              │
│  - Entity: 数据库实体                                        │
│  - DTO: 数据传输对象                                         │
│  - VO: 视图对象                                              │
│  - Query: 查询参数对象                                       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Controller 规范

```java
@Tag(name = "预约管理")
@RestController
@RequestMapping("/api/v1/reservations")
@RequiredArgsConstructor
public class ReservationController {

    private final ReservationService reservationService;

    /**
     * 分页查询预约记录
     */
    @Operation(summary = "分页查询预约记录")
    @GetMapping
    public Result<PageResult<ReservationVO>> page(@Valid ReservationQuery query) {
        return Result.success(reservationService.pageReservations(query));
    }

    /**
     * 获取预约详情
     */
    @Operation(summary = "获取预约详情")
    @GetMapping("/{id}")
    public Result<ReservationVO> getById(@PathVariable Long id) {
        return Result.success(reservationService.getReservationById(id));
    }

    /**
     * 创建预约
     */
    @Operation(summary = "创建预约")
    @PostMapping
    public Result<Long> create(@Valid @RequestBody ReservationDTO dto) {
        return Result.success(reservationService.createReservation(dto));
    }

    /**
     * 取消预约
     */
    @Operation(summary = "取消预约")
    @PutMapping("/{id}/cancel")
    public Result<Void> cancel(@PathVariable Long id) {
        reservationService.cancelReservation(id);
        return Result.success();
    }
}
```

### 2.3 Service 规范

```java
public interface ReservationService {
    
    /**
     * 分页查询预约记录
     *
     * @param query 查询条件
     * @return 分页结果
     */
    PageResult<ReservationVO> pageReservations(ReservationQuery query);
    
    /**
     * 创建预约
     *
     * @param dto 预约信息
     * @return 预约ID
     */
    Long createReservation(ReservationDTO dto);
}

@Slf4j
@Service
@RequiredArgsConstructor
public class ReservationServiceImpl implements ReservationService {

    private final ReservationMapper reservationMapper;
    private final SeatService seatService;
    private final ReservationConverter reservationConverter;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long createReservation(ReservationDTO dto) {
        // 1. 参数校验
        validateReservationTime(dto);
        
        // 2. 检查座位是否可用
        if (!seatService.isSeatAvailable(dto.getSeatId(), dto.getStartTime(), dto.getEndTime())) {
            throw new BusinessException(ResultCode.SEAT_NOT_AVAILABLE);
        }
        
        // 3. 创建预约
        Reservation reservation = reservationConverter.toEntity(dto);
        reservation.setStatus(ReservationStatusEnum.PENDING.getCode());
        reservationMapper.insert(reservation);
        
        // 4. 锁定座位
        seatService.lockSeat(dto.getSeatId(), dto.getStartTime(), dto.getEndTime());
        
        log.info("预约创建成功, reservationId={}, userId={}", reservation.getId(), dto.getUserId());
        return reservation.getId();
    }
    
    private void validateReservationTime(ReservationDTO dto) {
        LocalDateTime now = LocalDateTime.now();
        if (dto.getStartTime().isBefore(now)) {
            throw new BusinessException(ResultCode.INVALID_RESERVATION_TIME);
        }
        if (dto.getEndTime().isBefore(dto.getStartTime())) {
            throw new BusinessException(ResultCode.INVALID_TIME_RANGE);
        }
    }
}
```

### 2.4 Model 规范

```java
// Entity - 数据库实体，与表结构一一对应
// 【注意】表名禁止使用 t_ 前缀
@Data
@TableName("reservation")
public class Reservation {
    
    @TableId(type = IdType.ASSIGN_ID)
    private Long id;
    
    private Long userId;
    
    private Long seatId;
    
    private LocalDateTime startTime;
    
    private LocalDateTime endTime;
    
    private Integer status;
    
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
    
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
    
    @TableLogic
    private Integer deleted;
}

// DTO - 数据传输对象，用于接收请求参数
@Data
public class ReservationDTO {
    
    @NotNull(message = "座位ID不能为空")
    private Long seatId;
    
    @NotNull(message = "开始时间不能为空")
    @Future(message = "开始时间必须是将来时间")
    private LocalDateTime startTime;
    
    @NotNull(message = "结束时间不能为空")
    private LocalDateTime endTime;
}

// VO - 视图对象，用于返回给前端
@Data
public class ReservationVO {
    
    private Long id;
    
    private Long userId;
    
    private String userName;
    
    private Long seatId;
    
    private String seatName;
    
    private String classroomName;
    
    private LocalDateTime startTime;
    
    private LocalDateTime endTime;
    
    private String statusName;
    
    private LocalDateTime createdTime;
}

// Query - 查询参数对象
@Data
@EqualsAndHashCode(callSuper = true)
public class ReservationQuery extends PageQuery {
    
    @Schema(description = "用户ID")
    private Long userId;
    
    @Schema(description = "座位ID")
    private Long seatId;
    
    @Schema(description = "状态")
    private Integer status;
    
    @Schema(description = "开始时间-起")
    private LocalDateTime startTimeBegin;
    
    @Schema(description = "开始时间-止")
    private LocalDateTime startTimeEnd;
}
```

### 2.5 Converter 规范

```java
@Mapper(componentModel = "spring")
public interface ReservationConverter {
    
    Reservation toEntity(ReservationDTO dto);
    
    ReservationVO toVO(Reservation entity);
    
    List<ReservationVO> toVOList(List<Reservation> entities);
}
```

---

## 3. 异常处理规范

### 3.1 异常分类

```java
// 业务异常 - 可预期的业务错误
public class BusinessException extends RuntimeException {
    
    private final String code;
    private final String message;
    
    public BusinessException(IResultCode resultCode) {
        super(resultCode.getMessage());
        this.code = resultCode.getCode();
        this.message = resultCode.getMessage();
    }
    
    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
        this.message = message;
    }
}

// 系统异常 - 不可预期的系统错误
public class SystemException extends RuntimeException {
    
    public SystemException(String message) {
        super(message);
    }
    
    public SystemException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### 3.2 错误码规范

```java
public interface IResultCode {
    String getCode();
    String getMessage();
}

@Getter
@AllArgsConstructor
public enum ResultCode implements IResultCode {
    
    // 通用错误码 00xxx
    SUCCESS("00000", "操作成功"),
    SYSTEM_ERROR("00001", "系统异常"),
    PARAM_ERROR("00002", "参数错误"),
    UNAUTHORIZED("00003", "未授权"),
    FORBIDDEN("00004", "禁止访问"),
    NOT_FOUND("00005", "资源不存在"),
    
    // 用户模块 01xxx
    USER_NOT_FOUND("01001", "用户不存在"),
    USER_DISABLED("01002", "用户已禁用"),
    PASSWORD_ERROR("01003", "密码错误"),
    
    // 预约模块 02xxx
    SEAT_NOT_AVAILABLE("02001", "座位不可用"),
    INVALID_RESERVATION_TIME("02002", "预约时间无效"),
    INVALID_TIME_RANGE("02003", "时间范围无效"),
    RESERVATION_NOT_FOUND("02004", "预约记录不存在"),
    RESERVATION_CANNOT_CANCEL("02005", "预约无法取消"),
    
    // 座位模块 03xxx
    SEAT_NOT_FOUND("03001", "座位不存在"),
    SEAT_LOCKED("03002", "座位已被锁定"),
    ;
    
    private final String code;
    private final String message;
}
```

### 3.3 全局异常处理

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * 业务异常
     */
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        log.warn("业务异常: code={}, message={}", e.getCode(), e.getMessage());
        return Result.fail(e.getCode(), e.getMessage());
    }

    /**
     * 参数校验异常
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining(", "));
        log.warn("参数校验失败: {}", message);
        return Result.fail(ResultCode.PARAM_ERROR.getCode(), message);
    }

    /**
     * 约束违反异常
     */
    @ExceptionHandler(ConstraintViolationException.class)
    public Result<Void> handleConstraintViolationException(ConstraintViolationException e) {
        String message = e.getConstraintViolations().stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.joining(", "));
        log.warn("约束违反: {}", message);
        return Result.fail(ResultCode.PARAM_ERROR.getCode(), message);
    }

    /**
     * 系统异常
     */
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail(ResultCode.SYSTEM_ERROR);
    }
}
```

---

## 4. 日志规范

### 4.1 日志级别使用

| 级别 | 使用场景 |
|------|----------|
| ERROR | 系统错误、异常堆栈、需要人工介入处理 |
| WARN | 业务异常、可恢复错误、性能警告 |
| INFO | 关键业务流程、状态变更、外部调用 |
| DEBUG | 调试信息、详细流程、变量值 |
| TRACE | 最详细的跟踪信息，一般不使用 |

### 4.2 日志格式

```java
// 【强制】使用 SLF4J 门面
private static final Logger log = LoggerFactory.getLogger(XxxService.class);
// 或使用 Lombok @Slf4j 注解

// 【强制】使用占位符方式，避免字符串拼接
log.info("用户登录成功, userId={}, ip={}", userId, ip);

// 【强制】异常日志必须包含异常堆栈
log.error("处理订单失败, orderId={}", orderId, e);

// 【强制】敏感信息脱敏
log.info("用户注册, phone={}", DesensitizeUtil.phone(phone));
```

### 4.3 关键日志点

```java
@Slf4j
@Service
public class ReservationServiceImpl implements ReservationService {

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long createReservation(ReservationDTO dto) {
        // 方法入口日志
        log.info("创建预约开始, userId={}, seatId={}, startTime={}, endTime={}", 
                dto.getUserId(), dto.getSeatId(), dto.getStartTime(), dto.getEndTime());
        
        long startMs = System.currentTimeMillis();
        
        try {
            // 业务逻辑...
            
            // 关键状态变更日志
            log.info("预约创建成功, reservationId={}, userId={}", reservation.getId(), dto.getUserId());
            
            return reservation.getId();
        } catch (BusinessException e) {
            // 业务异常日志
            log.warn("创建预约失败, userId={}, reason={}", dto.getUserId(), e.getMessage());
            throw e;
        } catch (Exception e) {
            // 系统异常日志
            log.error("创建预约异常, userId={}, seatId={}", dto.getUserId(), dto.getSeatId(), e);
            throw new SystemException("创建预约失败", e);
        } finally {
            // 性能日志
            long costMs = System.currentTimeMillis() - startMs;
            if (costMs > 1000) {
                log.warn("创建预约耗时过长, userId={}, costMs={}", dto.getUserId(), costMs);
            }
        }
    }
}
```

### 4.4 链路追踪

```java
// 使用 MDC 传递链路信息
public class TraceIdFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        try {
            String traceId = UUID.randomUUID().toString().replace("-", "");
            MDC.put("traceId", traceId);
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}

// logback 配置
// <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{traceId}] [%thread] %-5level %logger{36} - %msg%n</pattern>
```

---

## 5. 注释规范

### 5.1 类注释

```java
/**
 * 预约服务实现类
 * <p>
 * 负责处理座位预约相关的业务逻辑，包括：
 * <ul>
 *     <li>预约创建与取消</li>
 *     <li>预约状态管理</li>
 *     <li>预约冲突检测</li>
 * </ul>
 *
 * @author TouHouQing
 * @since 1.0.0
 */
@Service
public class ReservationServiceImpl implements ReservationService {
    // ...
}
```

### 5.2 方法注释

```java
/**
 * 创建座位预约
 * <p>
 * 业务流程：
 * 1. 校验预约时间有效性
 * 2. 检查座位是否可用
 * 3. 创建预约记录
 * 4. 锁定座位
 *
 * @param dto 预约信息
 * @return 预约ID
 * @throws BusinessException 当座位不可用或时间无效时抛出
 */
@Override
@Transactional(rollbackFor = Exception.class)
public Long createReservation(ReservationDTO dto) {
    // ...
}
```

### 5.3 代码注释

```java
// 【强制】单行注释使用 //，多行注释使用 /* */
// 【强制】注释与代码同步更新，过时注释比没有注释更有害

// 检查座位是否在预约时间段内被占用
boolean isOccupied = reservationMapper.existsConflict(
        dto.getSeatId(), 
        dto.getStartTime(), 
        dto.getEndTime()
);

/*
 * 信用分扣减规则：
 * - 提前2小时取消：不扣分
 * - 提前1小时取消：扣5分
 * - 提前30分钟取消：扣10分
 * - 未签到自动取消：扣20分
 */
int deductPoints = calculateDeductPoints(reservation);
```

---

## 6. 并发编程规范

### 6.1 线程池规范

```java
// 【强制】线程池必须通过 ThreadPoolExecutor 创建，不允许使用 Executors
@Configuration
public class ThreadPoolConfig {
    
    /**
     * 业务线程池
     * 核心线程数 = CPU核心数
     * 最大线程数 = CPU核心数 * 2
     * 队列容量根据业务场景设置
     */
    @Bean("businessExecutor")
    public ThreadPoolExecutor businessExecutor() {
        int corePoolSize = Runtime.getRuntime().availableProcessors();
        int maxPoolSize = corePoolSize * 2;
        int queueCapacity = 1000;
        
        return new ThreadPoolExecutor(
                corePoolSize,
                maxPoolSize,
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(queueCapacity),
                new ThreadFactoryBuilder().setNameFormat("business-pool-%d").build(),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
    
    /**
     * IO密集型线程池
     */
    @Bean("ioExecutor")
    public ThreadPoolExecutor ioExecutor() {
        int corePoolSize = Runtime.getRuntime().availableProcessors() * 2;
        int maxPoolSize = corePoolSize * 2;
        
        return new ThreadPoolExecutor(
                corePoolSize,
                maxPoolSize,
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(2000),
                new ThreadFactoryBuilder().setNameFormat("io-pool-%d").build(),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

### 6.2 分布式锁规范

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class DistributedLockService {
    
    private final StringRedisTemplate redisTemplate;
    
    /**
     * 获取分布式锁
     *
     * @param lockKey    锁键
     * @param requestId  请求标识（用于释放锁时验证）
     * @param expireTime 过期时间（秒）
     * @return 是否获取成功
     */
    public boolean tryLock(String lockKey, String requestId, long expireTime) {
        Boolean result = redisTemplate.opsForValue().setIfAbsent(
                lockKey, 
                requestId, 
                expireTime, 
                TimeUnit.SECONDS
        );
        return Boolean.TRUE.equals(result);
    }
    
    /**
     * 释放分布式锁
     * 使用 Lua 脚本保证原子性
     */
    public boolean releaseLock(String lockKey, String requestId) {
        String script = """
                if redis.call('get', KEYS[1]) == ARGV[1] then
                    return redis.call('del', KEYS[1])
                else
                    return 0
                end
                """;
        Long result = redisTemplate.execute(
                new DefaultRedisScript<>(script, Long.class),
                Collections.singletonList(lockKey),
                requestId
        );
        return Long.valueOf(1L).equals(result);
    }
}

// 使用示例
@Service
public class SeatLockService {
    
    private static final String SEAT_LOCK_PREFIX = "lock:seat:";
    
    @Autowired
    private DistributedLockService lockService;
    
    public boolean lockSeat(Long seatId, LocalDateTime startTime, LocalDateTime endTime) {
        String lockKey = SEAT_LOCK_PREFIX + seatId + ":" + startTime.toLocalDate();
        String requestId = UUID.randomUUID().toString();
        
        // 锁定时间应略大于业务处理时间
        if (lockService.tryLock(lockKey, requestId, 30)) {
            try {
                // 执行业务逻辑
                return doLockSeat(seatId, startTime, endTime);
            } finally {
                lockService.releaseLock(lockKey, requestId);
            }
        }
        throw new BusinessException(ResultCode.SEAT_LOCKED);
    }
}
```

### 6.3 并发安全规范

```java
// 【强制】SimpleDateFormat 是线程不安全的，使用 DateTimeFormatter
private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd");

// 【强制】HashMap 在并发场景使用 ConcurrentHashMap
private final Map<String, Object> cache = new ConcurrentHashMap<>();

// 【强制】ArrayList 在并发场景使用 CopyOnWriteArrayList 或同步包装
private final List<String> list = new CopyOnWriteArrayList<>();

// 【强制】计数器使用 AtomicLong 或 LongAdder
private final LongAdder counter = new LongAdder();

// 【强制】双重检查锁定必须使用 volatile
private volatile Singleton instance;
```

---

## 7. ORM 规范

### 7.1 核心原则：简单查询优先

> ⚠️ **重要原则**：SQL 越简单越好，能不写复杂查询就不写。复杂查询会导致：
> - 数据库性能瓶颈
> - 难以优化和维护
> - 无法有效利用缓存
> - 水平扩展困难

#### 7.1.1 禁止使用的 SQL 特性

| 特性 | 说明 | 替代方案 |
|------|------|----------|
| **JOIN** | 禁止多表关联查询 | 分多次单表查询，在 Service 层组装 |
| **UNION / UNION ALL** | 禁止联合查询 | 分多次查询，在 Service 层合并 |
| **子查询** | 禁止嵌套子查询 | 拆分为多次简单查询 |
| **MySQL 函数** | 禁止在 WHERE 中使用函数 | 在应用层处理或预先计算 |
| **GROUP BY + HAVING** | 避免复杂聚合 | 使用缓存预计算或应用层聚合 |
| **DISTINCT** | 避免去重操作 | 业务层去重或优化数据模型 |
| **ORDER BY 多字段** | 避免复杂排序 | 单字段排序 + 应用层二次排序 |

#### 7.1.2 反面案例（禁止）

```xml
<!-- ❗ 错误示例1：JOIN 多表关联 -->
<select id="selectReservationWithUser" resultType="ReservationVO">
    SELECT r.*, u.user_name, u.avatar, c.classroom_name
    FROM reservation r
    INNER JOIN sys_user u ON r.user_id = u.id
    INNER JOIN classroom c ON r.classroom_id = c.id
    WHERE r.id = #{id}
</select>

<!-- ❗ 错误示例2：UNION 联合查询 -->
<select id="selectAllTags" resultType="TagVO">
    SELECT id, name, user_id, 0 AS is_collaborative
    FROM personal_tag
    WHERE user_id = #{userId}
    UNION ALL
    SELECT t.id, t.name, t.user_id, 1 AS is_collaborative
    FROM personal_tag t
    INNER JOIN collaborator_relation cr ON cr.inviter_user_id = t.user_id
    WHERE cr.invited_user_id = #{currentUserId}
</select>

<!-- ❗ 错误示例3：WHERE 中使用函数（索引失效） -->
<select id="selectByDate" resultType="Reservation">
    SELECT * FROM reservation
    WHERE DATE(create_time) = #{date}
    AND YEAR(start_time) = #{year}
</select>

<!-- ❗ 错误示例4：复杂子查询 -->
<select id="selectActiveUsers" resultType="User">
    SELECT * FROM sys_user
    WHERE id IN (
        SELECT DISTINCT user_id FROM reservation
        WHERE status = 1 AND create_time > (
            SELECT MAX(create_time) - INTERVAL 7 DAY FROM reservation
        )
    )
</select>

<!-- ❗ 错误示例5：LOCATE/FIND_IN_SET 等函数 -->
<select id="selectByTagIds" resultType="Tag">
    SELECT * FROM tag
    WHERE LOCATE(CONCAT(',', id, ','), #{tagIds}) > 0
</select>
```

#### 7.1.3 正确做法（推荐）

```java
// ✅ 正确示例1：分多次单表查询，Service 层组装
@Service
public class ReservationServiceImpl implements ReservationService {
    
    @Override
    public ReservationVO getReservationDetail(Long id) {
        // 1. 查询预约基本信息
        Reservation reservation = reservationMapper.selectById(id);
        if (reservation == null) {
            return null;
        }
        
        // 2. 查询用户信息（可走缓存）
        User user = userService.getById(reservation.getUserId());
        
        // 3. 查询教室信息（可走缓存）
        Classroom classroom = classroomService.getById(reservation.getClassroomId());
        
        // 4. 在 Service 层组装 VO
        ReservationVO vo = reservationConverter.toVO(reservation);
        vo.setUserName(user.getUserName());
        vo.setAvatar(user.getAvatar());
        vo.setClassroomName(classroom.getName());
        
        return vo;
    }
}

// ✅ 正确示例2：批量查询 + 内存组装（避免 N+1）
@Service
public class ReservationServiceImpl implements ReservationService {
    
    @Override
    public List<ReservationVO> listReservations(ReservationQuery query) {
        // 1. 查询预约列表
        List<Reservation> reservations = reservationMapper.selectByCondition(query);
        if (CollectionUtils.isEmpty(reservations)) {
            return Collections.emptyList();
        }
        
        // 2. 批量查询关联数据
        Set<Long> userIds = reservations.stream()
            .map(Reservation::getUserId)
            .collect(Collectors.toSet());
        Set<Long> classroomIds = reservations.stream()
            .map(Reservation::getClassroomId)
            .collect(Collectors.toSet());
        
        // 批量查询，转为 Map
        Map<Long, User> userMap = userService.listByIds(userIds).stream()
            .collect(Collectors.toMap(User::getId, Function.identity()));
        Map<Long, Classroom> classroomMap = classroomService.listByIds(classroomIds).stream()
            .collect(Collectors.toMap(Classroom::getId, Function.identity()));
        
        // 3. 内存组装
        return reservations.stream().map(r -> {
            ReservationVO vo = reservationConverter.toVO(r);
            User user = userMap.get(r.getUserId());
            Classroom classroom = classroomMap.get(r.getClassroomId());
            if (user != null) {
                vo.setUserName(user.getUserName());
            }
            if (classroom != null) {
                vo.setClassroomName(classroom.getName());
            }
            return vo;
        }).collect(Collectors.toList());
    }
}

// ✅ 正确示例3：替代 UNION，分别查询后合并
@Service
public class TagServiceImpl implements TagService {
    
    @Override
    public List<TagVO> listAllTags(Long userId) {
        // 1. 查询自己的标签
        List<Tag> myTags = tagMapper.selectByUserId(userId);
        
        // 2. 查询协作标签（先查协作关系，再查标签）
        List<Long> collaboratorUserIds = collaboratorMapper.selectInviterUserIds(userId);
        List<Tag> collaborativeTags = Collections.emptyList();
        if (!collaboratorUserIds.isEmpty()) {
            collaborativeTags = tagMapper.selectByUserIds(collaboratorUserIds);
        }
        
        // 3. 内存合并
        List<TagVO> result = new ArrayList<>();
        myTags.forEach(tag -> {
            TagVO vo = tagConverter.toVO(tag);
            vo.setIsCollaborative(false);
            result.add(vo);
        });
        collaborativeTags.forEach(tag -> {
            TagVO vo = tagConverter.toVO(tag);
            vo.setIsCollaborative(true);
            result.add(vo);
        });
        
        // 4. 内存排序
        result.sort(Comparator.comparing(TagVO::getIsCollaborative)
            .thenComparing(TagVO::getId));
        
        return result;
    }
}

// ✅ 正确示例4：时间范围查询（避免函数）
@Override
public List<Reservation> listByDate(LocalDate date) {
    // 在应用层计算时间范围
    LocalDateTime startOfDay = date.atStartOfDay();
    LocalDateTime endOfDay = date.plusDays(1).atStartOfDay();
    
    return reservationMapper.selectByTimeRange(startOfDay, endOfDay);
}
```

```xml
<!-- ✅ 正确的 SQL：简单单表查询 -->
<select id="selectByCondition" resultType="Reservation">
    SELECT id, user_id, seat_id, classroom_id, start_time, end_time, status, create_time
    FROM reservation
    WHERE deleted = 0
    <if test="userId != null">
        AND user_id = #{userId}
    </if>
    <if test="status != null">
        AND status = #{status}
    </if>
    ORDER BY create_time DESC
</select>

<!-- ✅ 正确的时间范围查询 -->
<select id="selectByTimeRange" resultType="Reservation">
    SELECT id, user_id, seat_id, start_time, end_time, status
    FROM reservation
    WHERE deleted = 0
    AND create_time >= #{startTime}
    AND create_time &lt; #{endTime}
</select>

<!-- ✅ 正确的批量 ID 查询 -->
<select id="selectByUserIds" resultType="Tag">
    SELECT id, name, user_id, create_time
    FROM tag
    WHERE deleted = 0
    AND user_id IN
    <foreach collection="userIds" item="userId" open="(" separator="," close=")">
        #{userId}
    </foreach>
</select>
```

### 7.2 MyBatis-Plus 使用规范

```java
// 【强制】禁止使用 * 查询所有字段
// 错误示例
List<User> users = userMapper.selectList(null);

// 正确示例 - 指定需要的字段
List<User> users = userMapper.selectList(
    new LambdaQueryWrapper<User>()
        .select(User::getId, User::getUserName, User::getStatus)
);

// 【强制】分页查询必须使用分页插件
@Override
public PageResult<ReservationVO> pageReservations(ReservationQuery query) {
    Page<Reservation> page = new Page<>(query.getPageNum(), query.getPageSize());
    
    LambdaQueryWrapper<Reservation> wrapper = new LambdaQueryWrapper<Reservation>()
            .eq(query.getUserId() != null, Reservation::getUserId, query.getUserId())
            .eq(query.getStatus() != null, Reservation::getStatus, query.getStatus())
            .ge(query.getStartTimeBegin() != null, Reservation::getStartTime, query.getStartTimeBegin())
            .le(query.getStartTimeEnd() != null, Reservation::getStartTime, query.getStartTimeEnd())
            .orderByDesc(Reservation::getCreateTime);
    
    Page<Reservation> result = reservationMapper.selectPage(page, wrapper);
    
    // 分开查询关联数据，避免 JOIN
    List<ReservationVO> voList = assembleReservationVOList(result.getRecords());
    
    return PageResult.of(result.getTotal(), voList);
}

// 【强制】批量操作使用批量方法，避免循环单条操作
// 错误示例
for (Reservation reservation : reservations) {
    reservationMapper.insert(reservation);
}

// 正确示例
reservationService.saveBatch(reservations, 500); // 每批500条
```

### 7.3 SQL 编写规范

```xml
<!-- 【强制】禁止使用 SELECT * -->
<select id="selectByCondition" resultType="Reservation">
    SELECT id, user_id, seat_id, start_time, end_time, status, create_time
    FROM reservation
    WHERE deleted = 0
    <if test="userId != null">
        AND user_id = #{userId}
    </if>
    <if test="status != null">
        AND status = #{status}
    </if>
    ORDER BY create_time DESC
</select>

<!-- 【强制】大表查询必须使用索引，避免全表扫描 -->
<!-- 【强制】IN 查询元素数量不超过 1000 -->
<select id="selectByIds" resultType="Reservation">
    SELECT id, user_id, seat_id, start_time, end_time, status
    FROM reservation
    WHERE deleted = 0
    AND id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>

<!-- 【强制】UPDATE/DELETE 必须带 WHERE 条件 -->
<!-- 【注意】NOW() 是少数允许使用的函数，因为不影响索引 -->
<update id="updateStatus">
    UPDATE reservation
    SET status = #{status}, update_time = NOW()
    WHERE id = #{id} AND deleted = 0
</update>
```

### 7.4 复杂查询的正确处理方式

| 场景 | 错误做法 | 正确做法 |
|------|----------|----------|
| 关联查询 | JOIN 多表 | 分次查询 + Service 层组装 |
| 合并数据 | UNION ALL | 分次查询 + 内存合并 |
| 按日期筛选 | WHERE DATE(time) = ? | WHERE time >= ? AND time < ? |
| 字符串匹配 | LOCATE/FIND_IN_SET | 应用层处理或改数据模型 |
| 统计聚合 | GROUP BY + 复杂计算 | 预计算 + 缓存 |
| 排行榜 | 复杂 SQL 实时计算 | Redis ZSet 缓存 |
| 去重 | SELECT DISTINCT | 业务层去重或优化数据模型 |

### 7.5 事务规范

```java
// 【强制】事务注解加在实现类方法上，不要加在接口上
// 【强制】事务方法必须是 public 的
// 【强制】指定 rollbackFor = Exception.class

@Override
@Transactional(rollbackFor = Exception.class)
public Long createReservation(ReservationDTO dto) {
    // 业务逻辑
}

// 【强制】只读事务使用 readOnly = true
@Override
@Transactional(readOnly = true)
public ReservationVO getReservationById(Long id) {
    // 查询逻辑
}

// 【强制】避免大事务，事务方法应尽量短小
// 【强制】事务内避免远程调用、文件操作等耗时操作

// 错误示例 - 大事务
@Transactional(rollbackFor = Exception.class)
public void processOrder(OrderDTO dto) {
    saveOrder(dto);           // 数据库操作
    sendNotification(dto);    // 远程调用 - 不应该在事务内
    uploadFile(dto);          // 文件操作 - 不应该在事务内
}

// 正确示例 - 拆分事务
public void processOrder(OrderDTO dto) {
    Long orderId = saveOrderWithTransaction(dto);  // 事务方法
    sendNotification(orderId);                      // 非事务方法
    uploadFile(orderId);                            // 非事务方法
}

@Transactional(rollbackFor = Exception.class)
public Long saveOrderWithTransaction(OrderDTO dto) {
    // 仅包含数据库操作
}
```

---

## 8. 缓存规范

### 8.1 缓存 Key 规范

```java
/**
 * 缓存 Key 常量
 * 格式: 业务:模块:标识
 */
public class CacheKeyConstant {
    
    // 用户相关
    public static final String USER_INFO = "user:info:";           // user:info:{userId}
    public static final String USER_TOKEN = "user:token:";         // user:token:{token}
    public static final String USER_PERMISSION = "user:perm:";     // user:perm:{userId}
    
    // 预约相关
    public static final String RESERVATION_INFO = "rsv:info:";     // rsv:info:{reservationId}
    public static final String SEAT_STATUS = "seat:status:";       // seat:status:{seatId}:{date}
    public static final String SEAT_LOCK = "seat:lock:";           // seat:lock:{seatId}:{timeSlot}
    
    // 统计相关
    public static final String STATS_DAILY = "stats:daily:";       // stats:daily:{date}
    public static final String RANK_WEEKLY = "rank:weekly:";       // rank:weekly:{week}
}
```

### 8.2 缓存使用规范

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class UserCacheService {
    
    private final StringRedisTemplate redisTemplate;
    private final UserMapper userMapper;
    
    private static final long USER_CACHE_TTL = 3600L; // 1小时
    private static final long NULL_CACHE_TTL = 60L;   // 空值缓存1分钟，防止缓存穿透
    
    /**
     * 获取用户信息（带缓存）
     */
    public UserVO getUserById(Long userId) {
        String cacheKey = CacheKeyConstant.USER_INFO + userId;
        
        // 1. 查询缓存
        String cacheValue = redisTemplate.opsForValue().get(cacheKey);
        if (cacheValue != null) {
            if ("NULL".equals(cacheValue)) {
                return null; // 空值缓存
            }
            return JSON.parseObject(cacheValue, UserVO.class);
        }
        
        // 2. 查询数据库
        User user = userMapper.selectById(userId);
        
        // 3. 写入缓存
        if (user != null) {
            UserVO vo = userConverter.toVO(user);
            redisTemplate.opsForValue().set(
                    cacheKey, 
                    JSON.toJSONString(vo), 
                    USER_CACHE_TTL, 
                    TimeUnit.SECONDS
            );
            return vo;
        } else {
            // 缓存空值，防止缓存穿透
            redisTemplate.opsForValue().set(cacheKey, "NULL", NULL_CACHE_TTL, TimeUnit.SECONDS);
            return null;
        }
    }
    
    /**
     * 更新用户信息（删除缓存）
     */
    @Transactional(rollbackFor = Exception.class)
    public void updateUser(UserDTO dto) {
        // 1. 更新数据库
        userMapper.updateById(userConverter.toEntity(dto));
        
        // 2. 删除缓存（延迟双删可选）
        String cacheKey = CacheKeyConstant.USER_INFO + dto.getId();
        redisTemplate.delete(cacheKey);
    }
}
```

### 8.3 本地缓存规范

```java
/**
 * 使用 Caffeine 作为本地缓存
 * 适用于：读多写少、数据量小、对实时性要求不高的场景
 */
@Configuration
public class LocalCacheConfig {
    
    /**
     * 字典缓存
     * 最大容量 1000，写入后 10 分钟过期
     */
    @Bean
    public Cache<String, List<DictVO>> dictCache() {
        return Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(10, TimeUnit.MINUTES)
                .recordStats()
                .build();
    }
    
    /**
     * 配置缓存
     * 最大容量 500，写入后 30 分钟过期
     */
    @Bean
    public Cache<String, String> configCache() {
        return Caffeine.newBuilder()
                .maximumSize(500)
                .expireAfterWrite(30, TimeUnit.MINUTES)
                .build();
    }
}

@Service
@RequiredArgsConstructor
public class DictService {
    
    private final Cache<String, List<DictVO>> dictCache;
    private final DictMapper dictMapper;
    
    public List<DictVO> getDictByType(String dictType) {
        return dictCache.get(dictType, key -> {
            // 缓存未命中，从数据库加载
            return dictMapper.selectByType(key);
        });
    }
    
    public void refreshCache(String dictType) {
        dictCache.invalidate(dictType);
    }
}
```

---

## 9. 安全规范

### 9.1 参数校验

```java
// 【强制】Controller 层使用 @Valid 校验
@PostMapping
public Result<Long> create(@Valid @RequestBody ReservationDTO dto) {
    return Result.success(reservationService.createReservation(dto));
}

// 【强制】DTO 使用 JSR-303 注解
@Data
public class ReservationDTO {
    
    @NotNull(message = "座位ID不能为空")
    @Positive(message = "座位ID必须为正数")
    private Long seatId;
    
    @NotNull(message = "开始时间不能为空")
    @Future(message = "开始时间必须是将来时间")
    private LocalDateTime startTime;
    
    @NotNull(message = "结束时间不能为空")
    private LocalDateTime endTime;
    
    @Size(max = 200, message = "备注不能超过200字")
    private String remark;
}

// 【强制】Service 层进行业务校验
public Long createReservation(ReservationDTO dto) {
    // 业务校验
    if (dto.getEndTime().isBefore(dto.getStartTime())) {
        throw new BusinessException(ResultCode.INVALID_TIME_RANGE);
    }
    // ...
}
```

### 9.2 SQL 注入防护

```java
// 【强制】使用参数化查询，禁止拼接 SQL
// 错误示例
String sql = "SELECT * FROM user WHERE name = '" + name + "'";

// 正确示例 - MyBatis 参数绑定
@Select("SELECT * FROM user WHERE name = #{name}")
User selectByName(@Param("name") String name);

// 【强制】动态排序字段必须白名单校验
public PageResult<UserVO> pageUsers(UserQuery query) {
    // 白名单校验排序字段
    Set<String> allowedSortFields = Set.of("id", "created_time", "user_name");
    if (!allowedSortFields.contains(query.getSortField())) {
        query.setSortField("id"); // 默认排序字段
    }
    // ...
}
```

### 9.3 XSS 防护

```java
// 【强制】输出到页面的内容必须转义
@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        // 配置 XSS 过滤
        SimpleModule module = new SimpleModule();
        module.addSerializer(String.class, new XssStringSerializer());
        mapper.registerModule(module);
        return mapper;
    }
}

public class XssStringSerializer extends JsonSerializer<String> {
    
    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider provider) 
            throws IOException {
        if (value != null) {
            gen.writeString(HtmlUtils.htmlEscape(value));
        }
    }
}
```

### 9.4 敏感数据处理

```java
// 【强制】敏感数据日志脱敏
@Slf4j
public class UserService {
    
    public void register(UserDTO dto) {
        log.info("用户注册, phone={}, email={}", 
                DesensitizeUtil.phone(dto.getPhone()),
                DesensitizeUtil.email(dto.getEmail()));
    }
}

// 【强制】敏感数据返回脱敏
@Data
public class UserVO {
    
    private Long id;
    
    private String userName;
    
    @JsonSerialize(using = PhoneDesensitizeSerializer.class)
    private String phone;
    
    @JsonSerialize(using = EmailDesensitizeSerializer.class)
    private String email;
    
    @JsonSerialize(using = IdCardDesensitizeSerializer.class)
    private String idCard;
}

// 脱敏工具类
public class DesensitizeUtil {
    
    /**
     * 手机号脱敏: 138****1234
     */
    public static String phone(String phone) {
        if (StringUtils.isBlank(phone) || phone.length() != 11) {
            return phone;
        }
        return phone.substring(0, 3) + "****" + phone.substring(7);
    }
    
    /**
     * 邮箱脱敏: z***@example.com
     */
    public static String email(String email) {
        if (StringUtils.isBlank(email) || !email.contains("@")) {
            return email;
        }
        int atIndex = email.indexOf("@");
        if (atIndex <= 1) {
            return email;
        }
        return email.charAt(0) + "***" + email.substring(atIndex);
    }
}
```

---

## 10. 性能规范

### 10.1 接口性能要求

| 接口类型 | P99 响应时间 | P95 响应时间 |
|----------|--------------|--------------|
| 简单查询 | < 50ms | < 30ms |
| 复杂查询 | < 200ms | < 100ms |
| 写入操作 | < 100ms | < 50ms |
| 批量操作 | < 500ms | < 300ms |

### 10.2 数据库性能优化

```java
// 【强制】查询必须走索引，禁止全表扫描
// 【强制】单表数据量超过 500 万考虑分表
// 【强制】单次查询数据量不超过 1000 条

// 【强制】批量插入使用 BATCH 模式
@Transactional(rollbackFor = Exception.class)
public void batchSaveReservations(List<ReservationDTO> dtos) {
    List<Reservation> entities = reservationConverter.toEntityList(dtos);
    
    // 分批插入，每批 500 条
    int batchSize = 500;
    for (int i = 0; i < entities.size(); i += batchSize) {
        int end = Math.min(i + batchSize, entities.size());
        List<Reservation> batch = entities.subList(i, end);
        reservationMapper.insertBatchSomeColumn(batch);
    }
}

// 【强制】大数据量导出使用流式查询
public void exportReservations(OutputStream out, ReservationQuery query) {
    try (Cursor<Reservation> cursor = reservationMapper.selectCursor(query)) {
        cursor.forEach(reservation -> {
            // 写入输出流
            writeToStream(out, reservation);
        });
    }
}
```

### 10.3 缓存性能优化

```java
// 【强制】热点数据必须缓存
// 【强制】缓存命中率目标 > 95%

// 【推荐】使用多级缓存
@Service
public class SeatStatusService {
    
    private final Cache<String, SeatStatus> localCache;  // L1: 本地缓存
    private final StringRedisTemplate redisTemplate;      // L2: Redis 缓存
    private final SeatMapper seatMapper;                  // L3: 数据库
    
    public SeatStatus getSeatStatus(Long seatId, LocalDate date) {
        String cacheKey = "seat:status:" + seatId + ":" + date;
        
        // L1: 本地缓存
        SeatStatus status = localCache.getIfPresent(cacheKey);
        if (status != null) {
            return status;
        }
        
        // L2: Redis 缓存
        String redisValue = redisTemplate.opsForValue().get(cacheKey);
        if (redisValue != null) {
            status = JSON.parseObject(redisValue, SeatStatus.class);
            localCache.put(cacheKey, status);
            return status;
        }
        
        // L3: 数据库
        status = seatMapper.selectStatus(seatId, date);
        if (status != null) {
            redisTemplate.opsForValue().set(cacheKey, JSON.toJSONString(status), 5, TimeUnit.MINUTES);
            localCache.put(cacheKey, status);
        }
        
        return status;
    }
}
```

### 10.4 异步处理

```java
// 【推荐】非核心流程异步处理
@Service
@RequiredArgsConstructor
public class ReservationServiceImpl implements ReservationService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long createReservation(ReservationDTO dto) {
        // 核心流程：创建预约
        Reservation reservation = doCreateReservation(dto);
        
        // 非核心流程：发布事件异步处理
        eventPublisher.publishEvent(new ReservationCreatedEvent(reservation));
        
        return reservation.getId();
    }
}

@Slf4j
@Component
@RequiredArgsConstructor
public class ReservationEventListener {
    
    private final NotificationService notificationService;
    private final StatisticsService statisticsService;
    
    @Async("businessExecutor")
    @EventListener
    public void handleReservationCreated(ReservationCreatedEvent event) {
        Reservation reservation = event.getReservation();
        
        try {
            // 发送通知
            notificationService.sendReservationNotification(reservation);
            
            // 更新统计
            statisticsService.incrementReservationCount(reservation.getSeatId());
        } catch (Exception e) {
            log.error("处理预约创建事件失败, reservationId={}", reservation.getId(), e);
        }
    }
}
```

### 10.5 连接池优化

```yaml
# HikariCP 连接池配置
spring:
  datasource:
    hikari:
      # 最小空闲连接数
      minimum-idle: 10
      # 最大连接数 = CPU核心数 * 2 + 有效磁盘数
      maximum-pool-size: 20
      # 连接超时时间
      connection-timeout: 30000
      # 空闲连接超时时间
      idle-timeout: 600000
      # 连接最大生命周期
      max-lifetime: 1800000
      # 连接测试查询
      connection-test-query: SELECT 1

# Redis 连接池配置
  redis:
    lettuce:
      pool:
        # 最大连接数
        max-active: 20
        # 最大空闲连接数
        max-idle: 10
        # 最小空闲连接数
        min-idle: 5
        # 获取连接最大等待时间
        max-wait: 3000ms
```

