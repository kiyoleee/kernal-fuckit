# RustActionEngine

RustActionEngine 是 TriScope 项目的目标程序执行、测试、输入生成、重放与行为归一化模块。它使用 Rust 开发，负责稳定、可重复地触发目标程序行为，并将 WinKernelEngine 与 GoNetEngine 产生的原始事件转换为统一行为模型。

---

## 1. 模块定位

RustActionEngine 的定位是：

```text
Behavior Test & Normalization Engine
行为触发、测试执行与归一化引擎
```

它回答三个问题：

```text
1. 如何稳定运行目标程序？
2. 如何通过输入、参数、环境变化触发更多行为？
3. 如何把底层事件归一化为统一的行为时间线？
```

---

## 2. 为什么使用 Rust

Rust 适合该模块的原因：

```text
- 强类型适合建模复杂行为事件
- 内存安全适合处理不可信输入和样本输出
- Result / Option 有利于清晰错误处理
- 适合写状态机、解析器和执行编排器
- 适合构建可靠的命令行工具
- 适合做跨平台本地执行组件
```

---

## 3. 主要功能

```text
- 目标程序启动
- 运行参数控制
- 工作目录管理
- 环境变量管理
- 超时控制
- stdout/stderr 捕获
- 退出码记录
- 崩溃检测
- 输入文件管理
- 简单输入变异
- 测试用例保存
- 行为重放
- 原始事件读取
- 事件归一化
- 行为时间线构建
```

---

## 4. 推荐目录结构

```text
RustActionEngine/
├── README.md
├── Cargo.toml
├── crates/
│   ├── action-cli/
│   │   └── src/
│   ├── action-core/
│   │   └── src/
│   ├── action-runner/
│   │   └── src/
│   ├── action-mutator/
│   │   └── src/
│   ├── action-normalizer/
│   │   └── src/
│   └── action-replay/
│       └── src/
├── configs/
│   └── default.toml
├── tests/
└── examples/
```

各 crate 职责：

```text
action-cli:
- 命令行入口

action-core:
- 数据模型
- 错误类型
- 公共 trait
- 配置结构

action-runner:
- 目标程序执行
- timeout
- stdout/stderr
- exit code

action-mutator:
- 输入生成
- 参数变异
- 环境变量组合

action-normalizer:
- 读取原始事件
- 转换为 BehaviorEvent
- 构建时间线

action-replay:
- 根据 run metadata 重放分析任务
```

---

## 5. CLI 设计

推荐命令名：

```text
actionctl
```

基础命令：

```bash
actionctl run --target C:\Sandbox\sample.exe --run-id run-20260706-0001 --timeout 60s

actionctl run --target C:\Sandbox\parser.exe --input seeds\input.bin --timeout 10s

actionctl mutate --seed seeds\input.bin --output runs\run-001\inputs --count 100

actionctl normalize --run-id run-20260706-0001

actionctl replay --run-id run-20260706-0001

actionctl timeline --run-id run-20260706-0001
```

---

## 6. 配置文件示例

```toml
[run]
timeout_seconds = 60
working_dir = "C:\\Sandbox\\workdir"
capture_stdout = true
capture_stderr = true
kill_process_tree_on_timeout = true

[input]
enable_file_input = true
seed_dir = "seeds"
max_input_size = 1048576
mutation_count = 32

[environment]
clear_env = false

[normalizer]
kernel_events = "events/kernel_events.jsonl"
network_events = "events/network_events.jsonl"
output = "events/normalized_events.jsonl"
```

---

## 7. 执行元数据

每次执行生成 `metadata.json`：

```json
{
  "run_id": "run-20260706-0001",
  "target": {
    "path": "C:\\Sandbox\\sample.exe",
    "args": ["input.bin"],
    "working_dir": "C:\\Sandbox\\workdir"
  },
  "execution": {
    "start_time": 1783300000000,
    "end_time": 1783300060000,
    "duration_ms": 60000,
    "exit_code": 0,
    "timeout": false,
    "crashed": false
  },
  "artifacts": {
    "stdout": "logs/stdout.log",
    "stderr": "logs/stderr.log",
    "inputs": "inputs/"
  }
}
```

---

## 8. 事件归一化

RustActionEngine 将不同来源的事件转换成统一格式。

输入：

