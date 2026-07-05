# kernal-fuckit
A modulized kernal kit which can cowork with LLM in the future

> A modular Windows kernel security observatory for game client security research, runtime telemetry, evidence-chain reconstruction, and future LLM-assisted analysis.

## 1. 项目简介

**kernal-fuckit** 是一个面向 Windows 游戏客户端安全研究的模块化内核工具箱式引擎。项目目标不是复刻现有逆向工具、调试器或进程查看器，而是构建一个可扩展的底层安全观测引擎，将 Windows 客户端运行过程中的进程、线程、模块、文件、注册表、对象访问、网络和驱动通信行为统一抽象为结构化事件流，并进一步支持基线比对、行为回放、证据链构建、规则分析和后续 LLM 接入。

本项目采用 **C++23-first** 的设计思想：

* 用户态引擎、事件存储、图谱构建、规则引擎、CLI 工具优先使用 C++23；
* 内核态驱动采用 WDK/KMDF 兼容的受限 C++ 子集；
* 所有内核事件统一通过事件总线进入用户态 Broker；
* 分析逻辑尽量放在用户态，内核层只负责可靠观测、低开销过滤和安全传输。

## 2. 项目目标

本项目希望解决的问题是：

> 如何将 Windows 游戏客户端运行过程中的底层行为转化为可查询、可回放、可解释、可接入 LLM 的安全语义数据。

核心目标包括：

1. 建立模块化内核观测能力；
2. 统一进程、模块、文件、注册表、网络等事件格式；
3. 支持客户端运行过程的基线、差异、回放和证据链分析；
4. 为后续 LLM 安全分析助手提供结构化事实基础；
5. 构建一个长期可扩展的 Windows 游戏客户端安全研究平台。

## 3. 非目标与安全边界

本项目仅用于授权环境下的 Windows 客户端安全研究、防护验证、内核开发学习和实验平台建设。

本项目不实现、不提供、不支持以下功能：

* 外挂开发；
* 游戏作弊功能；
* 注入第三方进程；
* 任意读写其他进程内存；
* 隐藏进程、隐藏驱动、隐藏模块；
* 绕过反作弊；
* 绕过驱动签名；
* PatchGuard 绕过；
* SSDT Hook；
* Inline Hook；
* 未授权分析真实商业游戏客户端；
* 对真实用户环境进行未授权监控。

推荐实验对象为：

* 自研 Toy Game Client；
* 自建测试程序；
* 明确授权的实验样本；
* Windows 虚拟机测试环境。

## 4. 核心设计思想

### 4.1 内核只做观测，不做复杂推理

内核层模块负责：

* 采集事件；
* 初步过滤；
* 统一封装；
* 安全传输到用户态。

复杂分析逻辑放在用户态完成：

* 规则匹配；
* 基线比对；
* 事件关联；
* 图谱构建；
* 风险解释；
* 报告生成；
* LLM 接入。

这样可以降低内核崩溃风险，并提升系统可维护性。

### 4.2 统一事件模型

所有内核模块都不直接输出私有格式，而是统一生成结构化事件：

```json
{
  "version": 1,
  "event_id": 10001,
  "event_type": "image_load",
  "timestamp": 1720000000000,
  "source": "kimage",
  "process": {
    "pid": 4321,
    "create_time": 1720000000000,
    "image_path": "C:\\Lab\\ToyGameClient\\ToyGameClient.exe"
  },
  "thread": {
    "tid": 8848
  },
  "object": {
    "type": "image",
    "path": "C:\\Lab\\ToyGameClient\\game_core.dll"
  },
  "operation": "load",
  "result": "success",
  "status": "0x00000000"
}
```

统一事件模型是后续图谱构建、行为回放、LLM 接入的基础。

### 4.3 事件驱动架构

系统整体数据流如下：

```text
Kernel Sensors
    ↓
KEventBus
    ↓
User-mode Broker
    ↓
EventStore
    ↓
GraphBuilder / RuleEngine / ReplayEngine
    ↓
ReportEngine / LLM Adapter
```

### 4.4 模块化工具箱

每个内核模块都是一个独立传感器：

