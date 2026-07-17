# 52-JIRA项目管理进阶

> 课时：55 min | 难度：★★★★☆

## 学习目标

- 掌握 JIRA 项目配置（Scrum/Kanban）
- 理解 Issue 类型层级关系
- 掌握 JQL 常用查询语句
- 能够定制工作流和自动化规则

## 核心概念

### JIRA Issue 类型层级

```
Epic（史诗）
  └── Story（用户故事）
        ├── Task（任务）
        ├── Bug（缺陷）
        └── Sub-task（子任务）
```

| Issue 类型 | 用途 | 示例 |
|-----------|------|------|
| Epic | 大型功能模块，跨多 Sprint | "用户中心重构" |
| Story | 用户可感知的功能单元 | "用户可以通过邮箱重置密码" |
| Task | 技术性工作项（用户不可见） | "升级数据库驱动版本" |
| Bug | 缺陷/问题 | "登录页面在 IE11 下布局错乱" |
| Sub-task | Story 或 Task 的拆分 | "编写重置密码接口" |

### 工作流（Workflow）

标准 JIRA 工作流状态与转换：

```
                    ┌─────────┐
                    │  To Do  │
                    └────┬────┘
                         │ Start Progress
                    ┌────▼────┐
         ┌─────────│ In Prog │─────────┐
         │         └────┬────┘         │
    Blocked            │              Done
         │    ┌───────▼───────┐       │
         └───►│  In Review    │───────┘
              └───────────────┘
```

工作流配置要点：
- **状态（Status）**：代表工作项所处阶段
- **转换（Transition）**：状态间的移动路径
- **条件（Condition）**：谁可以执行转换（如仅经办人）
- **验证器（Validator）**：转换前检查（如必填字段）
- **后处理（Post Function）**：转换后自动执行的动作

### JQL（JIRA Query Language）

JQL 是 JIRA 的高级查询语言，语法类似 SQL WHERE 子句。

**查询结构**：
```
<字段> <运算符> <值> [AND/OR <字段> <运算符> <值> ...] ORDER BY <字段> [ASC|DESC]
```

**常用 JQL 查询**：

```sql
-- 1. 当前 Sprint 中所有未完成的测试任务
project = "MYPROJECT" 
  AND issuetype = Task 
  AND labels = test 
  AND status != Done 
  AND sprint in openSprints()

-- 2. 分配给当前用户的高优先级 Bug
assignee = currentUser() 
  AND issuetype = Bug 
  AND priority in (High, Critical) 
  AND status not in (Done, Closed)

-- 3. 本周创建的所有问题
created >= startOfWeek() AND created < endOfWeek()

-- 4. 超过 7 天未更新的 In Progress 问题
status = "In Progress" 
  AND updated <= -7d 
  AND assignee is not EMPTY

-- 5. 当前 Sprint 中估算超过 5 点的 Story
sprint in openSprints() 
  AND issuetype = Story 
  AND "Story Points" > 5

-- 6. 某 Epic 下所有未完成的子任务
"Epic Link" = MYPROJECT-100 
  AND status != Done

-- 7. 没有验收标准的 Story
issuetype = Story 
  AND "Acceptance Criteria" is EMPTY

-- 8. 最近 30 天由我解决的缺陷
issuetype = Bug 
  AND resolver = currentUser() 
  AND resolved >= -30d

-- 9. 测试环境发现的缺陷
issuetype = Bug 
  AND environment ~ "test" 
  AND status != Closed

-- 10. 本季度创建的 Epic
issuetype = Epic 
  AND created >= startOfQuarter()

-- 11. 无经办人的未解决问题
project = "MYPROJECT" 
  AND assignee is EMPTY 
  AND status not in (Done, Closed)

-- 12. Sprint 中已逾期的工作项
sprint in openSprints() 
  AND duedate < now() 
  AND status != Done

-- 13. Story 超过 8 点的（需拆分提醒）
issuetype = Story 
  AND "Story Points" > 8 
  AND sprint in openSprints()

-- 14. 重复打开的缺陷（resolved 后再次打开）
issuetype = Bug 
  AND status CHANGED FROM "Done" TO "In Progress" 
  AFTER -30d

-- 15. Sprint 剩余工作项（未开始）
sprint in openSprints() 
  AND status = "To Do"

-- 16. 自动化测试覆盖的需求
issuetype in (Story, Epic) 
  AND labels in ("automated", "e2e-covered")

-- 17. 某目标的测试报告链接为空的问题
issuetype = Story 
  AND "Test Report Link" is EMPTY 
  AND status = Done

-- 18. 在多个 Sprint 中出现的工作项
issue in wasInSprints() 
  AND sprint in openSprints()

-- 19. Sprint 超负荷成员（使用 JQL + 仪表板）
sprint in openSprints() 
  AND assignee = specific_user

-- 20. 缺陷逃逸到生产环境的（需复盘）
issuetype = Bug 
  AND environment ~ "production" 
  AND created >= startOfMonth()
```

