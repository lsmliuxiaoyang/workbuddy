# OpenClaw 私人家庭教师 Agent · Skills 技能模块设计文档

> **版本：** v1.0 | **作者：** 刘潇洋 · 蔡长青 | **日期：** 2026-06

---

## 一、概述

OpenClaw 的 **Skills（技能系统）** 是五大核心组件之一，负责为 Agent 提供"可执行能力单元"。

每个 Skill 是一个独立的、可复用的功能模块，Agent 在"思考-行动"循环中按需调用。对于私人家庭教师 Agent，Skills 是实现"主动辅导 → 进度监控 → 智能提醒"的核心抓手。

```
Agent（推理引擎）
    ↓  调用
Skills（技能系统）
    ├── homework_monitor      作业进度监控
    ├── active_tutoring       主动辅导触发
    ├── progress_alert        拖延预警推送
    ├── daily_report          每日进度报告
    ├── weak_point_push       薄弱知识点推送
    └── parent_notify         家长通知推送
```

---

## 二、Skill 文件结构规范

每个 Skill 遵循以下目录结构：

```
skills/
└── skill_name/
    ├── skill.yaml        # 技能声明（元数据 + 触发条件 + 参数定义）
    ├── handler.py        # 执行逻辑（Python 实现）
    └── prompts/
        └── system.txt    # 提示词模板（可选，用于 LLM 推理类技能）
```

---

## 三、各技能模块详细设计

### 3.1 作业进度监控（homework_monitor）

**职责：** 实时采集并跟踪学生的作业完成状态，是所有监控功能的数据基础。

```yaml
# skill.yaml
name: homework_monitor
version: "1.0.0"
description: 实时监控学生作业完成进度，采集已完成/进行中/未完成状态及耗时数据
category: monitoring

triggers:
  - event: schedule          # 定时触发
    cron: "*/5 * * * *"      # 每 5 分钟轮询一次
  - event: student_login     # 学生登录时触发

parameters:
  student_id:
    type: string
    required: true
    description: 学生唯一标识符
  subject_list:
    type: array
    required: false
    description: 监控的学科列表，为空时监控全部学科

outputs:
  homework_status:
    type: object
    schema:
      total: integer           # 总作业数
      completed: integer       # 已完成数
      in_progress: integer     # 进行中数
      pending: integer         # 未开始数
      completion_rate: float   # 完成率（0~1）
      avg_duration_min: float  # 平均耗时（分钟）

integrations:
  - system: 学生学习数据管理系统
    api: GET /api/homework/status
    auth: service_token
```

```python
# handler.py（简化示例）
from openclaw.skills import BaseSkill
from openclaw.integrations import LearningDataSystem

class HomeworkMonitorSkill(BaseSkill):
    name = "homework_monitor"

    async def execute(self, student_id: str, subject_list: list = None) -> dict:
        """
        从学生学习数据管理系统拉取作业状态
        返回结构化进度数据供其他 Skills 消费
        """
        system = LearningDataSystem()
        raw_data = await system.get_homework_status(
            student_id=student_id,
            subjects=subject_list
        )
        return self._normalize(raw_data)

    def _normalize(self, data: dict) -> dict:
        total = len(data["homeworks"])
        completed = sum(1 for h in data["homeworks"] if h["status"] == "done")
        in_progress = sum(1 for h in data["homeworks"] if h["status"] == "doing")
        return {
            "total": total,
            "completed": completed,
            "in_progress": in_progress,
            "pending": total - completed - in_progress,
            "completion_rate": completed / total if total > 0 else 0.0,
            "avg_duration_min": data.get("avg_duration_min", 0),
        }
```

---

### 3.2 主动辅导触发（active_tutoring）

**职责：** 根据学生当前作业内容和知识点掌握情况，主动推送辅导内容，替代家长日常答疑。

