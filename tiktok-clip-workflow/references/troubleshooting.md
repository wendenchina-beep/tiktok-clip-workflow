# Troubleshooting 常见问题排查

## 问题分类

### 1. 剪映草稿相关

#### 1.1 剪映打开报"草稿损坏 / 无法读取"

**根因**：`draft_content.json` 或 `draft_meta_info.json` 里的三个关键 ID 不一致

**排查**：
```bash
# 打开草稿目录
cd ~/Movies/JianyingPro/User\ Data/Projects/com.lveditor.draft/<草稿名>/

# 拿三个 ID 对比
grep '"id"' draft_content.json | head -3
grep '"id"' draft_meta_info.json | head -3
```

**修复**：手动改成相同 UUID，或者让 Skill 重跑 Step 4

#### 1.2 素材显示红色叉号（找不到素材）

**根因 A**：路径失效（用户后来移动了素材）  
**根因 B**：草稿里写的是相对路径

**排查**：
```bash
# 检查草稿里引用的素材路径是否存在
python3 -c "
import json
with open('draft_content.json') as f:
    data = json.load(f)
for v in data['materials']['videos']:
    import os
    exists = os.path.exists(v['path'])
    print(('✅' if exists else '❌'), v['path'])
"
```

**修复**：
- 缺文件 → 重新复制素材到原位置，或改草稿里的路径
- 相对路径 → 让 Skill 重跑 Step 4 强制用绝对路径

#### 1.3 打开草稿后发现覆盖了之前的草稿

**根因**：新草稿 ID 跟已有草稿冲突

**排查**：
```bash
# 检查 draft_root 下所有草稿的 ID
for d in ~/Movies/JianyingPro/User\ Data/Projects/com.lveditor.draft/*/; do
    id=$(grep -o '"id":"[^"]*"' "$d/draft_meta_info.json" 2>/dev/null | head -1)
    echo "$id -> $d"
done | sort
```

**预防**：Step 4.1 的 UUID 冲突检查必须做，代码见 `step-4-jianying-draft.md`

**恢复丢的草稿**：剪映会自动备份到 `~/Movies/JianyingPro/User Data/Backup/`

#### 1.4 时间轴对不上（配音跟画面差几秒）

**根因**：时间单位混淆——剪映用**微秒**（μs），不是毫秒（ms）

**排查**：
```python
# 打开 draft_content.json 看某个 segment 的 duration
# 3 秒应该是 3000000 不是 3000
```

**修复**：Step 4 生成的所有时长值都要 × 1000

### 2. 素材匹配相关

#### 2.1 匹配结果全部低置信度（< 0.60）

**根因 A**：素材库分类不够细  
**根因 B**：素材库覆盖的场景不够全  
**根因 C**：参考视频跟素材库完全不是一个赛道（如参考是户外风景，素材库全是室内产品）

**排查**：
```bash
# 看素材库结构
tree assets/ -L 2 | head -30
# 看总素材数
find assets/ -type f \( -name "*.mp4" -o -name "*.mov" -o -name "*.jpg" \) | wc -l
```

**修复**：
- 素材数 < 30 → 补素材（每个类别至少 5-10 个）
- 分类太粗 → 按 SKILL.md 建议的子目录重新分类
- 参考视频不匹配你的素材赛道 → 换个更相关的参考视频

#### 2.2 相邻镜头看起来太相似（明显机器感）

**根因**：多样性铁律没生效

**排查**：
```bash
# 看 matches.json 里连续镜头的 matched_asset
python3 -c "
import json
with open('matches.json') as f:
    data = json.load(f)
last_asset = None
for shot_id in sorted(data.keys()):
    asset = data[shot_id].get('matched_asset', 'MISSING')
    marker = '⚠️ 连续' if asset == last_asset else ''
    print(shot_id, asset, marker)
    last_asset = asset
"
```

**修复**：让 Skill 强制应用多样性铁律，重新跑 Step 2

