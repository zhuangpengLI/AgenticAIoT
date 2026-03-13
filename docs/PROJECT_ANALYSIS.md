# Oktopus 项目代码目录与功能分析

## 1. 项目概述

Oktopus 是一个开源的 **USP (TR-369) 控制器** 和 **CWMP (TR-069) 兼容** 的多厂商 CPE/IoT 设备管理平台。目标是将设备管理统一到单一软件解决方案中，提供丰富的洞察和配置能力，适用于 CSP/ISP 场景。

---

## 2. 顶层目录结构

```
oktopus/
├── agent/                  # USP Agent (OB-USP-A) Docker 构建
├── backend/                # 后端微服务 (Go + Node.js)
│   └── services/
│       ├── acs/            # CWMP ACS 服务
│       ├── bulkdata/       # 批量数据采集服务
│       ├── controller/     # 核心控制器 (REST API)
│       ├── mtp/            # 消息传输协议层
│       │   ├── adapter/    # MTP 通用适配器
│       │   ├── mqtt/       # MQTT Broker
│       │   ├── mqtt-adapter/  # MQTT-NATS 桥接适配器
│       │   ├── ws/         # WebSocket 服务器
│       │   ├── ws-adapter/ # WebSocket-NATS 桥接适配器
│       │   ├── stomp/      # STOMP 服务器
│       │   └── stomp-adapter/ # STOMP-NATS 桥接适配器
│       └── utils/
│           ├── socketio/   # Socket.IO 实时通信服务
│           └── file-server/# 静态文件服务器
├── frontend/               # 前端 Web 应用 (Next.js + React + MUI)
├── build/                  # 全局构建脚本 (Makefile)
├── deploy/                 # 部署配置
│   ├── compose/            # Docker Compose 编排
│   └── kubernetes/         # Kubernetes 部署清单
├── .circleci/              # CI/CD 配置
└── .github/                # GitHub 工作流
```

---

## 3. 后端微服务详解

### 3.1 Controller (核心控制器)

- **路径**: `backend/services/controller/`
- **语言**: Go
- **核心依赖**: gorilla/mux, NATS JetStream, MongoDB
- **主要功能**:
  - **REST API 网关**: 对外提供统一的 HTTP API，端口 8000
  - **USP 协议处理**: 构建/解析 USP (TR-369) 消息 (protobuf)，支持 Get/Set/Add/Delete/Operate/Notify 等操作
  - **CWMP 协议处理**: 构建/解析 TR-069 SOAP XML 消息，支持 GetParameterValues/SetParameterValues/AddObject/DeleteObject 等 RPC
  - **设备管理**: 设备列表查询、状态追踪、别名设置、过滤选项、WiFi 管理
  - **固件更新**: 设备远程固件升级接口
  - **用户认证**: JWT Token 生成、用户注册/登录/删除/改密、管理员用户管理
  - **仪表盘信息**: 厂商统计、设备状态统计、产品类别统计、总览信息
  - **消息模板**: 支持 USP 消息模板的增删改查
  - **NATS 桥接**: 通过 NATS 消息总线与 MTP 层双向通信

**API 路由概览**:

| 路径前缀 | 功能 |
|---|---|
| `/api/auth/*` | 用户认证 (登录/注册/改密/删除) |
| `/api/device/*` | 设备管理 (CRUD/USP/CWMP 操作) |
| `/api/device/{sn}/{mtp}/*` | USP 设备消息操作 (get/set/add/del/notify/operate) |
| `/api/device/cwmp/{sn}/*` | CWMP 设备消息操作 |
| `/api/info/*` | 仪表盘统计信息 |
| `/api/users` | 用户列表 |

