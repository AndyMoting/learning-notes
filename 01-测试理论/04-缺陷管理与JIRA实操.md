# 04-缺陷管理与JIRA实操

> 课时：120 min | 难度：★★★★

## 学习目标

- 掌握缺陷生命周期的完整状态流转，能绘制状态转换图并说明各状态含义
- 掌握Bug报告的6个核心要素，能独立撰写符合规范的缺陷报告
- 能在JIRA中完成项目创建、工作流配置、缺陷创建与跟踪全流程
- 能区分Severity（严重程度）与Priority（优先级），并正确赋值
- 能对比分析优秀与低质Bug报告，识别低质报告的问题并修正

## 核心概念

### 缺陷生命周期（Bug Lifecycle）

**定义**：缺陷从发现到关闭所经历的状态序列，反映缺陷在团队中的流转过程。

**标准状态图**：

```
                    ┌─────────────────────────────────────────────┐
                    │                                             │
                    ▼                                             │
    ┌───────┐   ┌───────┐   ┌───────┐   ┌───────┐   ┌───────┐   │
    │  New  │──▶│ Open  │──▶│ Fixed │──▶│Verified│──▶│Closed │   │
    └───────┘   └───────┘   └───────┘   └───────┘   └───────┘   │
                    │             │             │             │
                    │             │             │             │
                    ▼             │             │             │
              ┌──────────┐       │             │             │
              │ Deferred │       │             │             │
              └──────────┘       │             │             │
                    │             │             │             │
                    ▼             │             │             │
              ┌──────────┐       │             │             │
              │ Rejected │       │             │             │
              └──────────┘       │             │             │
                    │             │             │             │
                    ▼             ▼             ▼             ▼
              ┌──────────────────────────────────────────────────┐
              │              Closed / Resolved                   │
              └──────────────────────────────────────────────────┘
```

**状态说明**：

| 状态 | 含义 | 触发条件 |
|------|------|----------|
| New | 新建 | 测试人员提交缺陷 |
| Open | 已确认 | 开发负责人确认缺陷有效，分配处理人 |
| Fixed | 已修复 | 开发人员修复代码并提交 |
| Verified | 已验证 | 测试人员验证修复通过 |
| Closed | 已关闭 | 缺陷确认关闭，归档 |
| Deferred | 延期 | 当前版本不修复，推迟到后续版本 |
| Rejected | 拒绝 | 非缺陷（如需求误解、重复提交） |
| Reopened | 重开 | 验证未通过，退回给开发 |

**状态转换规则**：

```
New ──(确认有效)──▶ Open
New ──(非缺陷)──▶ Rejected
New ──(重复)──▶ Closed（标记为Duplicate）
Open ──(分配)──▶ In Progress（可选中间状态）
In Progress ──(修复完成)──▶ Fixed
Fixed ──(验证通过)──▶ Verified
Fixed ──(验证失败)──▶ Reopened
Verified ──(确认关闭)──▶ Closed
Open ──(决定延期)──▶ Deferred
Deferred ──(版本排期)──▶ Open
Reopened ──(重新修复)──▶ Fixed
```

### Bug报告六要素

| 要素 | 说明 | 示例 |
|------|------|------|
| **Summary（摘要）** | 一句话描述缺陷现象+位置 | "登录页-输入错误密码3次后未锁定账户" |
| **Steps to Reproduce（复现步骤）** | 有序操作步骤，精确到点击/输入 | 1. 打开登录页 2. 输入用户名test 3. 输入错误密码 4. 点击登录 5. 重复步骤2-4共3次 |
| **Expected Result（预期结果）** | 需求规格中定义的正确行为 | 第3次错误后提示"账户已锁定，请30分钟后重试" |
| **Actual Result（实际结果）** | 实际观察到的错误行为 | 第3次错误后仅提示"密码错误"，可继续尝试 |
| **Severity（严重程度）** | 缺陷对系统功能的影响等级 | Critical / Major / Minor / Trivial |
| **Screenshot/Log（截图/日志）** | 缺陷现象的可视化证据 | 截图+控制台日志+网络请求记录 |