#### 2.3 明明有很合适的素材，Skill 没匹配到

**根因**：视觉匹配算法权重不适合当前场景

**排查**：手动选一个"应该匹配到但没匹到"的素材，让 Skill 打分对比

**修复**：改 `references/step-2-match-assets.md` 里的权重公式：
- 参考视频强调**色彩** → 颜色权重从 40% 提到 50%
- 参考视频强调**构图** → 构图权重从 20% 提到 30%

### 3. 配音相关

#### 3.1 配音跟画面完全对不上

**根因**：Step 3 的节奏对齐没跑，`matches.json` 的 `duration` 没按配音时长更新

**排查**：
```bash
# 对比配音时长跟视频镜头时长
python3 -c "
import json
with open('voice/timing.json') as f:
    voice = json.load(f)
with open('matches.json') as f:
    match = json.load(f)
for sid in sorted(voice.keys()):
    v = voice[sid]['actual_duration']
    m = match[sid]['duration']
    print(sid, f'voice={v}s', f'shot={m}s', '❌ 差距' if abs(v-m) > 0.5 else '✅')
"
```

**修复**：让 Skill 重跑 Step 3 的 4.5 节（节奏对齐更新 matches.json）

#### 3.2 配音有爆音 / 噪声

**根因**：TTS 参数不合适

**豆包**：
- 调 `rate` 到 24000 或 44100（默认 16000 音质差）
- 调 `speed_ratio` 到 0.9-1.1 之间

**VoxCPM**：
- 参考音频质量差 → 换清晰的音频重新克隆
- 音频跑一次 loudness 归一化：`ffmpeg -i in.mp3 -af loudnorm out.mp3`

#### 3.3 读错专业词（品牌名、产品名读错）

**修复**：在文案里加拼音标注：
```
错：这个吹风机
对：这个吹风机(chuī fēng jī)
```

或者用**多音字词典**替换（如 "SK-II" 读成 "SK 二" → 写成 "SK 二"）

### 4. 拆解参考视频相关

#### 4.1 拆出来的镜头都是零点几秒（切太碎）

**根因**：FFmpeg 场景检测阈值太低

**修复**：把 `references/step-1-decompose.md` 里的阈值从 `0.3` 提到 `0.4`

#### 4.2 明显应该切开的镜头没切开

**根因**：阈值太高，或者视频本身镜头切换很软（渐变过渡）

**修复**：
- 阈值降到 `0.2-0.25`
- 或者手动指定切点告诉 Skill："第 12 秒到 15 秒之间切开"

#### 4.3 ASR 结果乱码

**根因 A**：视频音频编码不兼容  
**根因 B**：视频没有音频（纯 BGM 无对白）

**排查**：
```bash
ffprobe -v error -show_streams reference.mp4 | grep codec_name
# 期望看到 aac 或 mp3
```

**修复**：
- 音频编码不对 → 转码：`ffmpeg -i in.mp4 -c:v copy -c:a aac out.mp4`
- 视频无对白 → 用 OCR 从字幕提取文字，或者用户手写文案

### 5. 环境相关

#### 5.1 `ffmpeg: command not found`

装 FFmpeg（见 `references/environment-setup.md`）

#### 5.2 `ModuleNotFoundError: No module named 'cv2'`

```bash
pip3 install opencv-python
```

#### 5.3 剪映草稿目录不存在

**macOS**：先打开一次剪映（会自动创建目录）  
**Windows**：同上

## 通用排查思路

遇到没在这上面的问题时：

1. **看错误信息**：Skill 会把错误信息完整贴出来，别忽略
2. **回到中间产物**：每一步都有 JSON 输出，打开看数据是否合理
3. **单步重跑**：不要重跑整个流水线，只重跑出错的那一步
4. **保留原始文件**：所有操作用 `cp`，不用 `mv`，出问题可以回滚
5. **实在不行**：把 `work/<日期>/` 整个目录发给 Skill，让它自查