### 报告类型

| 报告 | 用途 | 关键指标 |
|------|------|---------|
| Sprint Burndown | 跟踪 Sprint 进度 | 剩余工作量随时间递减趋势 |
| Velocity Chart | 团队速率趋势 | 每 Sprint 完成的故事点 |
| Cumulative Flow | 工作流健康度 | 各状态的工作项分布变化 |
| Epic Burndown | Epic 进度 | Epic 内 Story 完成速率 |

## 动手实操

### 实操1：JIRA 项目配置清单

```markdown
## JIRA 测试管理项目配置步骤

### 1. 创建项目
- 选择 "Scrum software development" 模板
- 项目名称：质量保障中心
- 项目键：QA

### 2. 配置 Issue Type Scheme
- 启用类型：Epic, Story, Task, Bug, Sub-task
- 自定义类型：Test Case, Test Execution

### 3. 配置工作流
- To Do → In Progress → In Review → Done
- 添加条件：
  - "In Review" 转换需要 "Code Review" 字段为通过
  - "Done" 转换需要 "Test Report" 字段非空

### 4. 配置字段
- 添加自定义字段：
  - Story Points（数字）
  - Acceptance Criteria（文本区域）
  - Test Coverage（单选：Full/Partial/None）
  - Environment（多选：Dev/Staging/Production）
  - Regression Risk（单选：High/Medium/Low）

### 5. 配置看板列
- To Do | In Progress | Code Review | Testing | Done
- 设置 WIP 限制：
  - In Progress: 3
  - Testing: 2

### 6. 配置权限
- 测试人员组：可创建 Bug、可执行所有工作流转换
- 开发人员组：可创建 Story/Task、可编辑 Story Points
```

### 实操2：自动化规则（Automation Rules）

```yaml
# JIRA Automation Rules 配置（伪代码格式）
# 在 JIRA → Project settings → Automation 中配置

rules:
  - name: "新建 Bug 自动分配给测试负责人"
    trigger: issue_created
    condition: issue_type == "Bug"
    action: assign_to = "qa-lead"
    
  - name: "Bug 修复完成通知测试人员"
    trigger: field_changed
    field: status
    from: "In Progress"
    to: "In Review"
    action: 
      - send_notification: "@qa-team 缺陷已修复，请验证"
      - add_comment: "已标记为待测试，请验证后更新状态"
    
  - name: "Sprint 结束时关闭未完成项提醒"
    trigger: sprint_closed
    action:
      - search_issues: "sprint = {{sprint.id}} AND status != Done"
      - send_notification_to: "scrum-master"
      - message: "Sprint 结束但有以下未完成项：{{issues}}"
    
  - name: "SLA 超期报警 - Critical Bug"
    trigger: scheduled
    schedule: "every 1 hour"
    condition: 
      - priority == "Critical"
      - created <= -4h
      - status != "Done"
    action:
      - send_slack_notification: "#critical-bugs"
      - escalate_to: "engineering-manager"
    
  - name: "Story 完成时检查验收标准"
    trigger: status_changed
    to: "Done"
    condition: 
      - issue_type == "Story"
      - acceptance_criteria is empty
    action:
      - block_transition: "请填写验收标准后再关闭"
```

### 实操3：Velocity 分析脚本

