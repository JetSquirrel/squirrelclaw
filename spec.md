# catpaw 项目技术规格说明书

## 1. 项目概述

### 1.1 项目定位

**catpaw** 是一个轻量级智能主机监控 Agent，使用 Go 语言开发，提供单二进制部署方式。项目定位类似于 Nagios/Sensu，但更现代化，专注于异常检测而非指标采集。

### 1.2 核心特性

- **🪶 轻量无重依赖**：单二进制文件，无需复杂的运行时环境
- **🔌 插件化监控**：25+ 检查插件，按需启用
- **🤖 AI 自动诊断**：告警触发后自动调用 AI 进行根因分析
- **💬 AI 交互排障**：命令行对话式故障排查，AI + 工具联动
- **🩺 主动健康巡检**：支持按需对目标执行 AI 驱动的深度检查
- **🛠️ 70+ 诊断工具**：覆盖系统、网络、存储、安全、进程、内核等各个维度
- **🔗 MCP 集成**：通过 Model Context Protocol 接入 Prometheus、Jaeger、CMDB 等外部数据源
- **📡 灵活通知**：支持控制台、通用 WebAPI、Flashduty、PagerDuty 等多种通知渠道
- **🔄 自监控友好**：适合监控系统的监控系统，避免循环依赖

### 1.3 技术栈

- **语言**：Go 1.25.5
- **主要依赖**：
  - BurntSushi/toml：配置文件解析
  - shirou/gopsutil：系统信息采集
  - go.uber.org/zap：结构化日志
  - prometheus-community/pro-bing：ICMP ping 功能
  - ergochat/readline：交互式 CLI
  - golang.org/x/sys：底层系统调用

## 2. 系统架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        catpaw agent                             │
│                                                                 │
│  ┌─────────────┐   告警    ┌──────────────┐    AI + 工具       │
│  │  25+ 检查   │ ────────── │  AI 诊断    │ ──────────────┐   │
│  │    插件     │   触发     │    引擎     │               │   │
│  └──────┬──────┘            └──────────────┘               │   │
│         │                                                  ▼   │
│         │ 事件      ┌──────────────┐         ┌───────────────┐ │
│         └────────── │   通知渠道   │         │  70+ 诊断    │ │
│                     │  （多选）    │         │     工具     │ │
│                     └──────────────┘         └───────┬───────┘ │
│                                                      │         │
│  ┌─────────────┐                            ┌────────┴───────┐ │
│  │  AI Chat    │ ───── 交互式排障 ───────── │  MCP 外部     │ │
│  │  (命令行)   │                            │  数据源       │ │
│  └─────────────┘                            └────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心模块

#### 2.2.1 目录结构

```
catpaw/
├── main.go              # CLI 入口：run/chat/inspect/diagnose/selftest/mcptest
├── agent/               # Agent 生命周期管理、插件加载、Runner 调度
├── engine/              # 事件处理引擎：去重、告警判定、恢复、触发诊断
├── plugins/             # 25+ 检查插件，每个子目录一个插件
│   └── plugins.go       # 插件注册表 + 核心接口定义
├── diagnose/            # AI 诊断子系统
│   ├── engine.go        # 诊断引擎主循环
│   ├── aggregator.go    # 告警聚合器
│   ├── registry.go      # 工具注册表
│   ├── prompt.go        # AI 提示词模板
│   ├── executor.go      # 工具执行路由
│   └── types.go         # 核心类型定义
├── chat/                # 交互式 AI Chat REPL
├── mcp/                 # MCP (Model Context Protocol) 客户端
├── notify/              # 通知后端
│   ├── console.go       # 控制台输出
│   ├── webapi.go        # 通用 HTTP 推送
│   ├── flashduty.go     # Flashduty 平台
│   └── pagerduty.go     # PagerDuty 平台
├── config/              # 配置结构定义与解析
├── types/               # 核心类型：Event、状态常量
├── logger/              # 日志封装
├── pkg/                 # 通用工具包
├── conf.d/              # 默认配置目录
│   ├── config.toml      # 全局配置
│   └── p.<plugin>/      # 各插件配置
├── state.d/             # 运行时状态
├── design.d/            # 设计原则文档
└── docs/                # 用户文档
```

