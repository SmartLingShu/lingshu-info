# Git 工作流与代码审查规范

> 适用于团队协作开发，以代码质量、协作效率为核心原则。

## 目录

- [1. 分支管理规范](#1-分支管理规范)
- [2. 提交规范](#2-提交规范)
- [3. 代码审查规范](#3-代码审查规范)
- [4. 发布流程规范](#4-发布流程规范)
- [5. 冲突处理规范](#5-冲突处理规范)

---

## 1. 分支管理规范

### 1.1 分支类型

| 分支类型 | 命名规则 | 说明 | 生命周期 |
|----------|----------|------|----------|
| main | main | 主分支，生产环境代码 | 永久 |
| develop | develop | 开发分支，最新开发代码 | 永久 |
| feature | feature/xxx | 功能分支 | 开发完成后删除 |
| bugfix | bugfix/xxx | Bug修复分支 | 修复完成后删除 |
| hotfix | hotfix/xxx | 紧急修复分支 | 修复完成后删除 |
| release | release/x.x.x | 发布分支 | 发布完成后删除 |

### 1.2 分支命名

```bash
# 功能分支：feature/模块-功能描述
feature/reservation-create
feature/user-credit-system
feature/seat-recommendation

# Bug修复分支：bugfix/issue编号-描述
bugfix/issue-123-login-error
bugfix/issue-456-reservation-conflict

# 紧急修复分支：hotfix/版本号-描述
hotfix/v1.0.1-security-fix
hotfix/v1.0.2-data-migration

# 发布分支：release/版本号
release/v1.0.0
release/v1.1.0
```

### 1.3 分支工作流

```
main ─────────────────────────────────────────────────────────►
  │                                              ▲
  │                                              │ merge
  ▼                                              │
develop ──────────────────────────────────────────────────────►
  │         ▲           ▲           ▲
  │         │ merge     │ merge     │ merge
  ▼         │           │           │
feature/a ──┘           │           │
                        │           │
feature/b ──────────────┘           │
                                    │
bugfix/c ───────────────────────────┘

# 工作流程：
# 1. 从 develop 创建 feature 分支
# 2. 在 feature 分支开发
# 3. 开发完成后提交 PR 到 develop
# 4. Code Review 通过后合并
# 5. 删除 feature 分支
# 6. 发布时从 develop 创建 release 分支
# 7. release 测试通过后合并到 main 和 develop
```

### 1.4 分支操作命令

```bash
# 创建功能分支
git checkout develop
git pull origin develop
git checkout -b feature/reservation-create

# 开发完成后推送
git push origin feature/reservation-create

# 合并后删除本地分支
git branch -d feature/reservation-create

# 删除远程分支
git push origin --delete feature/reservation-create

# 紧急修复流程
git checkout main
git pull origin main
git checkout -b hotfix/v1.0.1-security-fix
# 修复后
git checkout main
git merge hotfix/v1.0.1-security-fix
git checkout develop
git merge hotfix/v1.0.1-security-fix
```

---

## 2. 提交规范

### 2.1 Commit Message 格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

### 2.2 Type 类型

| Type | 说明 | 示例 |
|------|------|------|
| feat | 新功能 | feat(reservation): 添加预约创建功能 |
| fix | Bug修复 | fix(auth): 修复登录Token过期问题 |
| docs | 文档更新 | docs(readme): 更新部署文档 |
| style | 代码格式 | style(user): 格式化代码 |
| refactor | 重构 | refactor(seat): 重构座位选择逻辑 |
| perf | 性能优化 | perf(query): 优化预约列表查询性能 |
| test | 测试 | test(reservation): 添加预约服务单元测试 |
| chore | 构建/工具 | chore(deps): 升级Spring Boot版本 |
| revert | 回滚 | revert: feat(reservation): 添加预约创建功能 |

### 2.3 Scope 范围

```
# 后端模块
reservation  - 预约模块
seat         - 座位模块
classroom    - 教室模块
user         - 用户模块
auth         - 认证模块
system       - 系统模块
common       - 公共模块

# 前端模块
components   - 组件
views        - 页面
stores       - 状态管理
api          - 接口
utils        - 工具
styles       - 样式
```

### 2.4 提交示例

```bash
# 新功能
feat(reservation): 添加座位预约功能

- 实现预约创建接口
- 添加时间冲突检测
- 集成座位锁定机制

Closes #123

# Bug修复
fix(auth): 修复Token刷新失败问题

Token刷新时未正确更新Redis缓存，导致刷新后的Token无法使用。

修复方案：
1. 刷新Token时同步更新Redis缓存
2. 添加缓存更新失败的重试机制

Fixes #456

# 性能优化
perf(reservation): 优化预约列表查询性能

- 添加复合索引 idx_user_date
- 使用覆盖索引避免回表
- 查询时间从 500ms 降至 50ms

# 重构
refactor(seat): 重构座位状态管理

将座位状态管理从Service层抽离到独立的SeatStatusManager类，
提高代码复用性和可测试性。

BREAKING CHANGE: SeatService.updateStatus() 方法签名变更
```

### 2.5 提交规范检查

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf', 'test', 'chore', 'revert']
    ],
    'scope-enum': [
      2,
      'always',
      ['reservation', 'seat', 'classroom', 'user', 'auth', 'system', 'common',
       'components', 'views', 'stores', 'api', 'utils', 'styles']
    ],
    'subject-max-length': [2, 'always', 72],
    'body-max-line-length': [2, 'always', 100]
  }
};
```

---

## 3. 代码审查规范

### 3.1 PR 提交规范

```markdown
## PR 标题
feat(reservation): 添加座位预约功能

## 变更描述
简要描述本次变更的内容和目的。

## 变更类型
- [x] 新功能 (feat)
- [ ] Bug修复 (fix)
- [ ] 文档更新 (docs)
- [ ] 代码重构 (refactor)
- [ ] 性能优化 (perf)

## 变更范围
- [ ] 后端
- [ ] 前端-Web
- [ ] 前端-小程序
- [ ] 数据库

## 测试情况
- [x] 单元测试通过
- [x] 集成测试通过
- [ ] 手动测试通过

## 相关 Issue
Closes #123

## 截图（如有UI变更）
<!-- 添加截图 -->

## 检查清单
- [x] 代码符合开发规范
- [x] 已添加必要的注释
- [x] 已更新相关文档
- [x] 无敏感信息泄露
```

### 3.2 审查要点

#### 代码质量

```
□ 代码是否符合命名规范
□ 代码是否有充分的注释
□ 代码是否有重复逻辑可抽取
□ 代码是否有未使用的变量/导入
□ 代码是否有硬编码的魔法值
□ 异常处理是否完善
□ 日志记录是否充分
```

#### 性能考量

```
□ 是否有 N+1 查询问题
□ 是否有全表扫描风险
□ 是否需要添加缓存
□ 是否需要异步处理
□ 循环内是否有数据库/网络操作
□ 大数据量是否分批处理
```

#### 安全考量

```
□ 是否有 SQL 注入风险
□ 是否有 XSS 风险
□ 敏感数据是否脱敏
□ 接口是否有权限校验
□ 是否有敏感信息泄露
```

#### 可维护性

```
□ 代码是否易于理解
□ 函数/方法是否过长（建议不超过50行）
□ 类是否过大（建议不超过500行）
□ 是否符合单一职责原则
□ 是否有足够的单元测试
```

### 3.3 审查流程

```
1. 提交 PR
   ↓
2. 自动化检查（CI）
   - 代码风格检查（ESLint/Checkstyle）
   - 单元测试
   - 构建检查
   ↓
3. 指定审查人（至少1人）
   ↓
4. 代码审查
   - 审查人提出修改意见
   - 开发者修改代码
   - 重复直到通过
   ↓
5. 审查通过（Approve）
   ↓
6. 合并到目标分支
   ↓
7. 删除功能分支
```

### 3.4 审查意见模板

```markdown
# 必须修改 (Must Fix)
> 影响功能正确性或有安全风险

**文件**: src/service/ReservationService.java
**行号**: 45-50
**问题**: SQL 拼接存在注入风险
**建议**: 使用参数化查询

# 建议修改 (Suggestion)
> 可以改进但不阻塞合并

**文件**: src/controller/ReservationController.java
**行号**: 30
**问题**: 方法名不够清晰
**建议**: `get` 改为 `getReservationById`

# 疑问 (Question)
> 需要开发者解释

**文件**: src/service/SeatService.java
**行号**: 80
**问题**: 这里为什么要加锁？
```

---

## 4. 发布流程规范

### 4.1 版本号规范

```
# 语义化版本：MAJOR.MINOR.PATCH
# MAJOR: 不兼容的 API 变更
# MINOR: 向后兼容的功能新增
# PATCH: 向后兼容的问题修复

v1.0.0  # 首个正式版本
v1.1.0  # 新增功能
v1.1.1  # Bug修复
v2.0.0  # 重大变更，不兼容旧版本
```

### 4.2 发布流程

```bash
# 1. 创建发布分支
git checkout develop
git pull origin develop
git checkout -b release/v1.1.0

# 2. 更新版本号
# 修改 pom.xml / package.json 中的版本号

# 3. 测试验证
# 在测试环境部署并验证

# 4. 修复发布分支上的问题
git commit -m "fix(release): 修复xxx问题"

# 5. 合并到 main
git checkout main
git merge release/v1.1.0
git tag -a v1.1.0 -m "Release v1.1.0"
git push origin main --tags

# 6. 合并回 develop
git checkout develop
git merge release/v1.1.0
git push origin develop

# 7. 删除发布分支
git branch -d release/v1.1.0
git push origin --delete release/v1.1.0
```

### 4.3 发布检查清单

```
□ 版本号已更新
□ CHANGELOG 已更新
□ 数据库脚本已准备
□ 配置变更已记录
□ 单元测试全部通过
□ 集成测试全部通过
□ 性能测试已完成
□ 安全扫描已完成
□ 文档已更新
□ 回滚方案已准备
```

### 4.4 CHANGELOG 格式

```markdown
# Changelog

## [1.1.0] - 2024-01-15

### Added
- 新增座位预约功能 (#123)
- 新增学习时长统计 (#124)
- 新增信用积分系统 (#125)

### Changed
- 优化预约列表查询性能 (#126)
- 重构座位状态管理 (#127)

### Fixed
- 修复登录Token过期问题 (#128)
- 修复预约时间冲突检测bug (#129)

### Security
- 修复SQL注入漏洞 (#130)

## [1.0.0] - 2024-01-01

### Added
- 初始版本发布
- 基础预约功能
- 用户管理功能
```

---

## 5. 冲突处理规范

### 5.1 预防冲突

```bash
# 【强制】开发前先同步最新代码
git checkout develop
git pull origin develop
git checkout feature/xxx
git rebase develop

# 【强制】频繁提交，小步快跑
# 【强制】及时合并，避免长期分支
# 【推荐】复杂功能拆分为多个小PR
```

### 5.2 解决冲突

```bash
# 1. 拉取最新代码
git fetch origin

# 2. 变基到最新 develop
git rebase origin/develop

# 3. 解决冲突
# 编辑冲突文件，保留正确代码

# 4. 标记冲突已解决
git add .

# 5. 继续变基
git rebase --continue

# 6. 强制推送（仅限个人分支）
git push origin feature/xxx --force-with-lease
```

### 5.3 冲突解决原则

```
1. 【强制】理解双方代码意图后再解决
2. 【强制】解决后必须测试功能正常
3. 【强制】不确定时与相关开发者沟通
4. 【禁止】直接删除他人代码
5. 【禁止】使用 --force 推送公共分支
```

---

## 附录

### A. Git 常用命令

```bash
# 查看分支
git branch -a

# 查看提交历史
git log --oneline -10

# 查看文件变更
git diff

# 暂存变更
git stash
git stash pop

# 撤销未提交的修改
git checkout -- .

# 撤销已提交（未推送）
git reset --soft HEAD~1

# 修改最后一次提交
git commit --amend

# 合并多个提交
git rebase -i HEAD~3
```

### B. Git 配置

```bash
# 用户配置
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# 别名配置
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --all"

# 自动处理换行符
git config --global core.autocrlf input  # Mac/Linux
git config --global core.autocrlf true   # Windows
```
