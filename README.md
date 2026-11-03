# 支持工单分诊 Skill

将原始工单转为分类、评分、回复草稿和人工核实项。技能入口为 [SKILL.md](SKILL.md)，分类与评分、回复措辞按需读取 references/。此文件供使用者阅读。

## 使用

在项目中请求“帮我分诊这条工单”并提供原文。单条默认返回 JSON 对象，多条独立工单返回按输入排序的 JSON 数组。草稿不代表已经回复、转交或开始处理。

## 示例

输入：“我一个人看到设置页的帮助图标颜色不对，不影响使用，不着急。”

```json
{
  "category": "功能缺陷",
  "impact": 1,
  "urgency": 1,
  "priority_score": 1,
  "priority": "low",
  "summary": "客户反馈设置页帮助图标颜色异常；仅单个用户且不影响使用，impact=1；明确不紧急，urgency=1。是否为缺陷待核实。",
  "next_action": "建议产品支持核对图标预期样式及复现情况，处理时效待确认。",
  "draft_reply": "您好，感谢反馈帮助图标的显示问题。请补充产品版本及脱敏后的截图，便于产品支持核对预期样式。具体处理时效需核实后确认。",
  "human_review": ["核实图标颜色是否符合设计预期"],
  "missing_info": ["产品版本", "脱敏后的截图"]
}
```

## 评分与后端边界

乘积 16–25 为 high，6–15 为 medium，1–5 为 low。维度缺少关键证据时暂用 3 分，并在摘要说明；严重风险需额外人工核实。

当前 agent_lab/main.py 的乘积函数相同，但 mock_model 将所有低于 16 分的结果归为 medium，未实现 low。
/agent/analyze 接收 impact、urgency 和可选 max_steps，仅返回 summary、priority、next_action。完整分诊 JSON 不是该接口的请求或响应契约，接入需做字段映射并统一低分等级。