### 2.3 数据流

```
Plugins.Gather() → types.Event → engine.PushRawEvents()
  → handleAlertEvent() → notify.Forward()
  → mayTriggerDiagnose() → DiagnoseAggregator → DiagnoseEngine
  → AI 多轮对话 → 诊断报告 Event → notify.Forward()
```

详细流程：

1. **采集阶段**：`agent.PluginRunner` 按 interval 定时调用 `Instance.Gather(queue)`
2. **事件生成**：插件将检查结果封装为 `types.Event`，推入 queue
3. **事件处理**：`engine.PushRawEvents()` 消费 queue
   - 补充时间戳、合并 Labels
   - 计算 AlertKey（Labels 排序拼接 → MD5）
   - Ok 事件 → `handleRecoveryEvent()`
   - 告警事件 → `handleAlertEvent()`
4. **通知发送**：满足条件则调用 `notify.Forward()`
5. **AI 诊断**：告警发送后 → `mayTriggerDiagnose()`
6. **聚合处理**：聚合器按 `plugin::target` 在时间窗口（默认 5s）内聚合
7. **诊断执行**：`DiagnoseEngine.Submit()` → 信号量控制并发 → `RunDiagnose()`
8. **生成报告**：AI 多轮对话 → 生成报告 → 为每个 AlertKey 创建新事件推送

## 3. 核心功能规格

### 3.1 监控插件系统

#### 3.1.1 插件接口定义

```go
// 核心接口
type Plugin interface {
    GetLabels() map[string]string
    GetInterval() int64
}

type Instance interface {
    GetLabels() map[string]string
    GetInterval() int64
    GetAlerting() *config.Alerting
    GetDiagnoseConfig() *config.DiagnoseConfig
}

// 必须实现
type Gatherer interface {
    Gather(*safe.Queue[*types.Event])
}

// 可选接口
type Initer interface {
    Init() error
}

type Dropper interface {
    Drop()
}

type Diagnosable interface {
    RegisterDiagnoseTools(registry *diagnose.ToolRegistry)
}

type InstancesGetter interface {
    GetInstances() []Instance
}

type IApplyPartials interface {
    ApplyPartials() error
}
```

#### 3.1.2 内置插件列表（25+）

| 插件 | 说明 | 平台 |
|------|------|------|
| `cert` | TLS 证书有效期检查（远程 + 本地文件） | 跨平台 |
| `conntrack` | 连接跟踪表使用率监控 | Linux |
| `cpu` | CPU 使用率、归一化每核 Load Average | 跨平台 |
| `disk` | 磁盘空间、inode、可写性检查 | 跨平台 |
| `dns` | DNS 解析检查 | 跨平台 |
| `docker` | Docker 容器监控 | 跨平台 |
| `exec` | 执行脚本/命令产生事件 | 跨平台 |
| `filecheck` | 文件存在性、mtime、checksum | 跨平台 |
| `filefd` | 系统级文件描述符使用率 | Linux |
| `http` | HTTP 可用性、状态码、响应体、证书 | 跨平台 |
| `journaltail` | journalctl 增量日志读取 | Linux |
| `logfile` | 日志文件监控（轮转、多编码） | 跨平台 |
| `mem` | 内存、Swap 使用率检查 | 跨平台 |
| `mount` | 挂载点基线检查 | Linux |
| `neigh` | ARP/邻居表使用率监控 | Linux |
| `net` | TCP/UDP 连通性与响应时间 | 跨平台 |
| `netif` | 网卡健康检查 | Linux |
| `ntp` | NTP 同步状态、时钟偏移 | Linux |
| `ping` | ICMP 可达性、丢包率、时延 | 跨平台 |
| `procfd` | 进程级 fd 使用率监控 | Linux |
| `procnum` | 进程数量检查 | 跨平台 |
| `redis` | Redis 监控（单机/主从/集群） | 跨平台 |
| `redis_sentinel` | Redis Sentinel 监控 | 跨平台 |
| `scriptfilter` | 脚本输出行过滤匹配 | 跨平台 |
| `secmod` | SELinux/AppArmor 基线检查 | Linux |
| `sockstat` | TCP listen 队列溢出检测 | Linux |
| `sysctl` | 内核参数基线检查 | Linux |
| `systemd` | systemd 服务状态检查 | Linux |
| `tcpstate` | TCP 连接状态监控 | Linux |
| `uptime` | 系统异常重启检测 | 跨平台 |
| `zombie` | 僵尸进程检测 | 跨平台 |

