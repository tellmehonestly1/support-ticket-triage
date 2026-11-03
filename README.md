# 支持工单分诊 2.0

入口为 [SKILL.md](SKILL.md)，分类与评分规则按需读取 references/。
在项目中请求“帮我分诊这条工单”，附上原文；默认单条返回 JSON 对象，多条独立工单按顺序返回数组。

## 2.0 的变化

- 证据不足的分值用 null，不再默认补 3 分。
- assessment 独立保存两个分值及各自理由。
- escalation 单独判断人工升级，未知分数也能触发立即核实。
- next_action 拆分负责角色和具体行动。
- 回复根据证据自然起草，取消分类套话模板。
- schema_version 标明输出契约版本，技能 metadata.version 为 2.0.0。

这是输出契约的不兼容升级。1.0 消费者不能继续按旧字段读取。

## 示例：信息不足

输入：“我们部门无法登录 OA，提示账号锁定。”

```json
{
  "schema_version": "2.0",
  "category": "登录与认证",
  "summary": "客户反馈部门无法登录 OA，提示账号锁定；业务阻塞程度、替代方案及发生时间尚不明确。",
  "assessment": {
    "impact": null,
    "impact_reason": "客户报告影响一个部门，但核心业务是否受阻及替代方案未知，无法区分影响等级。",
    "urgency": null,
    "urgency_reason": "缺少发生时间、当前阻塞情况和业务期限，无法判断紧急程度。",
    "priority_score": null,
    "priority": null
  },
  "escalation": {
    "level": "normal",
    "reason": "需要技术支持核实登录问题，暂无明确安全事件或严重生产事故证据。"
  },
  "next_action": {
    "owner_role": "技术支持",
    "action": "建议核实受影响用户范围、报错时间、核心业务阻塞和替代登录方式。"
  },
  "draft_reply": "您好，理解部门无法登录带来的不便。请补充首次出现问题的时间、受影响范围，以及是否阻塞当前业务、是否有其他可用登录方式，便于进一步判断。",
  "missing_info": ["首次发生时间及当前是否持续", "受影响用户范围", "核心业务是否受阻及是否有替代方案"],
  "human_review": ["核实账号锁定状态及影响范围"]
}
```

## 示例：分数未知但需立即核实

输入：“我现在好像能打开其他客户的资料，不知道是什么数据。”

```json
{
  "schema_version": "2.0",
  "category": "安全与合规",
  "summary": "客户报告当前疑似可以访问其他客户资料；是否真实越权及数据类型未核实。",
  "assessment": {
    "impact": null,
    "impact_reason": "真实性、数据敏感程度和范围未知，不能确认泄露后果。",
    "urgency": null,
    "urgency_reason": "报告指向当前疑似越权，但尚无法确认持续泄露或实际损失。",
    "priority_score": null,
    "priority": null
  },
  "escalation": {
    "level": "immediate",
    "reason": "疑似当前越权访问，即使影响和紧急程度尚无法量化，也需立即人工核实。"
  },
  "next_action": {
    "owner_role": "安全负责人",
    "action": "建议立即核实访问权限边界、是否仍可复现，并评估必要的隔离措施。"
  },
  "draft_reply": "您好，您反馈的疑似跨客户访问情况需要尽快由安全负责人核实。请勿继续访问或传播他人资料，并通过官方安全支持渠道提供发生时间及脱敏后的页面路径；不要发送他人的原始数据。",
  "missing_info": ["首次发现时间及页面路径", "是否为授权共享场景"],
  "human_review": ["核实是否越权及是否持续", "确认受影响数据范围并决定处置措施"]
}
```

## 示例：证据足够的低优先级

输入：“我一个人看到设置页的帮助图标颜色不对，不影响使用，不着急。”

```json
{
  "schema_version": "2.0",
  "category": "功能缺陷",
  "summary": "客户反馈个人设置页帮助图标颜色异常，不影响使用且明确不着急。",
  "assessment": {
    "impact": 1,
    "impact_reason": "客户报告仅个人非核心显示问题，不影响使用。",
    "urgency": 1,
    "urgency_reason": "客户明确不着急。",
    "priority_score": 1,
    "priority": "low"
  },
  "escalation": {
    "level": "normal",
    "reason": "无需立即升级，但图标是否符合设计预期需要产品支持核实。"
  },
  "next_action": {
    "owner_role": "产品支持",
    "action": "建议核对帮助图标的设计预期，并在必要时获取产品版本和脱敏截图。"
  },
  "draft_reply": "您好，感谢反馈帮助图标的颜色问题。请提供产品版本及脱敏后的截图，便于产品支持核对预期样式。",
  "missing_info": [
    "产品版本",
    "脱敏后的截图"
  ],
  "human_review": [
    "核实图标是否符合设计预期"
  ]
}
```

## 1.0 消费者迁移

| 1.0 | 2.0 |
|---|---|
| impact、urgency | assessment.impact、assessment.urgency，允许 null |
| priority_score、priority | 移入 assessment；任一评分未知时均为 null |
| summary 中混合评分理由 | summary 描述现象；理由分别放入 assessment |
| next_action 字符串 | next_action.owner_role 与 next_action.action |
| 无独立升级字段 | escalation.level 与 escalation.reason |
| 无版本字段 | schema_version 固定为 "2.0" |

category、draft_reply、missing_info、human_review 保留原有基本类型。
接入方必须显式处理 null，不能静默替换为 0、3 或 low。
需要保留输入工单 ID 时，在调用侧按顺序关联原始 ID；本契约不虚构标识。

## 当前后端限制

现有 /agent/analyze 接收整数 impact、urgency 和可选 max_steps；
响应仅含 summary、priority、next_action。agent_lab/main.py 的乘积算法相同，
但 mock_model 只区分 high（>=16）和 medium，没有 low。

2.0 完整输出不是该接口契约，不能直接传入。
评分含 null 时应先补信息，不调用只接收整数的评分接口；
评分完整时也需显式映射字段，并统一 low 的等级逻辑。
后端升级和接口迁移不由本次技能文档修改自动完成。