* KProcess：进程与线程观测；
* KImage：镜像与模块加载观测；
* KObject：对象访问审计；
* KFile：文件系统观测；
* KRegistry：注册表观测；
* KNetwork：网络元数据观测；
* KMemoryMap：内存区域元数据观测；
* KIoctlAudit：驱动通信面审计；
* KTrace：驱动自身可观测性；
* KPolicy：轻量过滤策略。

模块之间不直接耦合，所有模块统一向 KEventBus 提交事件。

## 5. 项目架构

```text
WinKernelSecEngine/
├── apps/
│   ├── wks_cli/                    # 命令行入口
│   ├── wks_broker/                 # 用户态事件 Broker
│   └── toy_game_client/            # 授权实验客户端
│
├── core/
│   ├── event_model/                # 统一事件模型
│   ├── event_store/                # 事件存储
│   ├── graph_builder/              # 行为图谱构建
│   ├── rule_engine/                # 规则分析引擎
│   ├── replay_engine/              # 行为回放引擎
│   ├── report_engine/              # 报告生成
│   └── common/                     # 通用工具库
│
├── sensors/
│   ├── pe_sensor/                  # PE 静态信息采集
│   ├── process_sensor/             # 用户态进程信息采集
│   ├── file_integrity_sensor/      # 用户态文件完整性扫描
│   └── memory_map_sensor/          # 用户态内存区域元数据采集
│
├── kernel/
│   ├── kcore/                      # 内核基础框架
│   ├── keventbus/                  # 内核事件总线
│   ├── kprocess/                   # 进程/线程观测
│   ├── kimage/                     # 镜像加载观测
│   ├── kobject/                    # 对象访问审计
│   ├── kfile/                      # 文件系统 minifilter
│   ├── kregistry/                  # 注册表观测
│   ├── knetwork/                   # WFP 网络元数据观测
│   ├── kmemorymap/                 # 内存区域元数据观测
│   ├── kioctl_audit/               # IOCTL 通信审计
│   ├── ktrace/                     # 内核追踪与诊断
│   └── kpolicy/                    # 内核轻量策略过滤
│
├── schemas/
│   ├── event.schema.json           # 事件格式定义
│   ├── graph.schema.json           # 图谱格式定义
│   ├── rule.schema.json            # 规则格式定义
│   └── report.schema.json          # 报告格式定义
│
├── rules/
│   ├── baseline.yaml               # 基线规则
│   ├── module_rules.yaml           # 模块加载规则
│   ├── file_rules.yaml             # 文件完整性规则
│   └── object_access_rules.yaml    # 对象访问规则
│
├── docs/
│   ├── design.md                   # 总体设计文档
│   ├── event_model.md              # 事件模型说明
│   ├── kernel_modules.md           # 内核模块设计
│   ├── threat_model.md             # 安全边界与威胁模型
│   ├── llm_adapter_design.md       # 后续 LLM 接入设计
│   └── development_guide.md        # 开发指南
│
├── tests/
│   ├── unit/                       # 单元测试
│   ├── integration/                # 集成测试
│   └── samples/                    # 测试样本
│
├── scripts/
│   ├── build.ps1
│   ├── format.ps1
│   ├── test.ps1
│   └── setup_dev_env.ps1
│
├── CMakeLists.txt
├── README.md
├── LICENSE
└── SECURITY.md
```

## 6. 模块设计

### 6.1 KCore：内核基础框架

KCore 是所有内核模块的基础运行时。

职责：

* 驱动加载与卸载；
* 模块生命周期管理；
* 内核日志封装；
* 内存分配封装；
* 锁与同步封装；
* 配置管理；
* 统一错误码；
* IOCTL 通信基础；
* 健康状态统计。

KCore 原则：

* 禁止各模块直接随意分配内存；
* 禁止各模块直接定义私有通信协议；
* 禁止各模块直接向用户态输出非统一格式数据；
* 所有模块统一通过 KEventBus 提交事件。

### 6.2 KEventBus：内核事件总线

KEventBus 负责接收各内核模块提交的事件，并将事件安全传输到用户态 Broker。

职责：

* 事件格式校验；
* 事件版本管理；
* 事件缓存；
* 批量传输；
* 丢失统计；
* 事件过滤；
* 背压控制。

基础事件头：

```cpp
struct EventHeader {
    uint16_t version;
    uint16_t type;
    uint32_t size;
    uint64_t sequence_id;
    uint64_t timestamp;
    uint32_t process_id;
    uint32_t thread_id;
    uint32_t flags;
};
```

