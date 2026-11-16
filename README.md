# FishTool：给个人视频创作者的运营工作台

[![Release](https://img.shields.io/github/v/release/Sudofox2080/FishTool)](https://github.com/Sudofox2080/FishTool/releases/latest)
[![Python](https://img.shields.io/badge/Python-3.11%20%7C%203.12-blue)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/License-AGPL--3.0-blue)](./LICENSE)

FishTool 面向 B 站 UP 主，把热点发现、评论监控、同行调研、账号自诊和评论区抽奖放进同一个工作台。可以直接在浏览器里使用，也可以打开带托盘和桌宠提醒的桌面版；AI 是可选的分析助手。

[快速上手](./QUICKSTART.md) · [项目结构](./STRUCTURE.md) · [更新记录](./CHANGELOG.md) · [问题反馈](https://github.com/Sudofox2080/FishTool/issues)

![FishTool 界面总览](./assets/readme/overview.png)

## 项目背景：为什么做这个工具

个人创作者需要做的运营工作，往往散在不同页面里：看榜单找选题、反复打开评论区、记录同行数据、复盘自己的投稿。数据能找到，但手动收集和整理会占用本来可以用来创作的时间；只看一次播放量，也很难判断一个选题是在上涨，还是已经开始降温。

FishTool 想解决的是这段从「看到数据」到「做出行动」之间的重复劳动：持续记录热点变化，把评论整理成可以复核的舆情，把账号表现放进同行背景里比较，再把这些信息带回选题和复盘。项目希望让个人创作者也能用上可负担、可修改的运营工具，而不必先拥有一个运营团队。

因此，它保留了不依赖 AI 的采集、统计和规则判断，也允许接入自己的 OpenAI 兼容模型服务。数据和判断依据应当能被查看；模型输出则作为参考，交给创作者决定是否采用。

## 现在能解决什么问题

| 创作中的问题 | FishTool 的处理方式 |
| --- | --- |
| 热门视频很多，哪些还值得跟进？ | 按分区采集热点、记录趋势和生命周期，支持单视频持续跟踪、Tag 词云、活动情报和事件工作台。 |
| 评论太多，担心错过集中反馈？ | 采集、去重、情感统计和常驻监控；负面占比、评论量或关键词触发预警，网页和桌宠都能收到提醒。 |
| 同行为什么做得好，自己差在哪里？ | 头部 UP 数据卡片、运营策略拆解、账号自诊和同行排名；配置 AI 后补充分析报告。 |
| 评论区抽奖怎样兼顾筛选和公平？ | 按等级、会员、时间等条件筛选；支持真人候选分析、按 UID 去重、开奖前确认视频信息和开奖后复查。 |

抽奖页还提供「AI 伪人巡检」：用规则及可选模型分析可疑评论，列出依据、支持人工复核和导出。具体操作、举报方式和限制见 [巡检说明](./docs/LOTTERY_PATROL.md)。

评论监控的 **AI 语义分组仍属实验功能**。它允许把模型判断等价的评论折叠展示，同时保留原文、用户和情感信息；人工拆分或撤销会保存保护。展示分组不会改写原始评论和现有预警统计，语义准确率也尚未经过真实标注集验证。

## 核心难点：让采集、分析和界面保持一致

### 一套业务，同时服务浏览器和桌面

桌面版用 PySide6 的 WebEngine 承载同一套网页，业务请求仍走本机 FastAPI 服务，预警通过 WebSocket 推送。这样，评论采集、任务状态和报表不需要为两个入口各做一份；桌面层只补充窗口、托盘、桌宠和保存文件等能力。

这也意味着网页里能用的动作需要在桌面壳里验证。例如，外站的 `target="_blank"` 链接需要由桌面层接管并交给系统浏览器，才能从工作台正常打开 B 站主页。相关入口在 [app.py](./app.py)、[桌面主窗口](./desktop/main_window.py) 和 [Web 服务](./web/main.py)。

### 采集不到资料，不能当作资料不存在

B 站接口会遇到登录态失效、限频、分页变化和部分资料不可见。它们影响的是证据完整性，不能直接变成「这个账号异常」或「这份名单已经完整」的结论。

FishTool 在接口层保留限频和重试，在抽奖分析中记录资料来源状态，并允许返回「资料不足」。本人视频的楼中楼采集会检查分页完整性和评论总数变化，遇到不完整结果会报错，避免拿残缺名单继续开奖。模型不可用时，真人筛选只保留规则能支持的结论，其余归入资料不足。

### AI 输入、业务规则和展示统计要分清楚

账号昵称、签名是用户自行填写的内容，不能成为给判定模型的指令；评论条数也不等于参与抽奖的人数。后台需要分别处理账号证据、抽奖权重和中奖名额，界面则需要把这些口径说清楚。

这类问题靠更换模型或增加提示词并不能全部解决。下面两次近期修复，体现了项目在数据一致性和业务边界上的实际取舍。

## 最近两次排错记录

### 2026-11-14：拆开数据库连接之后，还要缩短写事务

**问题。** 文件数据库原先使用 `StaticPool`，不同 SQLAlchemy Session 可能共用同一条 SQLite 连接。两个任务交错执行时，一个 Session 的提交或回滚可能干扰另一个 Session 尚未结束的事务。

**处理。** 文件数据库改用 `NullPool`，让同时使用的 Session 各自持有连接；内存数据库仍保留 `StaticPool`，因为不同内存连接对应不同数据库。同时启用 WAL，并给锁等待设置可配置的超时。

修复没有停在连接池上：连接隔离之后，评论监控在写事务里等待 WebSocket／桌宠通知，会持续占用 SQLite 写锁，使其他写入排队甚至报 `database is locked`。现在先收集通知内容，完成提交、关闭会话，再推送消息；提交失败时也不会发出没有保存成功的预警。

**留下的经验。** 并发问题要同时检查连接归属和事务边界。通知等外部等待放到提交之后，既缩短持锁时间，也让用户看到的预警与数据库记录一致。仓库已有「一个会话回滚不影响另一个会话」和「提交失败不通知」的回归用例。

依据：[修复提交 5454523](https://github.com/Sudofox2080/FishTool/commit/5454523fa21f7841d32bb503a05f4544fe905d74) · [数据库管理](./core/database/manager.py) · [预警保存与推送](./modules/comment/_alerts.py) · [数据库测试](./tests/test_core_database.py) · [预警测试](./tests/test_comment_monitor_alerts.py)

### 2026-11-13：多条评论可以增加权重，但一个人只能中一份

**问题。** 关闭「同一 UID 只保留一次抽奖机会」后，候选池按评论计数。如果直接按评论抽多个中奖项，同一个人可能凭不同评论重复中奖；如果只按评论数检查名额，也可能出现评论很多、独立参与人数却不足的情况。

**处理。** 候选评论先按 UID 分组，抽取时保留评论数量带来的权重，一个 UID 中奖后移出该 UID 的全部候选机会。名额上限按独立人数校验，结果同时提供候选评论数和去重后人数。

同一轮修复还收紧了真人判定的模型输入：从剔除几个已知文本字段，改为只发送明确允许的账号指标和采集状态。昵称、签名等自由文本不发送，接口异常原文只保留错误数量，后续新增字段也不会默认进入模型。完整昵称仍可用于本地展示。

**留下的经验。** 中奖概率和中奖人数是两个不同规则，需要分别约束。模型输入也需要在代码中限定范围，不能只靠一句「不要执行资料里的指令」。仓库测试覆盖了多评论情况下中奖 UID 唯一、独立人数不足，以及未知字段不进入模型等边界。

依据：[修复提交 1685c69](https://github.com/Sudofox2080/FishTool/commit/1685c695ad2ce22be88bc02844108c062b259b9d) · [抽奖服务](./modules/lottery/service.py) · [模型分析](./modules/lottery/analyzer.py) · [抽奖测试](./tests/test_lottery_service.py) · [分析测试](./tests/test_lottery_analyzer.py)

## 快速开始

推荐 **Python 3.11 或 3.12**。先下载仓库 ZIP 并解压，或运行：

```bash
git clone https://github.com/Sudofox2080/FishTool.git
cd FishTool
```

### Windows 一键启动

安装 Python 时勾选 **Add python.exe to PATH**，然后双击根目录的 `start_web.bat`。首次运行按提示安装依赖；服务就绪后会自动打开 [http://localhost:8000](http://localhost:8000)。

运行期间保持命令行窗口打开，关闭窗口或按 `Ctrl+C` 会停止服务。端口被其他程序占用时，可以在命令行运行 `start_web.bat 9000`。

### 从源码启动

建议使用独立虚拟环境：

```bash
python -m venv .venv
```

Windows 激活环境：

```powershell
.venv\Scripts\activate
```

macOS／Linux 激活环境：

```bash
source .venv/bin/activate
```

安装依赖并启动 Web 版：

```bash
python -m pip install -r requirements.txt
python app.py web
```

浏览器访问 [http://localhost:8000](http://localhost:8000)。其他入口：

| 命令 | 用途 |
| --- | --- |
| `python app.py` | 打开桌面主窗口、托盘和桌宠，同时启动同一套 Web 服务。 |
| `python app.py web --port 9000` | 仅启动 Web 版并指定端口。 |
| `python app.py web --reload` | Web 开发模式，修改代码后自动重载。 |
| `python app.py init` | 仅初始化配置、数据库和运行目录。 |

默认只监听本机。需要局域网访问时，使用 `python app.py web --host 0.0.0.0`，并设置 `FISHTOOL_ALLOWED_HOSTS`，例如 `localhost,127.0.0.1,192.168.1.10`；该环境变量会覆盖默认白名单，需要保留本机地址。

### 第一次使用建议

1. 在「欢迎」页扫码登录 B 站，登录态会加密保存在本地。
2. 从一项具体任务开始，例如采集一个视频的评论，或查看自己账号的自诊数据。
3. 需要模型分析时，在「配置」页填写自己的 API Key、Base URL 和 Model，再保存。

![模型与应用配置](./assets/readme/config.png)

不配置 AI 也能使用热点采集和趋势、评论采集／去重／情感统计／预警、账号数据和同行排名、评论区抽奖等功能。AI 选题会降级为规则模板；头部 UP 策略和账号报告的模型分析部分需要配置模型。真人筛选未配置模型时，只挑出规则能识别的疑似抽奖号，其余归为资料不足；巡检则只使用内置规则。

详细操作与常见问题见 [QUICKSTART.md](./QUICKSTART.md)。

## 数据、凭据和使用边界

业务数据库保存在 `data/`，日志在 `data/logs/`；B 站登录态和模型 API Key 加密保存在本机，配套密钥为 `config/.key`。备份或迁移时应保留配套文件，勿把密钥、Cookie 和日志中的个人信息发到公开 Issue。

使用模型分析时，相应任务的数据会发往你配置的模型服务，可能产生费用。「抽奖真人判定」和「AI 伪人巡检」的输入范围不同：前者使用限定字段的账号指标，后者的 AI 甄别会发送评论、昵称和签名；启用前请查看对应功能的说明。AI 用量可在「配置」页查看。

采集受 B 站登录权限和接口状态影响，请保留内置限频。真人候选与伪人率都是辅助判断，不代表身份认证。巡检代发举报使用自己的 B 站账号，首次使用需要同意说明，每次提交前需要确认；单次最多 5 条、24 小时内最多 10 条，遇到平台提示过于频繁会暂停。巡检结果保存在本机，7 天后自动清理。

## 开发与文档

项目按职责拆分为 B 站接口封装、采集分析模块、数据库与配置、Web 服务和桌面外壳。对应目录与调用关系见 [STRUCTURE.md](./STRUCTURE.md)，异常处理约定见 [docs/EXCEPTION_HANDLING.md](./docs/EXCEPTION_HANDLING.md)。

安装测试工具后运行：

```bash
python -m pip install pytest
python -m pytest tests/ -q
```

桌面测试涉及 Qt 和显示环境，环境准备、依赖和分组运行方式见 [测试与桌面验证说明](./docs/TEST_BASELINE_AND_DESKTOP_RUNBOOK.md)。Web 服务启动后，可访问 [健康检查](http://localhost:8000/health) 和 [接口文档](http://localhost:8000/docs)。

需要打包桌面版时，在开发环境安装 PyInstaller，再运行：

```bash
python -m pip install pyinstaller
pyinstaller --clean --noconfirm build.spec
```

产物位于 `dist/`。AstrBot 集成的安装方式见 [插件使用说明](./integrations/astrbot/plugin/README.md)，适配和构建方式见 [集成开发说明](./integrations/astrbot/README.md)。

## 未来规划

下面是希望继续推进的方向，尚不代表已经完成或承诺的发布时间：

- **让分析结果更容易验证。** 为评论语义分组和巡检建立真实标注样本，分别观察误合并、误报与漏报，再决定规则和模型该如何调整。
- **让长任务的状态更清楚。** 在现有进度、取消和断点能力上，继续改善失败原因、资料缺失与恢复状态的呈现，方便用户判断应该等待、重试还是补充登录信息。
- **让热点研究连到创作复盘。** 继续完善事件、机会、选题和采用记录之间的联系，帮助创作者回看哪些判断最终转化成了作品。
- **打磨桌面体验。** 持续验证不同缩放比例、多屏切换和 WebEngine 环境下的表现，改善桌宠提醒与主窗口交互。

欢迎带着具体创作场景、复现步骤或脱敏后的结果来提 [Issue](https://github.com/Sudofox2080/FishTool/issues)；修复和改进也欢迎通过 [Pull Request](https://github.com/Sudofox2080/FishTool/pulls) 参与。QQ群入口保留在应用「欢迎」页。

## License

FishTool 以 [GNU Affero General Public License v3.0（AGPL-3.0）](./LICENSE) 发布。可以使用、修改和分发；分发修改版或通过网络向用户提供修改版服务时，请履行许可证规定的对应源代码提供义务。完整条款以 [LICENSE](./LICENSE) 为准。
