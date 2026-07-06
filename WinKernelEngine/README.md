# WinKernelEngine

WinKernelEngine 是 TriScope 项目的 Windows 高权限系统行为采集模块，主要使用 C++ / C++23 开发。它负责在授权环境中采集目标程序运行过程中的进程、线程、文件、注册表、模块加载、句柄访问和部分网络关联行为，并将这些底层事件输出为结构化事件流，供 RustActionEngine 和 GoNetEngine 进一步归一化、聚合和分析。

---

## 1. 模块定位

WinKernelEngine 的定位是：

```text
Windows 行为传感器
Windows Behavior Sensor
```

它只负责回答一个问题：

```text
目标程序在系统层面做了什么？
```

它不负责：

```text
- 判断程序是否恶意
- 生成最终报告
- 执行测试策略
- 分析网络协议细节
- 调用 LLM
- 实现隐蔽、绕过、对抗能力
```

---

## 2. 设计原则

### 2.1 采集层最小化

内核或高权限采集层应尽量简单，只做：

```text
- 事件采集
- 基础过滤
- 数据缓存
- 安全传输到用户态
```

复杂逻辑应放到 RustActionEngine 或 GoNetEngine 中。

### 2.2 稳定性优先

WinKernelEngine 的首要目标不是“功能最多”，而是：

```text
- 不蓝屏
- 不阻塞系统关键路径
- 不过度影响性能
- 输出事件可靠
- 出错时可恢复
```

### 2.3 授权与防御用途

本模块仅用于授权安全分析、程序行为审计、实验教学和防御研究。

不得用于：

```text
- 隐藏驱动
- 隐藏进程
- 隐藏文件
- 绕过安全软件
- 持久化驻留
- 未授权监控
```

---

## 3. 推荐架构

```text
WinKernelEngine/
├── driver/
│   ├── include/
│   ├── src/
│   └── CMakeLists.txt
├── service/
│   ├── include/
│   ├── src/
│   └── CMakeLists.txt
├── include/
│   └── winkernel_common/
├── tests/
├── tools/
│   └── winkernelctl/
├── docs/
└── README.md
```

建议拆成两层：

```text
Kernel Collector:
- 负责低层事件采集

User-mode Service:
- 负责接收事件
- 做基础格式化
- 输出 JSONL
- 与其他模块通信
```

---

## 4. 采集范围

### 4.1 进程事件

目标：

```text
- 记录目标程序创建的进程
- 记录父子进程关系
- 记录命令行
- 记录进程镜像路径
- 记录进程退出
```

事件类型：

```text
process_create
process_exit
```

示例事件：

```json
{
  "run_id": "run-20260706-0001",
  "source": "winkernel",
  "event_type": "process_create",
  "timestamp": 1783300000000,
  "pid": 4321,
  "ppid": 1000,
  "image": "C:\\Sandbox\\sample.exe",
  "command_line": "sample.exe input.bin"
}
```

### 4.2 文件事件

目标：

```text
- 文件创建
- 文件读取
- 文件写入
- 文件删除
- 文件重命名
- 可执行文件落地
- 临时目录写入
```

事件类型：

```text
file_create
file_read
file_write
file_delete
file_rename
```

示例：

```json
{
  "run_id": "run-20260706-0001",
  "source": "winkernel",
  "event_type": "file_write",
  "timestamp": 1783300001000,
  "pid": 4321,
  "path": "C:\\Users\\test\\AppData\\Local\\Temp\\a.dll",
  "attributes": {
    "size": "4096"
  }
}
```

### 4.3 注册表事件

目标：

```text
- 注册表键创建
- 注册表值设置
- 注册表值删除
- 自启动项相关变更
- 服务项相关变更
```

事件类型：

```text
registry_create_key
registry_set_value
registry_delete_value
registry_delete_key
```

### 4.4 模块加载事件

目标：

```text
- 记录 DLL 加载
- 记录模块路径
- 记录非系统目录模块加载
- 为后续规则提供依据
```

事件类型：

```text
module_load
```

### 4.5 句柄与跨进程访问事件

目标：

```text
- 记录目标进程打开其他进程句柄
- 记录其他进程访问目标进程
- 为后续分析跨进程行为提供线索
```

事件类型：

```text
handle_open
```

注意：该模块只记录行为元数据，不实现进程注入、隐藏、绕过等攻击性能力。

### 4.6 网络关联事件

WinKernelEngine 不负责深度网络协议分析，但可以记录基础关联信息：

```text
- 哪个 PID 发起连接
- 目标 IP
- 目标端口
- 协议类型
```

深度 DNS、HTTP、TLS、PCAP 分析由 GoNetEngine 完成。

---

## 5. 输出格式

WinKernelEngine 输出原始事件流：

```text
runs/<run_id>/events/kernel_events.jsonl
```

每一行是一个 JSON 对象：

```json
{"run_id":"run-001","source":"winkernel","event_type":"process_create","timestamp":1783300000000,"pid":4321}
{"run_id":"run-001","source":"winkernel","event_type":"file_write","timestamp":1783300001000,"pid":4321,"path":"C:\\Temp\\a.dll"}
```

---

## 6. 控制命令设计

推荐提供用户态控制工具 `winkernelctl`：

```bash
winkernelctl start --run-id run-20260706-0001 --output runs/run-20260706-0001/events/kernel_events.jsonl

winkernelctl status

winkernelctl stop --run-id run-20260706-0001

winkernelctl test --target-pid 4321
```

---

## 7. 配置文件示例

```toml
[collector]
run_id = "run-20260706-0001"
output = "runs/run-20260706-0001/events/kernel_events.jsonl"
buffer_size = 1048576

[filter]
target_pid = 4321
include_child_processes = true
capture_file_events = true
capture_registry_events = true
capture_module_events = true
capture_network_events = true

[performance]
drop_events_when_buffer_full = true
max_path_length = 4096
```

---

## 8. 与其他模块的关系

```text
WinKernelEngine
    ↓ kernel_events.jsonl
RustActionEngine
    ↓ normalized_events.jsonl
GoNetEngine
    ↓ report.html / report.md
```

WinKernelEngine 不直接生成最终结论，只提供证据。

---

## 9. 开发路线

### Phase 1：用户态 Mock Collector

```text
- 实现 winkernelctl
- 输出模拟 process/file 事件
- 跑通完整数据流
```

### Phase 2：ETW Consumer

```text
- 使用用户态方式采集部分系统事件
- 验证事件模型
- 减少内核开发早期风险
```

### Phase 3：内核采集能力

```text
- 引入必要内核采集组件
- 支持进程、文件、注册表等事件
- 与用户态服务通信
```

### Phase 4：性能与稳定性

```text
- 事件缓冲
- 丢弃策略
- 压测
- 异常恢复
- 日志与诊断
```

---

## 10. 测试计划

```text
- 创建普通进程，检查 process_create
- 创建子进程，检查 ppid 关联
- 写入临时文件，检查 file_write
- 修改测试注册表路径，检查 registry_set_value
- 加载 DLL，检查 module_load
- 执行大量文件操作，检查性能和丢事件情况
```

---

## 11. 安全说明

本模块运行权限较高，应始终在受控环境下开发和测试。

推荐环境：

```text
- Windows 虚拟机
- 开启快照
- 不保存敏感数据
- 测试样本来源明确
- 使用专门的分析用户
```

---

## 12. 总结

WinKernelEngine 是 TriScope 的底层感知模块。它的价值在于：

```text
- 可靠采集
- 清晰输出
- 稳定运行
- 不做过度分析
```

它应当尽可能像一个“传感器”，把复杂判断交给上层模块。