```yaml
# skill.yaml
name: active_tutoring
version: "1.0.0"
description: 根据学生作业内容主动触发个性化辅导，推送讲解、提示或例题
category: tutoring

triggers:
  - event: homework_started        # 学生开始写某科作业时触发
  - event: error_detected          # 检测到连续答错时触发
  - event: stuck_timeout           # 同一题停留超过阈值触发
    threshold_minutes: 5

parameters:
  student_id:
    type: string
    required: true
  homework_id:
    type: string
    required: true
  question_id:
    type: string
    required: false
    description: 当前卡住的题目 ID，为空则做整体辅导
  grade:
    type: string
    required: true
    description: "学生年级，如：小学三年级"
  subject:
    type: string
    required: true
    description: "学科，如：数学、语文、英语"

outputs:
  tutoring_content:
    type: object
    schema:
      type: enum              # hint（提示）/ explain（讲解）/ example（例题）
      content: string         # 辅导正文
      knowledge_point: string # 对应知识点
      difficulty: integer     # 难度评级 1-5
      source: string          # 来源（好未来学科资源库）

integrations:
  - system: 学生学习数据管理系统
    api: GET /api/student/knowledge_profile
  - system: 好未来学科资源库
    api: POST /api/resource/recommend
```

```python
# handler.py（简化示例）
from openclaw.skills import BaseSkill
from openclaw.llm import LLMClient
from openclaw.integrations import LearningDataSystem, TALResourceLibrary

class ActiveTutoringSkill(BaseSkill):
    name = "active_tutoring"

    async def execute(self, student_id: str, homework_id: str,
                      grade: str, subject: str, question_id: str = None) -> dict:
        # 1. 拉取学生知识点画像
        profile = await LearningDataSystem().get_knowledge_profile(student_id, subject)

        # 2. 从好未来资源库检索匹配内容
        resource = await TALResourceLibrary().recommend(
            grade=grade,
            subject=subject,
            question_id=question_id,
            weak_points=profile["weak_points"],
        )

        # 3. 通过 LLM 生成个性化辅导话术
        llm = LLMClient()
        content = await llm.generate(
            system_prompt=self.load_prompt("system.txt"),
            context={
                "student_grade": grade,
                "subject": subject,
                "resource": resource,
                "weak_points": profile["weak_points"],
            }
        )

        return {
            "type": resource["type"],
            "content": content,
            "knowledge_point": resource["knowledge_point"],
            "difficulty": resource["difficulty"],
            "source": resource["source_id"],
        }
```

```text
# prompts/system.txt
你是一位专业的${subject}辅导老师，正在帮助一位${student_grade}的学生完成作业。

学生的薄弱知识点：${weak_points}
参考资源内容：${resource}

请用简洁、鼓励的语气，针对该知识点给出一个简短的辅导提示（不超过150字）。
不要直接给出答案，而是引导学生自己思考。
```

---

### 3.3 拖延预警推送（progress_alert）

**职责：** 当检测到作业超时或进度滞后时，主动提醒学生加快节奏，同时通知家长。

```yaml
# skill.yaml
name: progress_alert
version: "1.0.0"
description: 检测作业拖延行为，向学生推送提醒，并实时通知家长，响应时间 ≤1分钟
category: alert

triggers:
  - event: homework_overtime
    description: 作业耗时超过预设阈值
  - event: long_idle
    description: 学生长时间无操作（超过 10 分钟）
  - event: scheduled_check
    cron: "*/1 * * * *"   # 每分钟检查一次

parameters:
  student_id:
    type: string
    required: true
  homework_id:
    type: string
    required: true
  overtime_minutes:
    type: integer
    default: 30
    description: 作业超时阈值（分钟），默认30分钟触发
  idle_minutes:
    type: integer
    default: 10
    description: 无操作超时阈值（分钟）

outputs:
  alert_sent:
    type: boolean
  alert_type:
    type: enum            # overtime / idle / delay
  student_message:
    type: string          # 推送给学生的提醒文本
  parent_message:
    type: string          # 推送给家长的通知文本
  triggered_at:
    type: datetime

integrations:
  - system: 用户反馈管理系统
    api: POST /api/alert/log    # 记录预警日志
  - system: 消息推送服务
    api: POST /api/notify/push
```

