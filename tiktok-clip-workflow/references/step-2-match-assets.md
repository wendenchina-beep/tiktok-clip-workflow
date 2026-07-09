# Step 2: 视觉匹配素材

## 目的

遍历用户素材库，给参考视频的每个镜头找到**最匹配**的替换素材。

## 输入 / 输出

**输入**：
- `work/<日期>/recipe.json` — Step 1 产物
- `assets/` — 用户素材库路径（含分类子目录）

**输出**：
- `work/<日期>/fragment_plan.json` — 每镜的素材需求
- `work/<日期>/matches.json` — 最终匹配结果
- `work/<日期>/selected/shot_001/xxx.mp4`, ... — 匹配到的素材**复制**到此（不是移动）

## 执行步骤

### 2.1 生成 fragment_plan.json

先把 `recipe.json` 里每镜的信息翻译成"素材需求"：

```json
{
  "shot_001": {
    "scene_type": "product_closeup",
    "duration": 3.2,
    "keyframe_features": {
      "dominant_color": "#f5e8d0",
      "brightness": 180,
      "composition": "center_focus"
    },
    "requirements": {
      "must_include": ["产品", "特写"],
      "should_avoid": ["人物", "室外", "深色背景"],
      "shot_type_preference": ["close_up", "product_shot"]
    },
    "text_content": "这个吹风机才 199 你敢信？"
  }
}
```

### 2.2 素材库预索引（可缓存）

第一次跑素材库 → 给每个素材文件提取关键帧 + 特征：

```python
# 每个素材提取 3 帧关键帧
# 计算: dominant_color, brightness, composition, has_face, has_text
# 存到 assets/.index.json 缓存
```

后续跑同一素材库直接读缓存，不用重复处理。

### 2.3 视觉匹配算法

对每个镜头：

1. 遍历 `assets/.index.json` 里的所有素材
2. 计算每个素材跟本镜头的**综合匹配得分**：
   - 颜色相似度（40%）
   - 亮度接近度（20%）
   - 构图匹配度（20%）
   - 分类匹配（20%，如需要产品特写就优先 `外观/` 目录）
3. 得分排序，取 Top 3
4. 应用**多样性铁律**（下一节）后取 Top 1

### 2.4 多样性铁律（必须遵守）

生成看起来不像"AI 生成"的核心：

- 🚫 **同一素材不允许连续 3 镜使用**
- 🚫 **同类型场景不允许连续超过 5 镜**
- 🚫 **同分类目录（如 `外观/`）不允许连续超过 3 镜**
- ✅ 相邻镜头**视觉风格必须有变化**：远景→特写→中景 交替
- ✅ 相邻镜头的**主色调**要有区分（避免连续 3 镜都是同色系）

违反铁律时，从 Top 3 里降级选次优素材；如果 Top 3 都违反，标记「缺素材」。

### 2.5 匹配置信度阈值

- **≥ 0.75**：高置信度，直接采用
- **0.60 - 0.75**：中置信度，采用但标记 `"downgraded": true`
- **< 0.60**：低置信度，**标记「缺素材」，不硬填**

**铁律：宁可留空素材，也不用无关素材硬凑。**

低置信度的镜头留空，让用户自己补。这样虽然多了一些手动工作，但最终的视频质量会好很多。

### 2.6 输出 matches.json

```json
{
  "shot_001": {
    "matched_asset": "assets/外观/product_A_closeup_001.mp4",
    "confidence": 0.87,
    "downgraded": false,
    "reason": "高置信度：颜色/亮度/构图全部匹配"
  },
  "shot_002": {
    "matched_asset": null,
    "confidence": 0.42,
    "downgraded": false,
    "reason": "MISSING: 素材库无匹配（需要户外使用场景，素材库缺）",
    "action_required": "人工补素材"
  },
  "shot_003": {
    "matched_asset": "assets/功能/demo_005.mp4",
    "confidence": 0.68,
    "downgraded": true,
    "reason": "中置信度：构图较匹配但色调偏冷"
  }
}
```

### 2.7 复制（不是移动）到 selected/

**铁律**：`cp`，不是 `mv`。破坏用户原始素材库是不可接受的。

```bash
mkdir -p work/<日期>/selected/shot_001
cp assets/外观/product_A_closeup_001.mp4 work/<日期>/selected/shot_001/
```

## Review Checkpoint（⏸ Gate）

匹配完让用户 review：

1. 打开 `matches.json`，看每镜的 `confidence` 和 `matched_asset`
2. 低置信度镜头（confidence < 0.60）→ 用户人工补
3. 用户觉得某镜匹配不满意 → 告诉 Skill "第 5 镜换成 XXX 素材"
4. 有 `MISSING` 标记 → 素材库不够全，去补对应类型素材

## Common Issues

- **匹配结果全部低置信度**：素材库分类不够细 / 覆盖不够全，参考 SKILL.md 的 assets 结构建议
- **相邻镜头看起来太相似**：多样性铁律没生效，检查是否算了 diversity penalty
- **明明有很合适的素材没匹配上**：颜色权重可能太高，改成 30-30-20-20 试试
- **素材被移走了**：绝对禁止 `mv`，一律用 `cp`；报错就查是不是环境的 alias
