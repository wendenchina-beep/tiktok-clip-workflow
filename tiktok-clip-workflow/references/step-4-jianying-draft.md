# Step 4: 生成剪映草稿

## 目的

把前 3 步的产物（匹配好的素材 + 配音 + 字幕）打包成一份**剪映可以直接打开**的草稿文件。

## 输入 / 输出

**输入**：
- `work/<日期>/matches.json` — 每镜用哪个素材（Step 2 输出，Step 3 更新过 duration）
- `work/<日期>/selected/` — 素材复制品
- `work/<日期>/voice/` — 配音文件
- `work/<日期>/voice/timing.json` — 每条配音时长

**输出**：
- `<剪映草稿目录>/<草稿名>/` — 剪映草稿目录（可以直接打开）
  - 结构：`draft_content.json` + `draft_meta_info.json` + 素材文件

## 剪映草稿目录在哪

**macOS**：`~/Movies/JianyingPro/User Data/Projects/com.lveditor.draft/`  
**Windows**：`C:\Users\<user>\AppData\Local\JianyingPro\User Data\Projects\com.lveditor.draft\`

Skill 生成完直接放到这个目录，剪映重启后就能看到。

## 剪映草稿铁律（不遵守就打不开）

### 铁律 1: 三个关键 ID 必须一致且不重复

```
content_id       = draft_content.json 里的 id 字段
metadata_id      = draft_meta_info.json 里的 id 字段  
root_index_id    = draft_content.json 里 tracks[0].id 字段
```

**规则**：
- 三个 ID 必须**完全一致**（都是同一个 UUID）
- 必须**不跟已有草稿冲突**（否则会覆盖之前的草稿）
- 生成方式：用 `uuid.uuid4()` 生成新的 UUID，然后遍历现有草稿目录检查冲突

### 铁律 2: 素材路径必须绝对路径

```json
{
  "video_source": {
    "path": "/Users/xxx/work/2026-07-09/selected/shot_001/product_A.mp4"
  }
}
```

- 🚫 不允许相对路径（剪映打开时找不到）
- 🚫 不允许移动素材文件（草稿引用绝对路径，移动即失效）
- ✅ 文件必须真实存在（先 `os.path.exists()` 检查）

### 铁律 3: 时间轴严格对齐（毫秒级）

三条轨道必须对齐：

- **视频轨** — 每镜按 `matches.json[shot_id].duration` 摆放
- **音频轨** — 每条配音起点 = 对应视频镜头起点，长度 = 配音实际时长
- **字幕轨** — 每条字幕起点/终点 = 对应配音起点/终点

**时间单位**：剪映用**微秒**（μs），不是毫秒。1 秒 = 1,000,000 μs

## 草稿结构

### draft_meta_info.json

```json
{
  "id": "<UUID>",
  "create_time": <timestamp_ms>,
  "duration": <total_duration_us>,
  "draft_name": "20260709_产品视频_01",
  "draft_root_path": "/Users/.../com.lveditor.draft",
  "materials": {
    "videos": [...],
    "audios": [...],
    "texts": [...]
  }
}
```

### draft_content.json

```json
{
  "id": "<UUID>",
  "canvas_config": {"width": 1080, "height": 1920, "ratio": "9:16"},
  "duration": <total_duration_us>,
  "tracks": [
    {
      "id": "<UUID>",
      "type": "video",
      "segments": [
        {
          "material_id": "<video_material_id>",
          "target_timerange": {"start": 0, "duration": 3500000},
          "source_timerange": {"start": 0, "duration": 3500000}
        }
      ]
    },
    {
      "id": "<UUID>",
      "type": "audio",
      "segments": [...]
    },
    {
      "id": "<UUID>",
      "type": "text",
      "segments": [...]
    }
  ],
  "materials": {
    "videos": [
      {
        "id": "<video_material_id>",
        "path": "/absolute/path/to/shot_001.mp4",
        "duration": 3500000
      }
    ],
    "audios": [...],
    "texts": [...]
  }
}
```

## 执行步骤

### 4.1 生成新 UUID + 冲突检查

```python
import uuid, os, json

def generate_unique_id(draft_root):
    while True:
        new_id = str(uuid.uuid4())
        # 检查所有已有草稿目录，避免冲突
        conflict = False
        for existing_draft in os.listdir(draft_root):
            meta_file = os.path.join(draft_root, existing_draft, "draft_meta_info.json")
            if os.path.exists(meta_file):
                with open(meta_file) as f:
                    if json.load(f).get("id") == new_id:
                        conflict = True
                        break
        if not conflict:
            return new_id
```

### 4.2 构造 draft_meta_info.json

按上面结构填字段，`materials.videos` 里放所有匹配到的素材（每个素材一个条目，含 id + 绝对路径 + 时长）。

### 4.3 构造 draft_content.json

- `tracks[0]`（视频轨）：每镜一个 segment，`target_timerange.start` 是累加时长
- `tracks[1]`（音频轨）：每条配音一个 segment，起点跟视频轨对齐
- `tracks[2]`（字幕轨）：每镜的文案，按配音时长摆放

### 4.4 落盘

```
<draft_root>/<草稿名>/
├── draft_content.json
├── draft_meta_info.json
```

素材文件**不需要**复制到草稿目录里（剪映走绝对路径引用）。

### 4.5 首次生成校验

```python
# 打开 draft_content.json 检查
# 1. 三个 ID 是否一致
# 2. 所有素材路径是否存在
# 3. 时间轴总时长 = 视频轨总时长 = 音频轨总时长（毫秒级偏差 < 100ms 可接受）
```

## Review Checkpoint（⏸ Gate）

生成完让用户手动打开剪映验证：

1. **打开剪映** → 首页应该能看到新草稿
2. **双击草稿** → 应该能正常打开，不报错
3. 检查：
   - [ ] 素材是不是都能正常显示（无红叉）
   - [ ] 时间轴每镜时长和顺序对不对
   - [ ] 字幕不遮关键内容
   - [ ] 配音和画面同步
4. 有问题：
   - 素材红叉 → Step 4.2 的路径写错了，重跑
   - 时间轴不对 → Step 3 的 timing.json 或 Step 4.3 累加错了
   - 字幕遮挡 → 调字幕轨的 `y` 坐标（默认底部 80% 位置）

## Common Failures

| 错误现象 | 根因 | 修复 |
| --- | --- | --- |
| 剪映打开报"草稿损坏" | 三个 ID 不一致 | 重新生成 UUID，三处同步 |
| 素材显示红色叉号 | 路径失效 / 相对路径 | 用绝对路径 + `os.path.exists()` 检查 |
| 打开后覆盖了之前的草稿 | ID 跟已有草稿冲突 | Step 4.1 的冲突检查必须做 |
| 时间轴对不上 | 视频/音频/字幕时长不一致 | Step 3 的节奏对齐没跑 |
| 打开后画面比例错 | `canvas_config` 没设对 | 抖音 9:16 → `1080x1920`；横版 16:9 → `1920x1080` |
| 字幕太靠上/下 | 文本 segment 的 y 坐标不对 | 底部字幕 y=0.85；顶部标题 y=0.15 |
| 配音跟画面差 1 秒 | 时间单位用错了 | 剪映用微秒（μs），不是毫秒（ms） |

## Tips

- **草稿名**建议：`YYYYMMDD_主题_序号`（如 `20260709_吹风机_01`）
- **首次生成慢**（10-15 分钟）是因为要拷贝所有素材元信息，后续同素材库复用会快
- **备份原始草稿目录**：Skill 第一次跑之前先 `cp -r <draft_root> <draft_root>.bak`
