# 模型调用参数（DeepSeek / 豆包语音）

> 本文件**刻意保留**外部模型的请求参数，便于复刻与替换；平台自身的接口路径、部署信息与任何密钥均不在其中。
> 密钥一律用环境变量或占位符表示，不要把真实值写进仓库。

## 0. 通用约定

- 文本模型走 OpenAI 兼容协议：`POST {base}/chat/completions`，`Authorization: Bearer <KEY>`。
- 语音服务走火山引擎 OpenSpeech：`POST https://openspeech.bytedance.com/api/v3/...`，鉴权用 `X-Api-Key`。
  **语音控制台的 API Key 与方舟（文本）API Key 是两套**，混用会 401。
- 统一封装签名（平台内部）：`call(system, user, model, maxTokens, jsonMode, noThink, timeoutMs, requestConfig)`。
- 计量：从响应的 `usage.prompt_tokens / completion_tokens` 累计；按 UTC+8 分日。
- 超时：对话约 60s；评分/规划 180s；长评估链路 600s。
- JSON 模式：请求体加 `response_format: {type: "json_object"}`，且提示词里必须出现 "json" 字样，否则部分网关会拒绝。

## 1. DeepSeek 文本（平台默认）

**端点**：`POST https://api.deepseek.com/chat/completions`

**请求头**
```
Content-Type: application/json
Authorization: Bearer $DEEPSEEK_API_KEY
```

**请求体（单轮，主力形态）**
```json
{
  "model": "deepseek-flash",
  "messages": [
    {"role": "system", "content": "系统提示（角色/约束/评分规则）"},
    {"role": "user",   "content": "用户输入（病例+对话+量规等）"}
  ],
  "temperature": 0.5,
  "max_tokens": 11095,
  "response_format": {"type": "json_object"},
  "thinking": {"type": "disabled"}
}
```

**参数说明**

| 字段 | 取值 | 说明 |
|---|---|---|
| `model` | `deepseek-flash`（当前硬编码） | 配置项不生效，见已知问题 |
| `temperature` | 0 / 0.2 / 0.4 / 0.5 / 0.7 | 评分用 0；结构化分析 0.2；对话 0.7 |
| `max_tokens` | 256 – 11095 | 清单判定 256；对话 1024；单题压测 2048；四评委 3072；规划 1200–1600；长评估 8192–11095 |
| `response_format` | `json_object` | 仅在需要结构化输出时加；纯文本评价不加 |
| `thinking` | `{"type":"disabled"}` | 关闭思考链，降低延迟；评分与分析类常用 |
| 超时 | 60s / 180s / 600s | 按任务复杂度 |

**多轮变体**：`messages` 数组直接传历史；固定 `thinking disabled`、`temperature 0.7`、`max_tokens 1024`（带对话记忆的短交互）。

**余额查询**：`GET https://api.deepseek.com/user/balance`，`Authorization: Bearer <KEY>`，超时 8s。

## 2. 豆包文本（火山方舟 Ark，可替换）

**端点**：`POST https://ark.cn-beijing.volces.com/api/v3/chat/completions`

- 协议与 DeepSeek 兼容（`messages`、`temperature`、`max_tokens`、`response_format` 同上）。
- `model` 填方舟的**接入点 ID**（endpoint id），不是模型名。
- 平台当前默认走 DeepSeek；Ark 作为可替换连接保留（配置 `api_base` + `api_key` 即可切换）。
- 方舟 Key 与语音 Key 分开管理。

## 3. 豆包语音

### 3.1 语音合成 TTS 2.0

**端点**：`POST https://openspeech.bytedance.com/api/v3/tts/unidirectional`

**请求头**
```
Content-Type: application/json
X-Api-Key: $VOLC_SPEECH_API_KEY
X-Api-Resource-Id: seed-tts-2.0
```

**请求体**
```json
{
  "user": {"uid": "medisim"},
  "req_params": {
    "text": "待合成文本",
    "speaker": "zh_female_cancan_uranus_bigtts",
    "audio_params": {"format": "mp3"}
  }
}
```