**内部模块结构**:
```
controller/internal/
├── api/            # HTTP API 路由和处理器
│   ├── auth/       # JWT 认证中间件
│   ├── cors/       # CORS 配置
│   └── middleware/  # 请求中间件
├── bridge/         # NATS 通信桥接 (USP/CWMP 消息转发)
├── config/         # 配置加载 (环境变量)
├── cwmp/           # CWMP SOAP XML 消息编解码
├── db/             # MongoDB 数据库操作 (设备/用户/模板)
├── entity/         # 数据实体定义 (Device/Msg/Status/MTP)
├── nats/           # NATS 客户端初始化
├── usp/            # USP protobuf 消息处理
│   ├── usp_msg/    # USP Message protobuf 定义 (v1.3)
│   ├── usp_record/ # USP Record protobuf 定义 (v1.3)
│   └── usp_utils/  # USP 工具函数
└── utils/          # 通用工具函数
```

---

### 3.2 ACS (Auto Configuration Server - CWMP)

- **路径**: `backend/services/acs/`
- **语言**: Go
- **主要功能**:
  - 实现 TR-069 CWMP ACS 服务器，端口 9292
  - 接收 CPE 的 Inform 请求并处理 CWMP 会话
  - CWMP SOAP 消息编解码 (GetParameterValues, SetParameterValues, GetParameterNames 等)
  - 通过 NATS 与 Controller 和 Adapter 通信
  - 设备连接请求 (Connection Request) 转发
  - 支持设备认证

**内部模块结构**:
```
acs/internal/
├── auth/           # 设备认证
├── bridge/         # NATS 桥接通信
├── config/         # 配置管理
├── cwmp/           # CWMP 协议实现 (SOAP XML)
├── nats/           # NATS 客户端
└── server/         # HTTP 服务器
    └── handler/    # CWMP 请求处理器
```

---

### 3.3 MTP (消息传输协议层)

MTP 层是 Oktopus 的通信骨架，负责设备与控制器之间的消息传输。采用**协议服务器 + 适配器**的架构模式。

#### 3.3.1 MQTT

- **mqtt** (`backend/services/mtp/mqtt/`): MQTT Broker 服务器，支持 MQTT/WebSocket/HTTP 多种监听方式，通过 Hook 机制记录设备连接/断开事件并通知 NATS
- **mqtt-adapter** (`backend/services/mtp/mqtt-adapter/`): MQTT-NATS 桥接，将 MQTT 主题上的 USP 消息转发到 NATS，反之亦然

#### 3.3.2 WebSocket

- **ws** (`backend/services/mtp/ws/`): WebSocket 服务器，支持 TLS，提供 `/ws/agent` 和 `/ws/controller` 端点，管理 Agent 和 Controller 的 WebSocket 连接
- **ws-adapter** (`backend/services/mtp/ws-adapter/`): WebSocket-NATS 桥接，将 WebSocket 消息转发到 NATS

#### 3.3.3 STOMP

- **stomp** (`backend/services/mtp/stomp/`): STOMP 协议服务器，基于 `go-stomp` 库，支持用户名密码认证
- **stomp-adapter** (`backend/services/mtp/stomp-adapter/`): STOMP-NATS 桥接，包含完整的 STOMP 帧解析实现

#### 3.3.4 Adapter (通用适配器)

- **路径**: `backend/services/mtp/adapter/`
- **功能**: 统一的 MTP 事件/请求处理适配器
  - 监听 NATS JetStream 事件 (设备上线/下线/消息)
  - 分别处理 USP 和 CWMP 两种协议的设备信息和状态
  - 将设备信息、状态持久化到 MongoDB
  - 处理来自 Controller 的请求并转发到对应协议

---

### 3.4 Bulkdata (批量数据采集)

- **路径**: `backend/services/bulkdata/http/`
- **语言**: Go
- **功能**: HTTP 批量数据采集器，接收设备上报的批量数据并通过 NATS 转发

---

### 3.5 Utils (工具服务)

#### 3.5.1 Socket.IO 服务

- **路径**: `backend/services/utils/socketio/`
- **语言**: Node.js (Express + Socket.IO)
- **端口**: 5000
- **功能**: 为前端提供实时通信能力，支持用户在线状态管理、WebRTC 信令 (callUser/answerCall)

#### 3.5.2 File Server (文件服务器)

