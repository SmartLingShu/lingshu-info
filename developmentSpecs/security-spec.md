# 安全开发规范

> 以安全编码、漏洞防护、数据保护为核心，覆盖前后端全链路安全。

## 目录

- [1. 认证授权安全](#1-认证授权安全)
- [2. 输入验证与输出编码](#2-输入验证与输出编码)
- [3. SQL 注入防护](#3-sql-注入防护)
- [4. XSS 防护](#4-xss-防护)
- [5. CSRF 防护](#5-csrf-防护)
- [6. 敏感数据保护](#6-敏感数据保护)
- [7. 接口安全](#7-接口安全)
- [8. 日志安全](#8-日志安全)

---

## 1. 认证授权安全

### 1.1 密码安全

```java
// 【强制】密码加密存储，使用 BCrypt
@Service
public class PasswordService {
    
    private final PasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
    
    public String encode(String rawPassword) {
        return passwordEncoder.encode(rawPassword);
    }
    
    public boolean matches(String rawPassword, String encodedPassword) {
        return passwordEncoder.matches(rawPassword, encodedPassword);
    }
}

// 【强制】密码强度校验
public class PasswordValidator {
    
    private static final Pattern PASSWORD_PATTERN = Pattern.compile(
        "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,20}$"
    );
    
    public static boolean isValid(String password) {
        // 至少8位，包含大小写字母、数字、特殊字符
        return PASSWORD_PATTERN.matcher(password).matches();
    }
}

// 【强制】登录失败限制
@Service
public class LoginAttemptService {
    
    private final StringRedisTemplate redisTemplate;
    private static final int MAX_ATTEMPTS = 5;
    private static final int LOCK_MINUTES = 30;
    
    public void recordFailedAttempt(String username) {
        String key = "login:failed:" + username;
        Long attempts = redisTemplate.opsForValue().increment(key);
        if (attempts == 1) {
            redisTemplate.expire(key, LOCK_MINUTES, TimeUnit.MINUTES);
        }
    }
    
    public boolean isBlocked(String username) {
        String key = "login:failed:" + username;
        String attempts = redisTemplate.opsForValue().get(key);
        return attempts != null && Integer.parseInt(attempts) >= MAX_ATTEMPTS;
    }
    
    public void clearAttempts(String username) {
        redisTemplate.delete("login:failed:" + username);
    }
}
```

### 1.2 Token 安全

```java
// 【强制】JWT Token 配置
@Component
public class JwtTokenProvider {
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.access-token-expire:7200}")  // 2小时
    private long accessTokenExpire;
    
    @Value("${jwt.refresh-token-expire:604800}")  // 7天
    private long refreshTokenExpire;
    
    public String createAccessToken(Long userId, List<String> roles) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + accessTokenExpire * 1000);
        
        return Jwts.builder()
            .setSubject(String.valueOf(userId))
            .claim("roles", roles)
            .setIssuedAt(now)
            .setExpiration(expiry)
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
    }
    
    public Claims parseToken(String token) {
        return Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody();
    }
}

// 【强制】Token 黑名单（登出/修改密码后失效）
@Service
public class TokenBlacklistService {
    
    private final StringRedisTemplate redisTemplate;
    
    public void addToBlacklist(String token, long expireSeconds) {
        String key = "token:blacklist:" + token;
        redisTemplate.opsForValue().set(key, "1", expireSeconds, TimeUnit.SECONDS);
    }
    
    public boolean isBlacklisted(String token) {
        return Boolean.TRUE.equals(redisTemplate.hasKey("token:blacklist:" + token));
    }
}
```

### 1.3 权限控制

```java
// 【强制】接口权限注解
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission {
    String value();
}

// 【强制】权限校验切面
@Aspect
@Component
@RequiredArgsConstructor
public class PermissionAspect {
    
    private final UserService userService;
    
    @Before("@annotation(requirePermission)")
    public void checkPermission(RequirePermission requirePermission) {
        String permission = requirePermission.value();
        Long userId = SecurityContextHolder.getUserId();
        
        if (!userService.hasPermission(userId, permission)) {
            throw new BusinessException(ResultCode.FORBIDDEN);
        }
    }
}

// 使用示例
@RequirePermission("reservation:create")
@PostMapping
public Result<Long> createReservation(@RequestBody ReservationDTO dto) {
    // ...
}

// 【强制】数据权限控制
@Service
public class ReservationServiceImpl implements ReservationService {
    
    @Override
    public ReservationVO getReservationById(Long id) {
        Reservation reservation = reservationMapper.selectById(id);
        
        // 校验数据权限：只能查看自己的预约
        Long currentUserId = SecurityContextHolder.getUserId();
        if (!reservation.getUserId().equals(currentUserId) 
                && !SecurityContextHolder.isAdmin()) {
            throw new BusinessException(ResultCode.FORBIDDEN);
        }
        
        return reservationConverter.toVO(reservation);
    }
}
```

---

## 2. 输入验证与输出编码

### 2.1 输入验证

```java
// 【强制】使用 JSR-303 注解校验
@Data
public class ReservationDTO {
    
    @NotNull(message = "座位ID不能为空")
    @Positive(message = "座位ID必须为正数")
    private Long seatId;
    
    @NotNull(message = "预约日期不能为空")
    @Future(message = "预约日期必须是将来日期")
    private LocalDate reservationDate;
    
    @NotBlank(message = "开始时间不能为空")
    @Pattern(regexp = "^([01]\\d|2[0-3]):[0-5]\\d$", message = "时间格式错误")
    private String startTime;
    
    @Size(max = 500, message = "备注不能超过500字")
    private String remark;
}

// 【强制】自定义校验器
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
public @interface Phone {
    String message() default "手机号格式错误";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneValidator implements ConstraintValidator<Phone, String> {
    
    private static final Pattern PHONE_PATTERN = Pattern.compile("^1[3-9]\\d{9}$");
    
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (StringUtils.isBlank(value)) {
            return true;  // 空值由 @NotBlank 校验
        }
        return PHONE_PATTERN.matcher(value).matches();
    }
}

// 【强制】白名单校验
public class SortFieldValidator {
    
    private static final Set<String> ALLOWED_SORT_FIELDS = Set.of(
        "id", "created_time", "updated_time", "status"
    );
    
    public static String validate(String sortField) {
        if (StringUtils.isBlank(sortField)) {
            return "id";
        }
        if (!ALLOWED_SORT_FIELDS.contains(sortField)) {
            throw new BusinessException(ResultCode.PARAM_ERROR, "非法排序字段");
        }
        return sortField;
    }
}
```

### 2.2 输出编码

```java
// 【强制】HTML 转义
public class HtmlUtils {
    
    public static String escape(String input) {
        if (input == null) {
            return null;
        }
        return input
            .replace("&", "&amp;")
            .replace("<", "&lt;")
            .replace(">", "&gt;")
            .replace("\"", "&quot;")
            .replace("'", "&#x27;");
    }
}

// 【强制】JSON 序列化时自动转义
@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
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
            gen.writeString(HtmlUtils.escape(value));
        }
    }
}
```

---

## 3. SQL 注入防护

### 3.1 参数化查询

```java
// 【强制】使用 MyBatis 参数绑定
// 正确：使用 #{} 参数绑定
@Select("SELECT * FROM t_user WHERE id = #{id}")
User selectById(@Param("id") Long id);

// 错误：使用 ${} 字符串拼接（有注入风险）
@Select("SELECT * FROM t_user WHERE id = ${id}")
User selectByIdUnsafe(@Param("id") Long id);

// 【强制】动态 SQL 安全写法
<select id="selectByCondition" resultType="User">
    SELECT id, user_name, phone, status
    FROM t_user
    WHERE is_deleted = 0
    <if test="userName != null and userName != ''">
        AND user_name LIKE CONCAT('%', #{userName}, '%')
    </if>
    <if test="status != null">
        AND status = #{status}
    </if>
    ORDER BY created_time DESC
</select>

// 【强制】IN 查询使用 foreach
<select id="selectByIds" resultType="User">
    SELECT * FROM t_user
    WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

### 3.2 动态排序安全处理

```java
// 【强制】排序字段白名单校验
public PageResult<UserVO> pageUsers(UserQuery query) {
    // 白名单校验
    String sortField = SortFieldValidator.validate(query.getSortField());
    String sortOrder = "desc".equalsIgnoreCase(query.getSortOrder()) ? "DESC" : "ASC";
    
    // 使用校验后的值
    query.setSortField(sortField);
    query.setSortOrder(sortOrder);
    
    return userMapper.selectPage(query);
}

// MyBatis 中使用 ${} 但已经过白名单校验
<select id="selectPage" resultType="User">
    SELECT * FROM t_user
    WHERE is_deleted = 0
    ORDER BY ${sortField} ${sortOrder}
    LIMIT #{offset}, #{pageSize}
</select>
```

---

## 4. XSS 防护

### 4.1 后端防护

```java
// 【强制】XSS 过滤器
@Component
public class XssFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        chain.doFilter(new XssHttpServletRequestWrapper((HttpServletRequest) request), response);
    }
}

public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {
    
    public XssHttpServletRequestWrapper(HttpServletRequest request) {
        super(request);
    }
    
    @Override
    public String getParameter(String name) {
        String value = super.getParameter(name);
        return cleanXss(value);
    }
    
    @Override
    public String[] getParameterValues(String name) {
        String[] values = super.getParameterValues(name);
        if (values == null) {
            return null;
        }
        return Arrays.stream(values)
            .map(this::cleanXss)
            .toArray(String[]::new);
    }
    
    private String cleanXss(String value) {
        if (value == null) {
            return null;
        }
        // 使用 OWASP Java HTML Sanitizer
        PolicyFactory policy = Sanitizers.FORMATTING.and(Sanitizers.LINKS);
        return policy.sanitize(value);
    }
}
```

### 4.2 前端防护

```typescript
// 【强制】使用 v-text 而非 v-html
// 正确
<div v-text="userInput"></div>

// 危险（除非内容已经过安全处理）
<div v-html="userInput"></div>

// 【强制】如必须使用 v-html，先进行消毒
import DOMPurify from 'dompurify';

const sanitizedHtml = DOMPurify.sanitize(userInput);

// 【强制】URL 参数编码
const url = `/search?keyword=${encodeURIComponent(keyword)}`;

// 【强制】CSP 配置
// index.html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';">
```

---

## 5. CSRF 防护

### 5.1 Token 验证

```java
// 【强制】CSRF Token 生成与验证
@Component
public class CsrfTokenService {
    
    private final StringRedisTemplate redisTemplate;
    
    public String generateToken(String sessionId) {
        String token = UUID.randomUUID().toString();
        String key = "csrf:token:" + sessionId;
        redisTemplate.opsForValue().set(key, token, 30, TimeUnit.MINUTES);
        return token;
    }
    
    public boolean validateToken(String sessionId, String token) {
        String key = "csrf:token:" + sessionId;
        String storedToken = redisTemplate.opsForValue().get(key);
        return token != null && token.equals(storedToken);
    }
}

// 【强制】CSRF 过滤器
@Component
public class CsrfFilter implements Filter {
    
    private final CsrfTokenService csrfTokenService;
    
    private static final Set<String> SAFE_METHODS = Set.of("GET", "HEAD", "OPTIONS");
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // 安全方法不校验
        if (SAFE_METHODS.contains(httpRequest.getMethod())) {
            chain.doFilter(request, response);
            return;
        }
        
        // 校验 CSRF Token
        String sessionId = httpRequest.getSession().getId();
        String token = httpRequest.getHeader("X-CSRF-Token");
        
        if (!csrfTokenService.validateToken(sessionId, token)) {
            HttpServletResponse httpResponse = (HttpServletResponse) response;
            httpResponse.setStatus(HttpServletResponse.SC_FORBIDDEN);
            httpResponse.getWriter().write("CSRF token invalid");
            return;
        }
        
        chain.doFilter(request, response);
    }
}
```

### 5.2 SameSite Cookie

```java
// 【强制】设置 SameSite Cookie 属性
@Configuration
public class CookieConfig {
    
    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setSameSite("Strict");
        serializer.setUseSecureCookie(true);
        serializer.setUseHttpOnlyCookie(true);
        return serializer;
    }
}
```

---

## 6. 敏感数据保护

### 6.1 数据脱敏

```java
// 【强制】返回数据脱敏
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

// 手机号脱敏：138****8000
public class PhoneDesensitizeSerializer extends JsonSerializer<String> {
    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider provider) 
            throws IOException {
        if (value != null && value.length() == 11) {
            gen.writeString(value.substring(0, 3) + "****" + value.substring(7));
        } else {
            gen.writeString(value);
        }
    }
}

// 邮箱脱敏：z***@example.com
public class EmailDesensitizeSerializer extends JsonSerializer<String> {
    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider provider) 
            throws IOException {
        if (value != null && value.contains("@")) {
            int atIndex = value.indexOf("@");
            if (atIndex > 1) {
                gen.writeString(value.charAt(0) + "***" + value.substring(atIndex));
                return;
            }
        }
        gen.writeString(value);
    }
}
```

### 6.2 数据加密

```java
// 【强制】敏感字段加密存储
@Component
public class AesEncryptor {
    
    @Value("${security.aes.key}")
    private String key;
    
    private static final String ALGORITHM = "AES/GCM/NoPadding";
    
    public String encrypt(String plainText) {
        try {
            SecretKeySpec secretKey = new SecretKeySpec(key.getBytes(), "AES");
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            byte[] iv = new byte[12];
            SecureRandom.getInstanceStrong().nextBytes(iv);
            cipher.init(Cipher.ENCRYPT_MODE, secretKey, new GCMParameterSpec(128, iv));
            byte[] encrypted = cipher.doFinal(plainText.getBytes());
            byte[] combined = new byte[iv.length + encrypted.length];
            System.arraycopy(iv, 0, combined, 0, iv.length);
            System.arraycopy(encrypted, 0, combined, iv.length, encrypted.length);
            return Base64.getEncoder().encodeToString(combined);
        } catch (Exception e) {
            throw new RuntimeException("加密失败", e);
        }
    }
    
    public String decrypt(String cipherText) {
        try {
            byte[] combined = Base64.getDecoder().decode(cipherText);
            byte[] iv = Arrays.copyOfRange(combined, 0, 12);
            byte[] encrypted = Arrays.copyOfRange(combined, 12, combined.length);
            SecretKeySpec secretKey = new SecretKeySpec(key.getBytes(), "AES");
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, secretKey, new GCMParameterSpec(128, iv));
            return new String(cipher.doFinal(encrypted));
        } catch (Exception e) {
            throw new RuntimeException("解密失败", e);
        }
    }
}

// MyBatis TypeHandler 自动加解密
@MappedTypes(String.class)
public class EncryptTypeHandler extends BaseTypeHandler<String> {
    
    @Autowired
    private AesEncryptor encryptor;
    
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) 
            throws SQLException {
        ps.setString(i, encryptor.encrypt(parameter));
    }
    
    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String value = rs.getString(columnName);
        return value != null ? encryptor.decrypt(value) : null;
    }
}
```

---

## 7. 接口安全

### 7.1 接口签名

```java
// 【强制】对外开放接口签名验证
@Component
public class SignatureService {
    
