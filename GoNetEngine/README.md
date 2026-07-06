# GoNetEngine

GoNetEngine 是 TriScope 项目的网络行为分析、任务调度、FakeNet、控制平面与报告生成模块。它使用 Go 开发，负责分析目标程序运行期间产生的 DNS、TCP、UDP、HTTP、TLS 等网络行为，并将系统行为、执行行为和网络行为聚合为可读报告。

---

## 1. 模块定位

GoNetEngine 的定位是：

```text
Network Analysis & Control Plane
网络分析与控制平面
```

它回答三个问题：

```text
1. 目标程序访问了哪些网络资源？
2. 网络行为与进程、文件、注册表行为之间有什么关联？
3. 如何把一次分析任务组织、聚合并生成报告？
```

---

## 2. 为什么使用 Go

Go 适合该模块的原因：

```text
- 标准库网络能力强
- goroutine 适合高并发 I/O
- 适合写 HTTP API 和控制平面
- 部署简单，容易编译成单文件
- 适合处理任务调度、日志聚合、报告服务
- 生态中有较多网络协议和 pcap 分析库
```

---

## 3. 核心功能

```text
- PCAP 文件读取
- 实时流量采集接口预留
- DNS 查询分析
- TCP/UDP 连接分析
- HTTP 请求与响应元数据提取
- TLS ClientHello、SNI、ALPN、证书信息提取
- 连接图构建
- Fake DNS 服务
- Fake HTTP 服务
- Fake HTTPS 服务预留
- 分析任务管理
- Web API
- CLI 控制端
- SQLite 存储
- Markdown / HTML / JSON 报告生成
```

---

## 4. 推荐目录结构

```text
GoNetEngine/
├── README.md
├── go.mod
├── cmd/
│   ├── gonetctl/
│   │   └── main.go
│   └── gonetd/
│       └── main.go
├── internal/
│   ├── analyzer/
│   │   ├── dns.go
│   │   ├── http.go
│   │   ├── tls.go
│   │   └── flow.go
│   ├── pcap/
│   │   ├── reader.go
│   │   └── decoder.go
│   ├── fakenet/
│   │   ├── dns_server.go
│   │   ├── http_server.go
│   │   └── config.go
│   ├── api/
│   │   ├── server.go
│   │   └── handlers.go
│   ├── storage/
│   │   └── sqlite.go
│   ├── report/
│   │   ├── markdown.go
│   │   ├── html.go
│   │   └── json.go
│   ├── model/
│   │   ├── event.go
│   │   ├── run.go
│   │   └── finding.go
│   └── scheduler/
│       └── worker.go
├── configs/
│   └── default.yaml
├── templates/
│   └── report.html
└── tests/
```

---

## 5. CLI 设计

推荐命令名：

```text
gonetctl
```

示例命令：

```bash
gonetctl run create --target C:\Sandbox\sample.exe

gonetctl fakenet start --run-id run-20260706-0001 --config configs/fakenet.yaml

gonetctl pcap analyze --run-id run-20260706-0001 --input runs/run-001/pcap/traffic.pcap

gonetctl events merge --run-id run-20260706-0001

gonetctl report --run-id run-20260706-0001 --format markdown

gonetctl serve --db triscope.db --listen 127.0.0.1:8080
```

---

## 6. 网络事件模型

示例 DNS 事件：

```json
{
  "run_id": "run-20260706-0001",
  "source": "gonet",
  "event_type": "dns_query",
  "timestamp": 1783300000000,
  "process": {
    "pid": 4321,
    "image": "C:\\Sandbox\\sample.exe"
  },
  "network": {
    "query": "update.example.com",
    "qtype": "A"
  }
}
```

示例 HTTP 事件：

```json
{
  "run_id": "run-20260706-0001",
  "source": "gonet",
  "event_type": "http_request",
  "timestamp": 1783300002000,
  "network": {
    "method": "GET",
    "host": "example.com",
    "path": "/update/check",
    "user_agent": "SampleClient/1.0"
  }
}
```

示例 TLS 事件：

```json
{
  "run_id": "run-20260706-0001",
  "source": "gonet",
  "event_type": "tls_client_hello",
  "timestamp": 1783300003000,
  "network": {
    "dst_ip": "1.2.3.4",
    "dst_port": 443,
    "sni": "api.example.com",
    "alpn": ["h2", "http/1.1"]
  }
}
```

---

## 7. FakeNet 功能

