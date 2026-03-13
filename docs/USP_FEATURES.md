# TR-369 USP (User Services Platform) 项目功能点分析

## 项目概述

本项目是 Broadband Forum (宽带论坛) 制定的 **TR-369 用户服务平台 (USP)** 规范仓库。USP 是一个标准化协议，用于管理、监控、更新和控制联网设备、IoT 终端、用户服务和家庭网络。它是 TR-069 (CWMP) 协议的演进版本，当前最新版本为 **v1.5.0** (2026-01-09发布)。

---

## 一、核心架构

### 1.1 端点 (Endpoints) 模型
- **Agent (代理端)**: 暴露一个或多个服务元素 (Service Elements) 给 Controller，包含实例化数据模型和支持的数据模型
- **Controller (控制端)**: 通过 Agent 操纵一组服务元素，可维护 Agent 数据库、其能力和状态
- **Endpoint ID (端点标识符)**: 支持多种标识方案 (`oui`, `cid`, `pen`, `self`, `user`, `os`, `ops`, `uuid`, `imei`, `proto`, `doc`, `fqdn`)

### 1.2 数据模型
- 基于 Device:2 数据模型 (TR-181) 的扩展
- 支持实例化数据模型 (Instantiated Data Model) 和支持数据模型 (Supported Data Model)
- 服务元素由对象 (Objects)、参数 (Parameters)、操作 (Operations) 和事件 (Events) 组成

---

## 二、消息系统 (CRUD-ON)

USP 采用基于 CRUD 模型扩展的消息体系，包含操作 (Operate) 和通知 (Notify)，简称 CRUD-ON。

### 2.1 基本消息类型

| 消息类型 | 说明 |
|---------|------|
| **Get / GetResp** | 读取参数值，支持 `max_depth` 限制响应深度 |
| **Set / SetResp** | 更新对象参数值，支持 `allow_partial` 部分成功模式 |
| **Add / AddResp** | 创建数据模型对象实例，支持搜索路径 |
| **Delete / DeleteResp** | 删除数据模型对象实例 |
| **Operate / OperateResp** | 执行数据模型定义的命令，支持同步和异步操作 |
| **Notify / NotifyResp** | Agent 向 Controller 发送通知 |
| **GetSupportedDM / GetSupportedDMResp** | 查询支持的数据模型信息，返回参数类型、访问权限、唯一键集合等 |
| **GetInstances / GetInstancesResp** | 获取多实例对象的当前实例信息 |
| **GetSupportedProtocol / GetSupportedProtocolResp** | 协商支持的 USP 协议版本 |
| **Register / RegisterResp** | 注册数据模型路径 (v1.3+) |
| **Deregister / DeregisterResp** | 注销数据模型路径 (v1.3+) |
| **Error** | 错误响应消息，包含错误码和参数级错误 |

### 2.2 通知子类型

| 通知类型 | 说明 |
|---------|------|
| **Event** | 事件通知 |
| **ValueChange** | 参数值变化通知 |
| **ObjectCreation** | 对象实例创建通知 |
| **ObjectDeletion** | 对象实例删除通知 |
| **OperationComplete** | 异步操作完成通知 |
| **OnBoardRequest** | 设备上线请求通知 |

---

## 三、消息传输协议 (MTP)

USP 支持多种消息传输协议：

| MTP 类型 | 状态 | 说明 |
|---------|------|------|
| **WebSocket** | 活跃 | 基于 RFC 6455，支持 TLS 加密 |
| **STOMP** | 活跃 | 简单文本面向消息协议 (v1.2) |
| **MQTT** | 活跃 | 支持 MQTT 3.1.1 和 MQTT 5.0，v1.5 新增 ALPN 支持 |
| **Unix Domain Socket (UDS)** | 活跃 | v1.3 新增，用于设备内部通信，v1.5 新增密码认证帧 |
| **CoAP** | 已废弃 | v1.2 标记为废弃，v1.3 正式淘汰 |