### 6.3 KProcess：进程与线程观测引擎

职责：

* 进程创建事件；
* 进程退出事件；
* 父子进程关系；
* 进程镜像路径；
* 进程创建时间；
* 线程创建事件；
* 线程退出事件；
* 进程身份归一化。

进程身份不应只依赖 PID，而应使用：

```text
ProcessIdentity = PID + CreateTime + ImagePath + ImageHash
```

原因：Windows PID 会复用，单独使用 PID 无法稳定表示一次进程生命周期。

### 6.4 KImage：镜像与模块加载观测引擎

职责：

* EXE 加载事件；
* DLL 加载事件；
* SYS 镜像加载事件；
* 模块路径；
* 模块基址；
* 模块大小；
* 加载进程；
* 加载时间；
* 与文件基线关联。

典型问题：

* 客户端加载了哪些 DLL？
* 这些 DLL 是否来自客户端目录？
* 是否加载了非基线模块？
* 是否存在未签名模块？
* 文件被修改后是否随后被加载？

签名校验和复杂哈希计算优先放在用户态完成。

### 6.5 KObject：对象访问审计引擎

职责：

* 进程对象访问审计；
* 线程对象访问审计；
* 源进程记录；
* 目标进程记录；
* 请求访问权限记录；
* 授予访问权限记录；
* 高权限访问模式识别。

第一阶段仅做审计与记录，不做强阻断。

示例问题：

* 哪些进程尝试访问 ToyGameClient？
* 是否有进程请求过高权限？
* 是否存在频繁对象访问行为？
* 目标线程是否被异常打开？

### 6.6 KFile：文件系统观测引擎

KFile 后期基于文件系统 minifilter 实现。

职责：

* 文件创建；
* 文件打开；
* 文件读取；
* 文件写入；
* 文件重命名；
* 文件删除；
* 文件关闭；
* 路径归一化；
* 写入者进程关联。

第一阶段可以先由用户态 FileIntegritySensor 实现目录扫描和基线比对，后续再引入 minifilter。

示例证据链：

```text
Process A writes game_core.dll
        ↓
game_core.dll hash differs from baseline
        ↓
ToyGameClient.exe loads game_core.dll
        ↓
Alert: modified module loaded by game client
```

### 6.7 KRegistry：注册表观测引擎

职责：

* 注册表键创建；
* 注册表值设置；
* 注册表值删除；
* 注册表键删除；
* 关键路径监控；
* 写入者进程关联。

关注路径示例：

* Run / RunOnce；
* Services；
* Drivers；
* Image File Execution Options；
* AppInit_DLLs；
* KnownDLLs；
* COM 相关注册项。

第一阶段仅做观测和告警，不修改注册表。

### 6.8 KNetwork：网络元数据观测引擎

KNetwork 后期基于 WFP 扩展。

第一阶段仅记录网络元数据，不记录 payload。

职责：

* 源进程；
* 本地地址；
* 远端地址；
* 本地端口；
* 远端端口；
* 协议；
* 方向；
* 连接时间；
* 连接状态。

示例问题：

* 客户端连接了哪些远端地址？
* 模块加载前后网络行为是否变化？
* 是否出现非基线端点？
* 某次运行是否比正常运行多出网络连接？

### 6.9 KMemoryMap：内存区域元数据观测引擎

该模块仅做元数据审计，不做任意读写进程内存。

职责：

* 内存区域摘要；
* 映像映射区域；
* 私有可执行区域；
* 区域权限；
* 可疑 RWX 区域元数据；
* 模块列表与内存区域对应关系。

禁止功能：

* 进程内存搜索；
* 特征码扫描真实游戏；
* 修改进程内存；
* 任意读写进程内存。

### 6.10 KIoctlAudit：驱动通信面审计引擎

该模块用于审计自研驱动或授权驱动的 IOCTL 接口面。

职责：

* 记录设备对象；
* 记录符号链接；
* 记录 IOCTL code；
* 记录输入缓冲区长度；
* 记录输出缓冲区长度；
* 记录调用进程；
* 记录返回状态；
* 生成接口文档；
* 检查危险通信模式。

第一阶段仅审计本项目内部驱动通信。

### 6.11 KTrace：内核追踪与诊断模块

职责：

