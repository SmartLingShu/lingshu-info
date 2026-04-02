# AI 助手联调验收清单

> 适用仓库：`lingshu-ai-assistant`、`lingshu-study-reservoir-mini`、`lingshu-vue3`、`lingshu-study-reservoir-java`

## 1. 验收目标

- 验证用户链路闭环：对话推荐 -> 确认预约 -> 取消预约 -> 学习统计 -> 反馈处置。
- 验证管理链路闭环：会话审计、风控命中、反馈工单化处理、看板指标更新。
- 验证故障链路闭环：AI 解析异常、预约服务异常、风控拦截时的降级行为。

## 2. 环境准备

1. 启动依赖：MySQL、Redis、预约后端。
2. 启动 `lingshu-ai-assistant`，确保 `DASHSCOPE_API_KEY`、`RESERVATION_API_BASE_URL` 已配置。
3. 启动小程序端与管理后台。
4. 准备一个可预约测试账号（会员 ID 固定）。

## 3. 核心验收用例

| 用例编号 | 场景 | 预期结果 |
| --- | --- | --- |
| A1 | 发送“预约明天上午 8-12 点” | 返回候选卡片，包含 `draftId` 与确认动作 |
| A2 | 确认 `draftId` | 创建预约成功，返回预约结果卡片 |
| A3 | 发起取消意图 | 返回可取消预约卡片 |
| A4 | 确认取消 | 预约取消成功，行为信号记录 `BOOKING_CANCELLED` |
| A5 | 查询学习统计 | 返回统计总览、趋势、排行榜、取消分析卡片 |
| A6 | 提交负反馈 | 反馈记录默认处置状态为 `OPEN` |
| A7 | 管理端处置反馈 | `OPEN -> IN_PROGRESS -> RESOLVED` 流转成功，`handledAt` 正确更新 |
| A8 | 看板核对 | 负反馈处置率、超时未处理、平均处置时长指标正确 |

## 4. 风控与失败兜底验收

| 用例编号 | 场景 | 预期结果 |
| --- | --- | --- |
| R1 | 高频重复确认预约 | 幂等生效，不重复创建预约 |
| R2 | 风控规则命中 BLOCK | 返回阻断信息，审计记录包含 `riskFlag` 和规则编码 |
| R3 | 预约后端返回异常 | 显示降级提示，支持重试或手动预约入口 |
| R4 | 参数非法 | 返回统一错误码，审计记录可追溯 |

## 5. 回归命令

在 `lingshu-ai-assistant` 执行：

```bash
mvn -q test
```

建议重点关注：

- `AssistantCoreFlowAcceptanceTest`
- `AssistantFeedbackAdminServiceTest`
- `AssistantDashboardAdminServiceTest`
- `AssistantRiskGuardServiceImplTest`

## 6. 发布前检查

1. 数据库表结构与 `schema.sql` 同步。
2. Redis key 前缀符合环境规范（避免跨环境冲突）。
3. 管理端权限项已配置：`assistant:*`。
4. 小程序基础地址配置可切换测试/生产环境。
5. 看板指标采集窗口和时区配置一致（默认 `Asia/Shanghai`）。
