# TriScope：多语言程序行为分析引擎

TriScope 是一个面向**授权环境、样本分析、软件行为审计与安全研究**的多语言程序行为分析引擎。项目使用 C++、Rust、Go 分别实现不同职责模块，通过统一的行为事件模型，将目标程序运行过程中的系统行为、执行行为、网络行为进行采集、归一化、关联分析与报告生成。

本项目的核心目标不是简单复刻某个已有逆向工具或扫描器，而是构建一个可扩展的行为分析平台：

```text
C++ 负责高权限系统行为采集
Rust 负责目标程序执行、测试、重放与行为归一化
Go 负责网络行为分析、任务调度、报告生成与控制平面
```

---

## 1. 项目定位

TriScope 主要用于以下场景：

```text
- 授权程序行为分析
- 可疑软件动态行为审计
- 软件安装包行为记录
- 本地程序安全测试
- 沙箱环境中的样本执行记录
- 程序网络行为分析
- 文件、注册表、进程、网络行为时间线构建
- 后续接入 LLM 进行行为解释与报告辅助生成
```

本项目坚持以下边界：

```text
- 仅用于授权环境下的安全研究、教学、测试和防御分析
- 不提供隐蔽驻留、绕过检测、恶意利用、未授权入侵等能力
- 内核侧仅做必要事件采集与低成本过滤，不实现攻击性功能
- 所有分析任务应在受控沙箱、虚拟机或明确授权环境中执行
```

---

## 2. 总体架构

```text
                           ┌──────────────────────────────┐
                           │          GoNetEngine          │
                           │ 网络分析 / 控制平面 / 报告生成 │
                           └───────────────┬──────────────┘
                                           │
                              JSONL / gRPC / Protobuf
                                           │
          ┌────────────────────────────────┼────────────────────────────────┐
          │                                │                                │
┌─────────▼─────────┐            ┌─────────▼─────────┐            ┌─────────▼─────────┐
│  WinKernelEngine  │            │ RustActionEngine  │            │   GoNetEngine     │
│ C++ 高权限采集层  │            │ Rust 执行测试层   │            │ Go 网络分析层     │
└─────────┬─────────┘            └─────────┬─────────┘            └─────────┬─────────┘
          │                                │                                │
          ▼                                ▼                                ▼
  进程/线程/文件/注册表/模块事件       执行/输入/崩溃/重放事件          DNS/HTTP/TLS/PCAP/连接事件

                                           │
                                           ▼

                             ┌────────────────────────┐
                             │ Unified Behavior Model │
                             │ 统一行为事件中间表示   │
                             └────────────────────────┘
                                           │
                                           ▼
                                时间线 / 行为图 / 风险报告
```

---

## 3. 模块职责

### 3.1 WinKernelEngine

语言：C++ / C++23  
定位：Windows 高权限系统行为采集引擎

主要负责：

```text
- 进程创建、退出、父子进程关系采集
- 线程创建与可疑线程行为记录
- 文件创建、读取、写入、删除、重命名记录
- 注册表键和值变更记录
- DLL / 模块加载记录
- 句柄访问与部分跨进程访问行为记录
- 网络连接与进程关联的基础采集
- 将底层事件输出为标准化原始事件流
```

不负责：

```text
- 漏洞利用
- 绕过安全软件
- 隐藏进程、隐藏文件、隐藏驱动
- 复杂策略判断
- 报告生成
- LLM 分析
```

### 3.2 RustActionEngine

语言：Rust  
定位：目标程序执行、测试、输入生成、重放与事件归一化引擎

主要负责：

```text
- 启动目标程序
- 管理运行环境
- 控制超时、退出码、stdout/stderr
- 生成 run_id 并管理单次分析目录
- 输入文件、参数、环境变量、配置文件的测试组合
- 简单变异测试
- 崩溃检测与崩溃输入保存
- 分析任务重放
- 读取 WinKernelEngine 和 GoNetEngine 的事件
- 将原始事件归一化为统一 BehaviorEvent
- 构建行为时间线
```

### 3.3 GoNetEngine

语言：Go  
定位：网络行为分析、FakeNet、任务调度、控制平面与报告生成

主要负责：

```text
- PCAP 文件读取与实时流量分析
- DNS 查询记录
- HTTP 请求与响应元数据分析
- TLS ClientHello、SNI、证书与协议元数据分析
- TCP/UDP 五元组提取
- 进程与网络连接关联
- Fake DNS / Fake HTTP / Fake HTTPS 测试环境
- Web API / CLI 控制平面
- 分析任务调度
- 事件聚合与存储
- Markdown / HTML / JSON 报告生成
```

---

## 4. 推荐仓库结构

```text
triscope/
├── README.md
├── LICENSE
├── docs/
│   ├── architecture.md
│   ├── event-model.md
│   ├── development-roadmap.md
│   ├── security-boundary.md
│   └── examples.md
├── schema/
│   ├── behavior_event.schema.json
│   ├── run_metadata.schema.json
│   └── finding.schema.json
├── engines/
│   ├── WinKernelEngine/
│   │   ├── README.md
│   │   ├── driver/
│   │   ├── service/
│   │   ├── include/
│   │   ├── tests/
│   │   └── CMakeLists.txt
│   ├── RustActionEngine/
│   │   ├── README.md
│   │   ├── crates/
│   │   │   ├── action-cli/
│   │   │   ├── action-core/
│   │   │   ├── action-runner/
│   │   │   ├── action-mutator/
│   │   │   ├── action-normalizer/
│   │   │   └── action-replay/
│   │   └── Cargo.toml
│   └── GoNetEngine/
│       ├── README.md
│       ├── cmd/
│       ├── internal/
│       │   ├── analyzer/
│       │   ├── fakenet/
│       │   ├── pcap/
│       │   ├── api/
│       │   ├── report/
│       │   └── storage/
│       └── go.mod
├── examples/
│   ├── sample-run/
│   │   ├── metadata.json
│   │   ├── kernel_events.jsonl
│   │   ├── action_events.jsonl
│   │   ├── network_events.jsonl
│   │   ├── normalized_events.jsonl
│   │   └── report.md
│   └── targets/
└── scripts/
    ├── build_all.ps1
    ├── run_demo.ps1
    └── clean_runs.ps1
```