    public String generateSign(Map<String, String> params, String secret) {
        // 1. 参数按 key 排序
        String sortedParams = params.entrySet().stream()
            .sorted(Map.Entry.comparingByKey())
            .map(e -> e.getKey() + "=" + e.getValue())
            .collect(Collectors.joining("&"));
        
        // 2. 拼接密钥
        String signStr = sortedParams + "&secret=" + secret;
        
        // 3. MD5 签名
        return DigestUtils.md5DigestAsHex(signStr.getBytes());
    }
    
    public boolean verifySign(Map<String, String> params, String sign, String secret) {
        String expectedSign = generateSign(params, secret);
        return expectedSign.equalsIgnoreCase(sign);
    }
}

// 签名验证拦截器
@Component
public class SignatureInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String appId = request.getHeader("X-App-Id");
        String timestamp = request.getHeader("X-Timestamp");
        String nonce = request.getHeader("X-Nonce");
        String sign = request.getHeader("X-Sign");
        
        // 1. 验证时间戳（防止重放攻击，5分钟内有效）
        long requestTime = Long.parseLong(timestamp);
        if (Math.abs(System.currentTimeMillis() - requestTime) > 5 * 60 * 1000) {
            throw new BusinessException("请求已过期");
        }
        
        // 2. 验证 nonce（防止重放攻击）
        if (nonceService.exists(nonce)) {
            throw new BusinessException("重复请求");
        }
        nonceService.save(nonce, 5, TimeUnit.MINUTES);
        
        // 3. 验证签名
        String secret = appService.getSecret(appId);
        Map<String, String> params = new HashMap<>();
        params.put("appId", appId);
        params.put("timestamp", timestamp);
        params.put("nonce", nonce);
        // 添加请求参数...
        
        if (!signatureService.verifySign(params, sign, secret)) {
            throw new BusinessException("签名验证失败");
        }
        
        return true;
    }
}
```

### 7.2 接口限流

```java
// 【强制】接口限流
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int limit() default 100;      // 限制次数
    int period() default 60;      // 时间窗口（秒）
    String key() default "";      // 限流 key
}