### 3.2 事件数据模型

#### 3.2.1 Event 结构

```go
type Event struct {
    EventTime         int64              // Unix 时间戳
    EventStatus       string             // Critical/Warning/Info/Ok
    AlertKey          string             // Labels 的 MD5，唯一标识
    Labels            map[string]string  // 身份标签，参与 AlertKey
    Attrs             map[string]string  // 展示属性，不参与 AlertKey
    Description       string             // 纯文本描述
    DescriptionFormat string             // text/markdown

    // 内部字段
    FirstFireTime     int64
    NotifyCount       int64
    LastSent          int64
}
```

#### 3.2.2 Labels 约定

**必须字段**：
- `from_plugin`：产出事件的插件名
- `from_agent`：固定为 `catpaw`
- `from_hostname`：主机名
- `from_hostip`：主机 IP
- `check`：检查维度，格式为 `plugin::dimension`（如 `disk::space_usage`）
- `target`：检查对象标识

**可选字段**：
- `protocol`：协议（net 插件特有）
- `method`：HTTP 方法（http 插件特有）
- 用户自定义标签

#### 3.2.3 Attrs 约定

动态度量数据，不参与 AlertKey 计算：
- `current_value`：触发告警的主指标值
- `threshold_desc`：人类可读的阈值描述
- `used_percent`：使用率
- `response_time`：响应时间
- `packet_loss`：丢包率
- 等等

#### 3.2.4 AlertKey 生成规则

```
sort labels by key → for each key: "key:value:" → MD5(concatenated string)
```

### 3.3 AI 诊断系统

#### 3.3.1 诊断工具分类（70+）

**⚙️ 系统与进程**：
- CPU Top、内存分布、OOM 历史
- cgroup 限制/用量
- 进程线程（含 wchan）
- 打开文件列表、环境变量
- PSI 压力指标

**🌐 网络**：
- ping、traceroute、DNS 解析
- ARP 邻居表、TCP 连接状态
- Socket 详情（RTT/cwnd）
- 重传率、连接延迟分布
- Listen 队列溢出
- TCP 内核调优检查
- softnet 统计、路由表
- IP 地址、网卡流量
- 防火墙规则

**💾 存储**：
- 磁盘 I/O 延迟
- 块设备拓扑树
- LVM 状态
- 挂载信息

**🔐 内核与安全**：
- dmesg 内核日志
- 中断分布
- conntrack 统计
- NUMA 内存分布
- 热区温度
- sysctl 快照
- SELinux/AppArmor 状态
- coredump 列表

**📜 日志**：
- 日志尾部读取
- 日志 grep（模式匹配）
- journald 查询

**🐳 服务**：
- systemd 服务状态
- 失败服务列表
- 定时器列表
- Docker ps/inspect

**🔌 远程插件专用工具**：
- Redis：INFO、CONFIG GET、SLOWLOG、CLIENT LIST 等
- Redis Sentinel：SENTINEL MASTERS、SENTINEL SLAVES 等

#### 3.3.2 DiagnoseTool 结构