### MTP 安全要求
- 跨网络边界传输时必须使用安全传输
- 支持 TLS 1.2 / TLS 1.3 加密
- X.509 证书认证

---

## 四、端到端消息交换

### 4.1 USP Record (记录层)
- 封装 USP 消息的传输容器
- 包含版本、源/目标端点ID、载荷安全类型、MAC/签名等字段
- 支持 `originator_id` 和 `destination_id` (v1.5+)

### 4.2 会话上下文 (Session Context)
- 支持有会话 (session_context) 和无会话 (no_session_context) 两种模式
- 有会话模式提供消息分段与重组 (SAR)、重传机制、序列号管理
- 支持载荷完整性保护

### 4.3 连接记录类型
- WebSocketConnectRecord
- MQTTConnectRecord
- STOMPConnectRecord
- UDSConnectRecord
- DisconnectRecord

---

## 五、消息编码

- 采用 **Protocol Buffers v3** 作为消息序列化格式
- 定义两套 Proto Schema:
  - `usp-msg-1-x.proto`: USP 消息 schema
  - `usp-record-1-x.proto`: USP 记录 schema
- 参数值使用 UTF-8 字符串表示，遵循 TR-106 数据类型规范

---

## 六、发现与通告

### 6.1 Controller 发现机制
- **DHCP 发现**: 通过 DHCPv4 (option 125) / DHCPv6 (option 17) 获取 Controller 信息
- **DNS 发现**: 通过 DNS SRV/TXT 记录解析
- **mDNS 发现**: 本地网络内的 DNS-SD 多播发现
- 预配置 / 固件内置
- 用户界面配置

### 6.2 Agent 通告
- Agent 可通过 mDNS 向本地网络通告自身存在
- Controller 在 Agent 初始连接时获取其信息

---

## 七、安全机制

### 7.1 认证 (Authentication)
- 基于 X.509 证书的端点认证
- UDS MTP 支持共享密钥认证
- 证书中必须包含端点 ID (subjectAltName 扩展)
- 支持信任首次使用 (TOFU) 策略

### 7.2 授权 - 基于角色的访问控制 (RBAC)
- 支持多角色定义 (如: 不受信任角色、完全访问角色、自定义角色)
- 参数/对象级别的权限控制 (读/写/创建/删除/执行/通知)
- SecuredRole 机制 (v1.3+)

### 7.3 信任证书机构 (Trusted CA)
- 预加载或安全方式配置可信 CA 证书
- CA 可用于认证 Controller 身份和/或授权角色

### 7.4 可信代理 (Trusted Brokers)
- 支持可信消息代理，代理可为端点身份担保
- v1.3 新增 R-SEC.4b 可信代理要求

---

## 八、扩展功能

### 8.1 设备代理 (Device Proxy)
- Agent 可代理其他设备的管理功能
- 支持 CoAP-STOMP MTP 代理等场景

### 8.2 消息代理 (Proxying)
- 支持 MTP 代理转发
- 代理架构支持跨协议消息路由

### 8.3 软件模块管理 (Software Module Management)
- 部署单元 (Deployment Unit) 生命周期管理 (安装/更新/卸载)
- 执行环境 (Execution Environment) 状态管理
- 执行单元 (Execution Unit) 状态管理
- v1.4 定义执行环境不再是静态的，通过 USP 管理

### 8.4 设备模块化 (Device Modularization)
- USP 服务应用框架
- Register/Deregister 消息用于 USP 服务注册
- USP Broker 架构支持
- v1.5 新增 USPServices.Trust 数据模型表用于访问控制

### 8.5 固件管理 (Firmware Management)
- 设备固件升级生命周期管理
- 支持固件镜像下载、安装、激活

### 8.6 HTTP 批量数据采集 (Bulk Data Collection)
- 支持 HTTP/HTTPS 批量数据上报
- 支持 MQTT 批量数据采集 (v1.2+)
- USPEventNotif 机制的 Push! 事件 (v1.2+)
- 支持 CSV 和 JSON 编码格式

