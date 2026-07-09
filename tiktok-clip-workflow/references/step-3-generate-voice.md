# Step 3: 生成配音

## 目的

用 TTS 生成每个镜头的配音，同时**按配音实际时长反向调整视频节奏**，保证画面和配音对齐。

## 输入 / 输出

**输入**：
- `work/<日期>/recipe.json` — 里面每镜的 `text` 字段（用户可先修正 ASR/OCR 误差）
- （可选）`work/<日期>/text.txt` — 用户手写的文案（覆盖 recipe.json 的 text）

**输出**：
- `work/<日期>/voice/voice_001.mp3`, `voice_002.mp3`, ... — 每镜配音
- `work/<日期>/voice/timing.json` — 每条配音的实际时长
- **更新** `work/<日期>/matches.json` — 按配音时长调整每镜的 `duration`

## 配音方式二选一

### 方式 A：豆包语音大模型（推荐电商带货）

**优势**：稳定 / 跟剪映音色同源 / 1 分钟约 4-8 毛

**申请**：字节火山引擎 → 豆包语音大模型 → 拿 API Key

**调用示例**：

```python
import requests
import json

url = "https://openspeech.bytedance.com/api/v1/tts"
headers = {
    "Authorization": f"Bearer;{TOKEN}",
    "Content-Type": "application/json"
}
payload = {
    "app": {"appid": APPID, "token": TOKEN, "cluster": "volcano_tts"},
    "user": {"uid": "user_id"},
    "audio": {
        "voice_type": "BV700_streaming",  # 剪映常用音色
        "encoding": "mp3",
        "speed_ratio": 1.0,
        "rate": 24000
    },
    "request": {
        "reqid": "req_001",
        "text": text,
        "operation": "query"
    }
}
resp = requests.post(url, headers=headers, json=payload)
audio_data = resp.json()["data"]
# base64 解码写入 voice_XXX.mp3
```

**常用音色**：
- `BV700_streaming` — 灿灿（女声，甜美，带货首选）
- `BV705_streaming` — 阳光青年（男声，讲解）
- `BV064_streaming` — 广告男声
- `BV407_streaming` — 商务女声

### 方式 B：VoxCPM（免费开源，声音克隆）

**优势**：完全免费 / 支持声音克隆 / 本地跑

**装法**：
```bash
git clone https://github.com/OpenBMB/VoxCPM
cd VoxCPM
pip install -r requirements.txt
```

**克隆自己声音**（先录一段 30 秒的清晰音频）：
```bash
python inference.py \
  --reference_audio my_voice_sample.wav \
  --text "要合成的文本" \
  --output voice/voice_001.wav
```

## 节奏对齐规则

生成完每条配音后拿实际时长，更新 `matches.json`：

```python
# 伪代码
for shot_id, config in matches.items():
    voice_file = f"voice/voice_{shot_id[-3:]}.mp3"
    actual_voice_duration = get_audio_duration(voice_file)
    original_duration = config["duration"]
    
    if config.get("text_content"):  # 有对话的镜头
        if actual_voice_duration > original_duration:
            # 配音比原镜头长：拉长镜头（放慢或延长）
            config["duration"] = actual_voice_duration
            config["speed_adjust"] = original_duration / actual_voice_duration
        else:
            # 配音比原镜头短：缩短镜头（保留 0.3s 自然结尾）
            config["duration"] = actual_voice_duration + 0.3
    else:
        # 无对话镜头（纯 B-roll）：保留原时长
        pass
```

## 输出 voice/timing.json

```json
{
  "shot_001": {
    "voice_file": "voice/voice_001.mp3",
    "actual_duration": 3.5,
    "original_shot_duration": 3.2,
    "adjusted_shot_duration": 3.5,
    "speed_adjust": 0.91
  },
  "shot_002": {
    "voice_file": null,
    "actual_duration": 0,
    "original_shot_duration": 2.5,
    "adjusted_shot_duration": 2.5,
    "note": "无对话镜头，保留原时长"
  }
}
```

## Review Checkpoint（⏸ Gate）

配音生成完，让用户**听一遍**：

1. 逐个播放 `voice/voice_XXX.mp3`
2. 检查：
   - 语音自然度（有无爆音、跳字、错字）
   - 语速（太快 / 太慢）
   - 音色（选的是否合适）
3. 有问题的镜头告诉 Skill：
   - "第 3 条配音语速太快，慢一点"
   - "第 5 条改成阳光青年音色"
   - "第 7 条读错字了，'吹风机' 读成了 '吹风鸡'"

Skill 用新参数重新生成对应条目。

## Common Issues

- **配音跟画面完全对不上**：Step 3 的节奏对齐没做，检查 `matches.json` 的 `duration` 是否更新
- **爆音 / 噪声**：豆包 API 参数 `rate` 改成 24000 或 44100；VoxCPM 换更清晰的参考音频
- **读错专业词**：在文案里加拼音标注，如"吹风机(chuī fēng jī)"
- **配音音量差异大**：跑一次 loudness normalization：`ffmpeg -i in.mp3 -af loudnorm out.mp3`
