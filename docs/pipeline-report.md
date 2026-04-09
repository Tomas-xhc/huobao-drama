# 火爆剧 — 生图到生视频完整流程技术报告

> 生成时间：2026-04-09  
> 代码库：Tomas-xhc/huobao-drama

---

## 目录

1. [总体数据流](#1-总体数据流)
2. [生图阶段（Image Generation）](#2-生图阶段)
   - 2.1 [图片生成接口](#21-图片生成接口)
   - 2.2 [frameType 说明](#22-frametype-说明)
   - 2.3 [各 Provider 请求详情](#23-各-provider-请求详情)
   - 2.4 [异步 vs 同步处理流程](#24-异步-vs-同步处理流程)
3. [宫格图生图（Grid Generation）](#3-宫格图生图)
   - 3.1 [模式说明](#31-模式说明)
   - 3.2 [完整 Prompt 模板](#32-完整-prompt-模板)
   - 3.3 [参考图收集机制](#33-参考图收集机制)
4. [生视频阶段（Video Generation）](#4-生视频阶段)
   - 4.1 [referenceMode 说明](#41-referencemode-说明)
   - 4.2 [各 Provider 请求详情](#42-各-provider-请求详情)
   - 4.3 [轮询 vs Webhook](#43-轮询-vs-webhook)
5. [图片标准化处理](#5-图片标准化处理)
6. [合成阶段（FFmpeg Compose）](#6-合成阶段)
7. [全集拼接（FFmpeg Merge）](#7-全集拼接)
8. [完整状态流转](#8-完整状态流转)
9. [Provider 能力对比](#9-provider-能力对比)

---

## 1. 总体数据流

```
分镜 imagePrompt / videoPrompt
         │
         ▼
  ┌─────────────────┐
  │  生图（Images）  │  POST /api/images
  │  宫格图（Grid）  │  POST /api/grid/generate
  └────────┬────────┘
           │ 图片下载到 data/static/images/
           │ 回写 storyboard.firstFrameImage / lastFrameImage
           ▼
  ┌──────────────────┐
  │  生视频（Videos） │  POST /api/videos
  └────────┬─────────┘
           │ 视频下载到 data/static/videos/
           │ 回写 storyboard.videoUrl
           ▼
  ┌───────────────────────┐
  │  FFmpeg 合成（Compose）│  POST /api/compose/storyboards/:id/compose
  │  视频 + TTS音频 + 字幕 │
  └────────┬──────────────┘
           │ 输出到 data/static/composed/
           │ 回写 storyboard.composedVideoUrl
           ▼
  ┌────────────────────┐
  │  FFmpeg 拼接（Merge）│  POST /api/merge/episodes/:id
  └────────┬───────────┘
           │ 输出到 data/static/merged/
           │ 回写 episode.videoUrl
           ▼
        最终成片
```

---

## 2. 生图阶段

### 2.1 图片生成接口

**入口**：`POST /api/images`

| 字段 | 类型 | 说明 |
|------|------|------|
| `prompt` | string（必填） | 图片生成描述 |
| `storyboard_id` | number | 关联分镜 ID（自动读取 episode.imageConfigId） |
| `drama_id` | number | 剧集 ID |
| `scene_id` | number | 场景 ID（完成后更新 scenes.imageUrl） |
| `character_id` | number | 角色 ID（完成后更新 characters.imageUrl） |
| `model` | string | 覆盖 config 中的模型 |
| `size` | string | 尺寸，默认 `1920x1080` |
| `reference_images` | string[] | 参考图 URL 或本地路径列表（最多6张） |
| `frame_type` | string | 帧类型：`first_frame` / `last_frame` / 空 |
| `config_id` | number | 指定 AI 配置 ID |

---

### 2.2 frameType 说明

| frameType | 存储字段 | 后续用途 |
|-----------|---------|---------|
| `first_frame` | `storyboard.firstFrameImage` | 视频生成的首帧参考图 |
| `last_frame` | `storyboard.lastFrameImage` | 视频生成的尾帧参考图 |
| 空（无）| `storyboard.composedImage` | 镜头封面图 |
| `grid_{mode}_{R}x{C}` | 仅 `imageGenerations` 记录 | 宫格拼图，不回写分镜 |

---

### 2.3 各 Provider 请求详情

#### MiniMax（异步为主）

- **端点**：`POST /v1/image_generation`
- **轮询**：`GET /v1/image_generation/task/{taskId}`
- **认证**：`Authorization: Bearer {apiKey}`

请求 Body：
```json
{
  "model": "model-name",
  "prompt": "your prompt here",
  "size": "1920x1080",
  "aspect_ratio": "1920/1080",
  "n": 1,
  "image": ["data:image/jpeg;base64,..."]
}
```

响应解析：
- 有 `task_id` / `id` → 异步，发起轮询
- 有 `data[0].url` / `url` → 同步，直接下载

---

#### OpenAI / chatfire（同步）

- **端点**：`POST /v1/images/generations`
- **认证**：`Authorization: Bearer {apiKey}`

请求 Body：
```json
{
  "model": "dall-e-3",
  "prompt": "your prompt here",
  "size": "1024x1024",
  "n": 1,
  "response_format": "url"
}
```

响应解析：
- `data[0].url` → 直接下载 URL
- `data[0].b64_json` → base64 图片数据（直接保存）
- 有 `task_id` / `id` → 异步轮询

---

#### Gemini（同步，返回 base64）

- **端点**：`POST /v1beta/models/{model}:generateContent?key={apiKey}`
- **认证**：URL 参数 `?key=` + Header `x-goog-api-key` + `Authorization: Bearer`
- **模型名称格式**：`models/gemini-2.5-flash-image`（自动加 `models/` 前缀）

请求 Body：
```json
{
  "contents": [{
    "parts": [
      {
        "inline_data": {
          "mime_type": "image/jpeg",
          "data": "base64EncodedReferenceImage"
        }
      },
      { "text": "your prompt here" }
    ]
  }],
  "generationConfig": {
    "responseModalities": ["IMAGE", "TEXT"],
    "imageConfig": {
      "aspectRatio": "16:9",
      "imageSize": "2K"
    }
  }
}
```

尺寸映射（`size` 宽度 → `imageSize`）：

| size 宽度 | imageSize |
|----------|-----------|
| ≥ 2048px | `4K` |
| ≥ 1024px | `2K` |
| ≥ 512px  | `1K` |
| < 512px  | `512` |

响应解析：
- `candidates[0].content.parts[].inlineData.data` → base64 数据，直接保存
- `finishReason` 不是 `STOP` / `MAX_TOKENS` → 抛出异常

---

#### 火山引擎 / VolcEngine（异步为主）

- **端点**：`POST /api/v3/images/generations`
- **轮询**：`GET /api/v3/images/generations/{taskId}`
- **认证**：`Authorization: Bearer {apiKey}`
- **默认模型**：`doubao-seedream-5-0-lite`

请求 Body：
```json
{
  "model": "doubao-seedream-5-0-lite",
  "prompt": "your prompt here",
  "width": 1920,
  "height": 1080
}
```

**注意**：不支持参考图。尺寸由 `size` 字段（如 `1920x1080`）拆分为 `width` / `height`。

轮询状态：`status === 'succeeded'` → 完成，`data[0].url` 取图片 URL。

---

#### 阿里云百炼 / Ali（强制异步）

- **端点**：`POST /api/v1/services/aigc/image-generation/generation`
- **轮询**：`GET /api/v1/tasks/{taskId}`
- **认证**：`Authorization: Bearer {apiKey}`
- **必须附加 Header**：`X-DashScope-Async: enable`
- **默认模型**：`wan2.6-t2i`

请求 Body：
```json
{
  "model": "wan2.6-t2i",
  "input": {
    "messages": [{
      "role": "user",
      "content": [{ "text": "your prompt here" }]
    }]
  },
  "parameters": {
    "size": "1696*960",
    "n": 1,
    "negative_prompt": "",
    "prompt_extend": true,
    "watermark": false,
    "seed": 1234567890
  }
}
```

尺寸映射（`size` → 阿里格式）：

| 宽高比 | 阿里尺寸 |
|--------|---------|
| 宽高比 > 1.7（横屏） | `1696*960`（16:9） |
| 宽高比 < 0.8（竖屏） | `960*1696`（9:16） |
| 其他 | `1280*1280`（1:1） |

轮询状态：`PENDING` / `RUNNING` → 处理中，`SUCCEEDED` → 完成，`FAILED` → 失败。

---

### 2.4 异步 vs 同步处理流程

```
同步模式（OpenAI / Gemini）：
  发请求 → 响应含图片 URL 或 base64
  → 下载/保存到本地
  → 更新数据库 status=completed

异步模式（MiniMax / 火山引擎 / 阿里云）：
  发请求 → 响应含 taskId
  → 更新 status=processing，taskId 入库
  → 每 5 秒轮询一次，最多轮询 120 次（10 分钟超时）
  → 轮询结果 completed → 下载图片
  → 轮询结果 failed → 更新 errorMsg
```

---

## 3. 宫格图生图

宫格图是一次生成 R×C 格的大图，再切割分配给各分镜的首帧 / 尾帧 / 参考图。

### 3.1 模式说明

| 模式 | 描述 | 每格含义 | 切割后 frame_type |
|------|------|---------|-----------------|
| `first_frame` | 每格 = 一个分镜首帧 | 各分镜开场画面 | `first_frame` |
| `first_last` | 成对首尾帧，奇偶交替 | 偶数格=首帧，奇数格=尾帧 | `first_frame` / `last_frame` |
| `multi_ref` | 同一场景多角度 | 同一分镜的不同角度 | `reference` |

---

### 3.2 完整 Prompt 模板

#### 宫格生图前：AI Agent 优先生成（grid_prompt_generator）

如果有 `grid_prompt_generator` Agent 配置，优先调用 Agent，用户 Prompt 如下：

```
请为宫格图生成提示词，并优先调用工具完成。
选中镜头ID：[1, 2, 3, ...]
行数：{rows}
列数：{cols}
模式：{mode}
参考图映射：图片1=镜头1首帧；图片2=角色A...
当提示词涉及到某个角色或场景时，直接把对应的图片编号写进提示词，例如：图片1中的角色A站了起来，图片3中的房间场景。不要只写名字，不写图片编号。
必须严格按 {rows}x{cols} 生成，总共 exactly {rows*cols} visible panels。不要合并格子，不要缺格。
必须返回 JSON，结构为：{"grid_prompt":"...","cell_prompts":[{"shot_number":1,"frame_type":"first_frame","prompt":"..."}]}
```

---

#### `first_frame` 模式 Fallback Prompt

```
{rows}x{cols} grid layout, consistent art style, {dramaStyle},
参考图映射：图片1=镜头1首帧；图片2=角色A；...
当画面涉及角色或场景时，优先使用对应的图片编号来约束一致性。
格1（row 1 col 1）: 参考图片1（镜头1首帧）、图片2（角色A），{imagePrompt/description}
格2（row 1 col 2）: 参考图片3（场景XX），{imagePrompt/description}
...
high quality, cinematic lighting, no text, no watermark
```

每格 cell_prompt：
```
格{N}（row R col C）：[参考图片X（标签）、...]，{imagePrompt}[, {location}][, {shotType}], opening scene
```

---

#### `first_last` 模式 Fallback Prompt

```
{rows}x{cols} grid layout, consistent art style, {dramaStyle},
参考图映射：...
first/last frame visual rhythm, alternating opening and closing beats across the grid,
格1（row 1 col 1）: 参考...，{desc}, opening moment
格2（row 1 col 2）: 参考...，{desc}, {action}, closing moment, subtle motion change
格3（row 2 col 1）: 参考...，{desc}, opening moment
格4（row 2 col 2）: 参考...，{desc}, {action}, closing moment, subtle motion change
...
continuous motion implied between left and right, high quality, no text
```

规律：偶数索引（0,2,4...）格 = `opening moment`（首帧），奇数索引（1,3,5...）格 = `{action}, closing moment, subtle motion change`（尾帧）。

---

#### `multi_ref` 模式 Fallback Prompt

```
{rows}x{cols} grid layout, same scene different angles and compositions, {dramaStyle},
参考图映射：...
main scene: {第一个分镜的 imagePrompt/description},
格1（row 1 col 1）: 参考图片1=...，{desc}, wide establishing shot
格2（row 1 col 2）: 参考图片1=...，{desc}, medium shot character focus
格3（row 1 col 3）: 参考图片1=...，{desc}, close-up detail
...（循环 25 种角度词）
consistent lighting and color palette, high quality, no text
```

25 种预设角度词（循环使用）：
```
wide establishing shot, medium shot character focus, close-up detail,
dramatic low angle, over-the-shoulder view, bird eye view, side profile,
atmospheric detail, extreme close-up, dutch angle, silhouette shot,
depth of field focus, symmetrical composition, leading lines,
negative space, high angle looking down, ground level, panoramic wide,
intimate two-shot, reflection shot, shadow play, backlit silhouette,
macro detail, split lighting, rim light portrait
```

---

### 3.3 参考图收集机制

参考图最多收集 6 张，优先级依次为（先到先得，自动去重）：

1. 分镜已有的 `firstFrameImage`（镜头N首帧）
2. 分镜已有的 `lastFrameImage`（镜头N尾帧）
3. 分镜已有的 `composedImage`（镜头N镜头图）
4. 分镜的 `referenceImages`（镜头N参考图）
5. 分镜关联场景的 `scene.imageUrl`（场景图）
6. 分镜关联角色的 `character.imageUrl`（角色图）

收集完成后生成参考图映射：`图片1=镜头1首帧；图片2=角色A；图片3=XX场景...`，嵌入到 Prompt 中。

---

## 4. 生视频阶段

### 4.1 referenceMode 说明

**入口**：`POST /api/videos`

| referenceMode | 必填字段 | 描述 |
|--------------|---------|------|
| `none` | 仅 `prompt` | 纯文字生视频 |
| `single` | `prompt` + `image_url` | 单参考图，图生视频（通常传 firstFrameImage） |
| `first_last` | `prompt` + `first_frame_url` + `last_frame_url` | 首帧+尾帧双约束，控制镜头起止 |
| `multiple` | `prompt` + `reference_image_urls[]` | 多参考图，角色/场景一致性 |

---

### 4.2 各 Provider 请求详情

#### MiniMax（异步）

- **端点**：`POST /v1/video_generation`
- **轮询**：`GET /v1/video_generation/task/{taskId}`
- **认证**：`Authorization: Bearer {apiKey}`

Prompt 格式（自动附加参数）：
```
{原始videoPrompt}  --ratio 16:9  --dur 5
```

`none` 模式请求 Body：
```json
{
  "model": "model-name",
  "content": [
    { "type": "text", "text": "prompt --ratio 16:9 --dur 5" }
  ]
}
```

`single` 模式请求 Body：
```json
{
  "model": "model-name",
  "content": [
    { "type": "text", "text": "prompt --ratio 16:9 --dur 5" },
    { "type": "image_url", "image_url": { "url": "base64..." }, "role": "reference_image" }
  ]
}
```

`first_last` 模式请求 Body：
```json
{
  "model": "model-name",
  "content": [
    { "type": "text", "text": "prompt --ratio 16:9 --dur 5" },
    { "type": "image_url", "image_url": { "url": "firstFrameBase64..." }, "role": "first_frame" },
    { "type": "image_url", "image_url": { "url": "lastFrameBase64..." }, "role": "last_frame" }
  ]
}
```

`multiple` 模式请求 Body：
```json
{
  "model": "model-name",
  "content": [
    { "type": "text", "text": "prompt --ratio 16:9 --dur 5" },
    { "type": "image_url", "image_url": { "url": "ref1Base64..." }, "role": "reference_image" },
    { "type": "image_url", "image_url": { "url": "ref2Base64..." }, "role": "reference_image" }
  ]
}
```

---

#### 火山引擎 Seedance（异步）

- **端点**：`POST /api/v3/contents/generations/tasks`
- **轮询**：`GET /api/v3/contents/generations/tasks/{taskId}`
- **认证**：`Authorization: Bearer {apiKey}`
- **默认模型**：`doubao-seedance-1-5-pro-251215`
- **时长限制**：4~12 秒

`none` / `single` / `first_last` 请求 Body：
```json
{
  "model": "doubao-seedance-1-5-pro-251215",
  "content": [
    { "type": "text", "text": "prompt" },
    { "type": "image_url", "image_url": { "url": "base64..." } },
    { "type": "image_url", "image_url": { "url": "..." }, "role": "first_frame" },
    { "type": "image_url", "image_url": { "url": "..." }, "role": "last_frame" }
  ],
  "generate_audio": true,
  "ratio": "adaptive",
  "duration": 5,
  "watermark": false
}
```

`multiple` 模式（无 `role` 字段）：
```json
{
  "model": "doubao-seedance-1-5-pro-251215",
  "content": [
    { "type": "text", "text": "prompt" },
    { "type": "image_url", "image_url": { "url": "ref1..." } },
    { "type": "image_url", "image_url": { "url": "ref2..." } }
  ],
  "generate_audio": true,
  "ratio": "adaptive",
  "duration": 5,
  "watermark": false
}
```

轮询状态：`succeeded` → 完成（`video_url` / `content.video_url` / `data.video_url`），`failed` → 失败。

---

#### Vidu（异步，Webhook 回调）

- **端点**：`POST /ent/v2/img2video`
- **认证**：`Authorization: Token {apiKey}`（**非** Bearer！）
- **默认模型**：`viduq3-turbo`
- **特殊**：**无轮询接口，通过 Webhook 回调** `POST /api/v1/webhooks/vidu` 更新状态

请求 Body：
```json
{
  "model": "viduq3-turbo",
  "prompt": "prompt",
  "images": ["imageUrl1", "imageUrl2"],
  "duration": 5,
  "resolution": "720p"
}
```

Webhook 回调解析：
- `state === 'success'` → 读取 `video_url`，更新完成
- `state === 'failed'` → 读取 `error`，更新失败

---

#### 阿里云百炼 / Ali（异步）

- **端点**：`POST /api/v1/services/aigc/video-generation/video-synthesis`
- **轮询**：`GET /api/v1/tasks/{taskId}`
- **认证**：`Authorization: Bearer {apiKey}`
- **默认模型**：`wan2.6-i2v-flash`
- **限制**：不支持 `multiple` 多参考图模式

请求 Body（`none` / `single` 模式）：
```json
{
  "model": "wan2.6-i2v-flash",
  "input": {
    "prompt": "prompt",
    "img_url": "imageUrl 或 firstFrameUrl"
  },
  "parameters": {
    "resolution": "1080P",
    "duration": 5,
    "watermark": false,
    "seed": 1234567890
  }
}
```

`first_last` 模式追加 `last_img_url`：
```json
{
  "model": "wan2.6-i2v-flash",
  "input": {
    "prompt": "prompt",
    "img_url": "firstFrameUrl",
    "last_img_url": "lastFrameUrl"
  },
  "parameters": { "resolution": "1080P", "duration": 5, "watermark": false }
}
```

分辨率映射：

| aspectRatio | 阿里 resolution |
|------------|----------------|
| `9:16` | `720P` |
| `1:1` | `720P` |
| `16:9`（默认）| `1080P` |

---

### 4.3 轮询 vs Webhook

| Provider | 方式 | 轮询间隔 | 最大次数 |
|----------|------|---------|---------|
| MiniMax | 轮询 | 10 秒 | 300 次（50 分钟） |
| 火山引擎 | 轮询 | 10 秒 | 300 次 |
| 阿里云 | 轮询 | 10 秒 | 300 次 |
| Vidu | **Webhook 回调** | N/A | N/A |

---

## 5. 图片标准化处理

在生视频时，本地图片（`static/` 路径）会经过压缩处理后再以 base64 传给 API：

```
最大宽度：768px
最大高度：768px
JPEG 压缩质量：68%
```

处理逻辑：

| 输入格式 | 处理方式 |
|---------|---------|
| `data:image/...` 前缀 | 已是 base64，直接使用 |
| `static/` 或 `/static/` 开头 | 读取本地文件，压缩为 base64 |
| 其他（HTTP URL）| 直接传原始 URL |

生图时的参考图同样走此流程（`normalizeReferenceImages`），最多保留 6 张，自动去重。

---

## 6. 合成阶段

**接口**：`POST /api/compose/storyboards/:id/compose`  
**批量接口**：`POST /api/compose/episodes/:id/compose-all`

### 步骤 1：解析对白，生成 TTS

对白格式：`角色名：台词内容（括号说明）`

自动过滤无需配音的情况：

| 过滤条件 | 示例 |
|---------|------|
| 角色名为环境音类 | `环境音:`, `BGM:`, `音效:`, `SFX:` |
| 台词内容为空/无意义 | `无`, `无对白`, `null`, `N/A` |

未过滤时，查找角色的 `voiceStyle`（默认 `alloy`），调用 MiniMax TTS：

- **端点**：`POST /v1/t2a_v2`
- **模型**：`speech-2.8-hd`

TTS 请求 Body：
```json
{
  "model": "speech-2.8-hd",
  "text": "纯净台词文本",
  "stream": false,
  "voice_setting": {
    "voice_id": "角色voiceStyle",
    "speed": 1,
    "vol": 1,
    "pitch": 0,
    "emotion": "happy"
  },
  "audio_setting": {
    "sample_rate": 32000,
    "bitrate": 128000,
    "format": "mp3",
    "channel": 1
  },
  "subtitle_enable": false
}
```

响应：hex 编码 mp3 音频，保存到 `static/tts/`。

---

### 步骤 2：生成 SRT 字幕

```
1
00:00:00,500 --> 00:00:{duration-1},000
{纯净台词文本}
```

字幕文件保存到 `static/subtitles/{uuid}.srt`。

---

### 步骤 3：FFmpeg 合成

```
输入：
  - data/static/videos/{视频文件}
  - data/static/tts/{音频文件}（若有对白）

字幕滤镜（若系统 ffmpeg 支持）：
  subtitles=filename='...',force_style='FontSize=20,PrimaryColour=&HFFFFFF&,OutlineColour=&H000000&,Outline=2'

编码参数（有音频）：
  -c:v libx264 -preset fast -crf 23
  -map 0:v -map 1:a -c:a aac -shortest

编码参数（无音频）：
  -c:v libx264 -preset fast -crf 23 -an

输出：data/static/composed/{uuid}.mp4
```

---

## 7. 全集拼接

**接口**：`POST /api/merge/episodes/:id`

将一集中所有 `composedVideoUrl` 非空的分镜视频按 `storyboardNumber` 顺序拼接为完整一集。

FFmpeg concat 参数：
```
-f concat -safe 0
-fflags +genpts
-c:v libx264 -preset medium -crf 23
-c:a aac -ar 48000 -b:a 192k
-movflags +faststart
```

输出：`data/static/merged/{uuid}.mp4`，回写到 `episode.videoUrl`，并记录时长到 `videoMerges.duration`。

---

## 8. 完整状态流转

### 图片（imageGenerations.status）

```
processing → completed
          → failed
```

### 视频（videoGenerations.status）

```
processing → completed
          → failed
```

### 分镜合成（storyboards.status）

```
（任意）→ compose_processing → compose_completed
                             → compose_failed
```

### 全集合并（videoMerges.status）

```
processing → completed
          → failed
```

---

## 9. Provider 能力对比

### 图片生成

| 能力 | MiniMax | OpenAI | Gemini | 火山引擎 | 阿里云 |
|------|:-------:|:------:|:------:|:-------:|:-----:|
| 参考图支持 | ✅ | ❌ | ✅（base64）| ❌ | ❌ |
| 同步返回 | ✅ | ✅ | ✅ | ✅ | ❌（强制异步）|
| 返回 base64 | ❌ | ✅（可选）| ✅ | ❌ | ❌ |

### 视频生成

| 能力 | MiniMax | 火山引擎 | Vidu | 阿里云 |
|------|:-------:|:-------:|:----:|:-----:|
| 纯文字（none）| ✅ | ✅ | ✅ | ✅（需img_url）|
| 单参考（single）| ✅ | ✅ | ✅ | ✅ |
| 首尾帧（first_last）| ✅ | ✅ | ✅ | ✅ |
| 多参考（multiple）| ✅ | ✅ | ✅ | ❌ |
| 生成音频 | ❌ | ✅（generate_audio）| ❌ | ❌ |
| Webhook 回调 | ❌ | ❌ | ✅（必须）| ❌ |