@Aspect
@Component
@RequiredArgsConstructor
public class RateLimitAspect {
    
    private final StringRedisTemplate redisTemplate;
    
    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint point, RateLimit rateLimit) throws Throwable {
        String key = buildKey(point, rateLimit);
        
        // 使用 Redis + Lua 实现滑动窗口限流
        String script = """
            local key = KEYS[1]
            local limit = tonumber(ARGV[1])
            local period = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            
            redis.call('ZREMRANGEBYSCORE', key, 0, now - period * 1000)
            local count = redis.call('ZCARD', key)
            
            if count < limit then
                redis.call('ZADD', key, now, now .. '-' .. math.random())
                redis.call('EXPIRE', key, period)
                return 1
            else
                return 0
            end
            """;
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(key),
            String.valueOf(rateLimit.limit()),
            String.valueOf(rateLimit.period()),
            String.valueOf(System.currentTimeMillis())
        );
        
        if (result == null || result == 0) {
            throw new BusinessException(ResultCode.TOO_MANY_REQUESTS);
        }
        
        return point.proceed();
    }
}

// 使用示例
@RateLimit(limit = 10, period = 60, key = "login")
@PostMapping("/login")
public Result<TokenVO> login(@RequestBody LoginDTO dto) {
    // ...
}
```

---

## 8. 日志安全

### 8.1 日志脱敏

```java
// 【强制】日志中敏感信息脱敏
@Slf4j
public class UserService {
    