```text
kernel_events.jsonl
network_events.jsonl
action_events.jsonl
```

输出：

```text
normalized_events.jsonl
```

归一化示例：

原始 WinKernelEngine 事件：

```json
{
  "source": "winkernel",
  "event_type": "file_write",
  "pid": 4321,
  "path": "C:\\Temp\\a.dll"
}
```

归一化后：

```json
{
  "run_id": "run-20260706-0001",
  "source": "kernel",
  "event_type": "file_write",
  "process": {
    "pid": 4321,
    "image": "C:\\Sandbox\\sample.exe"
  },
  "object": {
    "type": "file",
    "path": "C:\\Temp\\a.dll"
  },
  "tags": [
    "temp_directory",
    "dll_file"
  ],
  "severity_hint": "medium"
}
```

---

## 9. 测试策略设计

### 9.1 BasicRunStrategy

默认运行一次目标程序。

```text
输入：
- target path
- args
- timeout

输出：
- exit code
- stdout
- stderr
- action event
```

### 9.2 FileInputStrategy

将不同输入文件传给目标程序。

```bash
actionctl run --target parser.exe --input seeds/a.bin
```

### 9.3 ArgumentMutationStrategy

对命令行参数进行组合测试。

```text
sample.exe --help
sample.exe --version
sample.exe input.txt
sample.exe --config config.json
```

### 9.4 EnvironmentStrategy

改变环境变量观察行为变化。

```text
TEMP
PATH
APPDATA
USERPROFILE
```

### 9.5 ReplayStrategy

根据历史 metadata 重放任务。

```bash
actionctl replay --run-id run-20260706-0001
```

---

## 10. 崩溃检测与保存

当目标程序异常退出、超时或产生崩溃信号时，RustActionEngine 应保存：

```text
- 输入文件
- 命令行参数
- 环境变量
- stdout/stderr
- 相关事件片段
- metadata.json
```

目录示例：

```text
runs/run-001/crashes/crash-0001/
├── input.bin
├── metadata.json
├── stdout.log
├── stderr.log
└── related_events.jsonl
```

---

## 11. 与其他模块的关系

```text
GoNetEngine:
- 创建任务
- 启动网络环境
- 聚合报告

RustActionEngine:
- 执行目标
- 管理输入
- 归一化事件

WinKernelEngine:
- 提供系统行为原始事件
```

流程：

```text
GoNetEngine 创建 run_id
        ↓
RustActionEngine 启动目标程序
        ↓
WinKernelEngine 采集系统行为
        ↓
GoNetEngine 采集网络行为
        ↓
RustActionEngine 归一化事件
        ↓
GoNetEngine 生成报告
```

---

## 12. 开发路线

### Phase 1：基础 Runner

```text
- 实现 actionctl run
- 支持 target、args、timeout
- 捕获 stdout/stderr
- 输出 metadata.json
```

### Phase 2：输入管理

```text
- 支持 input 文件
- 支持 input corpus
- 支持多次执行
```

### Phase 3：变异测试

```text
- 实现简单 byte-level mutation
- 保存 interesting inputs
- 保存 crash inputs
```

### Phase 4：事件归一化

```text
- 读取 kernel_events.jsonl
- 读取 network_events.jsonl
- 输出 normalized_events.jsonl
```

### Phase 5：重放与时间线

```text
- replay run
- timeline build
- related event slicing
```

---

## 13. 测试计划

```text
- 运行正常程序，检查 exit code
- 运行超时程序，检查 timeout
- 运行崩溃程序，检查 crash 保存
- 输入不同文件，检查 input metadata
- 读取 mock kernel events，检查 normalized output
- 重放历史任务，检查结果目录一致性
```

---

## 14. 安全边界

RustActionEngine 可以运行未知或可疑目标程序，因此必须默认假设目标不可信。

建议：

```text
- 在虚拟机中运行
- 使用隔离目录
- 设置超时时间
- 限制输入大小
- 保留运行日志
- 不在含有敏感数据的主机上运行
```

---

## 15. 总结

RustActionEngine 是 TriScope 的实验执行核心。它的价值在于：

```text
- 稳定执行
- 可重复测试
- 可保存证据
- 可重放分析
- 可归一化行为
```

它让 TriScope 从“只观察一次行为”升级为“可控、可重放、可比较的行为分析系统”。
