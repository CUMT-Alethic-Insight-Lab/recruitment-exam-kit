# recruitment-exam-kit

Alethic Insight 招新机试的 **Windows 考生端、会话清洗器与离线安装器源码**。
考生在自己的电脑上使用 Kimi Code 或 opencode 作答，本项目提供本地考试页面、会话发现与清洗、预览、交卷和失败补传。

- 考生使用说明：[candidate-guide.md](candidate-guide.md)（安装包也携带此文件）。
- 本仓权威来源：<https://github.com/CUMT-Alethic-Insight-Lab/recruitment-exam-kit>。
- 服务端与 HTTP 契约权威来源：[alethicinsight-platform](https://github.com/CUMT-Alethic-Insight-Lab/alethicinsight-platform)。

## 职责与边界

| 目录 / 文件 | 职责 |
| --- | --- |
| `exam_server.py`、`config.json`、启动脚本 | 本机考试界面、平台 API 客户端、上传与 OTA 提示 |
| `cleaner.py` | Kimi Code / opencode 会话清洗与可预览导出 |
| `session_receiver.py` | 可信考场内网会话接收器，不是公网服务 |
| `installer/` | Windows 安装向导、离线组件清单与 PyInstaller 打包 |
| `tsnet-socks/` | 可选的用户态 P2P 上传辅助程序 |
| `test_*.py` | 合成数据驱动的单元与本地 HTTP 流程测试 |

本仓不包含平台服务端、报名/评分系统、成员数据、生产部署或前端模板。
公共 CLI 接口仍归平台仓的 `infra/interface/alethicinsight-cli`，不在这里拆分 CLI 或 MCP。
AI 工具和离线组件是第三方软件，不由本项目维护。

## 平台 API 与版本兼容

本次迁移不改变运行协议或 `KIT_VERSION`：基线为平台仓提交
[`c97b980d75c2eeb730c4ca68f240b9ca2b253a00`](https://github.com/CUMT-Alethic-Insight-Lab/alethicinsight-platform/commit/c97b980d75c2eeb730c4ca68f240b9ca2b253a00)
中的 `exam_kit/`，客户端版本为 `2026.10.13-1`。这是迁移兼容基线，**不是对所有后续平台版本的兼容承诺**；目前没有独立的 API 语义版本协商机制。

- `cloud_base` 是平台根前缀（默认示例 `https://alethicinsight.com/recruitment`），`campaign_id` 是届次；本地开发必须改为本地测试平台，不要使用真实考生账号或生产地址做冒烟测试。
- 登录依赖 `/api/auth/salt` 与 `/api/campaigns/{campaign_id}/auth/login` 的客户端 PBKDF2 散列契约；后续请求使用考生 Bearer token。
- 抽题、开考、换题、会话查询、交卷、项目提交以及可选 `exam/p2p-key` 和 `exam/kit-token` 端点的请求/响应，以 `exam_server.py` 的调用与平台对应版本的路由实现为准。默认云端上传、可选 LAN/P2P 的认证和失败回退必须一起验证。
- OTA 元数据位于 `{cloud_base}/static/kit-version.json`，包含 `version`、`url`、`sha256`、`notes`；受保护下载使用 `auth-required:kit-token`，不能把签发密钥放入客户端。
- 平台认证、字段、状态机或上传契约变更时，应同时评审两仓，记录可配对的平台提交与 kit 版本，运行本仓测试，并在隔离测试环境完成登录→选题→清洗预览→交卷→补传验证。当前测试使用本地替身平台，不能代替真实平台的跨仓验收。

## 开发与完整测试

推荐在 Windows 上使用 Python 3.12+；分发配置采用 Python 3.12.10，迁移测试环境为 Python 3.14。
核心 Python 程序只使用标准库；pytest 是开发测试依赖。

```powershell
# 在本仓根目录
python -m pip install pytest
python -m pytest -q

# 仅在配置为隔离测试平台后启动（本地页面默认 127.0.0.1:8899）
python exam_server.py
```

`test_*.py` 包括本地 HTTP、SQLite、清洗、LAN 接收与上传回退测试，使用临时目录和合成会话，不需要生产凭据。
不要对真实 AI 工具用户目录或真实成员数据执行测试。测试通过不代表安装组件、第三方服务或公网部署已验收。

## 构建与安装包发布

源码克隆不含运行时、组件缓存或安装包。组件版本、官方来源和 SHA256 以
[installer/COMPONENTS.md](installer/COMPONENTS.md) 与
[installer/components_spec.json](installer/components_spec.json) 为准；字段约定见
[installer/SPEC_README.md](installer/SPEC_README.md)，安装器说明见
[installer/installer_README.md](installer/installer_README.md)。不要把历史文档里的实测结果当作本次构建结果。

```powershell
# Windows 维护机：按 COMPONENTS.md 准备 installer/payload_cache/ 内组件
python -m pip install pyinstaller
cd installer
python build_installer.py --kit-only  # 校验组件并组装 payload，不生成 exe
python build_installer.py             # 校验、组装、打包、自检
```

正式产物是 `installer/dist/招新考试系统-安装程序.exe`。构建脚本会拒绝缺失或摘要不匹配的缓存；不要绕过校验。
如需重制 P2P 组件，在 `tsnet-socks/` 按其 `go.mod` 的 Go 版本运行
`CGO_ENABLED=0 go build -ldflags="-s -w" -o tsnet-socks.exe .`（Git Bash），再按 COMPONENTS.md 组装缓存 zip。
`build_package.ps1` 是旧式 embeddable Python zip 打包入口，不是完整离线安装器；正式发布使用上述 installer 流程。

发布由维护者显式执行，本仓迁移不触发发布或部署：

1. 修改 `exam_server.py` 的 `KIT_VERSION`，跑完整测试，在干净 Windows 环境按 [沙盒检查单](installer/SANDBOX_CHECKLIST.md) 验证安装和两种 AI 工具流程。
2. 构建安装器；检查产物内仅有可分发源码/组件、公共配置和考生指南，不含状态、会话、日志、附件或密钥。记录源码提交、平台配对提交、组件版本和 `certutil -hashfile <安装包.exe> SHA256` 摘要。
3. 安装包按组织发布策略上传至经批准的下载存储或 GitHub Release，**不提交进 Git 历史**。第三方组件分发前单独核对授权、通知和条款。
4. 由平台维护者在平台仓更新 `server/static/kit-version.json`，审核下载鉴权与摘要后通过平台的发布流程上线。不能只更新本仓就视为平台发布完成。考生只收到提示并手动下载安装，不自动替换或重启。

## 隐私与私有数据

本仓仅存公开源码、公共组件元数据与合成测试数据。以下内容不得提交或附在公开 issue 中：

- `state.json`（含考生 `auth_token`）、密码、API key、R2/签发凭据、tailnet 状态和环境私密配置；
- 原始或清洗后的真实会话、`pending_upload/` 的补传材料、附件、`received_sessions/`、学号与成员资料；
- 日志、考试工作目录、`runtime/`、`dist/`、安装器缓存和构建产物。

`.gitignore` 只防止误加入，不是脱敏器；提交/发布前还需检查暂存区与完整发布包。
清洗去除思考过程与工具输出正文，**不保证用户/助手文本没有个人信息或秘密**；交卷前应预览。
经用户选择上传的附件会单独传输，不能宣称只上传会话文字。LAN 接收器的列表/下载没有完整访问控制，仅限可信隔离内网，不应直接暴露公网。

## 历史来源、迁移与回滚

本仓从上述平台固定提交执行 `git subtree split --prefix=exam_kit`，拆分顶点为
`13e6d9fb7a0f9e846c69d5aa070b343af67ac481`。历史仅保留该子目录的提交，文件直接位于仓库根部；subtree 拆分会重写提交 ID，旧仓原提交仍是审计来源。后续改动以本仓为权威，不再同时修改旧仓 `exam_kit/`。
原仓及其未提交内容保留。原 `exam_kit/` 完整本地副本（含构建物、runtime、pending 等排除内容）现位于 `P:/AlethicInsightWorkspace/private-data/legacy-exam-kit-local`，不进入本公共仓；平台线上服务未因拆分改动。

- 源码回滚：使用已知良好提交/版本建立修复分支并通过普通 revert 或修复提交交付，不强推重写公共 `main`；需要比对原实现时检查上述 split 顶点。
- 运行包回滚：停掉本地考试服务，备份含隐私数据的状态/补传文件到受控位置，由维护者确认兼容性后重新安装已验证包，不覆盖正在运行的程序或丢弃未交卷材料。
- OTA 已发错版本时，平台可撤下提示，但客户端不会自动降级；需要重新分发修复包时使用更高 `KIT_VERSION`，保留旧安装包摘要和配对记录。平台自身回滚由平台仓负责。

## 许可状态

本次拆分没有新增或推定开源许可证；当前仓库未声明独立 LICENSE。公开可读不等于获得任意再分发授权。
第三方组件各自的许可证与分发条款仍适用；对外再分发前应由权利人确认本项目及组件的授权。
