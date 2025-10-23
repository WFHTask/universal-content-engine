# 通用内容引擎 v2.2 输入配置规范

## 1. 顶层结构

```json
{
  "phase": "idea_generation",
  "brand_profile": { ... },
  "content_task": { ... },
  "selected_idea": { ... }
}
```

- `phase` *(可选)*：`idea_generation` / `content_production`，默认 `idea_generation`  
- `brand_profile` *(必填)*：品牌背景及范文学习配置  
- `content_task` *(必填)*：本次内容任务定义  
- `selected_idea` *(阶段二必填)*：来自阶段一的创意对象

## 2. brand_profile

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| name | string | 是 | 品牌或业务名称 |
| description | string | 是 | 品牌概述、核心卖点 |
| audience | string | 是 | 目标受众描述，支持多语言 |
| tone_guidelines | object | 是 | 语气正负面清单 |
| tone_guidelines.positive | array\<string> | 是 | 建议采用的语气或关键词 |
| tone_guidelines.negative | array\<string> | 是 | 需避免的语气、敏感表述 |
| exemplars | array\<string> | 是 | 范文样本文本（≥1，建议 2-5 篇） |

### 示例

```json
"brand_profile": {
  "name": "Aurora Labs",
  "description": "为创作者提供 AI 赋能的视频剪辑工具",
  "audience": "中小型内容创作者、教育类视频制作人",
  "tone_guidelines": {
    "positive": ["真诚指导", "鼓励语气", "轻度专业术语"],
    "negative": ["夸大其词", "过度营销", "冷冰冰的机器人口吻"]
  },
  "exemplars": [
    "范文 1 ...",
    "范文 2 ..."
  ]
}
```

## 3. content_task

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| objective | string | 是 | 内容核心目标，如 `product_promotion` / `brand_building` / `community_activation` |
| source_type | string | 是 | 热点来源类型，如 `trend`, `event`, `hashtag`, `insight` |
| source_value | string | 是 | 热点来源具体值，如话题标签、事件名称 |
| output_platforms | array\<string> | 是 | 需要产出的平台列表，须覆盖文本或视频渠道 |

### output_platforms 建议
- 文本类：`X`, `LinkedIn`, `WeChat`, `XiaoHongShu`, `Weibo`, `Instagram`
- 视频类：`YouTube`, `Bilibili`, `Douyin`, `TikTok`

当包含视频平台（YouTube / Bilibili 等）时，工作流会要求额外生成 `video_title`、`video_description`、`video_script_outline`。

## 4. selected_idea（阶段二专用）
- 来源：阶段一 `ideas` 数组中的任意对象  
- 推荐字段：`id`, `title`, `hook`, `summary`, `target_platforms`, `rationale`, `language`  
- 阶段二触发时需将完整 JSON 放入 `selected_idea`，工作流会自动带入提示词

## 5. JSON Schema 参考

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["brand_profile", "content_task"],
  "properties": {
    "phase": {
      "type": "string",
      "enum": ["idea_generation", "content_production"],
      "default": "idea_generation"
    },
    "brand_profile": {
      "type": "object",
      "required": ["name", "description", "audience", "tone_guidelines", "exemplars"],
      "properties": {
        "name": { "type": "string", "minLength": 1 },
        "description": { "type": "string", "minLength": 1 },
        "audience": { "type": "string", "minLength": 1 },
        "tone_guidelines": {
          "type": "object",
          "required": ["positive", "negative"],
          "properties": {
            "positive": {
              "type": "array",
              "minItems": 1,
              "items": { "type": "string" }
            },
            "negative": {
              "type": "array",
              "minItems": 1,
              "items": { "type": "string" }
            }
          }
        },
        "exemplars": {
          "type": "array",
          "minItems": 1,
          "items": { "type": "string", "minLength": 50 }
        }
      }
    },
    "content_task": {
      "type": "object",
      "required": ["objective", "source_type", "source_value", "output_platforms"],
      "properties": {
        "objective": { "type": "string", "minLength": 1 },
        "source_type": { "type": "string", "minLength": 1 },
        "source_value": { "type": "string", "minLength": 1 },
        "output_platforms": {
          "type": "array",
          "minItems": 1,
          "items": {
            "type": "string",
            "enum": [
              "X", "LinkedIn", "WeChat", "Weibo", "XiaoHongShu",
              "Instagram", "Douyin", "TikTok", "YouTube", "Bilibili"
            ]
          },
          "uniqueItems": true
        }
      }
    },
    "selected_idea": {
      "type": "object"
    }
  },
  "additionalProperties": false
}
```

> 该 Schema 可直接用于 IDE 校验或 n8n 侧输入检测。
