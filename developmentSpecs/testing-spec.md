# 测试规范

> 以代码质量、回归保障、持续集成为核心，覆盖单元测试、集成测试、端到端测试。

## 目录

- [1. 测试原则](#1-测试原则)
- [2. 单元测试规范](#2-单元测试规范)
- [3. 集成测试规范](#3-集成测试规范)
- [4. 前端测试规范](#4-前端测试规范)
- [5. 测试覆盖率](#5-测试覆盖率)
- [6. 持续集成](#6-持续集成)

---

## 1. 测试原则

### 1.1 测试金字塔

```
        /\
       /  \        E2E 测试 (10%)
      /----\       - 关键业务流程
     /      \      - 用户场景验证
    /--------\     
   /          \    集成测试 (20%)
  /------------\   - API 接口测试
 /              \  - 数据库交互测试
/----------------\ 
                   单元测试 (70%)
                   - 业务逻辑测试
                   - 工具类测试
```

### 1.2 测试原则

| 原则 | 说明 |
|------|------|
| FIRST | Fast(快速)、Independent(独立)、Repeatable(可重复)、Self-validating(自验证)、Timely(及时) |
| AAA | Arrange(准备)、Act(执行)、Assert(断言) |
| 单一职责 | 一个测试方法只测试一个场景 |
| 边界测试 | 测试边界条件和异常情况 |
| 可读性 | 测试代码也是文档 |

### 1.3 命名规范

```java
// 测试类命名：被测类名 + Test
ReservationServiceTest
UserControllerTest

// 测试方法命名：should_预期结果_when_条件
@Test
void should_create_reservation_when_seat_available() { }

@Test
void should_throw_exception_when_seat_occupied() { }

@Test
void should_return_empty_list_when_no_data() { }
```

---

## 2. 单元测试规范

### 2.1 Service 层测试

```java
@ExtendWith(MockitoExtension.class)
class ReservationServiceTest {

    @Mock
    private ReservationMapper reservationMapper;
    
    @Mock
    private SeatService seatService;
    
    @Mock
    private ReservationConverter reservationConverter;
    
    @InjectMocks
    private ReservationServiceImpl reservationService;

    @Test
    @DisplayName("创建预约 - 座位可用时应成功")
    void should_create_reservation_when_seat_available() {
        // Arrange
        ReservationDTO dto = new ReservationDTO();
        dto.setSeatId(1L);
        dto.setUserId(100L);
        dto.setReservationDate(LocalDate.now().plusDays(1));
        dto.setStartTime(LocalTime.of(8, 0));
        dto.setEndTime(LocalTime.of(10, 0));
        
        Reservation entity = new Reservation();
        entity.setId(1L);
        
        when(seatService.isSeatAvailable(anyLong(), any(), any())).thenReturn(true);
        when(reservationConverter.toEntity(dto)).thenReturn(entity);
        when(reservationMapper.insert(any())).thenReturn(1);
        
        // Act
        Long reservationId = reservationService.createReservation(dto);
        
        // Assert
        assertThat(reservationId).isEqualTo(1L);
        verify(seatService).lockSeat(eq(1L), any(), any());
        verify(reservationMapper).insert(any());
    }

    @Test
    @DisplayName("创建预约 - 座位不可用时应抛出异常")
    void should_throw_exception_when_seat_not_available() {
        // Arrange
        ReservationDTO dto = new ReservationDTO();
        dto.setSeatId(1L);
        dto.setReservationDate(LocalDate.now().plusDays(1));
        dto.setStartTime(LocalTime.of(8, 0));
        dto.setEndTime(LocalTime.of(10, 0));
        
        when(seatService.isSeatAvailable(anyLong(), any(), any())).thenReturn(false);
        
        // Act & Assert
        assertThatThrownBy(() -> reservationService.createReservation(dto))
            .isInstanceOf(BusinessException.class)
            .hasFieldOrPropertyWithValue("code", "02001");
        
        verify(reservationMapper, never()).insert(any());
    }

    @Test
    @DisplayName("创建预约 - 时间范围无效时应抛出异常")
    void should_throw_exception_when_time_range_invalid() {
        // Arrange
        ReservationDTO dto = new ReservationDTO();
        dto.setSeatId(1L);
        dto.setReservationDate(LocalDate.now().plusDays(1));
        dto.setStartTime(LocalTime.of(10, 0));
        dto.setEndTime(LocalTime.of(8, 0));  // 结束时间早于开始时间
        
        // Act & Assert
        assertThatThrownBy(() -> reservationService.createReservation(dto))
            .isInstanceOf(BusinessException.class)
            .hasFieldOrPropertyWithValue("code", "02003");
    }

    @ParameterizedTest
    @DisplayName("取消预约 - 不同状态的处理")
    @CsvSource({
        "0, true",   // 待签到 - 可取消
        "1, false",  // 使用中 - 不可取消
        "2, false",  // 已完成 - 不可取消
        "3, false",  // 已取消 - 不可取消
    })
    void should_handle_cancel_based_on_status(int status, boolean canCancel) {
        // Arrange
        Reservation reservation = new Reservation();
        reservation.setId(1L);
        reservation.setStatus(status);
        
        when(reservationMapper.selectById(1L)).thenReturn(reservation);
        
        // Act & Assert
        if (canCancel) {
            assertThatCode(() -> reservationService.cancelReservation(1L))
                .doesNotThrowAnyException();
        } else {
            assertThatThrownBy(() -> reservationService.cancelReservation(1L))
                .isInstanceOf(BusinessException.class);
        }
    }
}
```

### 2.2 工具类测试

```java
class DesensitizeUtilTest {

    @Test
    @DisplayName("手机号脱敏 - 正常手机号")
    void should_desensitize_phone_correctly() {
        assertThat(DesensitizeUtil.phone("13800138000")).isEqualTo("138****8000");
    }

    @Test
    @DisplayName("手机号脱敏 - 空值")
    void should_return_null_when_phone_is_null() {
        assertThat(DesensitizeUtil.phone(null)).isNull();
    }

    @Test
    @DisplayName("手机号脱敏 - 长度不足")
    void should_return_original_when_phone_length_invalid() {
        assertThat(DesensitizeUtil.phone("1380013")).isEqualTo("1380013");
    }

    @ParameterizedTest
    @DisplayName("邮箱脱敏 - 多种格式")
    @CsvSource({
        "zhangsan@example.com, z***@example.com",
        "a@test.com, a***@test.com",
        "ab@test.com, a***@test.com",
    })
    void should_desensitize_email_correctly(String input, String expected) {
        assertThat(DesensitizeUtil.email(input)).isEqualTo(expected);
    }
}
```

### 2.3 Mock 使用规范

```java
// 【强制】使用 @Mock 注解而非手动创建
@Mock
private UserMapper userMapper;

// 【强制】使用 @InjectMocks 自动注入
@InjectMocks
private UserServiceImpl userService;

// 【强制】精确匹配参数
when(userMapper.selectById(eq(1L))).thenReturn(user);

// 【强制】验证方法调用
verify(userMapper).insert(any(User.class));
verify(userMapper, times(1)).insert(any());
verify(userMapper, never()).delete(any());

// 【强制】验证调用顺序
InOrder inOrder = inOrder(seatService, reservationMapper);
inOrder.verify(seatService).lockSeat(anyLong(), any(), any());
inOrder.verify(reservationMapper).insert(any());

// 【强制】捕获参数
ArgumentCaptor<Reservation> captor = ArgumentCaptor.forClass(Reservation.class);
verify(reservationMapper).insert(captor.capture());
assertThat(captor.getValue().getStatus()).isEqualTo(0);
```

---

## 3. 集成测试规范

### 3.1 Controller 层测试

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
class ReservationControllerTest {

    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @MockBean
    private ReservationService reservationService;

    @Test
    @DisplayName("创建预约 - 成功")
    @WithMockUser(username = "testuser", roles = {"USER"})
    void should_create_reservation_successfully() throws Exception {
        // Arrange
        ReservationDTO dto = new ReservationDTO();
        dto.setSeatId(1L);
        dto.setReservationDate(LocalDate.now().plusDays(1));
        dto.setStartTime(LocalTime.of(8, 0));
        dto.setEndTime(LocalTime.of(10, 0));
        
        when(reservationService.createReservation(any())).thenReturn(123L);
        
        // Act & Assert
        mockMvc.perform(post("/api/v1/reservations")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(dto)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value("00000"))
            .andExpect(jsonPath("$.data").value(123));
    }

    @Test
    @DisplayName("创建预约 - 参数校验失败")
    @WithMockUser(username = "testuser", roles = {"USER"})
    void should_return_error_when_params_invalid() throws Exception {
        // Arrange
        ReservationDTO dto = new ReservationDTO();
        // seatId 为空
        
        // Act & Assert
        mockMvc.perform(post("/api/v1/reservations")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(dto)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value("00002"));
    }

    @Test
    @DisplayName("创建预约 - 未认证")
    void should_return_401_when_not_authenticated() throws Exception {
        mockMvc.perform(post("/api/v1/reservations")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @DisplayName("分页查询预约")
    @WithMockUser(username = "testuser", roles = {"USER"})
    void should_return_page_result() throws Exception {
        // Arrange
        PageResult<ReservationVO> pageResult = PageResult.of(100L, List.of(
            createReservationVO(1L),
            createReservationVO(2L)
        ));
        
        when(reservationService.pageReservations(any())).thenReturn(pageResult);
        
        // Act & Assert
        mockMvc.perform(get("/api/v1/reservations")
                .param("pageNum", "1")
                .param("pageSize", "10"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.total").value(100))
            .andExpect(jsonPath("$.data.records").isArray())
            .andExpect(jsonPath("$.data.records.length()").value(2));
    }
    
    private ReservationVO createReservationVO(Long id) {
        ReservationVO vo = new ReservationVO();
        vo.setId(id);
        vo.setSeatName("A-01");
        vo.setStatus(0);
        return vo;
    }
}
```

### 3.2 数据库集成测试

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class ReservationMapperTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("test")
        .withUsername("test")
        .withPassword("test");

    @Autowired
    private ReservationMapper reservationMapper;

    @Test
    @DisplayName("插入预约记录")
    void should_insert_reservation() {
        // Arrange
        Reservation reservation = new Reservation();
        reservation.setUserId(1L);
        reservation.setSeatId(100L);
        reservation.setReservationDate(LocalDate.now());
        reservation.setStartTime(LocalTime.of(8, 0));
        reservation.setEndTime(LocalTime.of(10, 0));
        reservation.setStatus(0);
        
        // Act
        int result = reservationMapper.insert(reservation);
        
        // Assert
        assertThat(result).isEqualTo(1);
        assertThat(reservation.getId()).isNotNull();
    }

    @Test
    @DisplayName("查询用户预约列表")
    @Sql("/sql/reservation-test-data.sql")
    void should_select_by_user_id() {
        // Act
        List<Reservation> reservations = reservationMapper.selectByUserId(1L);
        
        // Assert
        assertThat(reservations).hasSize(3);
        assertThat(reservations).allMatch(r -> r.getUserId().equals(1L));
    }

    @Test
    @DisplayName("检查时间冲突")
    @Sql("/sql/reservation-test-data.sql")
    void should_check_time_conflict() {
        // Act
        boolean hasConflict = reservationMapper.existsConflict(
            100L,
            LocalDate.of(2024, 1, 15),
            LocalTime.of(9, 0),
            LocalTime.of(11, 0)
        );
        
        // Assert
        assertThat(hasConflict).isTrue();
    }
}
```

### 3.3 Redis 集成测试

```java
@SpringBootTest
@Testcontainers
class CacheServiceTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", redis::getFirstMappedPort);
    }

    @Autowired
    private UserCacheService userCacheService;
    
    @Autowired
    private StringRedisTemplate redisTemplate;

    @BeforeEach
    void setUp() {
        redisTemplate.getConnectionFactory().getConnection().flushAll();
    }

    @Test
    @DisplayName("缓存用户信息")
    void should_cache_user_info() {
        // Arrange
        Long userId = 1L;
        
        // Act
        UserVO user = userCacheService.getUserById(userId);
        
        // Assert
        assertThat(user).isNotNull();
        String cacheKey = "user:info:" + userId;
        assertThat(redisTemplate.hasKey(cacheKey)).isTrue();
    }

    @Test
    @DisplayName("缓存命中")
    void should_hit_cache() {
        // Arrange
        Long userId = 1L;
        userCacheService.getUserById(userId);  // 第一次查询，写入缓存
        
        // Act
        UserVO user = userCacheService.getUserById(userId);  // 第二次查询，命中缓存
        
        // Assert
        assertThat(user).isNotNull();
        // 验证数据库只查询了一次（通过 Mock 或日志验证）
    }
}
```

---

## 4. 前端测试规范

### 4.1 组件测试

```typescript
// ReservationCard.spec.ts
import { describe, it, expect, vi } from 'vitest';
import { mount } from '@vue/test-utils';
import ReservationCard from '@/components/ReservationCard.vue';

describe('ReservationCard', () => {
  const defaultProps = {
    reservation: {
      id: 1,
      seatName: 'A-01',
      classroomName: '图书馆3楼',
      startTime: '08:00',
      endTime: '10:00',
      status: 0,
      statusName: '待签到',
    },
  };

  it('should render reservation info correctly', () => {
    const wrapper = mount(ReservationCard, { props: defaultProps });
    
    expect(wrapper.text()).toContain('A-01');
    expect(wrapper.text()).toContain('图书馆3楼');
    expect(wrapper.text()).toContain('08:00');
    expect(wrapper.text()).toContain('10:00');
  });

  it('should show cancel button when status is pending', () => {
    const wrapper = mount(ReservationCard, { props: defaultProps });
    
    expect(wrapper.find('[data-test="cancel-btn"]').exists()).toBe(true);
  });

  it('should hide cancel button when status is completed', () => {
    const props = {
      reservation: { ...defaultProps.reservation, status: 2 },
    };
    const wrapper = mount(ReservationCard, { props });
    
    expect(wrapper.find('[data-test="cancel-btn"]').exists()).toBe(false);
  });

  it('should emit cancel event when cancel button clicked', async () => {
    const wrapper = mount(ReservationCard, { props: defaultProps });
    
    await wrapper.find('[data-test="cancel-btn"]').trigger('click');
    
    expect(wrapper.emitted('cancel')).toBeTruthy();
    expect(wrapper.emitted('cancel')![0]).toEqual([1]);
  });
});
```

### 4.2 Composable 测试

```typescript
// usePagination.spec.ts
import { describe, it, expect } from 'vitest';
import { usePagination } from '@/composables/usePagination';

describe('usePagination', () => {
  it('should initialize with default values', () => {
    const { pagination } = usePagination();
    
    expect(pagination.pageNum).toBe(1);
    expect(pagination.pageSize).toBe(10);
    expect(pagination.total).toBe(0);
  });

  it('should initialize with custom page size', () => {
    const { pagination } = usePagination({ defaultPageSize: 20 });
    
    expect(pagination.pageSize).toBe(20);
  });

  it('should update total correctly', () => {
    const { pagination, setTotal } = usePagination();
    
    setTotal(100);
    
    expect(pagination.total).toBe(100);
  });

  it('should reset page number', () => {
    const { pagination, reset } = usePagination();
    pagination.pageNum = 5;
    
    reset();
    
    expect(pagination.pageNum).toBe(1);
  });

  it('should handle page size change', () => {
    const { pagination, handleSizeChange } = usePagination();
    pagination.pageNum = 3;
    
    handleSizeChange(20);
    
    expect(pagination.pageSize).toBe(20);
    expect(pagination.pageNum).toBe(1);  // 重置到第一页
  });
});
```

### 4.3 API Mock 测试

```typescript
// reservationApi.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { createReservation, pageReservations } from '@/api/modules/reservation';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.post('/api/v1/reservations', () => {
    return HttpResponse.json({
      code: '00000',
      message: '创建成功',
      data: 123,
    });
  }),
  http.get('/api/v1/reservations', () => {
    return HttpResponse.json({
      code: '00000',
      message: '操作成功',
      data: {
        total: 100,
        records: [
          { id: 1, seatName: 'A-01' },
          { id: 2, seatName: 'A-02' },
        ],
      },
    });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('Reservation API', () => {
  it('should create reservation', async () => {
    const result = await createReservation({
      seatId: 1,
      reservationDate: '2024-01-15',
      startTime: '08:00:00',
      endTime: '10:00:00',
    });
    
    expect(result.code).toBe('00000');
    expect(result.data).toBe(123);
  });

  it('should fetch reservations', async () => {
    const result = await pageReservations({ pageNum: 1, pageSize: 10 });
    
    expect(result.code).toBe('00000');
    expect(result.data.total).toBe(100);
    expect(result.data.records).toHaveLength(2);
  });
});
```

---

## 5. 测试覆盖率

### 5.1 覆盖率要求

| 模块 | 行覆盖率 | 分支覆盖率 |
|------|----------|------------|
| Service | ≥ 80% | ≥ 70% |
| Controller | ≥ 70% | ≥ 60% |
| Utils | ≥ 90% | ≥ 80% |
| 前端组件 | ≥ 70% | ≥ 60% |

### 5.2 JaCoCo 配置

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.10</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 5.3 Vitest 覆盖率配置

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: ['node_modules/', 'src/types/', '**/*.d.ts'],
      thresholds: {
        lines: 70,
        branches: 60,
        functions: 70,
        statements: 70,
      },
    },
  },
});
```

---

## 6. 持续集成

### 6.1 GitHub Actions 配置

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  backend-test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: test
        ports:
          - 3306:3306
      redis:
        image: redis:7
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Cache Maven packages
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
      
      - name: Run tests
        run: mvn test -B
      
      - name: Upload coverage report
        uses: codecov/codecov-action@v3
        with:
          files: target/site/jacoco/jacoco.xml

  frontend-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm run test:coverage
      
      - name: Upload coverage report
        uses: codecov/codecov-action@v3
        with:
          files: coverage/lcov.info
```

### 6.2 测试报告

```yaml
# 测试报告生成
- name: Generate test report
  run: |
    mvn surefire-report:report
    
- name: Upload test report
  uses: actions/upload-artifact@v3
  with:
    name: test-report
    path: target/site/surefire-report.html
```

---

## 附录

### A. 测试检查清单

```
□ 是否覆盖正常流程
□ 是否覆盖异常流程
□ 是否覆盖边界条件
□ 是否有参数化测试
□ Mock 是否正确
□ 断言是否充分
□ 测试是否独立
□ 测试是否可重复
□ 覆盖率是否达标
```

### B. 常用测试工具

| 工具 | 用途 |
|------|------|
| JUnit 5 | Java 单元测试 |
| Mockito | Mock 框架 |
| AssertJ | 断言库 |
| Testcontainers | 容器化测试 |
| Vitest | Vue 测试 |
| Vue Test Utils | 组件测试 |
| MSW | API Mock |
| Playwright | E2E 测试 |
