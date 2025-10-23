# 通用内容引擎 v2.2 工作流设计说明

## 1. 总览
- **文件**：`n8n.json`  
- **触发方式**：`Webhook Trigger`（POST `/content-engine-v2_2`）  
- **核心能力**：单一工作流驱动创意生成与内容生产两个阶段，支持多语言、范文风格学习、视频平台适配  
- **模型节点**：基于 n8n LangChain Agent + OpenAI Chat Model，可替换为其他兼容模型

## 2. 主要节点与职责
| 节点名称 | 类型 | 说明 |
| --- | --- | --- |
| Webhook Trigger | Webhook | 接收标准 JSON 输入，并在流程结束时作为响应出口 |
| Validate Input | Function | 校验 `brand_profile`、`content_task`、`exemplars`、`objective` 等必填字段 |
| Phase Switch | Switch | 根据 `phase` 路由到创意生成或内容生产分支 |
| Idea Prompt Builder | Function | 自动检测语言并组装创意阶段提示词（确保 ≥3 个创意） |
| AI Agent + OpenAI Chat Model | LangChain Agent | 生成结构化创意 JSON，限定输出为纯 JSON |
| Format Idea Output | Function | 抽取并验证创意数组，同时附加阶段标识 |
| Ensure Selected Idea | Function | 阶段二入口，确认请求携带 `selected_idea` |
| Content Prompt Builder | Function | 汇总品牌信息、范文与目标，动态加入视频平台要求 |
| AI Agent1 + OpenAI Chat Model1 | LangChain Agent | 结合范文风格生成多平台内容 |
| Format Content Output | Function | 校验平台字段齐全，视频平台补充标题/描述/脚本大纲 |
| Final Response Dispatcher | Function | 收敛任一分支输出，补充时间戳与版本信息并统一响应 |

## 3. 数据流程
1. **阶段判断**  
   `Webhook → Validate Input → Phase Switch`  
   - 默认 `phase` 为 `idea_generation`  
   - 强制校验 `exemplars`、`objective`、`output_platforms` 等关键字段

2. **创意生成分支**  
   `Phase Switch(创意) → Idea Prompt Builder → AI Agent → Format Idea Output → Final Response Dispatcher`  
   - Prompt 包含品牌亮点、语气黑白名单、任务来源、目标平台等信息  
   - 自动识别语言并要求 LLM 使用对应语言输出

3. **内容生产分支**  
   `Phase Switch(生产) → Ensure Selected Idea → Content Prompt Builder → AI Agent1 → Format Content Output → Final Response Dispatcher`  
   - 将 `selected_idea` 与 `exemplars` 注入提示词，强调“人性化、去 AI 味”  
   - 当包含视频平台时，提示词自动追加视频标题/描述/脚本大纲要求  
   - `Format Content Output` 校验所有平台内容后再返回

## 4. 范文学习策略
- `Content Prompt Builder` 将 `brand_profile.exemplars` 顺序嵌入提示词，突出语气、节奏与句型参考  
- LLM 节点被要求吸收范文风格，避免生硬的 AI 口吻  
- 结果校验节点阻止缺失平台内容或视频字段的情况，保证范文学习效果可追溯

## 5. 多语言适配
- 借助 `detectLanguage` 辅助函数识别常见字符集（中文/日文/韩文/俄文/西欧语言）  
- Prompt 明确要求使用检测到的语言输出，避免上下文语言错配  
- 阶段二额外引用 `selected_idea.summary`，辅助模型保持语境一致

## 6. 扩展与定制
- **模型替换**：AI Agent 节点可切换为其他 OpenAI 兼容模型或自建推理服务  
- **上游对接**：`content_task.source_type/value` 可对接热点抓取、社媒监听等上游任务  
- **多品牌管理**：可在 Webhook 外增设 Router 或数据库查询节点，根据品牌 ID 动态注入配置  
- **监控与重试**：建议开启 n8n 执行日志或接入外部监控告警

## 7. 验证建议
1. 使用 `phase=idea_generation` 的示例配置调用 Webhook，确认 `ideas` 数量 ≥3、结构完整、语言正确  
2. 复制其中一个 `idea` 作为 `selected_idea`，触发 `phase=content_production`，验证所有 `output_platforms` 均返回内容；若包含视频平台，应出现 `video_title`、`video_description`、`video_script_outline`  
3. 检查响应中的 `workflow_version` 与 `timestamp` 字段，确保 Dispatcher 正常汇总  
4. 人工审核输出内容是否继承范文风格，如需强化可调整温度或补充提示语

## 8. 统一响应结构
Webhook 最终响应固定包含：

```json
{
  "phase": "idea_generation | content_production",
  "idea_generation_result": { ... } | null,
  "content_production_result": { ... } | null,
  "workflow_version": "v2.2",
  "timestamp": "2025-..."
}
```

- 创意阶段：`idea_generation_result` 含 `ideas` 数组，`content_production_result` 为 `null`  
- 内容阶段：`content_production_result` 含 `overall_strategy` 与 `platforms`，`idea_generation_result` 为 `null`  
- 调用方无需额外判断即可读取两个阶段的输出或空值