```go
type DiagnoseTool struct {
    Name        string
    Description string
    Parameters  []ToolParam
    Scope       ToolScope              // Local / Remote
    Execute     func(ctx, args) (string, error)
    RemoteExecute func(ctx, session, args) (string, error)
}
```

#### 3.3.3 诊断触发机制

1. **自动触发**：告警发送后自动调用
2. **手动巡检**：`catpaw inspect <plugin> [target]`
3. **交互 Chat**：`catpaw chat` 命令行对话

#### 3.3.4 诊断聚合

- 按 `plugin::target` 分组
- 时间窗口内聚合（默认 5s）
- 单个诊断会话包含该时间窗口内的所有相关告警

#### 3.3.5 并发控制

- 信号量控制并发诊断数（默认 3）
- 冷却期机制（默认 30m）
- 每日 Token 额度限制

### 3.4 MCP 集成

#### 3.4.1 MCP 协议

通过 stdio JSON-RPC 2.0 与外部 MCP Server 通信。

#### 3.4.2 支持的数据源

- Prometheus（历史指标）
- Jaeger（链路追踪）
- Nightingale（监控平台）
- CMDB（配置管理数据库）
- 任何符合 MCP 协议的服务

#### 3.4.3 配置示例

```toml
[ai.mcp]
enabled = true

[[ai.mcp.servers]]
name = "prometheus"
command = "/usr/local/bin/mcp-prometheus"
args = ["serve"]
identity = 'instance="${IP}:9100"'
[ai.mcp.servers.env]
PROMETHEUS_URL = "http://127.0.0.1:9090"
```

#### 3.4.4 工具管理

- 工具注册到 `ToolRegistry`
- 类别前缀 `mcp:`
- 工具名前缀 `{server}_`
- 白名单过滤：`tools_allow` 配置
- 身份标识：`identity` 字段

### 3.5 通知系统

#### 3.5.1 支持的通知渠道

| 渠道 | 配置段 | 说明 |
|------|--------|------|
| Console | `[notify.console]` | 终端彩色输出（默认启用） |
| WebAPI | `[notify.webapi]` | 通用 HTTP 推送 |
| Flashduty | `[notify.flashduty]` | Flashduty 告警平台 |
| PagerDuty | `[notify.pagerduty]` | PagerDuty 事件管理 |

#### 3.5.2 通知接口

```go
type Notifier interface {
    Name() string
    Forward(event *types.Event) error
}
```

#### 3.5.3 特性

- 支持多渠道同时启用
- HTTP 类 Notifier 支持重试退避
- 支持超时控制
- 支持自定义 Headers

## 4. CLI 命令规格

### 4.1 命令列表

| 命令 | 说明 | 主要参数 |
|------|------|----------|
| `catpaw run` | 启动监控 Agent | `--interval`、`--plugins` |
| `catpaw chat` | 交互式 AI 对话 | `-v`、`--model` |
| `catpaw inspect` | 主动 AI 健康检查 | `<plugin>`、`[target]` |
| `catpaw diagnose list` | 列出诊断记录 | - |
| `catpaw diagnose show` | 查看诊断详情 | `<id>` |
| `catpaw selftest` | 诊断工具冒烟测试 | `[filter]`、`-q` |
| `catpaw mcptest` | 测试 MCP 连接 | - |
| `catpaw help` | 查看帮助 | `[command]` |

### 4.2 全局参数

- `--configs <dir>`：配置目录（默认：conf.d）
- `--loglevel <lvl>`：日志级别（debug/info/warn/error）
- `--version`：显示版本

### 4.3 命令详细说明

#### 4.3.1 catpaw run

启动监控 Agent，执行采集、告警、诊断等所有功能。

```bash
catpaw run [flags]

Flags:
  --interval <sec>    覆盖全局采集间隔（秒）
  --plugins <list>    插件过滤器，冒号分隔（如 redis:cpu:disk）
```

#### 4.3.2 catpaw chat

启动交互式 AI 对话会话，用于故障排查。

