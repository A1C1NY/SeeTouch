# SeeTouch

Vision-driven mobile GUI agent that sees your screen and operates your phone via natural language.

SeeTouch 通过屏幕截图和视觉语言模型（VLM）理解手机界面，将自然语言指令转换为 Android 操作。项目仓库：[seetouch-agent/SeeTouch](https://github.com/seetouch-agent/SeeTouch)。

## 当前支持的运行方式

| 入口 | 设备控制 | 当前推理后端 | 敏感动作确认 |
|---|---|---|---|
| PC 命令行 `run` | ADB + uiautomator2 | 默认 Gemini，可用 `--reasoner doubao` 切换 | 命中关键词规则后在终端确认 |
| PC Web 调试器 `debug` | ADB + uiautomator2 | 固定 Doubao，尚无模型选择参数 | **当前自动批准，尚未接入交互确认** |
| Android 独立 App | 无障碍服务 + Kotlin/Python 桥接 | 固定 Doubao，通过 App 设置配置 | App 内确认，5 分钟超时拒绝 |

Android App 不依赖电脑执行任务，但仍需联网调用云端模型。构建与安装见 [Android 指南](android/README.md)。

本说明以 2026-10-12 检查的 `main`（代码基线 `de755f7`）为准；其他分支的能力不视为已合并功能。

## PC 快速开始

需要 Python 3.10+、Android Platform Tools（`adb`），以及已开启 USB 调试并授权电脑的 Android 设备。先用 `adb devices` 确认连接状态。

```bash
git clone https://github.com/seetouch-agent/SeeTouch.git seetouch
cd seetouch
python -m venv .venv
```

Windows PowerShell：

```powershell
.\.venv\Scripts\Activate.ps1
Copy-Item .env.example .env
```

macOS / Linux：

```bash
source .venv/bin/activate
cp .env.example .env
```

安装依赖，然后在本地 `.env` 中填入所选服务的 API key：

```bash
python -m pip install -e .
```

### Gemini：CLI 默认后端

从 [Google AI Studio](https://aistudio.google.com/) 获取你有权使用的凭据。以下模型名是代码中的默认值，实际可用性、收费和配额以你的 API 项目为准；项目不保证固定免费额度。

```dotenv
GEMINI_API_KEY=replace-with-your-key
GEMINI_MODEL_ID=gemini-3.1-flash-lite
```

```bash
python -m seetouch run "打开设置"
python -m seetouch run "在哔哩哔哩搜索采莲曲" --reasoner gemini
```

配置与实现边界见 [Gemini 指南](docs/gemini-integration.md)。

### Doubao：CLI 可选，调试器和 Android App 使用

```dotenv
VLM_API_KEY=replace-with-your-key
DOUBAO_MODEL_ID=doubao-seed-1-6-vision-250815
DOUBAO_API_URL=https://ark.cn-beijing.volces.com/api/v3
SEETOUCH_THINKING_MODE=enabled
```

```bash
python -m seetouch run "打开设置" --reasoner doubao
```

`SEETOUCH_THINKING_MODE` 控制 Doubao，Gemini 适配器不读取它。代码接受 `enabled`、`disabled`、`auto`；历史测试中的默认 Doubao Vision 模型不支持 `auto`，其他模型的支持情况需单独核实。

### CLI 参数和环境变量

```bash
python -m seetouch run --help
python -m seetouch run "打开设置" --reasoner doubao --max-steps 20 --runs-dir ./runs
```

| 设置 | 默认值 / 含义 |
|---|---|
| `--reasoner` | `gemini`；也可选 `doubao`，不是环境变量自动选择 |
| `--serial` / `SEETOUCH_DEVICE_SERIAL` | 指定设备；未指定时由 uiautomator2 连接 |
| `--max-steps` / `SEETOUCH_MAX_STEPS` | 45 |
| `--runs-dir` / `SEETOUCH_RUNS_DIR` | `runs` |
| `SEETOUCH_LOG_LEVEL` | `INFO` |
| `SEETOUCH_APPLIST_PATH` | PC 应用索引路径，默认 `~/.seetouch/applist.json` |

已有环境变量优先于 `.env`。Gemini key 按 `GEMINI_API_KEY → VLM_API_KEY` 读取；Doubao 按 `VLM_API_KEY → DOUBAO_API_KEY` 读取。不同厂商凭据不能混用。

`UnifiedConfig` 已存在，但 CLI 仍使用兼容的 `AppSettings` 和各 Reasoner 配置。该类中的额外环境变量并未全部接入运行入口，详见 [技术决策](memory/technical_decisions.md)。

### 应用索引和运行记录

安装 SeeTouch Android APK 后，可从仓库根目录导出手机应用显示名索引：

```bash
python scripts/pull_applist.py
```

脚本默认写入当前目录 `applist.json`，同时同步到 `~/.seetouch/applist.json`。文件通道失败时可显式改用 `--channel logcat`；PC 脚本不会自动从失败的文件通道切换。索引包含应用和设备信息，不应直接公开。

每次运行在 `runs/<task_id>/` 保存 `step_001.png` 等截图、`trace.jsonl` 和 `summary.json`。回放是打印已有记录，不会重新操作手机：

```bash
python scripts/replay.py runs/<task_id>
```

环境诊断入口为 `python scripts/doctor.py`。它会连接设备，当前仅检查 Doubao 凭据且打印 key 前缀；仅配置 Gemini 时可能报缺少 key，分享输出前应脱敏。

## Web 调试器

```bash
python -m pip install -e ".[debugger]"
python -m seetouch debug --port 5000
```

浏览器访问 `http://localhost:5000`。支持 Step / Run / Pause / Stop、截图动作标注、步骤时间线、模型输入输出、Doubao 返回的推理内容、token 统计和实时日志。

当前调试器固定使用 Doubao；仅设置 `GEMINI_API_KEY` 不能切换后端。服务监听 `0.0.0.0`，允许任意 CORS 来源且没有用户认证；敏感动作确认回调直接返回同意。应仅在受控开发环境运行，避免暴露到不可信网络，不用它执行支付、发消息或删除等敏感任务。

## 架构与动作

```text
seetouch/
├── core/           # Action、Task、Runner、Session、EventBus、异常类型
├── device/
│   ├── base.py     # DeviceController Protocol
│   ├── android/    # PC uiautomator2、AppIndex、AppLauncher
│   └── android_native/  # Android App 的 Kotlin 桥接适配器
├── perception/     # screen.py：缩放、坐标换算、图像编码
├── reasoning/      # base.py、doubao.py、gemini.py、parser.py、prompts.py
├── safety/         # 关键词规则和确认回调
├── debugger/       # Flask / SocketIO + 原生 Web UI
└── cli/            # 命令行组装
android/            # Kotlin / Compose / Chaquopy App
scripts/            # doctor.py、pull_applist.py、replay.py
tests/              # 当前维护的 Python 测试入口
```

| 动作 | 参数 | 含义 |
|---|---|---|
| `CLICK` | `{"point": [x, y]}` | 点击 |
| `TYPE` | `{"text": "..."}` | 输入文本 |
| `SCROLL` | `{"start_point": [x, y], "end_point": [x, y]}` | 滑动 |
| `OPEN` | `{"app_name": "设置"}` | 按本机显示名启动应用 |
| `BACK` | `{}` | 系统返回 |
| `WAIT` | `{}` 或 `{"seconds": 1.5}` | 默认 1.5 秒，执行时限制在 0.5–5 秒 |
| `COMPLETE` | `{}` | 模型声明结束；不是独立验证的成功指标 |

坐标采用 0–1000 归一化；设备层按实际屏幕尺寸换算。Runner 将截图宽度降至最多 720 像素，保持比例，供模型和本地记录使用。

OPEN 顺序为学习缓存、本机索引归一化精确匹配、唯一子串匹配、候选反馈；索引缺失或累计多次未命中后回桌面视觉查找。内部调用保留已安装包名直通，模型提示要求输出显示名。旧静态映射表已退出启动路径。

## 已知限制与数据处理

- **完成状态可能误报**：两个 Reasoner 在捕获 API 异常时均返回 `COMPLETE`，Runner 随后会记为 completed；这是待修问题。模型输出解析失败才降级为 `WAIT`，不能把它描述成可靠的 API 重试。
- Guard 依赖任务文本和模型摘要中的关键词，可能漏判；各入口的确认方式不同，见首表。
- 截图、任务指令及历史摘要会发送给所选云端模型；本地截图、日志、trace 也可能包含聊天内容、账号或输入文本，当前没有自动脱敏。
- Android key 使用普通 SharedPreferences 保存；备份规则和 exported 诊断入口尚待加固，见 [隐私审计](docs/privacy-audit.md)。
- 暂未实现通用 OpenAI 后端入口、多轮澄清、用户偏好、Playbook、Web/iOS/桌面控制；Python Protocol 提供扩展点。

## 开发与验证

```bash
python -m pip install -e ".[dev]"
python -m pytest tests -q
python -m ruff check seetouch
```

2026-10-12 在上述代码基线上：`tests/` **88 passed**，`ruff check seetouch` 通过。扩大到 `ruff check seetouch tests scripts` 时有 4 项既存测试文件 lint 问题。上述结果来自离线单元 / mock 测试，不证明真实模型服务或手机任务成功。

早期记录包含 Doubao + PC 路径的真机试跑，以及 Android 模拟器闭环；本轮未重新构建 APK、连接手机或调用真实模型。历史结果见 [开发历程](memory/development_history.md)，不能作为当前 Gemini 或 Android 真机回归结论。

贡献方式见 [CONTRIBUTING.md](CONTRIBUTING.md)，近期工作见 [路线图](memory/future_roadmap.md)。问题反馈：[GitHub Issues](https://github.com/seetouch-agent/SeeTouch/issues)。

## 许可声明

`pyproject.toml` 声明 Apache-2.0，但当前工作树中缺少 `LICENSE` 正文。应由维护者确认并补齐正式许可证文件；本次文档审计不改变许可选择。
