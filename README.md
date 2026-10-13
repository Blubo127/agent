# monitor-agent

[Blubo127/agent](https://github.com/Blubo127/agent) 是基于
[monitor-probe/agent](https://github.com/monitor-probe/agent) 维护的 fork，配套
[Blubo127/monitor](https://github.com/Blubo127/monitor) 使用。采集 Linux 主机指标，经 WebSocket 上报 hub。

定制开发分支为 `diy`，GitHub 默认分支目前为 `main`；获取本 fork 的定制代码时请显式选择 `diy`。
安装包从本仓库的 [Releases](https://github.com/Blubo127/agent/releases) 获取，问题和改进建议提交到
[Issues](https://github.com/Blubo127/agent/issues)。本文的安装、构建和发布说明以本 fork 为准。

Release 产物为静态链接单文件，无运行时依赖，常驻内存数 MB。

## 特性

- 直接读 `/proc`、`/sys/class/net` 与 `statvfs`，不依赖 sysinfo
- 内存对齐 `free(1)` 的 used 列，磁盘对齐 `df(1)` 的 Used 列
- 无状态：不写文件，不保存跨重启的数据，流量累加由 hub 负责
- token 走 `Authorization` 请求头，不放在 URL 中
- 默认拒绝非回环地址的明文 `ws://`

## 安装

在你部署的 [Blubo127/monitor](https://github.com/Blubo127/monitor) 面板添加节点，
复制生成的命令，在目标 Linux 主机以 root 身份执行：

`https://your-hub` 应替换为你部署的 hub 访问地址，例如 `https://monitor.example.com`。
这是提供安装脚本和监控接口的服务器地址，GitHub 仓库地址用于分发源码和安装包。
`<token>` 替换为面板中该节点的 token。

```bash
curl -fsSL https://your-hub/install.sh | sh -s -- --server https://your-hub --token '<token>'
```

安装脚本识别 systemd 与 OpenRC，二进制装到 `/opt/monitor/monitor-agent`，token 写入
`/opt/monitor/agent.env`（0600）。hub 与 agent 安装在同一台主机时共用 `/opt/monitor` 目录。

### 安装包来源

面板安装链路为：本机拉取 hub 的 `/install.sh`，脚本再从 hub 的 `/agent/<arch>` 下载二进制；
hub 根据源码常量 `AGENT_REPO` 中继 GitHub Releases 安装包。使用本项目时，配套 hub 源码
`src/main.rs` 中 `AGENT_REPO` 应为 `Blubo127/agent`，`HUB_REPO` 应为 `Blubo127/monitor`。
这两个值是编译时常量，不能通过环境变量修改。
只修改本仓库文档或 Git remote，不能改变已部署 hub 的下载来源；hub 修改后需要重新构建和部署。

本仓库发布的安装包名称及下载地址如下：

| 平台 | 安装包 |
|---|---|
| Linux x86_64 | [monitor-agent-x86_64-unknown-linux-musl](https://github.com/Blubo127/agent/releases/latest/download/monitor-agent-x86_64-unknown-linux-musl) |
| Linux aarch64 | [monitor-agent-aarch64-unknown-linux-musl](https://github.com/Blubo127/agent/releases/latest/download/monitor-agent-aarch64-unknown-linux-musl) |
| SHA-256 校验清单 | [sha256sums.txt](https://github.com/Blubo127/agent/releases/latest/download/sha256sums.txt) |

Fork 不会自动复制上游 Releases。首次使用前，需要在本仓库发布含上述文件的正式 Release，
并确认它被标记为 Latest；仅有 tag、草稿或预发布版不足以提供上述下载入口。发布前这些地址可能返回 404。
手动下载固定版本时，把链接中的 `latest/download` 改为 `download/<tag>`，例如
`download/v1.2.0`，并使用同一版本的校验清单。面板安装脚本通过 hub 下载的始终是 Latest，
没有指定版本的参数；固定版本需要手动部署对应二进制。

## 运行

```bash
/opt/monitor/monitor-agent --server https://your-hub --token '<token>'
```

通过安装脚本安装后，二进制的完整路径为 `/opt/monitor/monitor-agent`。

| 参数 | 默认 | 说明 |
|---|---|---|
| `--server` | 必填 | hub 地址，也可用 `MONITOR_SERVER` |
| `--token` | 必填 | 节点 token，也可用 `MONITOR_TOKEN` |
| `--interval` | 1 | 上报间隔（秒）；直接运行二进制时，非负整数会被限制到 1–3600，安装脚本要求输入在该范围内 |
| `--iface` | 空 | 流量统计的网卡，逗号分隔，也可用 `MONITOR_IFACE`；见下 |
| `--insecure` | 关闭 | 允许远程明文 `ws://`，token 与监控数据将明文传输；仅用于可信隔离网络 |

### 统计哪些网卡的流量

默认规则是同一份线上的字节只数一次：lo、容器与虚拟机网卡、隧道不计；bond、网桥、VLAN、macvlan
这类叠在别的网卡上的设备也不计，只计它们底下的那块。除了按名字，还按内核给出的链路类型、`DEVTYPE`、
`lower_*` 链接和所属网桥判断，改过名的隧道、网桥，以及挂在网桥上的虚拟机 tap（如 libvirt 的 `vnet0`）
和容器 veth 同样认得出。PPPoE 只认 OpenWrt 的 `pppoe-wan`：pppd 拨号的
`ppp0` 默认照计，因为 LTE 拨号时它是唯一的链路，用 pppd 拨 PPPoE 的机器要用 `--iface` 指定。

转发流量的机器（软路由、桥接了软路由的宿主机）上，同一个包会经过两块真网卡，哪块面向运营商只有
使用者知道，这时用 `--iface`：

- `--iface eth1` 或 `--iface pppoe-wan`：只统计列出的网卡，写全名时内置规则不再生效
- `--iface -eth0`：从本来会计入的网卡里去掉一块（如软路由的 LAN 口），`-` 开头的都是排除，排除优先于列出。
  一批机器都有一块同名的内网网卡时，一条批量命令就能在每台上去掉它
- 只认完整的网卡名，不支持通配

启动时打印一行 `counting traffic on: ...`，列出此刻计入的网卡。所计网卡的集合一旦变化（改了 `--iface`、
网卡增减或被重新归类），上报的 `boot_id` 也跟着变，hub 重新对基线，不会把新旧两组读数之差记成流量。

## 上报字段

[src/collect.rs](src/collect.rs) 中的 `Facts` 与 `Metrics` 两个 struct 直接序列化为线上 JSON，是字段的权威定义。

- **`Facts`** 每次连接成功时上报一次：主机名、系统、内核、架构、虚拟化类型、CPU 型号与核数、内存 / swap / 磁盘总量、agent 版本、本机 IPv4 / IPv6（每族最多一个）。地址优先选公网；公网属性相同时，IPv6 优先选稳定地址，临时或已废弃地址仍可能被选中。
- **`Metrics`** 首次在连接成功约 1 秒后上报，之后每 `--interval` 秒上报：CPU、负载、内存、swap、磁盘、网卡收发速率与内核累计计数器、TCP / UDP 套接字数、进程数、运行时间。TCP 统计包含 TIME_WAIT。

`net_rx_total` / `net_tx_total` 为所计网卡的内核 lifetime 计数器之和，原样上报；`boot_id` 取自
`/proc/sys/kernel/random/boot_id`，后接 `/` 与所计网卡集合的摘要，标明这两个读数在哪段区间内可以相减，
hub 用它识别主机重启或统计网卡集合变化，并结合计数器是否回退决定是否重新建立基线，**不要删**。
`iface` 回报当前的 `--iface`，供面板显示与预填。

连接 hub 时逐个尝试解析出的地址，除最后一个外每个限 5 秒。网卡上只有内网 IPv4（NAT）时先连 hub 的
IPv4：NAT 的公网地址不在网卡上，hub 只有看到一条 IPv4 连接才知道它。

## 协议

协议实现见 [src/main.rs](src/main.rs)。agent 连接 hub 的 `/api/agent/ws`，
使用 `Authorization: Bearer <token>` 请求头认证。
`https://` 转为 `wss://`，裸主机地址默认使用 TLS，非回环地址的明文连接需要显式启用 `--insecure`。

应用消息为 WebSocket 文本帧，格式为 JSON-RPC 2.0 通知。例如一次 TCP 探测结果：

```json
{"jsonrpc":"2.0","method":"ping.result","params":{"task_id":1,"latency_ms":12}}
```

各方法的 `params` 如下：

| 方向 | 方法 | `params` |
|---|---|---|
| agent → hub | `hello` | `Facts`，每次连接完成后发送一次 |
| agent → hub | `report` | `Metrics`，按 `--interval` 周期上报 |
| hub → agent | `ping.tasks` | 探测任务数组，每个任务为 `{id, target, interval}`，`target` 为 `host:port` |
| agent → hub | `ping.result` | `{task_id, latency_ms}`，连接失败时 `latency_ms` 为 `-1` |

探测只建立 TCP 连接，不读取目标的响应内容。最多运行 64 个任务，间隔限制为 5–3600 秒；
DNS 解析超时会跳过本轮样本。hub 可指定任意可达的主机和端口，部署时应按需要限制探测目标。
hub 使用 WebSocket Ping 保持连接，agent 自动回复 Pong；超过 90 秒未收到 hub 帧会重连。

## 构建

在 Linux 上准备 Rust stable、C 编译器和系统链接器，拉取本 fork 的定制分支后构建：

```bash
git clone --branch diy https://github.com/Blubo127/agent.git
cd agent
cargo build --release --locked
```

开发检查需要 Rust 的 `rustfmt`、`clippy` 组件。完整测试还会调用 `free -b`、
GNU `df -B1 --output=size,used` 和 `systemd-detect-virt`，缺少这些命令会失败；
请先安装提供这些命令的系统软件包。这些工具用于构建和测试，发布二进制运行时不依赖它们。

```bash
rustup component add rustfmt clippy
cargo fmt --all --check
cargo test --locked
cargo clippy --locked --all-targets -- -D warnings
```

本机构建产物为 `target/release/monitor-agent`，使用 Rust 工具链的默认目标平台；
它不一定是静态链接二进制。发布工作流通过 `cross` 为两种架构构建 musl 静态链接产物。

## 发布与 CI

推送 `v*` tag 由 [发布工作流](.github/workflows/release.yml) 构建。
如果 fork 后 GitHub Actions 尚未启用，先在本仓库的 Actions 页面启用工作流。

发布工作流使用 `GITHUB_REPOSITORY` 作为目标，在本仓库执行发布步骤时会发布到 `Blubo127/agent`。
tag 必须与 `Cargo.toml` 的 `version` 一致，例如版本 `1.2.0` 对应 tag `v1.2.0`；
tag 应指向包含定制修改的提交。修改版本号时同步更新 `Cargo.lock` 中本包的版本，确保 `--locked` 构建可用。

| 触发方式 | 工作流成功后的产物 |
|---|---|
| 推送匹配版本的 `v*` tag，或以该 tag 手动运行 | 两种架构的二进制、`sha256sums.txt`，并创建 GitHub Release |
| 从分支手动运行 | 两种架构的二进制 Actions artifacts；不生成校验清单，也不创建 Release |

仅推送分支不会触发发布工作流。手动运行入口要求默认分支上已有 `workflow_dispatch` 工作流，
运行时确认选中了包含所需修改的分支或 tag。
发布步骤使用 `gh release create`；已有同名 Release 时，重新运行不会覆盖它，创建步骤会失败。

[CI 工作流](.github/workflows/ci.yml) 会检查推送到 `main`、`diy` 的代码变更和 Pull Request；
仅改 Markdown 或 `LICENSE` 时跳过。端到端检查从 `Blubo127/monitor` 的默认分支获取测试脚本
（工作流未指定 `ref`，不会随 agent 切换到 `diy`），并下载该仓库 Latest Release 中的
`monitor-hub-x86_64-unknown-linux-musl`。缺少该发布物时下载步骤会失败。

## 检查记录

[源码安全检查报告](check_summary.md) 记录了对特定历史提交的静态检查，适用范围和未完成的验证见报告正文。

## 许可

采用 [MIT License](LICENSE)。本项目基于 `monitor-probe/agent`，保留原作者
`stqfdyr` 的版权和许可声明；本 fork 的修改在 `Blubo127/agent` 中维护。
