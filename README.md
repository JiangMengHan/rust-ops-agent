# rust-ops-agent

使用 Rust 构建通用 Linux 服务器诊断子 agent，让办公端 AI 按需获取机器信息、读取授权文件并搜索日志，用于排查服务器错误。

节点不运行模型，不后台采集业务指标或建立常驻日志索引。AI 在办公端选择工具与分析，节点在本机执行有界只读操作。

## 项目状态

当前已建立 Git 仓库、目录骨架和设计文档。Cargo workspace、Rust 源码、可执行程序、CI、部署脚本和实机测试尚未实现；空目录由 `.gitkeep` 保留。此时不能运行 `cargo build` 或诊断服务。

## 两份核心文档

- [架构设计](docs/ARCHITECTURE.md)：组件、协议、权限、SQLite、机器信息、文件与日志、资源边界、部署和恢复。
- [按开发顺序的任务明细](docs/TASKS.md)：前置依赖、开发交付物、验证要求，以及最终完整代码审查、测试环境部署、故障/资源/稳定性测试和回归关闭。

架构定义最终系统；任务文档定义实现顺序。任务顺序不代表产品分成不同功能版本。后续开发应从 T01 开始，满足每项验收后再更新状态。

## 工作方式

```text
办公端 AI ← MCP stdio → ops-mcp ← HTTP → ops-gateway
                                            ↑
                                  节点主动 WS / 节点凭证
                                            │
                                         ops-node
```

网关负责身份、授权、路由、总预算和审计。SQLite 保存管理元数据及操作记录，默认不保存文件或日志正文。节点不需要开放公网诊断端口。办公端直接配置 `http://<GATEWAY_IP>:<PORT>`，节点配置对应 `ws://` 地址，不要求域名和 HTTPS 证书；防火墙限制允许来源，HTTP/WS 本身不加密传输。

## 通用能力范围

- CPU 型号/核数、OS/内核、内存、负载和按需采样的使用率。
- 磁盘设备/挂载、容量、inode、读写 IO，以及网卡状态/流量/错误。
- 有界进程信息、只读 systemd 服务状态及授权系统错误日志。
- 授权目录浏览、文件属性、分段/尾部/命中上下文读取。
- 关键词、受限正则、事件时间、多行堆栈、中文编码、gzip/zstd 搜索。
- 有界多节点查询、取消、可靠续查、部分覆盖和未知状态说明。
- 节点与用户身份、目录/服务/日志源权限、脱敏、审计、凭证轮换和部署维护。

不包含任意 Shell、文件写入、服务启停、自动处置或数据库/消息队列产品专有诊断。

## 目录设计

```text
rust-ops-agent/
├── .github/workflows/       # CI；在工程任务中实现
├── .editorconfig
├── .gitattributes
├── .gitignore
├── README.md
├── docs/
│   ├── ARCHITECTURE.md      # 唯一架构文档
│   └── TASKS.md             # 唯一有序开发任务清单
├── crates/
│   ├── ops-protocol/src/    # 协议、schema、错误与预算
│   ├── ops-node/src/        # Linux 节点、系统读取、文件与日志
│   ├── ops-gateway/src/     # HTTP/WS、认证授权、SQLite、审计
│   └── ops-mcp/src/         # 办公端 MCP stdio 桥接
├── configs/
│   ├── examples/           # 仅无真实凭证的配置模板
│   └── local/              # 本机真实配置；忽略，按需创建
├── deploy/systemd/         # 节点/网关服务单元
├── scripts/                # 构建、部署、验收入口
├── tests/
│   ├── integration/        # T01 接入为独立 Rust 测试 workspace member
│   ├── e2e/                # 真实链路、部署与故障测试
│   ├── fixtures/           # 可公开的合成/脱敏测试样本
│   └── performance/        # 资源、稳定性与性能验收
└── .local/                 # 凭证、数据库、日志、原始测试证据；忽略
```

T01 创建 Cargo manifests、工具链与实际源码入口。虚拟 workspace 不会自动发现仓库根的测试目录，`tests/integration` 需要显式接入。`.local/` 和 `configs/local/` 不创建可跟踪占位文件。

## Git 与公开内容约定

`findings.md`、`progress.md`、`task_plan.md` 为本机工作记录，保留在本地，不进入 Git。真实配置、秘密凭证、SQLite、环境变量文件、构建/覆盖率输出和原始运行材料同样忽略。初始提交使用 GitHub noreply 邮箱。

应用项目的 `Cargo.lock` 在生成后应提交。测试所需的合成/脱敏 `.log`、`.gz`、`.zst` 不做全局忽略；`tests/fixtures` 保留原始字节，避免换行转换破坏偏移和编码用例。真实日志及秘密材料放在被忽略的本地目录，忽略规则不能替代发布前的内容检查。

## 开发与验收

每个任务应记录实现位置、执行命令和证据，不能仅凭代码存在就标记完成。资源占用以包含 HTTP/WS、认证授权、日志解析和压缩能力的完整程序测量为准。

最终验收包括独立完整代码审查及修复、在已授权 Linux 测试环境部署、真实 AI/MCP → HTTP 网关 → WS 节点调用、权限/文件/故障测试、资源和长时间稳定性测试、修复后回归与复审。服务器身份、地址和登录配置通过私有渠道提供，不写入公开仓库。
