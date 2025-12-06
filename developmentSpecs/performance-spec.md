# 性能优化规范

> 以高并发、低延迟、高可用为目标，覆盖前后端全链路性能优化。

## 目录

- [1. 性能指标](#1-性能指标)
- [2. 后端性能优化](#2-后端性能优化)
- [3. 前端性能优化](#3-前端性能优化)
- [4. 数据库性能优化](#4-数据库性能优化)
- [5. 缓存优化](#5-缓存优化)
- [6. 网络优化](#6-网络优化)
- [7. 监控与调优](#7-监控与调优)

---

## 1. 性能指标

### 1.1 响应时间目标

| 接口类型 | P50 | P95 | P99 |
|----------|-----|-----|-----|
| 简单查询 | < 20ms | < 50ms | < 100ms |
| 复杂查询 | < 50ms | < 150ms | < 300ms |
| 写入操作 | < 30ms | < 80ms | < 150ms |
| 批量操作 | < 200ms | < 400ms | < 800ms |
| 页面加载 | < 1s | < 2s | < 3s |

### 1.2 吞吐量目标

| 场景 | 目标 RPS | 说明 |
|------|----------|------|
| 单实例 | > 1000 | 普通业务接口 |
| 热点接口 | > 5000 | 如座位状态查询 |
| 写入接口 | > 500 | 如创建预约 |

### 1.3 资源利用率

| 指标 | 正常范围 | 告警阈值 |
|------|----------|----------|
| CPU | < 60% | > 80% |
| 内存 | < 70% | > 85% |
| 磁盘 IO | < 60% | > 80% |
| 网络带宽 | < 50% | > 70% |
| 数据库连接 | < 70% | > 85% |

---

## 2. 后端性能优化

### 2.1 代码层优化

```java
// 【强制】避免在循环中进行数据库/网络操作
// 错误示例
for (Long userId : userIds) {
    User user = userMapper.selectById(userId);  // N次查询
    // ...
}

// 正确示例
List<User> users = userMapper.selectBatchIds(userIds);  // 1次查询
Map<Long, User> userMap = users.stream()
    .collect(Collectors.toMap(User::getId, Function.identity()));

// 【强制】使用批量操作
// 错误示例
for (Reservation r : reservations) {
    reservationMapper.insert(r);  // N次插入
}

// 正确示例
reservationService.saveBatch(reservations, 500);  // 批量插入

// 【强制】合理使用并行流
// 适用于 CPU 密集型、数据量大、无共享状态
List<ReservationVO> vos = reservations.parallelStream()
    .map(this::convertToVO)
    .collect(Collectors.toList());

// 【强制】避免创建不必要的对象
// 错误示例
String result = "";
for (String s : list) {
    result += s;  // 每次创建新对象
}

// 正确示例
StringBuilder sb = new StringBuilder();
for (String s : list) {
    sb.append(s);
}
String result = sb.toString();
```

### 2.2 异步处理

```java
// 【强制】非核心流程异步处理
@Service
@RequiredArgsConstructor
public class ReservationServiceImpl implements ReservationService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long createReservation(ReservationDTO dto) {
        // 核心流程：同步执行
        Reservation reservation = doCreateReservation(dto);
        
        // 非核心流程：异步执行
        eventPublisher.publishEvent(new ReservationCreatedEvent(reservation));
        
        return reservation.getId();
    }
}

@Async("businessExecutor")
@EventListener
public void handleReservationCreated(ReservationCreatedEvent event) {
    // 发送通知
    notificationService.send(event.getReservation());
    // 更新统计
    statisticsService.increment(event.getReservation());
}

// 【强制】线程池配置
@Bean("businessExecutor")
public ThreadPoolExecutor businessExecutor() {
    int coreSize = Runtime.getRuntime().availableProcessors();
    return new ThreadPoolExecutor(
        coreSize,
        coreSize * 2,
        60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(1000),
        new ThreadFactoryBuilder().setNameFormat("business-%d").build(),
        new ThreadPoolExecutor.CallerRunsPolicy()
    );
}
```

### 2.3 连接池优化

```yaml
# HikariCP 配置
spring:
  datasource:
    hikari:
      minimum-idle: 10
      maximum-pool-size: 20
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      connection-test-query: SELECT 1

# Redis 连接池配置
  redis:
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 3000ms

# HTTP 连接池配置（RestTemplate/WebClient）
@Bean
public RestTemplate restTemplate() {
    PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
    cm.setMaxTotal(200);
    cm.setDefaultMaxPerRoute(50);
    
    CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(cm)
        .build();
    
    HttpComponentsClientHttpRequestFactory factory = 
        new HttpComponentsClientHttpRequestFactory(httpClient);
    factory.setConnectTimeout(3000);
    factory.setReadTimeout(5000);
    
    return new RestTemplate(factory);
}
```

### 2.4 JVM 优化

```bash
# 堆内存配置
-Xms4g -Xmx4g                    # 堆大小固定，避免动态调整
-XX:NewRatio=2                   # 新生代:老年代 = 1:2
-XX:SurvivorRatio=8              # Eden:Survivor = 8:1

# GC 配置（JDK 17 推荐 G1）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200         # 目标暂停时间
-XX:G1HeapRegionSize=16m         # Region 大小

# 元空间配置
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m

# GC 日志
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=5,filesize=100m

# 其他优化
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdump.hprof
-XX:+DisableExplicitGC           # 禁止显式 GC
```

---

## 3. 前端性能优化

### 3.1 加载优化

```typescript
// 【强制】路由懒加载
const routes = [
  {
    path: '/reservation',
    component: () => import('@/views/reservation/index.vue'),
  },
];

// 【强制】组件懒加载
const HeavyComponent = defineAsyncComponent(() =>
  import('@/components/HeavyComponent.vue')
);

// 【强制】图片懒加载
<img v-lazy="imageUrl" />

// 【强制】按需导入
// 错误
import _ from 'lodash';

// 正确
import debounce from 'lodash/debounce';

// 【强制】Tree Shaking 友好的导出
// 使用具名导出而非默认导出
export { functionA, functionB };
```

### 3.2 渲染优化

```vue
<template>
  <!-- 【强制】v-if vs v-show -->
  <!-- 频繁切换用 v-show，条件很少改变用 v-if -->
  <div v-show="isVisible">频繁切换的内容</div>
  <div v-if="isLoggedIn">登录后才显示的内容</div>

  <!-- 【强制】列表渲染使用唯一 key -->
  <div v-for="item in list" :key="item.id">{{ item.name }}</div>

  <!-- 【强制】大列表使用虚拟滚动 -->
  <VirtualList :items="largeList" :item-height="50">
    <template #default="{ item }">
      <ListItem :data="item" />
    </template>
  </VirtualList>

  <!-- 【强制】使用 v-memo 缓存 -->
  <div v-for="item in list" :key="item.id" v-memo="[item.id, item.status]">
    <ExpensiveComponent :data="item" />
  </div>
</template>

<script setup lang="ts">
// 【强制】使用 shallowRef 优化大对象
const largeList = shallowRef<Item[]>([]);

// 【强制】使用 computed 缓存计算
const filteredList = computed(() => {
  return list.value.filter(item => item.active);
});

// 【强制】避免不必要的响应式
const staticConfig = markRaw({
  // 不需要响应式的配置
});
</script>
```

### 3.3 网络请求优化

```typescript
// 【强制】请求防抖
const debouncedSearch = useDebounceFn(async (keyword: string) => {
  const result = await searchApi(keyword);
  // ...
}, 300);

// 【强制】请求取消
const controller = new AbortController();

async function fetchData() {
  try {
    const response = await fetch(url, {
      signal: controller.signal,
    });
    // ...
  } catch (error) {
    if (error.name === 'AbortError') {
      console.log('请求已取消');
    }
  }
}

// 组件卸载时取消请求
onUnmounted(() => {
  controller.abort();
});

// 【强制】请求缓存
const cache = new Map<string, { data: any; timestamp: number }>();
const CACHE_TTL = 5 * 60 * 1000;

async function fetchWithCache<T>(url: string): Promise<T> {
  const cached = cache.get(url);
  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    return cached.data;
  }
  
  const data = await fetch(url).then(r => r.json());
  cache.set(url, { data, timestamp: Date.now() });
  return data;
}

// 【强制】请求合并
const batchFetch = useDebounceFn(async (ids: number[]) => {
  const result = await batchApi(ids);
  // 分发结果
}, 50);
```

### 3.4 资源优化

```typescript
// 【强制】图片优化
// 1. 使用 WebP 格式
// 2. 响应式图片
<picture>
  <source srcset="image.webp" type="image/webp" />
  <source srcset="image.jpg" type="image/jpeg" />
  <img src="image.jpg" alt="description" />
</picture>

// 3. 图片压缩（构建时）
// vite.config.ts
import viteImagemin from 'vite-plugin-imagemin';

export default {
  plugins: [
    viteImagemin({
      gifsicle: { optimizationLevel: 7 },
      mozjpeg: { quality: 80 },
      pngquant: { quality: [0.8, 0.9] },
      webp: { quality: 80 },
    }),
  ],
};

// 【强制】代码分割
// vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['vue', 'vue-router', 'pinia'],
          'element-plus': ['element-plus'],
          'echarts': ['echarts'],
        },
      },
    },
  },
};

// 【强制】Gzip 压缩
import viteCompression from 'vite-plugin-compression';

export default {
  plugins: [
    viteCompression({
      algorithm: 'gzip',
      threshold: 10240,
    }),
  ],
};
```

---

## 4. 数据库性能优化

### 4.1 查询优化

```sql
-- 【强制】使用 EXPLAIN 分析
EXPLAIN SELECT * FROM t_reservation WHERE user_id = 1;

-- 关注指标：
-- type: const > eq_ref > ref > range > index > ALL
-- key: 使用的索引
-- rows: 扫描行数（越小越好）
-- Extra: 避免 Using filesort, Using temporary

-- 【强制】避免全表扫描
-- 错误：索引失效
SELECT * FROM t_reservation WHERE DATE(created_time) = '2024-01-01';

-- 正确：使用范围查询
SELECT * FROM t_reservation 
WHERE created_time >= '2024-01-01 00:00:00' 
  AND created_time < '2024-01-02 00:00:00';

-- 【强制】覆盖索引
-- 创建索引
CREATE INDEX idx_user_status ON t_reservation(user_id, status);

-- 查询只访问索引，不回表
SELECT user_id, status FROM t_reservation WHERE user_id = 1;

-- 【强制】深度分页优化
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

### 4.2 索引优化

```sql
-- 【强制】遵循最左前缀原则
-- 联合索引 (a, b, c)
-- 可用：WHERE a=1, WHERE a=1 AND b=2, WHERE a=1 AND b=2 AND c=3
-- 不可用：WHERE b=2, WHERE c=3, WHERE b=2 AND c=3

-- 【强制】区分度高的字段放前面
-- 错误：status 区分度低
CREATE INDEX idx_status_user ON t_reservation(status, user_id);

-- 正确：user_id 区分度高
CREATE INDEX idx_user_status ON t_reservation(user_id, status);

-- 【强制】避免冗余索引
-- 已有 (a, b)，不需要再建 (a)

-- 【强制】定期分析索引使用情况
SELECT 
    table_name,
    index_name,
    stat_value AS pages,
    stat_description
FROM mysql.innodb_index_stats
WHERE database_name = 'lingshu'
  AND stat_name = 'size';
```

### 4.3 写入优化

```sql
-- 【强制】批量插入
INSERT INTO t_reservation (user_id, seat_id, status) VALUES
    (1, 100, 0),
    (2, 101, 0),
    (3, 102, 0);

-- 【强制】分批处理大量数据
-- 每批 500-1000 条
DELIMITER //
CREATE PROCEDURE batch_update()
BEGIN
    DECLARE done INT DEFAULT 0;
    WHILE done = 0 DO
        UPDATE t_reservation 
        SET status = 4 
        WHERE status = 0 AND reservation_date < CURDATE()
        LIMIT 1000;
        
        IF ROW_COUNT() = 0 THEN
            SET done = 1;
        END IF;
    END WHILE;
END //
DELIMITER ;

-- 【强制】避免大事务
-- 事务时间控制在 1 秒内
```

---

## 5. 缓存优化

### 5.1 多级缓存架构

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   L1 Cache  │ --> │   L2 Cache  │ --> │   Database  │
│  (Caffeine) │     │   (Redis)   │     │   (MySQL)   │
│   本地缓存   │     │  分布式缓存  │     │    数据库    │
└─────────────┘     └─────────────┘     └─────────────┘
     10μs               1ms                 10ms
```

```java
@Service
@RequiredArgsConstructor
public class SeatCacheService {
    
    private final Cache<String, SeatStatus> localCache;
    private final StringRedisTemplate redisTemplate;
    private final SeatMapper seatMapper;
    
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
            redisTemplate.opsForValue().set(
                cacheKey, 
                JSON.toJSONString(status), 
                5, TimeUnit.MINUTES
            );
            localCache.put(cacheKey, status);
        }
        
        return status;
    }
}
```

### 5.2 缓存策略

```java
// 【强制】缓存穿透防护
public UserVO getUserById(Long userId) {
    String cacheKey = "user:info:" + userId;
    String cacheValue = redisTemplate.opsForValue().get(cacheKey);
    
    if (cacheValue != null) {
        if ("NULL".equals(cacheValue)) {
            return null;  // 空值缓存
        }
        return JSON.parseObject(cacheValue, UserVO.class);
    }
    
    User user = userMapper.selectById(userId);
    if (user != null) {
        redisTemplate.opsForValue().set(
            cacheKey, 
            JSON.toJSONString(user), 
            1, TimeUnit.HOURS
        );
    } else {
        // 缓存空值，防止穿透
        redisTemplate.opsForValue().set(cacheKey, "NULL", 1, TimeUnit.MINUTES);
    }
    
    return userConverter.toVO(user);
}

// 【强制】缓存击穿防护（热点数据）
public SeatStatus getHotSeatStatus(Long seatId) {
    String cacheKey = "seat:hot:" + seatId;
    String lockKey = "lock:seat:" + seatId;
    
    String cacheValue = redisTemplate.opsForValue().get(cacheKey);
    if (cacheValue != null) {
        return JSON.parseObject(cacheValue, SeatStatus.class);
    }
    
    // 分布式锁，只有一个线程去查数据库
    String requestId = UUID.randomUUID().toString();
    if (lockService.tryLock(lockKey, requestId, 10)) {
        try {
            // 双重检查
            cacheValue = redisTemplate.opsForValue().get(cacheKey);
            if (cacheValue != null) {
                return JSON.parseObject(cacheValue, SeatStatus.class);
            }
            
            SeatStatus status = seatMapper.selectStatus(seatId);
            redisTemplate.opsForValue().set(
                cacheKey, 
                JSON.toJSONString(status), 
                5, TimeUnit.MINUTES
            );
            return status;
        } finally {
            lockService.releaseLock(lockKey, requestId);
        }
    }
    
    // 获取锁失败，等待后重试
    Thread.sleep(100);
    return getHotSeatStatus(seatId);
}

// 【强制】缓存雪崩防护
// 1. 过期时间加随机值
long ttl = 3600 + new Random().nextInt(600);  // 1小时 + 0-10分钟随机
redisTemplate.opsForValue().set(key, value, ttl, TimeUnit.SECONDS);

// 2. 热点数据永不过期，后台异步更新
```

### 5.3 缓存更新策略

```java
// 【推荐】Cache Aside Pattern
// 读：先读缓存，缓存没有读数据库，然后写缓存
// 写：先更新数据库，再删除缓存

@Transactional(rollbackFor = Exception.class)
public void updateUser(UserDTO dto) {
    // 1. 更新数据库
    userMapper.updateById(userConverter.toEntity(dto));
    
    // 2. 删除缓存
    String cacheKey = "user:info:" + dto.getId();
    redisTemplate.delete(cacheKey);
}

// 【推荐】延迟双删（解决缓存不一致）
@Transactional(rollbackFor = Exception.class)
public void updateUserWithDoubleDelete(UserDTO dto) {
    String cacheKey = "user:info:" + dto.getId();
    
    // 1. 先删除缓存
    redisTemplate.delete(cacheKey);
    
    // 2. 更新数据库
    userMapper.updateById(userConverter.toEntity(dto));
    
    // 3. 延迟再删除缓存
    CompletableFuture.runAsync(() -> {
        try {
            Thread.sleep(500);
            redisTemplate.delete(cacheKey);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    });
}
```

---

## 6. 网络优化

### 6.1 HTTP 优化

```nginx
# Nginx 配置

# 开启 Gzip 压缩
gzip on;
gzip_min_length 1k;
gzip_comp_level 6;
gzip_types text/plain application/json application/javascript text/css;

# 开启 HTTP/2
listen 443 ssl http2;

# 静态资源缓存
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# API 响应头
location /api/ {
    add_header Cache-Control "no-cache, no-store, must-revalidate";
}

# 连接优化
keepalive_timeout 65;
keepalive_requests 1000;
```

### 6.2 CDN 优化

```
# 静态资源使用 CDN
# 1. JS/CSS 文件
# 2. 图片资源
# 3. 字体文件

# CDN 配置要点
# 1. 开启 Gzip/Brotli 压缩
# 2. 配置合理的缓存策略
# 3. 开启 HTTP/2
# 4. 配置回源策略
```

---

## 7. 监控与调优

### 7.1 监控指标

```yaml
# 应用指标
- 接口响应时间（P50/P95/P99）
- 接口 QPS
- 错误率
- 线程池使用率

# JVM 指标
- 堆内存使用率
- GC 次数和耗时
- 线程数

# 数据库指标
- 慢查询数量
- 连接池使用率
- QPS/TPS

# 缓存指标
- 命中率
- 内存使用率
- 连接数

# 系统指标
- CPU 使用率
- 内存使用率
- 磁盘 IO
- 网络带宽
```

### 7.2 性能测试

```bash
# 压测工具：JMeter / wrk / ab

# wrk 示例
wrk -t12 -c400 -d30s http://localhost:8080/api/v1/reservations

# 压测场景
# 1. 基准测试：单接口性能
# 2. 负载测试：逐步增加并发
# 3. 压力测试：找到系统瓶颈
# 4. 稳定性测试：长时间运行

# 压测报告关注点
# 1. 响应时间分布
# 2. 吞吐量
# 3. 错误率
# 4. 资源使用率
```

### 7.3 问题排查

```bash
# CPU 高
top -Hp <pid>                    # 找到高 CPU 线程
printf '%x\n' <tid>              # 转换为 16 进制
jstack <pid> | grep <tid_hex>    # 查看线程堆栈

# 内存泄漏
jmap -heap <pid>                 # 查看堆内存
jmap -histo:live <pid>           # 查看对象分布
jmap -dump:format=b,file=heap.hprof <pid>  # 导出堆转储

# GC 问题
jstat -gcutil <pid> 1000         # 监控 GC

# 慢查询
# 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;

# 分析慢查询
mysqldumpslow -s t /var/log/mysql/slow.log
```

---

## 附录

### A. 性能检查清单

```
□ 接口响应时间是否达标
□ 是否有 N+1 查询
□ 是否有全表扫描
□ 是否使用了缓存
□ 是否有大事务
□ 是否有内存泄漏风险
□ 线程池配置是否合理
□ 连接池配置是否合理
□ 是否开启了 Gzip 压缩
□ 静态资源是否使用 CDN
□ 前端是否做了代码分割
□ 图片是否做了优化
```

### B. 常用性能工具

| 工具 | 用途 |
|------|------|
| Arthas | Java 诊断工具 |
| JProfiler | Java 性能分析 |
| VisualVM | JVM 监控 |
| wrk | HTTP 压测 |
| JMeter | 压力测试 |
| Lighthouse | 前端性能分析 |
| Chrome DevTools | 前端调试 |
| Prometheus | 监控系统 |
| Grafana | 监控可视化 |