### 8.7 IoT 控制 (IoT Theory of Operations)
- USP Agent 作为 IoT 控制网关
- IoTCapability 数据模型绑定支持 (v1.5+)
- 智能家居设备管理
- 传感器数据采集与遥测

---

## 九、Controller REST API

项目提供 Controller REST API 规范 (Swagger v1.0.0)，主要接口：

| 接口路径 | 方法 | 说明 |
|---------|------|------|
| `/agents` | POST | 创建 USP Agent |
| `/agents/{endpointID}` | GET | 获取 Agent 详情 |
| `/agents/{endpointID}` | PUT | 更新 Agent 信息 |
| `/agents/{endpointID}` | DELETE | 删除 Agent |
| `/agents/{endpointID}/status` | GET | 获取 Agent 通信状态 |
| `/agents/{endpointID}/associatedAgents` | GET | 查询关联 Agent |
| `/agents/{endpointID}/serviceElements` | POST | 创建服务元素实例 |
| `/agents/{endpointID}/serviceElements/{name}` | GET/PATCH/DELETE | 服务元素 CRUD |
| `/agents/{endpointID}/commands` | POST | 执行服务元素操作 |
| `/agents/{endpointID}/commands/{asyncRequestID}` | GET | 获取异步操作结果 |

---

## 十、版本演进

| 版本 | 日期 | 主要变更 |
|------|------|---------|
| v1.0.0 | 2018-04-17 | 初始发布 |
| v1.1.0 | 2019-10-18 | 新增 MQTT MTP; IoT 控制理论 |
| v1.2.0 | 2022-01-27 | 废弃 CoAP; 新增连接/断开记录; GetSupportedDM 增强; max_depth |
| v1.3.0 | 2023-06-14 | 新增 UDS MTP; Register/Deregister 消息; 软件模块管理; SecuredRole |
| v1.4.0 | 2024-07-23 | UDS TLS 支持; GetSupportedDM 唯一键; Register 扩展; 36-bit OUI |
| v1.5.0 | 2026-01-09 | MQTT ALPN; UDS 密码认证; originator_id/destination_id; IoTCapability 绑定 |

---

## 十一、项目结构

```
usp/
├── PROJECT.yaml                    # 项目元数据与版本历史
├── README.md                       # 项目简介
├── CHANGELOG.md                    # 变更日志
├── common.yaml                     # 公共配置与引用定义
├── api/
│   └── swagger-usp-controller-v1.yaml  # Controller REST API 规范
├── specification/                  # USP 规范文档
│   ├── index.md                    # 规范首页 (术语、范围、引用)
│   ├── architecture/               # 架构定义
│   ├── messages/                   # 消息定义 (CRUD-ON)
│   ├── encoding/                   # 消息编码 (Protocol Buffers)
│   ├── e2e-message-exchange/       # 端到端消息交换
│   ├── discovery/                  # 发现与通告
│   ├── security/                   # 认证与授权
│   ├── mtp/                        # 消息传输协议
│   │   ├── websocket/
│   │   ├── stomp/
│   │   ├── mqtt/
│   │   ├── coap/ (已废弃)
│   │   └── unix-domain-socket/
│   ├── extensions/                 # 扩展功能
│   │   ├── proxying/               # 消息代理
│   │   ├── device-proxy/           # 设备代理
│   │   ├── software-module-management/  # 软件模块管理
│   │   ├── device-modularization/  # 设备模块化
│   │   ├── firmware-management/    # 固件管理
│   │   ├── http-bulk-data-collection/   # 批量数据采集
│   │   └── iot/                    # IoT 控制
│   ├── usp-msg-1-*.proto           # USP 消息 Protobuf Schema (各版本)
│   └── usp-record-1-*.proto        # USP 记录 Protobuf Schema (各版本)
└── docs/                           # 生成的文档与静态资源
```