* 驱动内部日志；
* 事件丢失统计；
* 模块状态；
* 性能计数；
* 错误路径记录；
* 调试辅助信息；
* 蓝屏前关键状态记录。

驱动开发必须从第一天开始考虑可观测性。

### 6.12 KPolicy：轻量内核策略过滤模块

KPolicy 只负责降噪，不负责复杂规则判断。

支持策略：

* 只关注指定进程；
* 只关注指定目录；
* 只关注指定扩展名；
* 只关注指定注册表路径；
* 只关注指定远端端口；
* 只关注高风险对象访问权限。

复杂规则放到用户态 RuleEngine。

## 7. 用户态核心模块

### 7.1 Broker

Broker 是内核事件和用户态分析引擎之间的中转层。

职责：

* 接收内核事件；
* 校验事件格式；
* 补充用户态上下文；
* 批量写入 EventStore；
* 下发 KPolicy；
* 处理驱动健康状态；
* 管理采集会话。

### 7.2 EventStore

EventStore 用于存储结构化事件。

初期支持：

* JSONL；
* SQLite。

后续可扩展：

* DuckDB；
* Parquet；
* Neo4j；
* 自定义二进制事件日志。

### 7.3 GraphBuilder

GraphBuilder 将事件流转换为行为图谱。

实体类型：

* Process；
* Thread；
* Image；
* File；
* RegistryKey；
* NetworkEndpoint；
* DriverDevice；
* IoctlInterface；
* RunSession。

关系类型：

* created；
* exited；
* loaded；
* opened；
* wrote；
* renamed；
* deleted；
* connected_to；
* accessed；
* called_ioctl；
* modified；
* derived_from。

### 7.4 RuleEngine

RuleEngine 负责确定性规则分析。

示例规则：

```yaml
id: modified_module_loaded
name: Modified module loaded by client
severity: high
when:
  - event: file_hash_mismatch
  - followed_by: image_load
condition:
  same_path: true
  process.name: ToyGameClient.exe
explain: >
  A file that differs from the baseline was later loaded as a module by the game client.
```

### 7.5 ReplayEngine

ReplayEngine 支持离线重放一次运行过程。

典型命令：

```powershell
wks replay --input runs/run_001/events.jsonl
wks diff --left runs/clean --right runs/suspicious
```

用途：

* 复盘实验；
* 比较不同版本客户端；
* 比较正常运行与异常运行；
* 为 LLM 提供稳定上下文；
* 生成可复现报告。

### 7.6 ReportEngine

ReportEngine 生成 Markdown / HTML / JSON 报告。

报告内容：

* 运行会话信息；
* 进程树；
* 模块加载列表；
* 文件完整性变化；
* 注册表变化；
* 网络元数据；
* 对象访问记录；
* 告警列表；
* 证据链；
* 复核建议。

### 7.7 LLM Adapter

LLM Adapter 是后期扩展模块。

LLM 不直接控制内核，不直接操作驱动，而是调用用户态只读查询工具。

未来可提供工具：

* list_runs；
* query_events；
* trace_process；
* trace_file；
* find_non_baseline_modules；
* get_evidence_chain；
* diff_runs；
* explain_alert；
* generate_report。

## 8. C++23 使用策略

### 8.1 用户态

用户态模块优先使用 C++23。

推荐特性：

* `std::expected`：表达错误返回；
* `std::span`：表达连续内存视图；
* `std::string_view`：减少不必要复制；
* `std::filesystem`：文件路径与目录遍历；
* `std::format`：日志与报告格式化；
* `std::ranges`：事件过滤和转换；
* `std::chrono`：统一时间处理；
* 强类型 enum；
* RAII；
* `std::unique_ptr`；
* `std::shared_ptr` 谨慎使用；
* 自定义 deleter 管理 Windows HANDLE。

示例：

```cpp
using UniqueHandle = std::unique_ptr<void, HandleDeleter>;

std::expected<EventBatch, WinError> read_events_from_driver();
```

### 8.2 内核态

内核态采用 WDK/KMDF 兼容的受限 C++ 子集。

内核态禁止或避免：

* STL 容器；
* C++ 异常；
* RTTI；
* 协程；
* iostream；
* 全局复杂对象；
* 隐式动态初始化；
* 未封装的 `new` / `delete`；
* 内核中复杂模板元编程。