```bash
catpaw chat [-v] [--model <name>]

Flags:
  -v, --verbose       详细模式（显示工具输出摘要）
  --model <name>      指定模型（跳过故障转移）

交互命令：
  /models             列出所有配置的模型
  /model <name>       切换到指定模型
  /model auto         恢复优先级故障转移
```

#### 4.3.3 catpaw inspect

执行主动健康巡检（AI 驱动）。

```bash
catpaw inspect <plugin> [target]

示例：
  catpaw inspect redis 10.0.0.1:6379   # 巡检 Redis 实例
  catpaw inspect cpu                    # 巡检本地 CPU
  catpaw inspect mem                    # 巡检本地内存
  catpaw inspect disk                   # 巡检本地磁盘
```

#### 4.3.4 catpaw diagnose

查看和管理诊断记录。

```bash
catpaw diagnose <command>

Commands:
  list          列出最近的记录（最多 50 条）
  show <id>     显示指定记录的完整详情
```

#### 4.3.5 catpaw selftest

对所有诊断工具执行冒烟测试。

```bash
catpaw selftest [filter] [-q]

Options:
  filter      只测试匹配此字符串的类别（如 sysdiag、cpu）
  -q          安静模式：只显示失败和摘要

退出码：
  0           所有测试通过（或跳过/警告）
  1           一个或多个测试失败
```

#### 4.3.6 catpaw mcptest

测试所有配置的 MCP server 连接。

```bash
catpaw mcptest

退出码：
  0           所有服务器连接成功
  1           一个或多个服务器失败
```

## 5. 配置规格

### 5.1 配置文件结构

```
conf.d/
├── config.toml           # 全局配置
├── config.local.toml     # 本地覆盖（git-ignored）
└── p.<plugin>/           # 插件配置目录
    ├── default.toml      # 默认配置
    └── custom.toml       # 自定义配置（多文件合并）
```

### 5.2 加载顺序

```
config.toml → conf.d/ 中其他文件 → config.local.toml
```

### 5.3 全局配置

```toml
[global]
interval = 60                # 默认采集间隔（秒）
labels = { env = "prod" }    # 全局标签

[log]
level = "info"               # 日志级别
filename = "logs/catpaw.log" # 日志文件
max_size = 100               # MB
max_backups = 3              # 保留数量
max_age = 7                  # 天数

[ai]
enabled = true               # 启用 AI 诊断
max_rounds = 20              # 最大对话轮数
aggregate_window = "5s"      # 聚合窗口
language = "zh"              # 语言（zh/en）
daily_token_limit = 1000000  # 每日 Token 额度
model_priority = ["default"] # 模型优先级

[ai.models.default]
base_url = "https://api.openai.com/v1"
api_key = "${OPENAI_API_KEY}"
model = "gpt-4o"
context_window = 128000
input_price = 2.5            # $ per 1M tokens
output_price = 10.0

[ai.mcp]
enabled = true

[[ai.mcp.servers]]
name = "prometheus"
command = "/usr/local/bin/mcp-prometheus"
args = ["serve"]
identity = 'instance="${IP}:9100"'
tools_allow = []             # 空表示全部允许
[ai.mcp.servers.env]
PROMETHEUS_URL = "http://127.0.0.1:9090"

[notify.console]
enabled = true

[notify.webapi]
url = "https://api.example.com/v1/events"
method = "POST"
timeout = "10s"
[notify.webapi.headers]
Authorization = "Bearer ${WEBAPI_TOKEN}"

[notify.flashduty]
integration_key = "your-integration-key"

[notify.pagerduty]
routing_key = "your-routing-key"
```

### 5.4 插件配置

```toml
[[instances]]
targets = ["localhost:6379"]
interval = "30s"
labels = { cluster = "cache" }

[instances.alerting]
for_duration = 0              # 持续时长（秒）
repeat_interval = "5m"        # 重复间隔
repeat_number = 3             # 最大重复次数
disabled = false              # 是否禁用告警
disable_recovery_notification = false

[instances.diagnose]
enabled = true                # 启用诊断
min_severity = "Warning"      # 最小严重级别
timeout = "120s"              # 超时时间
cooldown = "30m"              # 冷却期
```