- 响应可能是 `audio/*` 二进制，也可能是 `text/plain` 的 JSON 行（含 base64 分片）；两种都要处理。
- 音色目录属于 2.0 资源族（如 `zh_female_cancan_uranus_bigtts`、`zh_male_dayi_uranus_bigtts` 等十余个）；**用错资源族会被网关以 "resource ID is mismatched with speaker" 拒绝**。
- 平台做法：维护允许音色集合，不在集合内时回退到默认音色。

### 3.2 语音识别 ASR（录音文件 · 极速版）

**端点**：`POST https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash`

**请求头**
```
Content-Type: application/json
X-Api-Key: $VOLC_SPEECH_API_KEY
X-Api-Resource-Id: volc.bigasr.auc_turbo
X-Api-App-Key: $VOLC_APP_ID
X-Api-Request-Id: <uuid>
X-Api-Sequence: -1
```

**请求体**
```json
{
  "user": {"uid": "medisim"},
  "audio": {"data": "<base64 音频>"},
  "request": {"model_name": "bigmodel"}
}
```

- 返回 `result.text` 与 `audio_info.duration`；超时 60s。
- 适合"整段录音转写"场景（思考模式的一句话输入、复核用）。

### 3.3 语音识别 ASR（流式）

**端点**：`wss://openspeech.bytedance.com/api/v3/sauc/bigmodel_nostream`
**资源**：`volc.seedasr.sauc.duration`（流式识别 2.0 小时版）

- 用于按住说话/实时字幕；二进制协议，音频分片发送，返回中间与最终结果。

### 3.4 全双工实时语音（Realtime）

**端点**：`wss://openspeech.bytedance.com/api/v3/duplex/realtime/dialogue`
**鉴权**：握手头 `x-api-key: $VOLC_SPEECH_API_KEY`

**建会话事件**
```json
{
  "type": "session.create",
  "session": {
    "model": "1.2.6.1",
    "instructions": "病例约束 + 教学要求 + 学生锁定提示",
    "audio": {
      "input":  {"format": {"type": "pcm", "rate": 16000}},
      "output": {"format": {"type": "pcm", "rate": 24000}, "voice": "zh_female_cancan_uranus_bigtts"}
    }
  },
  "extension": {"asr": {}, "tts": {}, "dialog": {}}
}
```

- 事件：`session.created`（可回 `session.close`）→ 音频二进制帧双向传输 → `error`。
- 服务端职责：校验平台令牌与课程成员、注入病例约束、心跳（约 25s）、异常转发；**评分不在连接内做**。
- 违规检测：每累计约 4s 抽样学生音频，与患者/助手音色比对（阈值约 0.5），连续命中记为"AI 代答"。

## 4. 场景参数建议（复刻时可直接照用）

| 场景 | 模型 | temperature | max_tokens | JSON | 思考 | 超时 |
|---|---|---|---|---|---|---|
| 患者对话（逐句） | 文本 | 0.7 | 1024 | 否 | 关 | 60s |
| 四评委评分 | 文本 | 0 | 3072 | 是 | 关 | 180s |
| 写作质量评估 | 文本 | 0.2 | 11095 | 是 | 开 | 600s |
| 单题压力测试 | 文本 | 0.2 | 2048 | 否 | 关 | 600s |
| 执行质量总评 | 文本 | 0 | 8192 | 是 | 关 | 600s |
| 错因/复习规划 | 文本 | 0 | 1200–1600 | 是 | 关 | 120–180s |
| 图谱节点评价 | 文本 | 0.2 | 1200 | 是 | 关 | 120s |
| 小结评语 | 文本 | 0.5 | 900–1200 | 是 | 关 | 180s |
| 语音合成 | TTS 2.0 | — | — | — | — | 30s |
| 录音转写 | ASR flash | — | — | — | — | 60s |

## 5. 密钥与配置

- 环境变量覆盖（示例命名）：`DEEPSEEK_API_KEY`、`DEEPSEEK_API_BASE`、`DOUBAO_REALTIME_APP_ID`、`DOUBAO_REALTIME_API_KEY`、`DOUBAO_REALTIME_SPEAKER`。
- 语音 `App ID` 只用于部分接口的 `X-Api-App-Key`，不参与签名。
- 不要把 Key 写进技能、README、截图或日志；替换模型时只改连接配置，不改业务代码。
- 现状提醒：模型名硬编码、fallback 实为同模型重试（见 [known-issues.md](known-issues.md) A3）。