```python
# handler.py（简化示例）
from openclaw.skills import BaseSkill
from openclaw.integrations import NotifyService, AlertLogSystem
from datetime import datetime

class ProgressAlertSkill(BaseSkill):
    name = "progress_alert"

    STUDENT_MESSAGES = {
        "overtime": "⏰ 加油！这道题已经思考了{minutes}分钟，要不要换个角度试试？",
        "idle":     "📚 是不是走神了？回来继续吧，作业快写完了！",
        "delay":    "🚀 作业进度有点慢哦，冲刺一下，早点完成早点玩！",
    }
    PARENT_MESSAGES = {
        "overtime": "📊 提醒：{student_name}的{subject}作业已超时{minutes}分钟，请关注进度。",
        "idle":     "📊 提醒：{student_name}已超过{minutes}分钟无作业操作，可能需要引导。",
        "delay":    "📊 提醒：{student_name}今日作业完成率{rate}%，进度较慢。",
    }

    async def execute(self, student_id: str, homework_id: str,
                      overtime_minutes: int = 30, idle_minutes: int = 10) -> dict:
        alert_type = await self._detect_alert_type(
            student_id, homework_id, overtime_minutes, idle_minutes
        )
        if not alert_type:
            return {"alert_sent": False}

        ctx = await self._build_context(student_id, homework_id, alert_type)
        student_msg = self.STUDENT_MESSAGES[alert_type].format(**ctx)
        parent_msg = self.PARENT_MESSAGES[alert_type].format(**ctx)

        notify = NotifyService()
        await notify.push_to_student(student_id, student_msg)
        await notify.push_to_parent(student_id, parent_msg)
        await AlertLogSystem().log(student_id, alert_type, datetime.now())

        return {
            "alert_sent": True,
            "alert_type": alert_type,
            "student_message": student_msg,
            "parent_message": parent_msg,
            "triggered_at": datetime.now().isoformat(),
        }
```

---

### 3.4 每日进度报告（daily_report）

**职责：** 每日自动生成学生作业完成情况报告，推送至家长终端。

```yaml
# skill.yaml
name: daily_report
version: "1.0.0"
description: 每日自动汇总学生作业完成数据，生成可视化进度报告并推送给家长
category: reporting

triggers:
  - event: schedule
    cron: "0 21 * * *"    # 每天 21:00 自动触发

parameters:
  student_id:
    type: string
    required: true
  report_date:
    type: date
    required: false
    description: 报告日期，默认为当天

outputs:
  report:
    type: object
    schema:
      date: date
      completion_rate: float          # 当天完成率
      total_subjects: integer         # 今日学科数
      completed_subjects: integer     # 全部完成的学科数
      total_duration_min: integer     # 今日作业总耗时
      alert_count: integer            # 今日预警次数
      weak_points_today: array        # 今日新发现的薄弱点
      weekly_trend: array             # 近7天完成率趋势
      suggestion: string              # 明日辅导建议

integrations:
  - system: 学生学习数据管理系统
    api: GET /api/report/daily
  - system: 消息推送服务
    api: POST /api/notify/report
```

---

### 3.5 薄弱知识点推送（weak_point_push）

**职责：** 分析学生历史错题，识别薄弱知识点并主动推送强化练习。

```yaml
# skill.yaml
name: weak_point_push
version: "1.0.0"
description: 分析历史错题数据，识别薄弱知识点，主动推送针对性强化练习
category: tutoring

triggers:
  - event: homework_completed        # 每次完成作业后分析
  - event: schedule
    cron: "0 18 * * *"               # 每天 18:00 定时推送

parameters:
  student_id:
    type: string
    required: true
  subject:
    type: string
    required: false
    description: 指定学科，为空则分析全部学科
  push_limit:
    type: integer
    default: 3
    description: 每次最多推送的知识点数量

outputs:
  pushed_points:
    type: array
    items:
      knowledge_point: string   # 知识点名称
      mastery_rate: float       # 掌握程度 0~1
      practice_content: object  # 推荐练习题
      reason: string            # 推荐理由
```

---

### 3.6 家长通知推送（parent_notify）

**职责：** 统一管理向家长推送各类通知，支持多渠道（App、微信等）。