### 5.5 配置热加载

支持通过 SIGHUP 信号热加载插件配置：

```bash
kill -HUP $(pidof catpaw)
```

## 6. 设计原则

### 6.1 核心原则

1. **告警质量优先**：宁可漏报，不可误报；默认阈值保守
2. **Fail-open**：采集失败本身应产出告警事件，不能静默
3. **优雅降级**：单个 target/instance/plugin 失败不影响其他
4. **开箱即用**：默认配置下载即可运行，无需调整
5. **跨平台**：支持 Linux/Windows/macOS
6. **防 goroutine 泄漏**：可能 hang 的操作必须有 inFlight 防重入 + 超时保护

### 6.2 命名约定

- `check` 格式：`plugin::dimension`
- `threshold_desc` 格式：人类可读的阈值描述，如 `"Warning ≥ 80.0%, Critical ≥ 95.0%"` 或 `"Critical: state ≠ active"`
- 配置文件：`conf.d/p.<plugin>/*.toml`
- 插件包：`plugins/<plugin>/`

### 6.3 平台特定代码

使用 build tags 隔离平台特有逻辑：

```go
//go:build linux

package myplugin
```

## 7. 开发指南

### 7.1 构建和测试

```bash
# 构建
./build.sh

# 运行测试
go test ./...

# 快速验证
./catpaw run --plugins cpu:mem

# 测试诊断工具
./catpaw selftest

# 测试 MCP 连接
./catpaw mcptest
```

### 7.2 新增插件

1. 在 `plugins/<name>/` 创建目录
2. 实现必要接口（见 `docs/plugin-development.md`）
3. 在 `agent/agent.go` 中添加 blank import
4. 在 `conf.d/p.<name>/` 创建默认配置

参考实现：
- 简单 local 插件：`plugins/cpu/`、`plugins/mem/`
- 简单 remote 插件：`plugins/ping/`、`plugins/http/`
- 复杂 remote 插件：`plugins/redis/`（推荐标杆）

### 7.3 新增诊断工具

实现 `Diagnosable` 接口：

```go
func (ins *Instance) RegisterDiagnoseTools(registry *diagnose.ToolRegistry) {
    registry.Register(diagnose.DiagnoseTool{
        Name:        "my_tool",
        Description: "Tool description",
        Category:    "category_name",
        Parameters: []diagnose.ToolParam{
            {Name: "arg1", Type: "string", Required: true},
        },
        Scope:   diagnose.ToolScopeLocal,
        Execute: func(ctx context.Context, args map[string]string) (string, error) {
            // 实现逻辑
            return "result", nil
        },
    })
}
```

### 7.4 新增通知后端

1. 实现 `notify.Notifier` 接口
2. 在 `agent/agent.go` 中注册
3. 在 `config/config.go` 中添加配置结构

## 8. 部署方案

### 8.1 二进制部署

```bash
# 下载
wget https://github.com/cprobe/catpaw/releases/download/v1.0.0/catpaw-linux-amd64

# 添加执行权限
chmod +x catpaw-linux-amd64
mv catpaw-linux-amd64 /usr/local/bin/catpaw

# 运行
catpaw run
```

### 8.2 systemd 服务

```ini
[Unit]
Description=catpaw monitoring agent
After=network.target

[Service]
Type=simple
User=catpaw
WorkingDirectory=/opt/catpaw
ExecStart=/usr/local/bin/catpaw run --configs /etc/catpaw/conf.d
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### 8.3 Docker 部署

```dockerfile
FROM alpine:latest

COPY catpaw /usr/local/bin/
COPY conf.d /app/conf.d