内核态推荐：

* 简单类封装；
* RAII 风格资源管理，但必须确保兼容内核约束；
* 明确的 `Init()` / `Shutdown()` 生命周期；
* 非抛异常错误处理；
* `NTSTATUS` 返回值；
* 自定义内存分配封装；
* 明确 IRQL 约束；
* 明确锁顺序；
* 明确分页/非分页内存边界。

### 8.3 代码风格

用户态：

```cpp
std::expected<RunSession, Error> create_run_session(const RunConfig& config);
```

内核态：

```cpp
NTSTATUS KProcessInitialize(_In_ const KPROCESS_CONFIG* Config);
VOID KProcessShutdown();
```

用户态可以追求现代 C++ 表达力，内核态优先追求稳定、可控、可调试。

## 9. 编译环境

推荐环境：

* Windows 11；
* Visual Studio 2022 或更新版本；
* MSVC；
* Windows SDK；
* Windows Driver Kit；
* CMake 3.25+；
* Ninja；
* Git；
* WinDbg；
* Windows 虚拟机测试环境。

用户态建议使用：

```text
C++ standard: C++23
Runtime: MSVC
Build system: CMake
Package manager: vcpkg, optional
```

内核态建议使用：

```text
WDK
KMDF
Visual Studio Driver Project
Test-signing VM
WinDbg Kernel Debugging
```

## 10. 构建方式

### 10.1 用户态构建

```powershell
git clone <repo>
cd WinKernelSecEngine

cmake -S . -B build -G Ninja `
  -DCMAKE_BUILD_TYPE=RelWithDebInfo `
  -DCMAKE_CXX_STANDARD=23

cmake --build build
```

### 10.2 运行 CLI

```powershell
.\build\apps\wks_cli\wks.exe --help
```

示例命令：

```powershell
wks baseline create --path C:\Lab\ToyGameClient --out baseline.json
wks process list
wks pe inspect --file C:\Lab\ToyGameClient\ToyGameClient.exe
wks run record --target ToyGameClient.exe --out runs\run_001
wks report generate --run runs\run_001 --out report.md
```

### 10.3 内核驱动构建

内核驱动建议通过 Visual Studio + WDK Driver Project 构建。

构建目标：

```text
kernel/kcore
kernel/keventbus
kernel/kprocess
kernel/kimage
```

第一阶段驱动只实现：

* 驱动加载；
* 驱动卸载；
* IOCTL 查询版本；
* 进程创建事件；
* 镜像加载事件；
* 用户态 Broker 读取事件。

## 11. 运行环境建议

驱动测试必须在虚拟机中进行。

推荐环境：

```text
Host:
  Windows 11
  Visual Studio
  WDK
  WinDbg

Guest:
  Windows 11 VM
  Test Signing Enabled
  Kernel Debugging Enabled
```

不要在主力物理机上直接加载开发中的内核驱动。

## 12. 开发路线图

### v0.1：最小可运行内核事件链路

目标：

* KCore；
* KEventBus；
* KProcess；
* KImage；
* User-mode Broker；
* JSONL EventStore。

验收：

```text
启动 ToyGameClient 后，能够记录：
1. 进程创建；
2. 镜像加载；
3. 进程退出；
4. 事件写入 events.jsonl。
```

### v0.2：用户态分析基础

目标：

* PE Sensor；
* FileIntegritySensor；
* ProcessSensor；
* ReportEngine。

验收：

```text
能够对 ToyGameClient 生成初版安全报告。
```

### v0.3：行为图谱

目标：

* GraphBuilder；
* 实体归一化；
* 进程-模块-文件关系图；
* graph.json 输出。

验收：

```text
能够回答某个模块由哪个进程加载、对应哪个磁盘文件、是否偏离基线。
```

### v0.4：行为回放与差异分析

目标：

* ReplayEngine；
* DiffEngine；
* 正常运行与异常运行对比。

验收：

```text
能够比较两次 ToyGameClient 运行中的模块、文件、进程行为差异。
```

### v0.5：对象访问审计

目标：

* KObject；
* 进程对象访问记录；
* 线程对象访问记录；
* 高权限访问告警。

验收：

```text
能够记录哪些进程尝试访问 ToyGameClient 进程对象。
```

### v0.6：文件系统 minifilter

目标：

