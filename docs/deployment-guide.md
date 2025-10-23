# 通用内容引擎 v2.2 部署与运行指南

## 1. 环境准备
- 建议使用 n8n ≥ v1.30（支持新版 Webhook 响应与 LangChain 节点）  
- 准备 OpenAI API Key（或兼容的 Azure OpenAI 部署）  
- 部署方式不限：本地 Docker、Railway、Cloudflare Tunnel 等均可  
- 获取工程文件：`n8n.json`、`docs/*.md`、`configs/*.json`

## 2. 导入工作流
1. 登录 n8n，进入 **Workflows → Import from File**  
2. 选择项目根目录下的 `n8n.json` 完成导入  
3. 保存后工作流名称为 **Universal Content Engine v2.2**

## 3. 配置凭据
1. 打开任一 LLM 节点（如 `AI Agent`）  
2. 在 `Credentials` 中选择或新建 **OpenAI API** 凭据  
   - `API Key`: `sk-...`  
   - 若使用 Azure，切换至 Azure OpenAI 凭据并填写 `resource`、`deployment`  
3. 两个 LangChain 节点可共用同一凭据

## 4. Webhook 暴露方式
- 默认触发路径：`/webhook/content-engine-v2_2`  
- 测试模式：在编辑器中点击 *Test → Listen for Test Event*，使用 `/webhook-test/...` 地址  
- 生产模式：激活工作流后使用 `/webhook/...` 地址  
- 若运行在内网，可借助 Cloudflare Tunnel、ngrok 等工具暴露公网入口

## 5. 阶段测试流程

### 阶段一：创意生成
1. 准备 `configs/product_promotion/brand_profile.json` 与 `content_task.json`  
2. 组装请求体：
   ```json
   {
     "phase": "idea_generation",
     "brand_profile": { ... },
     "content_task": { ... }
   }
   ```
3. POST 至 Webhook URL  
4. 预期输出：`idea_generation_result` 中包含 `ideas`（数量 ≥3），`content_production_result` 为 `null`

### 阶段二：内容生产
1. 从阶段一响应中挑选一个 `idea` 赋值给 `selected_idea`  
2. 构造请求：
   ```json
   {
     "phase": "content_production",
     "brand_profile": { ... },
     "content_task": { ... },
     "selected_idea": { ... }
   }
   ```
3. POST 至同一 Webhook URL  
4. 预期输出：`content_production_result` 中包含 `overall_strategy` 与 `platforms`，且视频平台具有 `video_title` / `video_description` / `video_script_outline` 字段；`idea_generation_result` 为 `null`

## 6. Railway / Cloudflare 部署提示

### Railway
1. 使用官方 n8n Docker 模板或自建镜像  
2. 在环境变量中配置 `N8N_ENCRYPTION_KEY`、`OPENAI_API_KEY`、`N8N_BASIC_AUTH_*` 等  
3. 通过 Web UI 导入 `n8n.json`，或将其放置在挂载卷中  
4. 确保服务公开 HTTP 端口，可绑定自定义域名或使用 Railway 默认域名

### Cloudflare
1. 使用 Cloudflare Tunnel 将本地/远程 n8n 实例暴露到公网  
2. 若使用 Workers AI，可在工作流中调换为 Cloudflare AI 的 HTTP 请求节点  
3. 校验防火墙或 Access 策略，确保 Webhook 路径允许 POST + JSON

## 7. 演示视频建议
1. 录屏展示导入、凭据配置、阶段一/阶段二调用流程  
2. 若部署在 Railway 或 Cloudflare，可演示公网 URL 与执行日志  
3. 将视频上传至团队共享空间，便于验收

## 8. 维护与扩展
- 调整提示词：直接编辑 `Idea Prompt Builder` / `Content Prompt Builder` 节点的代码片段  
- 更换模型：在 `OpenAI Chat Model` / `OpenAI Chat Model1` 中修改 `model` 或温度参数  
- 扩展平台字段：同步更新 `Content Prompt Builder` 与 `Format Content Output` 的校验逻辑  
- 监控与重试：建议启用 n8n 执行日志或接入外部告警系统，保证长期开通稳定
