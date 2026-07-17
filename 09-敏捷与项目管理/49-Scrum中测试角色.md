# 49-Scrum中测试角色

> 课时：45 min | 难度：★★★☆☆

## 学习目标

- 理解 Scrum 框架的三角色、五事件、三工件
- 掌握测试人员在每个 Scrum 仪式中的具体职责
- 能够编写清晰的验收标准（Acceptance Criteria）
- 理解 Definition of Done 在测试维度的含义

## 核心概念

### Scrum 框架概览

Scrum 由三个角色、五个事件、三个工件组成：

| 类别 | 组成 | 说明 |
|------|------|------|
| 角色 | Product Owner、Scrum Master、开发团队 | 测试人员属于开发团队成员 |
| 事件 | Sprint、Sprint Planning、Daily Scrum、Sprint Review、Sprint Retrospective | 时间盒约束 |
| 工件 | Product Backlog、Sprint Backlog、Increment | 透明化工作 |

### 测试人员在 Scrum 中的角色

Scrum 不定义独立的"测试角色"，测试是开发团队集体的责任。但测试人员在以下维度承担主导责任：

- **质量倡导**：在 Sprint 中持续推动质量意识
- **测试策略制定**：定义测试层级、工具选择、环境需求
- **验收标准编写**：与 PO 协作将 User Story 转化为可测试的条件
- **自动化建设**：维护和扩展回归测试套件

### Sprint Planning 中的测试职责

1. **测试工作量估算**：结合历史数据（story point 或理想人天）
2. **容量计算**：从 Sprint 总容量中扣除会议、休假等非开发时间
3. **测试任务拆分**：将测试活动拆分为具体的 Sprint Backlog 项

```
测试工作量估算公式：
测试工作量 = 开发工作量 × 测试系数

测试系数参考值：
- 简单 CRUd 操作：0.3~0.5
- 中等业务逻辑：0.5~0.8
- 复杂算法/多系统集成：0.8~1.2
- 安全/性能敏感功能：1.0~1.5
```

### Daily Standup 中测试人员的报告

每日站会回答三个问题，测试人员的报告模式：

```
昨日完成：完成了订单模块接口测试，发现2个缺陷并已提交
今日计划：开始购物车功能的探索性测试
阻塞项：支付沙箱环境不可用，影响支付流程测试进度
```

### 验收标准编写（Acceptance Criteria）

采用 Given-When-Then 格式（BDD 风格）：

```gherkin
Feature: 用户登录

  Scenario: 使用正确凭证登录
    Given 用户已注册且账号未被锁定
    When 用户输入正确的用户名和密码并点击登录
    Then 系统跳转到首页
    And 显示欢迎消息包含用户名

  Scenario: 密码错误连续输入5次
    Given 用户已注册
    When 用户连续5次输入错误密码
    Then 账号被锁定30分钟
    And 显示"账号已锁定，请30分钟后重试"
```

### Definition of Done (DoD) 测试维度

```markdown
## 测试完成的定义 (Testing DoD)

- [ ] 所有验收标准已通过自动化测试验证
- [ ] 单元测试覆盖率达到团队约定阈值（≥80%）
- [ ] 代码静态分析无严重/阻断级别问题
- [ ] 所有已知缺陷已修复并回归验证
- [ ] 测试用例已更新至测试管理系统
- [ ] 自动化脚本已合并到主分支 CI 流水线
- [ ] 测试报告已生成并归档
```

## 动手实操

### 实操1：为 User Story 编写验收标准

原始 User Story：
> 作为用户，我希望能够重置密码，以便在忘记密码时恢复账号访问。

转化为验收标准：

```gherkin
Feature: 密码重置

  Scenario: 通过邮箱重置密码
    Given 用户已注册邮箱 user@example.com
    When 用户在登录页点击"忘记密码"
    And 输入已注册的邮箱地址
    And 点击"发送重置链接"
    And 查收邮件点击重置链接（链接有效期24小时）
    And 输入新密码"NewPass@123"并确认
    Then 页面显示"密码重置成功"
    And 用户可以使用新密码登录

  Scenario: 重置链接过期
    Given 用户在24小时前请求了密码重置
    When 用户点击已过期的重置链接
    Then 页面显示"链接已失效，请重新申请"
```

### 实操2：计算 Sprint 测试容量

```python
# sprint_capacity.py

def calculate_test_capacity(
    team_members: int,
    sprint_days: int,
    daily_standup_min: int,
    sprint_ceremonies_days: float,
    utilization_rate: float = 0.7
) -> float:
    """
    计算 Sprint 中的测试可用容量
    
    Args:
        team_members: 测试人员数量
        sprint_days: Sprint 总天数（通常10天=2周）
        daily_standup_min: 每日站会分钟数
        sprint_ceremonies_days: 仪式占用天数（计划会/评审会/回顾会）
        utilization_rate: 有效利用率（排除突发事件、沟通成本）
    
    Returns:
        可用测试人天
    """
    total_person_days = team_members * sprint_days
    
    # 扣除仪式时间
    ceremony_days = sprint_ceremonies_days  # 通常 1.5~2 天
    
    # 扣除每日站会（折算为天）
    standup_days = (daily_standup_min * sprint_days) / (8 * 60)
    
    available_days = (total_person_days - ceremony_days - standup_days) * utilization_rate
    
    return round(available_days, 1)


# 示例：2名测试人员，2周Sprint，15分钟站会，2天仪式时间
capacity = calculate_test_capacity(
    team_members=2,
    sprint_days=10,
    daily_standup_min=15,
    sprint_ceremonies_days=2.0
)
print(f"测试可用容量: {capacity} 人天")  # 输出: 10.3 人天
```

## 常见坑

1. **测试人员成为瓶颈**：团队将所有测试活动集中到个别成员身上，而非全员承担质量责任
2. **DoD 模糊**：Definition of Done 缺少测试维度，导致"开发完成"不等于"真正完成"
3. **验收标准缺失**：User Story 缺少明确的验收条件，测试无法判断是否通过
4. **Sprint 末才测试**：未践行持续测试，所有测试活动堆积在 Sprint 末尾
5. **估算偏差**：未考虑测试环境准备、数据构造等隐性时间成本

## 自测清单

- [ ] 能准确描述 Scrum 的三个角色、五个事件、三个工件
- [ ] 能为给定的 User Story 编写 Given-When-Then 格式的验收标准
- [ ] 能计算 Sprint 测试容量
- [ ] 能区分 Definition of Ready 和 Definition of Done
- [ ] 能说明测试人员在五个 Scrum 仪式中的具体贡献

## 延伸阅读

- [Scrum Guide 官方文档](https://scrumguides.org/)
- [Agile Testing: A Practical Guide for Testers and Agile Teams](https://www.amazon.com/Agile-Testing-Practical-Guide-Testers/dp/0321534468)
- [Acceptance Criteria vs. Definition of Done](https://www.agilealliance.org/glossary/definition-of-done/)
