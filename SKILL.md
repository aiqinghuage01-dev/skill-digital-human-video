---
name: 数字人剪辑成片
description: 用户明确提到“数字人”并且还要“剪辑/成片/字幕/BGM/卡片叠加”时使用。本技能先生成数字人口播视频，再用 remotion-poster 的后处理链路生成最终成品视频。声音默认来自数字人本身，禁止再重复走 MiniMax 克隆声音。
---

# 数字人剪辑成片

## 执行回执（硬规则）

只要本轮因为 `skill` / `技能` 指名，或因为关键词命中而进入本技能，第一行先写：
`已走技能：数字人成片`

## 适用场景

当用户要做这些事时，使用本技能：
- 生成数字人口播视频 **并且** 剪辑成完整成品
- 把文案做成数字人视频然后自动剪辑
- 一键出片（数字人 + 剪辑）
- 给数字人视频加字幕、背景音乐、片头片尾
- 数字人口播 + 卡片字幕/卡片叠加/后处理

**不适用**：
- 只生成数字人、不需要剪辑 → 用 `shiliu-digital-human` 技能
- 只写文案、不涉及视频 → 用文案类技能
- 没有提 `数字人`，只是要把文案做成视频 → 用 `remotion-poster`

## 硬规则

- 只要当前请求明确提到 `数字人`，本技能负责最终成片
- 本技能**禁止**再额外调用 MiniMax 克隆声音；数字人人声以数字人视频自带声音为准
- 如果请求只是“生成数字人口播视频”，不要走完整成片链，改用 `shiliu-digital-human`
- 如果请求是“先写文案，再做数字人成片”，先产出文案，再继续本技能

## 运行前提

与 `shiliu-digital-human` 共享基础设施：
- 本地 runtime：`/Users/black.chen/.openclaw/workspace/shared/shiliu-mcp`
- Python 3.12
- 环境变量：`SHILIU_API_KEY`
- 已配置好的默认 `avatar_id` / `speaker_id`
- FFmpeg（用于剪辑合成）

数字人 skill 的配置和脚本路径：
- 配置文件：`../shiliu-digital-human/data/runtime/config.json`
- 生成脚本：`../shiliu-digital-human/scripts/generate_video.sh`

## Workflow

### Step 1: 获取文案

判断文案来源（优先级从高到低）：
1. 用户直接贴了文案文本 → 直接使用
2. 用户指定了批次和条号 → 调用 `../shiliu-digital-human/scripts/select_copy.py` 解析
3. 上一轮对话中刚生成的文案 → 确认后使用
4. 都没有 → 明确告诉用户"请先提供文案"

**禁止**自己编文案或从知识库里猜。

### Step 2: 生成数字人视频

调用数字人生成：

```bash
../shiliu-digital-human/scripts/generate_video.sh --text "文案内容" --wait
```

等待生成完成后，再进入剪辑。

必须先看 `../shiliu-digital-human/scripts/generate_video.sh` 的 JSON 结果里的 `diagnosis`：

- `outcome = ready`
  - 已拿到 `video_id` 且石榴已返回完成状态/视频链接
  - 才能继续剪辑

- `outcome = submitted_waiting`
  - 石榴已接单，但当前仍在生成
  - 不要报失败，也不要继续剪辑；应告诉用户“已建单，稍后继续”

- `outcome = create_response_failed_uncertain`
  - 创建响应失败，当前没拿到 `video_id`
  - 不要断言 VPN、代理或域名拦截；只能告诉用户“当前无法自动确认是否已建单”

### Step 3: 用原始文案校正并生成卡片脚本

调用 `remotion-poster` 的 pipeline，不要自己重新发明流程：

```bash
python3 ../remotion-poster/scripts/full-pipeline.py \
  <数字人视频路径或URL> \
  --name <project_name> \
  --shiliu \
  --text "原始文案"
```

要求：
1. `--text` 必须传原始文案，用来校正 Whisper 转写错误
2. 如果用户给的是文案技能刚生成的结果，直接使用那份最终文案，不要再靠 Whisper 反推正文
3. 数字人模式下，Whisper 只用于时间戳对齐，不用于决定最终文案文字

### Step 4: 渲染最终成品

```bash
node ../remotion-poster/scripts/render-card.mjs \
  ../remotion-poster/src/data/<project_name>-script.json \
  --output <project_name>
```

### Step 5: 返回结果

成功后返回给用户：
- 使用的文案内容（前 50 字）
- 数字人 `video_id`
- 最终成品视频路径
- 视频时长和分辨率

## 默认参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 输出分辨率 | 1080x1920 | 竖屏短视频 |
| BGM 音量 | -15dB | 不抢人声 |
| 字幕字体 | 思源黑体 | 兼容中文 |
| 输出格式 | MP4 (H.264) | 全平台兼容 |

## 失败保护

- 数字人生成只要不是 `ready`，就不要进入剪辑
- `submitted_waiting` 不是失败，而是“已建单等待中”
- `create_response_failed_uncertain` 也不要说成“石榴一定没生成”或“网络一定拦截”
- FFmpeg 不可用 → 提示用户安装：`brew install ffmpeg`
- 缺少文案 → 不生成、不猜测，明确要求用户提供