FakeNet 用于在受控分析环境中模拟网络服务，使目标程序的网络行为更容易被观察。

支持目标：

```text
- 记录 DNS 查询
- 将域名解析到本地分析机
- 提供假 HTTP 响应
- 记录 HTTP 请求路径、Header 和 Body 摘要
- 模拟网络失败、超时、固定响应
```

示例配置：

```yaml
dns:
  listen: "0.0.0.0:53"
  default_ip: "192.168.56.1"
  log_queries: true

http:
  listen: "0.0.0.0:8080"
  default_status: 200
  default_body: "OK"
  log_headers: true
  log_body_limit: 4096

rules:
  - host: "update.example.com"
    path: "/check"
    status: 200
    body: "{\"version\":\"1.0.0\"}"
```

启动：

```bash
gonetctl fakenet start --run-id run-20260706-0001 --config configs/fakenet.yaml
```

---

## 8. 报告内容

GoNetEngine 最终生成报告，推荐包含：

```text
- 分析任务基本信息
- 目标程序信息
- 执行摘要
- 进程树
- 文件行为摘要
- 注册表行为摘要
- 网络行为摘要
- DNS 查询列表
- HTTP 请求列表
- TLS 连接列表
- 可疑行为时间线
- 风险提示
- 原始证据文件路径
```

Markdown 报告示例结构：

```markdown
# TriScope Analysis Report

## 1. Target

## 2. Execution Summary

## 3. Process Tree

## 4. File Behavior

## 5. Registry Behavior

## 6. Network Behavior

## 7. Timeline

## 8. Findings

## 9. Artifacts
```

---

## 9. 存储设计

推荐使用 SQLite 作为初期存储。

基础表：

```text
runs
targets
processes
events
network_flows
dns_queries
http_requests
tls_sessions
findings
artifacts
```

事件也应保留 JSONL 文件形式，便于跨语言调试和后续接入 LLM。

---

## 10. Web API 设计

预留 `gonetd` 作为服务端：

```text
POST   /api/runs
GET    /api/runs
GET    /api/runs/{run_id}
GET    /api/runs/{run_id}/events
GET    /api/runs/{run_id}/network
GET    /api/runs/{run_id}/report
POST   /api/runs/{run_id}/report
```

---

## 11. 与其他模块的关系

```text
RustActionEngine:
- 提供 action_events.jsonl
- 提供 normalized_events.jsonl
- 提供 metadata.json

WinKernelEngine:
- 提供 kernel_events.jsonl

GoNetEngine:
- 提供 network_events.jsonl
- 负责聚合
- 负责报告
- 负责控制平面
```

---

## 12. 开发路线

### Phase 1：报告生成器

```text
- 读取 metadata.json
- 读取 mock network_events.jsonl
- 生成 Markdown 报告
```

### Phase 2：PCAP 分析

```text
- 读取 pcap
- 提取 TCP/UDP 流
- 提取 DNS 查询
```

### Phase 3：HTTP/TLS 元数据

```text
- 提取 HTTP 请求元数据
- 提取 TLS SNI / ALPN
- 建立网络行为时间线
```

### Phase 4：FakeNet

```text
- 实现 Fake DNS
- 实现 Fake HTTP
- 记录请求
- 输出 network_events.jsonl
```

### Phase 5：控制平面

```text
- 实现 gonetd
- 实现 Web API
- 实现 SQLite 存储
- 实现 HTML 报告
```

---

## 13. 测试计划

```text
- 使用 mock pcap 验证 DNS 提取
- 使用本地 HTTP 请求验证 http_request 事件
- 使用 TLS 测试连接验证 SNI 提取
- 启动 Fake DNS，验证查询记录
- 启动 Fake HTTP，验证请求日志
- 读取 normalized_events.jsonl，验证报告生成
```

---

## 14. 安全边界

GoNetEngine 的 FakeNet 只用于受控分析环境，不应部署到公网。

注意：

```text
- 不记录敏感真实用户流量
- 不作为代理服务暴露到公网
- 不用于未授权流量截获
- 不用于中间人攻击
- 不生成攻击载荷
```

---

## 15. 总结

GoNetEngine 是 TriScope 的网络与平台大脑。它的价值在于：

```text
- 看懂网络行为
- 组织分析任务
- 聚合多源事件
- 生成可读报告
- 为后续 Web 控制台和 LLM 分析提供基础
```

它让 TriScope 从单机采集工具升级为完整行为分析平台。