### Severity vs Priority

| 维度 | Severity（严重程度） | Priority（优先级） |
|------|---------------------|-------------------|
| 定义 | 缺陷对系统功能/用户的影响程度 | 修复该缺陷的业务紧迫性 |
| 视角 | 技术视角（测试/开发） | 业务视角（产品/项目经理） |
| 赋值者 | 测试人员 | 产品经理/项目经理 |
| 分类 | Critical / Major / Minor / Trivial | P0 / P1 / P2 / P3 |
| 关注点 | "这个Bug有多严重？" | "这个Bug多快需要修？" |

**Severity 等级定义**：

| 等级 | 说明 | 示例 |
|------|------|------|
| Critical（致命） | 系统崩溃/数据丢失/核心功能不可用 | 支付失败导致订单丢失 |
| Major（严重） | 主要功能异常，无 workaround | 登录功能完全不可用 |
| Minor（一般） | 次要功能异常，有 workaround | 搜索排序结果不正确 |
| Trivial（轻微） | UI/文案问题，不影响功能 | 按钮文案拼写错误 |

**Priority 等级定义**：

| 等级 | 说明 | 响应时间 |
|------|------|----------|
| P0 | 阻塞性，必须立即修复 | 24小时内 |
| P1 | 高优先级，当前版本必须修复 | 当前迭代内 |
| P2 | 中优先级，计划修复 | 下一迭代 |
| P3 | 低优先级，可延后 | 后续版本 |

**Severity ≠ Priority 示例**：

| 场景 | Severity | Priority | 理由 |
|------|----------|----------|------|
| 首页Logo错位 | Trivial | P1 | 影响品牌形象，业务紧急 |
| 后台管理页按钮失效 | Major | P3 | 功能严重但仅内部使用，用户无影响 |
| 支付金额计算错误 | Critical | P0 | 资金损失，必须立即修复 |
| 404页面文案错误 | Trivial | P3 | 影响小且非核心功能 |

### JIRA工作流配置

**JIRA核心概念**：

| 概念 | 说明 |
|------|------|
| Project | 项目，包含Issue、工作流、权限等配置 |
| Issue | 工作项，包括Bug、Task、Story等类型 |
| Workflow | 工作流，定义Issue的状态与转换规则 |
| Screen | 屏幕，定义创建/编辑Issue时显示的字段 |
| Field | 字段，如Summary、Severity、Priority等 |

**标准JIRA工作流（Bug）**：

```
┌──────┐     ┌──────┐     ┌──────────┐     ┌───────┐     ┌──────────┐     ┌───────┐
│ To   │────▶│ In   │────▶│ In       │────▶│ Done  │────▶│ Verified │────▶│Closed │
│ Do   │     │ Prog │     │ Review   │     │       │     │          │     │       │
└──────┘     └──────┘     └──────────┘     └───────┘     └──────────┘     └───────┘
   │             │             │                │              │               │
   │             │             │                │              │               │
   ▼             ▼             ▼                ▼              ▼               ▼
(Start)     (Assign)      (Fix Done)      (Verify)       (Pass)          (Archive)
```

## 动手实操

### 任务1：在JIRA中创建测试项目并配置工作流

1. 创建项目

```
操作路径：JIRA → Projects → Create project
选择模板：Software Development → Scrum
项目名称：QA-Test-Project
项目键：QAT
```

2. 配置工作流

```
操作路径：Project settings → Workflows → Add workflow
工作流名称：Bug Workflow

添加状态：
  - To Do（待处理）
  - In Progress（处理中）
  - In Review（待审核）
  - Done（已完成）
  - Verified（已验证）
  - Closed（已关闭）

添加转换（Transitions）：
  - To Do → In Progress（名称：Start Progress）
  - In Progress → In Review（名称：Submit for Review）
  - In Review → Done（名称：Approve）
  - In Review → In Progress（名称：Reject）
  - Done → Verified（名称：Verify）
  - Verified → Closed（名称：Close）
  - Verified → In Progress（名称：Reopen）
```