    public void register(UserDTO dto) {
        log.info("用户注册, phone={}, email={}", 
            DesensitizeUtil.phone(dto.getPhone()),
            DesensitizeUtil.email(dto.getEmail()));
    }
}

// 【强制】禁止记录密码、Token 等敏感信息
// 错误
log.info("用户登录, username={}, password={}", username, password);

// 正确
log.info("用户登录, username={}", username);
```

### 8.2 审计日志

```java
// 【强制】关键操作记录审计日志
@Aspect
@Component
@RequiredArgsConstructor
public class AuditLogAspect {
    
    private final AuditLogMapper auditLogMapper;
    
    @AfterReturning(pointcut = "@annotation(auditLog)", returning = "result")
    public void afterReturning(JoinPoint point, AuditLog auditLog, Object result) {
        AuditLogEntity log = new AuditLogEntity();
        log.setUserId(SecurityContextHolder.getUserId());
        log.setOperation(auditLog.value());
        log.setMethod(point.getSignature().toString());
        log.setParams(JSON.toJSONString(point.getArgs()));
        log.setResult(JSON.toJSONString(result));
        log.setIp(RequestContextHolder.getClientIp());
        log.setCreatedTime(LocalDateTime.now());
        
        auditLogMapper.insert(log);
    }
}

// 使用示例
@AuditLog("删除用户")
@DeleteMapping("/{id}")
public Result<Void> deleteUser(@PathVariable Long id) {
    // ...
}
```

---

## 附录

### A. 安全检查清单

```
□ 密码是否加密存储
□ 是否有登录失败限制
□ Token 是否有过期机制
□ 是否有权限校验
□ 输入是否有校验
□ 是否有 SQL 注入风险
□ 是否有 XSS 风险
□ 是否有 CSRF 防护
□ 敏感数据是否脱敏
□ 日志是否脱敏
□ 接口是否有限流
□ 是否使用 HTTPS
```

### B. 常见漏洞类型

| 漏洞类型 | 防护措施 |
|----------|----------|
| SQL 注入 | 参数化查询 |
| XSS | 输入过滤、输出编码 |
| CSRF | Token 验证、SameSite Cookie |
| 越权访问 | 权限校验 |
| 敏感信息泄露 | 数据脱敏、加密 |
| 暴力破解 | 登录限制、验证码 |
| 重放攻击 | 时间戳、Nonce |