* KFile；
* 文件写入记录；
* 文件重命名记录；
* 文件删除记录；
* 写入者进程关联。

验收：

```text
能够记录客户端目录下关键文件被哪个进程修改。
```

### v0.7：注册表观测

目标：

* KRegistry；
* 关键注册表路径监控；
* 注册表变化事件。

验收：

```text
能够记录关键注册表项变化及其来源进程。
```

### v0.8：网络元数据观测

目标：

* KNetwork；
* WFP 元数据采集；
* 网络端点基线。

验收：

```text
能够记录 ToyGameClient 的网络连接元数据。
```

### v0.9：IOCTL 审计

目标：

* KIoctlAudit；
* 自研驱动接口文档生成；
* IOCTL 参数长度审计。

验收：

```text
能够生成本项目驱动的 IOCTL 接口审计报告。
```

### v1.0：LLM Adapter

目标：

* 只读查询工具；
* 事件检索；
* 证据链解释；
* 报告辅助生成。

验收：

```text
LLM 能基于结构化事件回答：
“本次运行中有哪些非基线模块被加载？”
“哪个文件先被修改后被客户端加载？”
“最重要的三条证据链是什么？”
```

## 13. 事件类型

初始事件类型包括：

```cpp
enum class EventType : uint16_t {
    ProcessCreate,
    ProcessExit,
    ThreadCreate,
    ThreadExit,
    ImageLoad,
    FileCreate,
    FileOpen,
    FileRead,
    FileWrite,
    FileRename,
    FileDelete,
    RegistryCreateKey,
    RegistrySetValue,
    RegistryDeleteValue,
    NetworkConnect,
    ObjectHandleCreate,
    MemoryRegionSummary,
    IoctlCall,
    DriverHealth,
    PolicyUpdate
};
```

## 14. 示例报告结构

```markdown
# WinKernelSec Run Report

## Run Summary

- Run ID: run_001
- Target: ToyGameClient.exe
- Start Time:
- End Time:
- Event Count:

## Process Tree

## Loaded Images

## File Integrity Changes

## Registry Changes

## Network Metadata

## Object Access Events

## Alerts

## Evidence Chains

## Recommendations
```

## 15. 规则示例

```yaml
id: non_baseline_module_loaded
name: Non-baseline module loaded by game client
severity: medium
description: >
  The target game client loaded a module that is not present in the known baseline.

match:
  event_type: image_load
  process.name: ToyGameClient.exe

condition:
  object.path.not_in_baseline: true

evidence:
  - process.image_path
  - object.path
  - object.hash
  - timestamp

recommendation: >
  Verify whether the module belongs to an authorized update, plugin, or test component.
```

```yaml
id: modified_file_loaded_as_module
name: Modified file loaded as module
severity: high
description: >
  A file whose hash differs from the baseline was later loaded as an image by the game client.

sequence:
  - event_type: file_hash_mismatch
  - event_type: image_load

join:
  left.object.path: right.object.path

condition:
  right.process.name: ToyGameClient.exe

recommendation: >
  Review the file modification source process and validate the file against the trusted baseline.
```

## 16. 测试策略

### 16.1 用户态测试

* EventModel 序列化/反序列化测试；
* EventStore 写入/读取测试；
* GraphBuilder 图构建测试；
* RuleEngine 规则匹配测试；
* ReplayEngine 回放测试；
* PE Sensor 样本解析测试；
* FileIntegritySensor 基线比对测试。

### 16.2 内核态测试

内核态测试必须在虚拟机中执行。

测试内容：

* 驱动加载/卸载；
* IOCTL 查询版本；
* Broker 连接；
* 事件缓冲区读写；
* KProcess 事件采集；
* KImage 事件采集；
* 高频事件下的丢失统计；
* 异常参数处理；
* 驱动卸载清理。

### 16.3 集成测试

* 启动 ToyGameClient；
* 采集运行事件；
* 修改测试文件；
* 重新运行；
* 生成差异报告；
* 检查证据链是否正确。

## 17. 代码规范

### 17.1 命名规范

用户态：

```cpp
class EventStore;
struct RunSession;
enum class EventType;
std::expected<RunReport, Error> generate_report(...);
```

内核态：

```cpp
NTSTATUS KProcessInitialize(...);
VOID KProcessShutdown();
typedef struct _WKS_EVENT_HEADER { ... } WKS_EVENT_HEADER;
```

