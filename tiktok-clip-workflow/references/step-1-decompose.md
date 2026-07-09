# Step 1: 拆解参考视频

## 目的

把一段完整的参考视频拆成一个个独立镜头，每镜提取关键帧，输出 `recipe.json` 作为后续所有步骤的**单一真相源**。

## 输入 / 输出

**输入**：
- `work/<日期>/reference.mp4` — 参考视频文件

**输出**：
- `work/<日期>/shots/shot_001.mp4`, `shot_002.mp4`, ... — 拆好的镜头片段
- `work/<日期>/keyframes/shot_001_key_0.jpg`, `..._key_1.jpg`, `..._key_2.jpg` — 每镜 3 张关键帧
- `work/<日期>/recipe.json` — 视频结构分析

## 执行步骤

### 1.1 场景检测拆分

用 FFmpeg 的 scene detection filter：

```bash
ffmpeg -i reference.mp4 -filter:v "select='gt(scene,0.3)',showinfo" \
  -vsync vfr keyframes/scene_%03d.jpg 2>&1 | grep "pts_time"
```

- 阈值 `0.3` 起（对话类视频；纪录片可降到 `0.2`；快剪类升到 `0.4`）
- 如果切太碎（大量 0.5s 以下镜头）→ 提高阈值到 0.4
- 如果切不开（明显应该切的地方没切）→ 降低阈值到 0.25

### 1.2 按检测点切分成小片段

```bash
# 用 ffprobe 拿到每个场景的起止时间
ffprobe -v error -select_streams v:0 \
  -show_entries frame=pkt_pts_time -of csv reference.mp4

# 用 ffmpeg 按时间点切分
ffmpeg -i reference.mp4 -ss <start> -to <end> -c copy shots/shot_001.mp4
```

### 1.3 每镜提取 3 张关键帧

```bash
# 开头 / 中间 / 结尾各一张
ffmpeg -i shots/shot_001.mp4 -vf "select='eq(n,0)+eq(n,middle)+eq(n,last)'" \
  -vsync vfr keyframes/shot_001_key_%d.jpg
```

关键帧用于 Step 2 的视觉匹配。

### 1.4 提取文案

**优先顺序**：先 ASR（音频转文字）→ 失败再 OCR（字幕识别）

**ASR 方案**：
```bash
# 用 Whisper（本地）
whisper reference.mp4 --language zh --output_format json

# 或用豆包 ASR API
```

**OCR 方案**（ASR 拿不到文字时降级用）：
```bash
# 对每张关键帧跑 OCR
python3 -c "import easyocr; reader = easyocr.Reader(['ch_sim']); print(reader.readtext('keyframes/shot_001_key_1.jpg'))"
```

**每镜挂对应文案**：把 ASR/OCR 结果按时间戳匹配到对应镜头。

### 1.5 输出 recipe.json

结构：

```json
{
  "reference_video": "work/2026-07-09/reference.mp4",
  "total_duration": 62.5,
  "total_shots": 18,
  "shots": [
    {
      "id": "shot_001",
      "start": 0.0,
      "end": 3.2,
      "duration": 3.2,
      "file": "shots/shot_001.mp4",
      "keyframes": [
        "keyframes/shot_001_key_0.jpg",
        "keyframes/shot_001_key_1.jpg",
        "keyframes/shot_001_key_2.jpg"
      ],
      "text": "这个吹风机才 199 你敢信？",
      "scene_type": "product_closeup",
      "transition_from_prev": "hard_cut"
    },
    {
      "id": "shot_002",
      "...": "..."
    }
  ]
}
```

**scene_type 分类**（Step 2 匹配用）：
- `product_closeup` — 产品特写
- `product_use` — 产品使用场景
- `person_talk` — 人物讲解
- `demo` — 功能演示
- `lifestyle` — 生活场景
- `text_only` — 纯文字画面
- `unknown` — 无法识别（要人工标）

## Review Checkpoint（⏸ Gate）

拆完后**必须**让用户 review：

1. 打开 `shots/` 目录，检查拆分是否合理
2. 打开 `recipe.json`，检查每镜的 `text` 字段是否准确
3. 有问题的镜头告诉 Skill：
   - "第 3 镜切太早了，往后挪 0.5 秒"
   - "第 5 镜的文案错了，应该是 XXX"
   - "第 7 和第 8 镜应该合成一个"

Skill 根据用户反馈更新 `shots/` 和 `recipe.json`，再次让用户 review。

**禁止**：跳过 Review 直接进 Step 2 —— 后续所有步骤都依赖 recipe.json，源头错了下游全错。

## Common Issues

- **场景切太碎**：阈值降到 0.2-0.25 时特别容易；改回 0.3-0.35
- **相似镜头没切开**：阈值升到 0.4-0.5，或者手动指定切点
- **ASR 结果乱码**：检查视频音频编码是否为 AAC/MP3；先转码 `ffmpeg -i in.mp4 -c:a aac out.mp4`
- **OCR 字幕识别不全**：字幕颜色跟背景反差小时会漏，可以先做二值化预处理
