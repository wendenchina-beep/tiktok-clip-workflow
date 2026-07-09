# 环境搭建

首次使用本 Skill 前的环境自检 + 装缺失工具的指引。

## 必装组件

### 1. FFmpeg

**macOS**：
```bash
# 用 Homebrew
brew install ffmpeg

# 没装 Homebrew：https://brew.sh/
```

**Windows**：
- 官网下载：https://ffmpeg.org/download.html
- 解压后把 `bin/` 加到系统 PATH

**Linux**：
```bash
sudo apt install ffmpeg    # Debian/Ubuntu
sudo yum install ffmpeg    # CentOS/RHEL
```

**验证**：
```bash
ffmpeg -version
# 期望输出: ffmpeg version X.X.X ...
```

### 2. Python ≥ 3.8

**验证**：
```bash
python3 --version
# 期望输出: Python 3.8.x 或更高
```

如果没装：
- **macOS**：`brew install python3`
- **Windows**：官网下载 https://www.python.org/downloads/
- **Linux**：`sudo apt install python3 python3-pip`

### 3. Python 依赖库

```bash
pip3 install opencv-python pillow numpy scikit-image
pip3 install requests   # 用豆包 TTS 时需要
pip3 install openai-whisper   # 用 Whisper ASR 时需要
pip3 install easyocr    # 用 OCR 提字幕时需要
```

### 4. 剪映专业版

- 中国大陆：https://www.capcut.cn/
- 海外：https://www.capcut.com/

**注意**：本 Skill 生成的草稿基于**剪映专业版**格式，不是移动端剪映。

## 可选组件（配音方式二选一）

### 方式 A：豆包语音大模型 API（推荐带货）

1. 注册字节火山引擎账号：https://www.volcengine.com/
2. 开通「语音技术」→「语音合成」
3. 拿到 `APPID` 和 `Access Token`
4. 存到环境变量：
   ```bash
   export DOUBAO_APPID="your_appid"
   export DOUBAO_TOKEN="your_token"
   ```
5. 或者写到 `AGENTS.md` 里的配置段

### 方式 B：VoxCPM（免费开源，声音克隆）

```bash
git clone https://github.com/OpenBMB/VoxCPM
cd VoxCPM
pip install -r requirements.txt
```

首次运行会下载模型（约 2GB），需要一定网速。

## 一键自检脚本

保存为 `check_env.sh`，跑一次看缺什么：

```bash
#!/usr/bin/env bash
set +e
echo "=== TikTok Clip Workflow 环境自检 ==="

check() {
    local name=$1
    local cmd=$2
    if eval "$cmd" >/dev/null 2>&1; then
        echo "✅ $name"
    else
        echo "❌ $name  (缺失，请先装)"
    fi
}

check "FFmpeg"           "ffmpeg -version"
check "Python ≥ 3.8"     "python3 -c 'import sys; sys.exit(0 if sys.version_info >= (3, 8) else 1)'"
check "opencv-python"    "python3 -c 'import cv2'"
check "pillow"           "python3 -c 'import PIL'"
check "numpy"            "python3 -c 'import numpy'"
check "requests"         "python3 -c 'import requests'"
check "剪映草稿目录"       "test -d ~/Movies/JianyingPro/User\ Data/Projects/com.lveditor.draft || test -d ~/AppData/Local/JianyingPro/User\ Data/Projects/com.lveditor.draft"

echo ""
echo "=== 可选：配音工具 ==="
check "DOUBAO_APPID 环境变量"  "test -n \"\$DOUBAO_APPID\""
check "VoxCPM 目录"             "test -d ./VoxCPM || test -d ~/VoxCPM"

echo ""
echo "自检完成。缺失项按上面文档补装。"
```

跑一次：
```bash
bash check_env.sh
```

## 直接让 Skill 自检

在 AI Agent 里说：

> 帮我跑一下 tiktok-clip-workflow 的环境自检

Skill 会自动跑 `check_env.sh`，缺什么直接告诉你怎么装。