3. 配置字段

```
操作路径：Project settings → Issue types → Bug → Fields

必填字段：
  - Summary（摘要）
  - Description（描述，含复现步骤）
  - Severity（严重程度，自定义字段）
  - Priority（优先级）
  - Assignee（指派人）
  - Sprint（迭代）
  - Attachment（附件，截图/日志）
```

4. 将工作流关联到项目

```
操作路径：Project settings → Workflows → Add workflow scheme
将 Bug Workflow 关联到 Bug Issue Type
Publish 发布
```

### 任务2：创建并跟踪一个Bug

1. 创建Bug

```
操作路径：JIRA → Create issue
Issue Type：Bug
Summary：[登录] 输入错误密码3次后账户未锁定
Severity：Major
Priority：P1
Environment：Chrome 120 / Windows 11 / 测试环境 v2.3.1
Description：
  **复现步骤：**
  1. 访问 https://test.example.com/login
  2. 输入用户名：testuser
  3. 输入错误密码：wrongpass
  4. 点击"登录"按钮
  5. 重复步骤2-4共3次

  **预期结果：**
  第3次错误后，页面提示"账户已锁定，请30分钟后重试"，账户状态变为locked。

  **实际结果：**
  第3次错误后，页面仅提示"用户名或密码错误"，账户状态仍为active，可继续尝试登录。

  **附件：**
  - screenshot_login_3times.png
  - console_log.txt
  - network_request.har

Assignee：dev-zhang
Sprint：Sprint 24
```

2. 跟踪Bug状态

```
操作路径：JIRA → Boards → Active sprints

状态流转：
  1. 开发将状态从 To Do → In Progress
  2. 开发修复后 In Progress → In Review
  3. 代码审核通过后 In Review → Done
  4. 测试验证：
     - 通过：Done → Verified → Closed
     - 失败：Verified → In Progress（Reopen）
```

3. 使用JQL查询缺陷

```sql
-- 查询当前迭代所有未关闭的Major及以上缺陷
project = QAT 
  AND issuetype = Bug 
  AND severity in (Critical, Major) 
  AND status not in (Closed, Rejected)
ORDER BY priority DESC, created DESC

-- 查询分配给当前用户且超过3天未更新的缺陷
project = QAT 
  AND assignee = currentUser() 
  AND status != Closed 
  AND updated < -3d
ORDER BY updated ASC

-- 查询本版本新增的缺陷趋势
project = QAT 
  AND issuetype = Bug 
  AND created >= 2026-07-01 
  AND created <= 2026-07-17
```

### 任务3：编写Python脚本生成Bug报告模板

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class BugReport:
    summary: str
    steps: list[str]
    expected: str
    actual: str
    severity: str  # Critical / Major / Minor / Trivial
    priority: str  # P0 / P1 / P2 / P3
    environment: str
    reporter: str
    attachments: list[str]
    created_at: datetime = None

    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now()

    def to_markdown(self) -> str:
        steps_str = "\n".join(f"{i+1}. {step}" for i, step in enumerate(self.steps))
        attachments_str = "\n".join(f"- {a}" for a in self.attachments)
        return f"""## Bug Report

**Summary:** {self.summary}
**Severity:** {self.severity} | **Priority:** {self.priority}
**Environment:** {self.environment}
**Reporter:** {self.reporter}
**Created:** {self.created_at.strftime('%Y-%m-%d %H:%M')}

### Steps to Reproduce
{steps_str}

### Expected Result
{self.expected}

### Actual Result
{self.actual}

### Attachments
{attachments_str}
"""

    def validate(self) -> list[str]:
        """验证Bug报告完整性"""
        errors = []
        if not self.summary or len(self.summary) < 10:
            errors.append("Summary过短，需至少10字符")
        if len(self.steps) < 2:
            errors.append("复现步骤至少2步")
        if not self.expected:
            errors.append("Expected Result不能为空")
        if not self.actual:
            errors.append("Actual Result不能为空")
        if self.severity not in ("Critical", "Major", "Minor", "Trivial"):
            errors.append(f"无效Severity: {self.severity}")
        if self.priority not in ("P0", "P1", "P2", "P3"):
            errors.append(f"无效Priority: {self.priority}")
        return errors