---

## 5. 统一行为事件模型

所有模块最终都应输出或转换为统一的 `BehaviorEvent`。

示例：

```json
{
  "run_id": "run-20260706-0001",
  "event_id": "evt-00000001",
  "timestamp": 1783300000000,
  "source": "kernel",
  "event_type": "file_write",
  "process": {
    "pid": 4321,
    "ppid": 1000,
    "image": "C:\\Sandbox\\sample.exe",
    "command_line": "sample.exe input.bin"
  },
  "object": {
    "type": "file",
    "path": "C:\\Users\\test\\AppData\\Local\\Temp\\drop.dll"
  },
  "attributes": {
    "access": "write",
    "size": "4096"
  },
  "tags": [
    "temp_directory",
    "dll_file"
  ],
  "severity_hint": "medium"
}
```

建议的事件类型：

```text
process_create
process_exit
thread_create
file_create
file_read
file_write
file_delete
file_rename
registry_create_key
registry_set_value
registry_delete_value
module_load
network_connect
dns_query
http_request
http_response
tls_client_hello
memory_alloc
memory_protect
handle_open
crash
timeout
runner_start
runner_exit
```

---

## 6. 单次分析目录结构

每次分析任务生成一个独立目录：

```text
runs/
└── run-20260706-0001/
    ├── metadata.json
    ├── target/
    │   └── sample.exe
    ├── inputs/
    │   ├── seed.bin
    │   └── mutated-0001.bin
    ├── logs/
    │   ├── stdout.log
    │   ├── stderr.log
    │   └── engine.log
    ├── events/
    │   ├── kernel_events.jsonl
    │   ├── action_events.jsonl
    │   ├── network_events.jsonl
    │   └── normalized_events.jsonl
    ├── crashes/
    ├── pcap/
    │   └── traffic.pcap
    ├── artifacts/
    └── report/
        ├── report.md
        ├── report.html
        └── summary.json
```

---

## 7. 开发阶段规划

### Phase 1：最小闭环

目标：三模块能通过文件事件流协作。

```text
WinKernelEngine:
- 可先使用用户态 mock collector 或 ETW consumer
- 输出简单 process/file 事件

RustActionEngine:
- 启动目标程序
- 控制 timeout
- 保存 stdout/stderr
- 输出 action_events.jsonl

GoNetEngine:
- 读取 network_events.jsonl mock 数据
- 生成基础 Markdown 报告
```

交付物：

```text
- 一个可执行目标程序
- 一次完整 run 目录
- 三类事件文件
- 一份基础报告
```

### Phase 2：真实系统事件采集

```text
WinKernelEngine:
- 增加真实系统行为采集
- 加强进程、文件、注册表事件

RustActionEngine:
- 增加事件归一化器
- 构建行为时间线

GoNetEngine:
- 增加事件聚合与 SQLite 存储
```

### Phase 3：网络分析闭环

```text
GoNetEngine:
- 支持 pcap 解析
- 支持 DNS/HTTP/TLS 元数据分析
- 支持 Fake DNS / Fake HTTP
```

### Phase 4：测试策略与重放

```text
RustActionEngine:
- 增加参数变异
- 增加文件输入变异
- 增加 corpus 管理
- 增加 crash 保存
- 增加 replay 支持
```

### Phase 5：报告、规则与 LLM 接入

```text
GoNetEngine:
- 生成 HTML 报告
- 增加规则引擎
- 增加行为图
- 预留 LLM 解释接口
```

---

## 8. 运行示例

```bash
# 1. 创建分析任务
gonetctl run create --target C:\Sandbox\sample.exe --profile default

# 2. 启动网络分析环境
gonetctl fakenet start --run-id run-20260706-0001

# 3. 启动系统行为采集
winkernelctl start --run-id run-20260706-0001

# 4. 启动目标程序测试执行
actionctl run --run-id run-20260706-0001 --target C:\Sandbox\sample.exe --timeout 60s

# 5. 停止采集并聚合事件
winkernelctl stop --run-id run-20260706-0001
gonetctl analyze --run-id run-20260706-0001

# 6. 生成报告
gonetctl report --run-id run-20260706-0001 --format html
```

---

## 9. 安全与合规说明

TriScope 仅面向授权分析、教学研究和防御型安全工程。

请不要将本项目用于：

```text
- 未授权目标分析
- 恶意程序开发
- 绕过安全产品
- 隐蔽执行
- 权限维持
- 数据窃取
- 入侵、破坏、横向移动等攻击行为
```

推荐运行环境：

```text
- Windows 虚拟机
- 快照可恢复环境
- 无敏感数据的隔离网络
- 受控 FakeNet 环境
- 明确授权的测试样本
```

---

## 10. 总结

TriScope 的核心不是简单地把 C++、Rust、Go 放在一个仓库里，而是用三种语言分别解决最适合它们的问题：

```text
C++：看得见底层行为
Rust：测得稳、可重放、可归一化
Go：管得住网络、任务和报告
```

最终目标是形成一个模块化、可扩展、可审计的程序行为分析平台。
