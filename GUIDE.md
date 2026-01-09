# GLM-Free-API 深度分析与优化指南

## 1. 项目深度分析

### 1.1 核心架构
本项目是一个基于 Koa 框架的中间件服务，其核心功能是将 OpenAI/Claude/Gemini 的 API 请求转换为智谱清言（GLM）官方网页版的后端 API 请求。

- **逆向工程**: 项目的核心在于逆向了 GLM 网页版的签名算法 (`generateSign` 函数)。该算法依赖于时间戳、随机数和一个硬编码的 `SIGN_SECRET`。
- **请求伪装**: 通过模拟浏览器的 Headers（包括 User-Agent, Origin, Referer 等）来绕过官方的爬虫检测。
- **流式转发**: 使用 Server-Sent Events (SSE) 技术，将 GLM 的流式响应实时转换为 OpenAI 兼容的流式格式。
- **适配器模式**: 
    - `chat.ts`: 处理标准 OpenAI 格式请求。
    - `claude-adapter.ts`: 将 Claude 格式请求转换为 GLM 格式，并将响应转回 Claude 格式。
    - `gemini-adapter.ts`: 同上，针对 Google Gemini 格式。

### 1.2 潜在风险与挑战
1.  **签名失效**: `SIGN_SECRET` 是硬编码的字符串。一旦官方更新了前端代码中的密钥或签名算法，服务将立即不可用。
2.  **账号封禁**: 高频调用或固定的指纹特征（如 User-Agent）可能导致账号被风控。
3.  **接口变动**: 这是一个非官方的逆向 API，官方随时可能调整后端接口参数，导致服务报错。

## 2. 优化实施说明

针对上述风险，本次会话中实施了以下优化：

### 2.1 安全性增强：签名密钥配置化
- **问题**: 原项目将 `SIGN_SECRET` 硬编码在代码中，一旦密钥失效，用户必须修改源码。
- **优化**: 修改了 `src/lib/environment.ts` 和 `src/api/controllers/chat.ts`，现在优先从环境变量 `SIGN_SECRET` 读取密钥。
- **配置方法**: 在 Docker 启动命令中添加 `-e SIGN_SECRET=your_new_secret` 即可更新密钥，无需重新编译镜像。

### 2.2 稳定性增强：User-Agent 随机化
- **问题**: 原项目使用固定的 User-Agent，容易被识别为机器人行为。
- **优化**: 引入了一个包含主流浏览器（Chrome, Firefox, Safari）的 User-Agent 池。每次发起请求（刷新 Token、对话、生成图片等）时，都会随机选择一个 User-Agent。
- **效果**: 降低了被官方风控的概率，模拟了真实的多用户访问场景。

## 3. 高级配置与使用指南

### 3.1 获取 Refresh Token
1. 访问 [智谱清言](https://chatglm.cn/) 并登录。
2. 按 `F12` 打开开发者工具，切换到 `Application` (应用) -> `Cookies`。
3. 找到 `chatglm_refresh_token`，复制其值。

### 3.2 多账号配置
本项目支持多账号轮询，以提高并发能力和每日配额。

**配置方式**:
在客户端请求的 `Authorization` Header 中，将多个 Token 用逗号分隔：
```
Authorization: Bearer token1,token2,token3
```
服务会自动解析并随机选择一个 Token 进行请求。

### 3.3 环境变量配置
建议使用环境变量来配置服务，优先级高于 `service.yml`。

| 环境变量 | 说明 | 默认值 |
| :--- | :--- | :--- |
| `SERVER_PORT` | 服务端口 | 8000 |
| `SIGN_SECRET` | 签名密钥 | (内置默认值) |
| `TZ` | 时区 | Asia/Shanghai |

### 3.4 Docker 部署建议
```bash
docker run -it -d \
  --name glm-free-api \
  -p 8000:8000 \
  -e TZ=Asia/Shanghai \
  -e SIGN_SECRET="如果你获取到了新的密钥" \
  akashrajpuroh1t/glm-free-api-fix
```

## 4. API 使用示例

### 4.1 OpenAI 兼容格式 (Python)
```python
from openai import OpenAI

client = OpenAI(
    api_key="your_refresh_token", # 这里填 refresh_token
    base_url="http://localhost:8000/v1"
)

response = client.chat.completions.create(
    model="glm-4-plus",
    messages=[
        {"role": "user", "content": "你好，请介绍一下你自己"}
    ],
    stream=True
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### 4.2 Claude 兼容格式 (Curl)
```bash
curl http://localhost:8000/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: your_refresh_token" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "glm-4-plus",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "Hello, Claude format!"}
    ]
  }'
```

### 4.3 Gemini 兼容格式 (Curl)
```bash
curl "http://localhost:8000/v1beta/models/glm-4-plus:generateContent?key=your_refresh_token" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "parts": [{"text": "Explain how AI works"}]
    }]
  }'
```

## 5. 进一步优化建议 (Roadmap)

如果需要继续改进本项目，可以考虑以下方向：

1.  **Redis 缓存集成**: 目前 Token 缓存在内存中，重启服务会导致 Token 需要重新刷新。引入 Redis 可以持久化 Token 状态，并支持多实例部署。
2.  **自动保活机制**: 增加定时任务，定期刷新 Token，防止 Token 在长时间不使用后过期。
3.  **异常熔断**: 记录每个 Token 的失败率，对连续失败的 Token 进行暂时屏蔽，提高整体服务成功率。