WORKDIR /app
CMD ["/usr/local/bin/catpaw", "run"]
```

## 9. 安全考虑

### 9.1 敏感信息

- API Key 通过环境变量注入：`${OPENAI_API_KEY}`
- 配置文件权限：建议 600
- `config.local.toml` 已加入 .gitignore

### 9.2 执行权限

- 诊断工具使用当前用户权限
- Shell 命令需要用户确认（交互模式）
- 不建议以 root 运行（除非必要）

### 9.3 网络安全

- HTTP Notifier 支持 TLS
- MCP Server 使用 stdio 通信（不暴露端口）
- 支持防火墙规则配置

## 10. 监控指标

### 10.1 内部指标

catpaw 自身不暴露 Prometheus 指标，但会通过事件系统报告自身状态：

- 插件执行失败
- 诊断超时
- Token 额度耗尽
- MCP 连接失败

### 10.2 推荐监控

- catpaw 进程存活
- 日志错误率
- 事件产出速率
- 诊断执行延迟

## 11. 性能特性

### 11.1 资源消耗

- **内存**：基础 50MB，每个插件 5-20MB
- **CPU**：采集时瞬时 5-10%，平时 < 1%
- **磁盘**：日志和状态文件，可配置轮转
- **网络**：仅通知和 AI API 调用

### 11.2 并发控制

- 插件采集：按 interval 顺序执行
- 诊断任务：信号量控制（默认 3）
- MCP 连接：每个 server 独立进程

### 11.3 规模限制

- 单实例建议监控目标数：< 1000
- 并发诊断数：默认 3（可配置）
- 事件队列：无界队列（内存保护）

## 12. 故障排查

### 12.1 常见问题

**Q: 插件采集失败？**
A: 检查 `logs/catpaw.log`，查看具体错误信息

**Q: AI 诊断不工作？**
A:
1. 检查 `[ai] enabled = true`
2. 验证 API Key 是否有效
3. 查看 Token 额度是否耗尽

**Q: 通知发送失败？**
A:
1. 检查网络连通性
2. 验证配置（URL、Key 等）
3. 查看日志详细错误

**Q: 配置热加载不生效？**
A:
1. 仅支持插件配置热加载
2. 全局配置需要重启
3. 检查配置文件语法

### 12.2 调试模式

```bash
catpaw run --loglevel debug
```

### 12.3 工具验证

```bash
# 验证所有诊断工具
catpaw selftest

# 验证特定类别
catpaw selftest sysdiag

# 测试 MCP 连接
catpaw mcptest
```

## 13. 版本和兼容性

### 13.1 Go 版本

- 最低要求：Go 1.25.5
- 推荐使用最新稳定版

### 13.2 平台支持

- **Linux**：完整功能支持（推荐）
- **macOS**：大部分功能支持（部分 Linux 特有插件不可用）
- **Windows**：基础功能支持（Windows 服务模式）

### 13.3 API 兼容性

- OpenAI API：Compatible with v1
- MCP Protocol：Model Context Protocol 规范
- Flashduty API：v1
- PagerDuty Events API：v2

## 14. 许可证

GNU Affero General Public License v3.0 (AGPL-3.0)

## 15. 社区和支持

- **GitHub**: https://github.com/cprobe/catpaw
- **Issues**: https://github.com/cprobe/catpaw/issues
- **微信群**: 添加 `picobyte`，备注 `catpaw`

## 16. 未来规划

基于项目文档，潜在的发展方向：

1. **更多监控插件**：MySQL、PostgreSQL、MongoDB 等
2. **更多 MCP 数据源**：扩展对接更多监控和可观测性平台
3. **Web UI**：提供 Web 界面查看诊断历史和配置
4. **集群模式**：支持多 Agent 协同和中心化管理
5. **自定义 AI 模型**：支持本地模型和私有化部署
6. **增强分析能力**：趋势分析、异常检测、预测性告警

---

**文档版本**: 1.0
**生成日期**: 2026-03-17
**基于项目**: https://github.com/cprobe/catpaw (commit: 5c34d48)