```yaml
# skill.yaml
name: parent_notify
version: "1.0.0"
description: 统一的家长通知推送技能，支持进度提醒、预警通知、报告推送等多场景
category: notification

triggers:
  - event: manual                    # 由其他 Skills 主动调用
  - event: progress_milestone        # 里程碑达成时（完成率达50%/100%）

parameters:
  student_id:
    type: string
    required: true
  notify_type:
    type: enum
    values: [progress, alert, report, milestone, weekly_summary]
    required: true
  content:
    type: string
    required: true
    description: 通知正文内容
  channels:
    type: array
    default: ["app_push"]
    description: "推送渠道，可选：app_push / wechat / sms"
  priority:
    type: enum
    values: [high, normal, low]
    default: normal

outputs:
  success: boolean
  channels_sent: array
  sent_at: datetime
```

---

## 四、技能调用关系图

```
用户事件 / 定时调度
        │
        ▼
homework_monitor（每5分钟轮询）
        │
        ├──→ active_tutoring（学生卡题 → 主动辅导）
        │           │
        │           └──→ parent_notify（通知家长辅导情况）
        │
        ├──→ progress_alert（超时/闲置 → 预警）
        │           │
        │           └──→ parent_notify（预警推送家长）
        │
        └──→ daily_report（每日21:00 → 生成报告）
                    │
                    ├──→ weak_point_push（推送薄弱点）
                    └──→ parent_notify（推送日报）
```

---

## 五、配置示例（config.yaml）

```yaml
# OpenClaw Agent 配置文件
agent:
  name: 私人家庭教师
  version: "1.0.0"
  description: 面向学生及家长的主动辅导与进度监控智能体

skills:
  enabled:
    - homework_monitor
    - active_tutoring
    - progress_alert
    - daily_report
    - weak_point_push
    - parent_notify

  homework_monitor:
    poll_interval_minutes: 5

  progress_alert:
    overtime_threshold_minutes: 30     # 超时30分钟触发预警
    idle_threshold_minutes: 10         # 无操作10分钟触发预警

  active_tutoring:
    stuck_timeout_minutes: 5           # 同一题5分钟无进展触发辅导
    max_hint_per_homework: 3           # 每次作业最多提示3次

  daily_report:
    push_time: "21:00"
    include_weekly_trend: true

  weak_point_push:
    max_points_per_day: 3
    push_time: "18:00"

integrations:
  learning_data_system:
    base_url: "${LEARNING_DATA_API_URL}"
    auth_token: "${SERVICE_TOKEN}"
  notify_service:
    base_url: "${NOTIFY_API_URL}"
    channels:
      - app_push
      - wechat
```

---

## 六、开发规范

### 6.1 Skill 基类接口

```python
from abc import ABC, abstractmethod

class BaseSkill(ABC):
    name: str               # 技能唯一标识
    version: str = "1.0.0"

    @abstractmethod
    async def execute(self, **kwargs) -> dict:
        """技能执行入口，返回标准化输出字典"""
        pass

    def load_prompt(self, filename: str) -> str:
        """加载 prompts/ 目录下的提示词文件"""
        import os
        prompt_path = os.path.join(os.path.dirname(__file__), "prompts", filename)
        with open(prompt_path, "r", encoding="utf-8") as f:
            return f.read()

    def validate_params(self, required: list, kwargs: dict):
        """参数校验工具方法"""
        for key in required:
            if key not in kwargs or kwargs[key] is None:
                raise ValueError(f"Missing required parameter: {key}")
```

### 6.2 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 技能目录名 | `snake_case` | `homework_monitor` |
| 技能类名 | `PascalCase + Skill` | `HomeworkMonitorSkill` |
| 触发事件名 | `snake_case` | `homework_started` |
| 输出字段名 | `snake_case` | `completion_rate` |

### 6.3 测试规范

每个 Skill 需包含：
- `test_normal_case.py`：正常流程测试
- `test_edge_case.py`：边界条件测试（空数据、接口超时等）
- `mock_data/`：模拟数据目录

---

*文档版本 v1.0 · 好未来网校增长研发部 · 2026-06*