- **路径**: `backend/services/utils/file-server/`
- **语言**: Go
- **功能**: 简单的静态文件服务器，用于分发固件文件和镜像资源

---

## 4. 前端应用

- **路径**: `frontend/`
- **技术栈**: Next.js + React 18 + MUI (Material UI) 5 + Emotion
- **功能页面**:

| 页面 | 路径 | 功能描述 |
|---|---|---|
| Overview | `/` | 仪表盘总览 (设备统计、厂商分布、在线状态) |
| Devices | `/devices` | 设备列表管理、搜索过滤 |
| Device Detail (USP) | `/devices/usp/[id]` | USP 设备详情 (参数发现、RPC 操作) |
| Device Detail (CWMP) | `/devices/cwmp/[id]` | CWMP 设备详情 (参数查询、WiFi 配置) |
| Credentials | `/credentials` | 设备认证凭据管理 |
| Users | `/access-control/users` | 用户访问控制管理 |
| Settings | `/settings` | 系统设置 (主题颜色、通知、密码) |
| Login/Register | `/auth/login`, `/auth/register` | 用户登录/注册 |
| Chat | `/chat` | 用户间实时通信 (WebRTC) |

**前端架构**:
```
frontend/src/
├── components/     # 通用组件 (Logo, Chart, Scrollbar, SeverityPill)
├── contexts/       # React Context (Auth, Backend API, SocketIO, Error)
├── guards/         # 路由守卫 (Auth Guard)
├── hocs/           # 高阶组件
├── hooks/          # 自定义 Hooks (useAuth, usePopover, useSelection)
├── layouts/        # 页面布局 (Dashboard 侧边栏/顶栏, Auth 布局)
├── pages/          # Next.js 页面路由
├── sections/       # 页面区块组件
│   ├── devices/    # 设备相关 (USP Discovery/RPC, CWMP RPC/WiFi)
│   ├── overview/   # 仪表盘图表和统计卡片
│   ├── credentials/# 凭据表格
│   ├── settings/   # 设置面板
│   └── account/    # 账户信息
├── theme/          # MUI 主题定制 (颜色/排版/组件/阴影)
└── utils/          # 工具函数
```

---

## 5. Agent (USP Agent)

- **路径**: `agent/`
- **功能**: 基于 OB-USP-A (Open Broadband USP Agent) 的 Docker 镜像构建
- **支持的 MTP**: MQTT, STOMP, WebSocket
- **用途**: 用于测试或在设备端运行 USP Agent，配置文件分别对应三种传输协议

---

## 6. 部署配置

### 6.1 Docker Compose

- **路径**: `deploy/compose/`
- **服务编排**: 包含全部微服务的 Docker Compose 配置
- **使用 profiles 管理可选组件**:
  - `nats`: NATS 消息代理
  - `controller`: Controller + MongoDB
  - `adapter`: MTP Adapter + MongoDB
  - `mqtt`: MQTT Broker + MQTT Adapter
  - `ws`: WebSocket 服务 + WebSocket Adapter
  - `stomp`: STOMP 服务 + STOMP Adapter
  - `cwmp`: ACS (CWMP) 服务
  - `frontend`: 前端 + Socket.IO
  - `portainer`: 容器管理 UI
- **基础设施**: NATS (消息总线), MongoDB (数据库), Nginx (反向代理)
- **网络**: 自定义 bridge 网络 `172.16.235.0/24`

### 6.2 Kubernetes

- **路径**: `deploy/kubernetes/`
- **包含**: Controller, Adapter, MQTT, MQTT-Adapter, WS, WS-Adapter, Frontend, Socket.IO, MongoDB, Ingress 等 YAML 清单

---

## 7. 构建系统

- **路径**: `build/Makefile`
- **命令**:
  - `make build`: 构建全部 Docker 镜像 (前端 + 后端)
  - `make build-backend`: 仅构建后端所有微服务镜像
  - `make build-frontend`: 仅构建前端镜像
  - `make release`: 构建并推送到 Docker Hub

---

## 8. 系统架构总览