```python
# jira_velocity_analysis.py
"""JIRA Sprint 速率分析脚本"""

from dataclasses import dataclass
from typing import List, Dict
import requests
from datetime import datetime


@dataclass
class Sprint:
    sprint_id: int
    name: str
    start_date: datetime
    end_date: datetime
    completed_points: float
    committed_points: float
    completed_issues: int
    total_issues: int
    
    @property
    def velocity(self) -> float:
        """实际速率：完成的故事点"""
        return self.completed_points
    
    @property
    def commitment_ratio(self) -> float:
        """承诺兑现率"""
        if self.committed_points == 0:
            return 0
        return self.completed_points / self.committed_points * 100


class JIRAVelocityAnalyzer:
    """JIRA 速率分析器"""
    
    def __init__(self, base_url: str, api_token: str, project_key: str):
        self.base_url = base_url
        self.auth = ("user@example.com", api_token)
        self.project_key = project_key
    
    def get_sprint_report(self, board_id: int) -> List[Sprint]:
        """获取所有 Sprint 报告"""
        url = f"{self.base_url}/rest/agile/1.0/board/{board_id}/sprint"
        response = requests.get(url, auth=self.auth)
        sprints_data = response.json().get("values", [])
        
        sprints = []
        for s in sprints_data:
            if s["state"] == "closed":
                report = self._get_sprint_metrics(s["id"])
                sprints.append(Sprint(
                    sprint_id=s["id"],
                    name=s["name"],
                    start_date=datetime.fromisoformat(s["startDate"].replace("Z", "+00:00")),
                    end_date=datetime.fromisoformat(s["endDate"].replace("Z", "+00:00")),
                    completed_points=report["completed_points"],
                    committed_points=report["committed_points"],
                    completed_issues=report["completed_issues"],
                    total_issues=report["total_issues"]
                ))
        return sprints
    
    def analyze_velocity(self, sprints: List[Sprint]) -> Dict:
        """分析速率趋势"""
        if not sprints:
            return {}
        
        velocities = [s.velocity for s in sprints]
        commitments = [s.commitment_ratio for s in sprints]
        
        return {
            "sprint_count": len(sprints),
            "avg_velocity": sum(velocities) / len(velocities),
            "min_velocity": min(velocities),
            "max_velocity": max(velocities),
            "avg_commitment_ratio": sum(commitments) / len(commitments),
            "velocity_trend": "上升" if velocities[-1] > velocities[0] else "下降",
            "recommendation": self._generate_recommendation(velocities, commitments)
        }
    
    def _generate_recommendation(self, velocities: List[float], commitments: List[float]) -> str:
        """生成 Sprint 规划建议"""
        avg_v = sum(velocities) / len(velocities)
        avg_c = sum(commitments) / len(commitments)
        
        if avg_c > 120:
            return f"承诺过载（平均兑现率 {avg_c:.0f}%），建议下一 Sprint 承诺量控制在 {avg_v * 0.8:.0f} 点"
        elif avg_c < 70:
            return f"承诺不足（平均兑现率 {avg_c:.0f}%），建议提高承诺量至 {avg_v:.0f} 点"
        else:
            return f"承诺合理，建议维持在 {avg_v:.0f} 点左右"
    
    def _get_sprint_metrics(self, sprint_id: int) -> Dict:
        """获取单个 Sprint 的详细指标"""
        # 实际实现调用 JIRA API
        # 这里返回模拟数据
        return {
            "completed_points": 35,
            "committed_points": 40,
            "completed_issues": 12,
            "total_issues": 15
        }


# 使用示例
if __name__ == "__main__":
    analyzer = JIRAVelocityAnalyzer(
        base_url="https://your-domain.atlassian.net",
        api_token="your-api-token",
        project_key="QA"
    )
    
    sprints = analyzer.get_sprint_report(board_id=1)
    analysis = analyzer.analyze_velocity(sprints)
    
    print(f"平均速率: {analysis['avg_velocity']:.1f} 故事点/Sprint")
    print(f"承诺兑现率: {analysis['avg_commitment_ratio']:.0f}%")
    print(f"趋势: {analysis['velocity_trend']}")
    print(f"建议: {analysis['recommendation']}")
```

## 常见坑

1. **工作流过度复杂**：添加过多状态导致跟踪负担重——保持 4~6 个状态为原则
2. **估算不一致**：团队对 Story Point 的理解不统一——需定期校准
3. **Sprint 目标漂移**：Sprint 中途插入新需求破坏冻结原则
4. **自动化规则冲突**：多条规则同时触发导致循环或矛盾
5. **权限配置混乱**：不经测试就给予所有人所有权限，导致数据质量下降

## 自测清单

- [ ] 能创建 JIRA Scrum 项目并配置 Issue 类型
- [ ] 能编写 10 条以上常用 JQL 查询
- [ ] 能配置包含条件和验证器的工作流
- [ ] 能解释 Velocity、Burndown、CFD 三种报告的用途
- [ ] 能设计自动化规则覆盖常见测试协作场景

## 延伸阅读

- [Atlassian JIRA Documentation](https://www.atlassian.com/software/jira/guides)
- [JQL Reference - Atlassian](https://support.atlassian.com/jira-software-cloud/docs/what-is-advanced-search-in-jira-cloud/)
- [JIRA Automation Guide](https://www.atlassian.com/software/jira/guides/automation/overview)