# 使用示例
bug = BugReport(
    summary="[登录] 输入错误密码3次后账户未锁定",
    steps=[
        "访问 https://test.example.com/login",
        "输入用户名：testuser",
        "输入错误密码：wrongpass",
        "点击登录按钮",
        "重复步骤2-4共3次",
    ],
    expected="第3次错误后提示账户已锁定，账户状态变为locked",
    actual="第3次错误后仅提示密码错误，账户状态仍为active",
    severity="Major",
    priority="P1",
    environment="Chrome 120 / Windows 11 / v2.3.1",
    reporter="qa-li",
    attachments=["screenshot.png", "console.log"],
)

# 验证
errors = bug.validate()
if errors:
    print("验证失败：")
    for e in errors:
        print(f"  - {e}")
else:
    print("验证通过，生成报告：")
    print(bug.to_markdown())
```

运行脚本：

```bash
python bug_report_template.py
```

## 常见坑

1. **Bug Summary过于笼统** — "登录有问题"无法传达具体现象，应写"登录页-输入错误密码3次后未锁定账户"。
2. **复现步骤缺少关键输入值** — 仅写"输入密码"而不写具体值，导致开发无法复现。
3. **Expected与Actual混淆** — Expected是需求定义的正确行为，Actual是实际观察到的错误行为，不可颠倒。
4. **Severity与Priority赋值随意** — 所有Bug都标Critical/P1会导致真正紧急的缺陷被淹没，需严格按标准赋值。
5. **Bug报告缺少环境信息** — 不写浏览器/OS/版本号，导致开发在正确环境下无法复现。
6. **验证失败后直接关闭** — 验证失败应Reopen而非关闭，否则缺陷会永久丢失。
7. **JIRA工作流未配置权限** — 未限制状态转换权限会导致任何人可随意修改状态，应配置Transition Conditions。

## 自测清单

- [ ] 能绘制完整的缺陷生命周期状态图（含所有状态与转换）
- [ ] 能独立撰写包含6要素的完整Bug报告
- [ ] 能在JIRA中完成项目创建、工作流配置、Bug创建与状态流转
- [ ] 能正确区分Severity与Priority并合理赋值
- [ ] 能识别低质Bug报告的问题并修正
- [ ] 能使用JQL编写常用缺陷查询语句
- [ ] 能用Python脚本生成标准化的Bug报告文档

## 延伸阅读

- [JIRA官方文档 - Bug Tracking](https://www.atlassian.com/software/jira/guides/use-cases/bug-tracking) — JIRA缺陷管理官方指南
- [JIRA Workflow Documentation](https://confluence.atlassian.com/jiracorecloud/working-with-workflows-765593140.html) — JIRA工作流配置详解
- [菜鸟教程 - 软件测试缺陷管理](https://www.runoob.com/software-testing/bug-management.html) — 缺陷管理基础概念
- [ISTQB - Incident Management](https://www.istqb.org/certifications/certified-tester-foundation-level) — ISTQB缺陷管理标准
- [Atlassian JQL Reference](https://confluence.atlassian.com/jiracorecloud/advanced-searching-765593140.html) — JQL完整语法参考
- [Writing a Good Bug Report](https://www.softwaretestinghelp.com/how-to-write-good-bug-report/) — 高质量Bug报告写作指南