```
                         ┌──────────────┐
                         │   Frontend   │ (Next.js)
                         │   :3000      │
                         └──────┬───────┘
                                │ HTTP / Socket.IO
                    ┌───────────┼───────────┐
                    │           │           │
             ┌──────▼─────┐ ┌──▼──────┐ ┌──▼──────────┐
             │ Controller │ │SocketIO │ │   Nginx     │
             │  REST API  │ │  :5000  │ │ (反向代理)   │
             │   :8000    │ └─────────┘ └─────────────┘
             └──────┬─────┘
                    │
              ┌─────▼─────┐
              │   NATS     │ (消息总线 / JetStream)
              │   :4222    │
              └─────┬──────┘
                    │
     ┌──────────────┼──────────────────────┐
     │              │              │        │
┌────▼────┐  ┌──────▼──────┐  ┌───▼───┐ ┌──▼───┐
│ Adapter │  │ ACS (CWMP)  │  │BulkD. │ │ MTP  │
│ (通用)  │  │   :9292     │  │       │ │ 层   │
└────┬────┘  └─────────────┘  └───────┘ └──┬───┘
     │                                     │
     │         ┌───────────────────────────┐│
     │         │     MTP 服务器 + 适配器   ││
     │         │ ┌────────┐ ┌──────────┐  ││
     │         │ │ MQTT   │ │MQTT-Adapt│  ││
     │         │ │ :1883  │ │  (桥接)  │  ││
     │         │ ├────────┤ ├──────────┤  ││
     │         │ │  WS    │ │WS-Adapter│  ││
     │         │ │ :8080  │ │  (桥接)  │  ││
     │         │ ├────────┤ ├──────────┤  ││
     │         │ │ STOMP  │ │STOMP-Adpt│  ││
     │         │ │:61613  │ │  (桥接)  │  ││
     │         │ └────────┘ └──────────┘  ││
     │         └───────────────────────────┘│
     │                                      │
  ┌──▼──────┐                        ┌──────▼──────┐
  │ MongoDB │                        │ CPE / IoT   │
  │ :27017  │                        │   设备       │
  └─────────┘                        └─────────────┘
```

---

## 9. 核心技术栈

| 层级 | 技术 |
|---|---|
| 前端 | Next.js, React 18, MUI 5, Socket.IO Client, ApexCharts, WebRTC (simple-peer) |
| 后端 API | Go, gorilla/mux, JWT 认证 |
| 消息总线 | NATS + JetStream (含 KeyValue Store) |
| 数据库 | MongoDB |
| 协议支持 | USP TR-369 (protobuf), CWMP TR-069 (SOAP XML) |
| MTP 传输 | MQTT, WebSocket, STOMP |
| 容器化 | Docker, Docker Compose, Kubernetes |
| CI/CD | CircleCI, GitHub Actions |
| 反向代理 | Nginx |

---

## 10. 主要功能点汇总

1. **多协议设备管理**: 同时支持 USP (TR-369) 和 CWMP (TR-069) 两种标准协议
2. **多传输层支持**: MQTT / WebSocket / STOMP 三种 MTP 可选
3. **设备生命周期管理**: 设备注册、状态监控 (在线/离线)、信息采集
4. **远程参数操作**: 参数读取/设置/发现、对象增删、实例查询
5. **远程固件升级**: 支持 OTA 固件更新
6. **WiFi 管理**: 远程查看和配置设备 WiFi 参数
7. **批量数据采集**: HTTP Bulk Data Collection 机制
8. **用户与权限**: 用户注册/登录、JWT 认证、管理员角色
9. **仪表盘与可视化**: 设备统计、厂商分布、在线率图表
10. **实时通信**: Socket.IO 推送、WebRTC 视频/音频通话
11. **消息模板**: 可复用的 USP 消息模板管理
12. **设备认证**: 设备接入认证与凭据管理
13. **微服务架构**: 通过 NATS 消息总线解耦，各服务独立部署扩展
14. **多种部署方式**: Docker Compose 一键部署、Kubernetes 集群部署