### 17.2 错误处理

用户态优先使用：

```cpp
std::expected<T, Error>
```

内核态使用：

```cpp
NTSTATUS
```

禁止在内核态使用 C++ 异常。

### 17.3 资源管理

用户态 Windows HANDLE 必须使用 RAII 封装：

```cpp
class unique_handle {
public:
    unique_handle() noexcept = default;
    explicit unique_handle(HANDLE handle) noexcept;
    ~unique_handle();

    unique_handle(const unique_handle&) = delete;
    unique_handle& operator=(const unique_handle&) = delete;

    unique_handle(unique_handle&& other) noexcept;
    unique_handle& operator=(unique_handle&& other) noexcept;

    HANDLE get() const noexcept;
    HANDLE release() noexcept;
    void reset(HANDLE handle = nullptr) noexcept;
};
```

内核态资源必须有明确释放路径：

```text
Init 成功的资源必须在 Shutdown 中释放；
任何中间失败路径必须回滚已申请资源；
所有回调必须在驱动卸载前注销；
所有队列和缓冲区必须在卸载前停止接收新事件。
```

## 18. 性能原则

内核层：

* 避免复杂字符串处理；
* 避免长时间持锁；
* 避免在高 IRQL 下执行复杂逻辑；
* 避免同步等待用户态；
* 避免无界队列；
* 所有事件必须可丢弃、可统计；
* 过滤策略尽量简单。

用户态：

* 批量读取事件；
* 批量写入存储；
* 支持事件压缩；
* 支持异步报告生成；
* 支持按 RunSession 分区。

## 19. 安全原则

* 默认只观测，不阻断；
* 默认只记录元数据，不记录敏感内容；
* 默认只监控授权目标；
* 驱动只在测试签名虚拟机中运行；
* 不提供攻击性自动化；
* 不提供绕过能力；
* 不对真实商业游戏进行未授权实验；
* 所有实验必须可复现、可解释、可审计。

## 20. 未来 LLM 接入方向

LLM Adapter 的设计原则：

* LLM 不直接访问驱动；
* LLM 不直接操作内核；
* LLM 只调用用户态只读查询工具；
* 所有结论必须绑定事件证据；
* 所有报告必须引用事件 ID 或证据链 ID。

未来可支持的问题：

```text
本次运行中有哪些异常模块？
哪些文件先被修改后被加载？
ToyGameClient 本次运行和基线相比有什么差异？
是否存在异常对象访问行为？
最值得人工复核的证据链是什么？
```

## 21. 当前状态

项目当前处于设计与早期开发阶段。

优先级：

```text
P0:
  KCore
  KEventBus
  KProcess
  KImage
  User-mode Broker
  EventStore

P1:
  PE Sensor
  FileIntegritySensor
  GraphBuilder
  RuleEngine
  ReplayEngine

P2:
  KObject
  KFile
  KRegistry
  KNetwork
  KIoctlAudit
  LLM Adapter
```

## 22. 贡献指南

欢迎贡献以下内容：

* 内核模块设计；
* 用户态 C++23 工程代码；
* 事件模型改进；
* 规则引擎规则；
* ToyGameClient 实验样本；
* 文档与测试；
* WinDbg 调试记录；
* 性能优化；
* 合规安全实验案例。

贡献要求：

1. 不提交攻击性代码；
2. 不提交绕过类代码；
3. 不提交针对真实游戏的未授权样本；
4. 新模块必须遵循统一事件模型；
5. 新内核模块必须提供卸载清理逻辑；
6. 新功能必须包含测试或最小复现实验。

## 23. License

待定。

建议选择：

* MIT：适合开放工具；
* Apache-2.0：适合带专利授权条款的工程项目；
* GPL-3.0：适合希望强制开源衍生版本的项目。

## 24. Disclaimer

This project is intended for authorized Windows security research, defensive engineering, kernel development learning, and controlled game client security experiments.

Do not use this project to develop cheats, malware, unauthorized monitoring tools, anti-cheat bypasses, process hiding techniques, driver hiding techniques, or any capability that violates software terms of service, applicable laws, or ethical research boundaries.

All experiments should be conducted in controlled environments, preferably using self-developed toy clients or explicitly authorized samples.
