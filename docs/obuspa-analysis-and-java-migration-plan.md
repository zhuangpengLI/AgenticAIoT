# OB-USP-AGENT 项目分析与 Java 改造计划

## 一、项目概述

**项目名称：** OB-USP-AGENT (Open Broadband User Services Platform Agent)

**项目来源：** Broadband Forum 开源项目

**许可证：** BSD 3-Clause

**编程语言：** C (ANSI C / C99)

**构建系统：** GNU Autotools (autoconf/automake) + CMake

**项目定位：** USP (User Services Platform) 协议的 Agent 端参考实现。USP 是一种远程管理和控制协议，管理实体被分为 Agent 和 Controller。Agent 负责暴露一组"服务元素"（由对象和参数组成的数据模型），供 Controller 消费。典型应用场景为家庭网络中的 CPE（客户终端设备），如宽带家用路由器、Wi-Fi 接入点、IoT 网关等。

**协议版本支持：** USP 1.0 / 1.1 / 1.2 / 1.3

---

## 二、项目结构

```
obuspa/
├── src/
│   ├── core/              # 核心USP Agent实现 (~80+ .c/.h 文件)
│   │   ├── main.c         # 程序入口
│   │   ├── data_model.c   # 数据模型引擎 (211KB, 核心模块)
│   │   ├── usp_broker.c   # USP Broker功能 (362KB, 最大文件)
│   │   ├── msg_handler.c  # USP消息路由处理
│   │   ├── path_resolver.c # 路径解析器
│   │   ├── device_*.c     # 设备数据模型组件 (10个文件)
│   │   ├── handle_*.c     # USP消息处理器 (8个文件)
│   │   ├── group_*_vector.c # 分组操作向量 (4个文件)
│   │   ├── stomp.c        # STOMP协议实现 (153KB)
│   │   ├── mqtt.c         # MQTT协议实现 (169KB)
│   │   ├── coap_*.c       # CoAP协议实现 (3个文件)
│   │   ├── wsclient.c     # WebSocket客户端 (101KB)
│   │   ├── wsserver.c     # WebSocket服务端 (80KB)
│   │   ├── uds.c          # Unix Domain Socket (109KB)
│   │   ├── database.c     # SQLite数据库层
│   │   ├── dm_*.c         # 数据模型辅助模块 (4个文件)
│   │   ├── usp_*.c        # USP基础设施 (7个文件)
│   │   └── [工具类文件]    # 各种向量、文本、网络工具
│   ├── include/           # 公共API头文件
│   │   ├── usp_api.h      # USP Agent核心API (供Vendor调用)
│   │   ├── usp_err_codes.h # 错误码定义
│   │   ├── vendor_api.h   # Vendor回调API
│   │   └── compiler.h     # 编译器兼容性宏
│   ├── vendor/            # 厂商自定义扩展
│   │   ├── vendor.c       # Vendor Hook入口
│   │   ├── vendor_defs.h  # Vendor可配置的宏定义 (数组大小、功能开关)
│   │   └── vendor_factory_reset_example.c  # 出厂复位示例
│   ├── libjson/           # JSON解析库 (CCAN json)
│   │   └── ccan/json/
│   └── protobuf-c/        # Protobuf-C序列化实现
│       ├── protobuf-c.c/h # Protobuf-C运行时库
│       ├── usp-msg.pb-c.c/h    # USP消息Protobuf定义
│       └── usp-record.pb-c.c/h # USP记录Protobuf定义
├── examples/
│   ├── plugin/            # Vendor插件开发示例
│   └── test_uds/          # UDS连接测试示例
├── ci/                    # CI/CD配置
│   ├── Dockerfile         # CI构建环境
│   ├── cdrouter-runner.py # CDRouter自动化测试
│   ├── configs/           # 各MTP协议测试配置
│   └── docker-compose.yml
├── configure.ac           # Autotools配置脚本
├── Makefile.am            # Automake构建定义
├── CMakeLists.txt         # CMake构建定义
├── Dockerfile             # Docker构建/运行
├── README.md
├── QUICK_START_GUIDE.md   # 快速入门指南
├── INTERNAL_SERVICES_GUIDE.md  # 内部服务开发指南
├── ROLES_AND_PERMISSIONS.md    # 角色和权限说明
├── CHANGELOG.md           # 变更日志
├── RELEASES.md            # 发布说明
└── conformance_test_results.txt  # TP-469一致性测试结果
```

### 代码规模统计

| 类别 | 文件数 | 说明 |
|------|--------|------|
| C 源文件 (.c) | ~90 | 核心代码约80个，示例/库约10个 |
| C 头文件 (.h) | ~67 | 公共API 4个，核心57个，库6个 |
| 核心代码总量 | ~4.5MB | 仅 src/core/ 目录 |
| 最大单文件 | 362KB | usp_broker.c |

---

## 三、主要功能模块分析

### 3.1 MTP (消息传输协议) 层

MTP 层负责 USP 消息的物理传输，支持5种传输协议：

| 协议 | 源文件 | 配置文件 | 说明 |
|------|--------|----------|------|
| **STOMP** | `stomp.c` (153KB), `device_stomp.c` | 默认启用 | 基于TCP的简单文本消息协议 |
| **MQTT** | `mqtt.c` (169KB), `device_mqtt.c` | `--enable-mqtt` | 基于libmosquitto，IoT常用 |
| **CoAP** | `coap_client.c`, `coap_server.c`, `coap_common.c` | `--enable-coap` | 受限网络轻量协议 |
| **WebSocket** | `wsclient.c` (101KB), `wsserver.c` (80KB) | `--enable-websockets` | 基于libwebsockets |
| **UDS** | `uds.c` (109KB), `device_uds.c` | `--enable-uds` | Unix Domain Socket，本地进程间通信 |

**MTP 执行引擎** (`mtp_exec.c`)：管理各MTP协议线程的生命周期、调度和唤醒。每种协议有独立的主循环线程。

### 3.2 数据模型引擎

USP Agent 的核心功能是维护和暴露一个分层的数据模型 (TR-181)。

| 模块 | 源文件 | 功能 |
|------|--------|------|
| **数据模型核心** | `data_model.c` (211KB) | 数据模型节点注册、查询、遍历、参数类型管理 |
| **路径解析器** | `path_resolver.c` (115KB) | 解析数据模型路径表达式（含搜索表达式、通配符） |
| **访问控制** | `dm_access.c` (37KB) | 基于角色的权限控制 (RBAC) |
| **执行引擎** | `dm_exec.c` (105KB) | 数据模型线程主循环，处理内部消息队列 |
| **实例管理** | `dm_inst_vector.c` (45KB) | 多实例对象的实例号管理 |
| **事务处理** | `dm_trans.c` (18KB) | 数据模型修改的事务支持 |

**数据模型节点类型：**
- 单实例对象 / 多实例对象
- 常量参数 / 数量参数（NumEntries）
- 数据库参数（只读/读写/安全/自动生成）
- 厂商参数（只读/读写）
- 同步/异步操作
- 事件

### 3.3 USP 消息处理

USP 协议定义了一组标准的 CRUD + 操作 + 通知消息类型：

| 消息类型 | 源文件 | 功能 |
|----------|--------|------|
| **Get** | `handle_get.c` | 获取参数值 |
| **Set** | `handle_set.c` | 设置参数值 |
| **Add** | `handle_add.c` | 添加对象实例 |
| **Delete** | `handle_delete.c` | 删除对象实例 |
| **Operate** | `handle_operate.c` | 执行同步/异步操作 |
| **Notify** | `handle_notify.c` | 通知和订阅处理 |
| **GetInstances** | `handle_get_instances.c` | 查询实例编号 |
| **GetSupportedDM** | `handle_get_supported_dm.c` | 查询支持的数据模型结构 |
| **GetSupportedProtocol** | `handle_get_supported_protocol.c` | 协议版本协商 |

**消息路由** (`msg_handler.c`, 82KB)：根据消息类型分发到对应的处理器，管理请求/响应匹配。

**消息工具** (`msg_utils.c`, 51KB)：消息构造、序列化/反序列化辅助函数。

**分组操作向量**：`group_get_vector.c`, `group_set_vector.c`, `group_add_vector.c`, `group_del_vector.c` -- 批量操作时将参数按 group_id 分组执行。

### 3.4 设备数据模型组件

实现 TR-181 数据模型的各个子树：

| 组件 | 源文件 | 数据模型路径 |
|------|--------|-------------|
| **本地代理** | `device_local_agent.c` (47KB) | `Device.LocalAgent.*` |
| **控制器管理** | `device_controller.c` (191KB) | `Device.LocalAgent.Controller.*` |
| **MTP配置** | `device_mtp.c` (85KB) | `Device.LocalAgent.MTP.*` |
| **安全/TLS** | `device_security.c` (139KB) | `Device.LocalAgent.Certificate.*`, TLS管理 |
| **控制器信任** | `device_ctrust.c` (132KB) | `Device.LocalAgent.ControllerTrust.*` |
| **订阅管理** | `device_subscription.c` (149KB) | `Device.LocalAgent.Subscription.*` |
| **批量数据采集** | `device_bulkdata.c` (125KB) | `Device.BulkData.*` |
| **STOMP配置** | `device_stomp.c` (52KB) | `Device.STOMP.*` |
| **MQTT配置** | `device_mqtt.c` (89KB) | `Device.MQTT.*` |
| **时间管理** | `device_time.c` (26KB) | 时间同步和维护 |
| **UDS配置** | `device_uds.c` (40KB) | `Device.UnixDomainSockets.*` |
| **请求管理** | `device_request.c` (26KB) | `Device.LocalAgent.Request.*` |

### 3.5 USP Broker / Service

| 模块 | 源文件 | 功能 |
|------|--------|------|
| **USP Broker** | `usp_broker.c` (362KB) | Agent作为Broker时的功能，管理多个USP Service的注册/注销、消息路由转发、数据模型代理 |
| **USP Service** | `usp_service.c` (55KB) | Agent作为USP Service时的功能，向Broker注册数据模型对象 |

### 3.6 基础设施层

| 模块 | 源文件 | 功能 |
|------|--------|------|
| **数据库** | `database.c` (56KB) | SQLite 数据库封装，参数持久化存储 |
| **错误处理** | `usp_err.c` (17KB) | 错误码管理、错误消息格式化 |
| **日志** | `usp_log.c` (20KB) | 日志系统（syslog/stdout/文件） |
| **内存管理** | `usp_mem.c` (24KB) | 内存分配/释放追踪（调试用内存泄漏检测） |
| **USP Record** | `usp_record.c` (13KB) | USP Record 编码/解码 |
| **E2E 上下文** | `e2e_context.c` (40KB) | 端到端会话管理 |
| **搜索缓存** | `se_cache.c` (59KB) | 搜索表达式结果缓存 |
| **CLI** | `cli_server.c` (73KB), `cli_client.c` (9KB) | 命令行接口（通过UDS通信） |
| **插件系统** | `plugin.c` (8KB) | 动态加载 .so 插件 |
| **协议追踪** | `proto_trace.c` (10KB) | USP 消息收发日志追踪 |

### 3.7 工具类

| 模块 | 源文件 | 功能 |
|------|--------|------|
| **文本工具** | `text_utils.c` (74KB) | 字符串操作、类型转换、路径操作 |
| **IP地址** | `nu_ipaddr.c` (46KB) | IP地址解析、接口枚举、DNS |
| **MAC地址** | `nu_macaddr.c` (5KB) | MAC地址获取 |
| **ISO 8601** | `iso8601.c` (9KB) | 日期时间解析 |
| **RFC 1123** | `rfc1123.c` (4KB) | HTTP日期格式化 |
| **OS工具** | `os_utils.c` (11KB) | 线程ID、信号处理 |
| **各类向量** | `str_vector.c`, `int_vector.c`, `kv_vector.c`, `expr_vector.c`, `subs_vector.c`, `sar_vector.c`, `fd_vector.c`, `inst_sel_vector.c` | 动态数组容器 |
| **双向链表** | `dllist.c` (9KB) | 双向链表实现 |
| **Socket集合** | `socket_set.c` (10KB) | select/poll封装 |
| **同步定时器** | `sync_timer.c` (13KB) | 定时器管理 |
| **重试等待** | `retry_wait.c` (8KB) | 指数退避重连 |

### 3.8 序列化层

| 模块 | 说明 |
|------|------|
| **protobuf-c** | Protocol Buffers C运行时，用于USP消息的序列化/反序列化 |
| **usp-msg.pb-c** | 由 .proto 文件生成的 USP Message 结构体和编解码函数 |
| **usp-record.pb-c** | 由 .proto 文件生成的 USP Record 结构体和编解码函数 |
| **libjson (CCAN)** | JSON 解析/生成，用于 Bulk Data Collection 等场景 |

### 3.9 外部依赖

| 依赖库 | 用途 |
|--------|------|
| **OpenSSL** | TLS/SSL 加密通信、证书管理、信任链验证 |
| **SQLite3** | 参数值持久化存储 |
| **libcurl** | HTTP/HTTPS 通信（Bulk Data Collection上报） |
| **zlib** | 数据压缩 |
| **libmosquitto** | MQTT 客户端协议实现 |
| **libwebsockets** | WebSocket 客户端/服务端实现 |
| **pthread** | POSIX 多线程 |
| **libdl** | 动态库加载（插件系统） |

---

## 四、线程架构

OB-USP-AGENT 采用多线程架构：

```
+------------------+     +------------------+     +------------------+
|   Data Model     |     |   STOMP MTP      |     |   MQTT MTP       |
|   Thread (Main)  |     |   Thread         |     |   Thread         |
|   dm_exec.c      |     |   stomp.c        |     |   mqtt.c         |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
         +----------+-------------+----------+-------------+
                    |                        |
              +-----------+            +-----------+
              | Message   |            | CoAP MTP  |
              | Queue     |            | Thread    |
              | (pipe fd) |            +-----------+
              +-----------+
                    |
         +----------+-------------+
         |                        |
+--------+---------+     +--------+---------+
|  WebSocket MTP   |     |   UDS MTP        |
|  Thread          |     |   Thread         |
+------------------+     +------------------+
```

- **数据模型线程 (主线程)：** 处理 USP 消息、数据模型操作、订阅通知、定时器
- **MTP 线程：** 每种启用的 MTP 协议有独立线程，通过 pipe 和消息队列与主线程通信
- **Bulk Data Collection 线程：** 独立线程处理数据采集和上报

---

## 五、Java 改造计划

### 5.1 改造目标

将 OB-USP-AGENT 从 C 语言改造为 Java 实现，保持功能完整性和协议兼容性，同时利用 Java 生态提升可维护性、可扩展性和跨平台能力。

### 5.2 技术选型

| 领域 | C 原实现 | Java 替代方案 |
|------|----------|---------------|
| **构建工具** | Autotools / CMake | Maven 或 Gradle |
| **序列化** | protobuf-c (手写C绑定) | protobuf-java (官方Java支持) |
| **数据库** | SQLite3 (C API) | SQLite JDBC 或 H2 Database |
| **MQTT** | libmosquitto | Eclipse Paho MQTT Client |
| **WebSocket** | libwebsockets | Java-WebSocket / Tyrus / Netty |
| **CoAP** | 自实现 (基于OpenSSL DTLS) | Eclipse Californium |
| **STOMP** | 自实现 | Spring STOMP Client 或自实现 |
| **TLS/SSL** | OpenSSL | Java SSE (javax.net.ssl) / Bouncy Castle |
| **HTTP** | libcurl | Java HttpClient (JDK 11+) / OkHttp |
| **JSON** | CCAN libjson | Jackson / Gson |
| **线程模型** | pthread | Java ThreadPoolExecutor / Virtual Threads (JDK 21+) |
| **日志** | 自实现 (syslog) | SLF4J + Logback |
| **CLI** | Unix Domain Socket | JLine3 / picocli |
| **插件系统** | dlopen/dlsym | Java SPI (ServiceLoader) / OSGi |
| **压缩** | zlib | java.util.zip |

### 5.3 项目结构设计

```
obuspa-java/
├── pom.xml (或 build.gradle)
├── src/main/java/org/broadbandforum/obuspa/
│   ├── OBUSPAgent.java                    # 应用入口
│   ├── config/
│   │   └── AgentConfig.java               # 配置管理
│   │
│   ├── datamodel/                         # 数据模型引擎
│   │   ├── DataModel.java                 # 数据模型核心 (对应 data_model.c)
│   │   ├── DataModelNode.java             # 数据模型节点定义
│   │   ├── DataModelNodeType.java         # 节点类型枚举
│   │   ├── PathResolver.java              # 路径解析器 (对应 path_resolver.c)
│   │   ├── DataModelAccess.java           # 访问控制 (对应 dm_access.c)
│   │   ├── DataModelExecutor.java         # 执行引擎 (对应 dm_exec.c)
│   │   ├── InstanceVector.java            # 实例号管理
│   │   ├── TransactionManager.java        # 事务管理 (对应 dm_trans.c)
│   │   └── SearchExpressionCache.java     # 搜索缓存
│   │
│   ├── mtp/                               # 消息传输协议层
│   │   ├── MtpProtocol.java               # MTP协议接口
│   │   ├── MtpExecutor.java               # MTP线程管理 (对应 mtp_exec.c)
│   │   ├── MtpSendItem.java               # 发送消息封装
│   │   ├── stomp/
│   │   │   ├── StompClient.java           # STOMP协议实现
│   │   │   └── StompConfig.java           # STOMP配置
│   │   ├── mqtt/
│   │   │   ├── MqttClient.java            # MQTT协议实现
│   │   │   └── MqttConfig.java
│   │   ├── coap/
│   │   │   ├── CoapClient.java            # CoAP客户端
│   │   │   ├── CoapServer.java            # CoAP服务端
│   │   │   └── CoapCommon.java
│   │   ├── websocket/
│   │   │   ├── WsClient.java              # WebSocket客户端
│   │   │   └── WsServer.java              # WebSocket服务端
│   │   └── uds/
│   │       └── UdsTransport.java          # UDS传输(Linux平台专用)
│   │
│   ├── handler/                           # USP消息处理器
│   │   ├── MessageHandler.java            # 消息路由 (对应 msg_handler.c)
│   │   ├── MessageUtils.java              # 消息工具
│   │   ├── GetHandler.java                # Get消息处理
│   │   ├── SetHandler.java                # Set消息处理
│   │   ├── AddHandler.java                # Add消息处理
│   │   ├── DeleteHandler.java             # Delete消息处理
│   │   ├── OperateHandler.java            # Operate消息处理
│   │   ├── NotifyHandler.java             # Notify消息处理
│   │   ├── GetInstancesHandler.java       # GetInstances处理
│   │   ├── GetSupportedDMHandler.java     # GetSupportedDM处理
│   │   └── GetSupportedProtocolHandler.java
│   │
│   ├── device/                            # 设备数据模型实现
│   │   ├── DeviceLocalAgent.java          # LocalAgent数据模型
│   │   ├── DeviceController.java          # Controller管理
│   │   ├── DeviceMtp.java                 # MTP配置
│   │   ├── DeviceSecurity.java            # 安全/TLS管理
│   │   ├── DeviceControllerTrust.java     # 控制器信任
│   │   ├── DeviceSubscription.java        # 订阅管理
│   │   ├── DeviceBulkData.java            # 批量数据采集
│   │   ├── DeviceStomp.java               # STOMP配置
│   │   ├── DeviceMqtt.java                # MQTT配置
│   │   ├── DeviceTime.java                # 时间管理
│   │   └── DeviceRequest.java             # 请求管理
│   │
│   ├── broker/                            # USP Broker / Service
│   │   ├── UspBroker.java                 # Broker功能
│   │   └── UspService.java               # Service功能
│   │
│   ├── persistence/                       # 持久化层
│   │   ├── Database.java                  # 数据库抽象接口
│   │   ├── SqliteDatabase.java            # SQLite实现
│   │   └── DatabaseMigration.java         # 数据库迁移
│   │
│   ├── security/                          # 安全模块
│   │   ├── TlsManager.java               # TLS连接管理
│   │   ├── CertificateManager.java        # 证书管理
│   │   └── TrustStore.java               # 信任库管理
│   │
│   ├── record/                            # USP Record层
│   │   ├── UspRecord.java                 # USP Record编解码
│   │   ├── E2EContext.java                # E2E会话上下文
│   │   └── SarVector.java                # 分段/重组
│   │
│   ├── cli/                               # CLI交互
│   │   ├── CliServer.java                 # CLI服务端
│   │   └── CliClient.java                 # CLI客户端
│   │
│   ├── plugin/                            # 插件系统
│   │   ├── PluginManager.java             # 插件加载管理
│   │   └── VendorPlugin.java             # Vendor插件接口
│   │
│   ├── util/                              # 工具类
│   │   ├── TextUtils.java                 # 文本处理工具
│   │   ├── IpAddressUtils.java            # IP地址工具
│   │   ├── MacAddressUtils.java           # MAC地址工具
│   │   ├── Iso8601Utils.java              # ISO8601日期时间
│   │   ├── RetryWait.java                 # 重试等待策略
│   │   ├── SyncTimer.java                 # 同步定时器
│   │   └── SocketSet.java                # Socket集合管理
│   │
│   ├── error/                             # 错误处理
│   │   ├── UspError.java                  # USP错误类
│   │   └── UspErrorCode.java             # 错误码枚举
│   │
│   └── proto/                             # Protobuf生成代码(自动生成)
│       ├── UspMsg.java
│       └── UspRecord.java
│
├── src/main/proto/                        # Protobuf定义文件
│   ├── usp-msg.proto
│   └── usp-record.proto
│
├── src/main/resources/
│   ├── application.yml                    # 应用配置
│   ├── factory_reset_default.txt          # 默认出厂复位参数
│   └── logback.xml                        # 日志配置
│
├── src/test/java/org/broadbandforum/obuspa/
│   ├── datamodel/
│   ├── handler/
│   ├── mtp/
│   └── ...                                # 单元测试
│
└── src/test/resources/                    # 测试资源
```

### 5.4 改造实施步骤

#### 阶段一：基础框架搭建

**步骤 1：项目初始化**
- 使用 Maven/Gradle 创建 Java 项目骨架
- 配置 protobuf-maven-plugin，从 `.proto` 文件生成 Java 类
- 引入基础依赖（SLF4J, Jackson, SQLite JDBC 等）
- 配置 JDK 版本（建议 JDK 17 LTS 或 JDK 21 LTS）

**步骤 2：Protobuf 消息层迁移**
- 获取 USP 官方 `.proto` 文件（usp-msg.proto, usp-record.proto）
- 使用 protobuf-java 编译生成 Java 消息类
- 验证序列化/反序列化与 C 版本的二进制兼容性

**步骤 3：错误码与基础类型**
- 迁移 `usp_err_codes.h` → `UspErrorCode.java` (枚举)
- 迁移基础数据结构：`kv_vector` → `Map<String, String>` 或自定义 `KvVector`
- `str_vector` → `List<String>`, `int_vector` → `List<Integer>`
- 迁移 `vendor_defs.h` 中的常量 → Java 常量类

#### 阶段二：核心引擎实现

**步骤 4：数据模型引擎**
- 设计 `DataModelNode` 类层次结构（利用 Java 继承/多态替代 C 的函数指针）
- 实现 `DataModel` 核心类：节点注册、路径查找、遍历
- 实现 `PathResolver`：搜索表达式、通配符解析
- 实现 `InstanceVector`：多实例对象管理
- 实现 `TransactionManager`：事务支持

**步骤 5：数据库层**
- 使用 JDBC + SQLite 实现 `Database` 接口
- 迁移数据库 schema 和 hash 函数
- 实现参数的 CRUD 操作和出厂复位逻辑

**步骤 6：访问控制**
- 实现 `DataModelAccess`：角色权限模型
- 迁移 `combined_role_t` 和权限位图

#### 阶段三：消息处理层

**步骤 7：USP 消息处理框架**
- 实现 `MessageHandler`：消息类型路由
- 定义 `UspMessageProcessor` 接口，各 handler 实现此接口
- 实现所有 9 种消息处理器（Get/Set/Add/Delete/Operate/Notify/GetInstances/GetSupportedDM/GetSupportedProtocol）
- 实现 `MessageUtils`：消息构造辅助

**步骤 8：USP Record 层**
- 实现 USP Record 编解码
- 实现端到端会话（E2E Context）
- 实现消息分段/重组（SAR）

#### 阶段四：MTP 传输层

**步骤 9：MTP 框架**
- 定义 `MtpProtocol` 接口（connect, disconnect, send, receive）
- 实现 `MtpExecutor`：管理 MTP 连接池和线程

**步骤 10：STOMP MTP**
- 实现 STOMP 1.2 客户端协议
- TLS 支持（Java SSE）
- 心跳机制、重连策略

**步骤 11：MQTT MTP**
- 基于 Eclipse Paho 实现 MQTT 客户端
- 支持 MQTT v3.1.1 和 v5.0
- 主题订阅、QoS 管理

**步骤 12：WebSocket MTP**
- 实现 WebSocket 客户端（连接 Controller）
- 实现 WebSocket 服务端（接收 Controller 连接）
- 子协议协商（`v1.usp`）

**步骤 13：CoAP MTP**
- 基于 Eclipse Californium 实现 CoAP
- DTLS 支持
- Block-wise 传输

**步骤 14：UDS MTP**
- 使用 JDK 16+ 的 UnixDomainSocketAddress
- 实现 USP Broker 本地通信

#### 阶段五：设备数据模型

**步骤 15：核心设备模型**
- `DeviceLocalAgent`：本地代理信息（EndpointID, 版本等）
- `DeviceController`：控制器管理（最复杂的设备模型，191KB）
- `DeviceMtp`：MTP 连接配置

**步骤 16：安全模型**
- `DeviceSecurity`：Java KeyStore 集成
- `DeviceControllerTrust`：角色和权限管理
- 证书管理（X.509）

**步骤 17：订阅与通知**
- `DeviceSubscription`：订阅生命周期
- ValueChange / ObjectCreation / ObjectDeletion / OperationComplete / Event 通知类型
- 通知重试机制

**步骤 18：其他设备模型**
- `DeviceBulkData`：数据采集和 HTTP 上报
- `DeviceStomp` / `DeviceMqtt`：协议配置数据模型
- `DeviceTime` / `DeviceRequest`

#### 阶段六：高级功能

**步骤 19：USP Broker**
- 实现 Broker 核心：USP Service 注册/注销
- 数据模型代理和消息转发
- Passthrough 请求处理

**步骤 20：USP Service 模式**
- 实现 Service 注册流程
- Broker-Service 通信（通过 UDS）

**步骤 21：CLI 系统**
- 使用 picocli 或 JLine3 实现命令行交互
- `dbget`, `dbset`, `dbdel` 等数据库操作命令
- `get`, `set`, `add`, `delete`, `operate` 等 USP 命令

**步骤 22：插件系统**
- 基于 Java SPI (ServiceLoader) 实现 Vendor 插件加载
- 定义 `VendorPlugin` 接口（替代 C 的函数指针回调）

#### 阶段七：测试与验证

**步骤 23：单元测试**
- 为每个核心模块编写 JUnit 5 单元测试
- 数据模型路径解析、消息编解码、数据库操作的测试覆盖

**步骤 24：集成测试**
- MTP 协议集成测试（连接真实 STOMP/MQTT/WebSocket 服务器）
- USP Broker-Service 集成测试

**步骤 25：协议一致性测试**
- 与 C 版本交叉测试（确保协议兼容性）
- 参照 TP-469 conformance test plan 验证

**步骤 26：性能测试**
- 消息吞吐量基准测试
- 内存使用对比分析
- 并发连接压力测试

#### 阶段八：打包与部署

**步骤 27：Docker 支持**
- 编写 Java 版 Dockerfile（基于 JRE 运行时镜像）
- 多阶段构建（构建 + 运行）

**步骤 28：文档**
- Java API 文档 (Javadoc)
- 配置说明和迁移指南
- Vendor 集成开发指南

### 5.5 关键设计决策

#### 5.5.1 数据模型节点设计

C 版本使用结构体 + 函数指针实现多态：
```c
typedef struct {
    dm_node_type_t type;
    dm_get_value_cb_t get_cb;
    dm_set_value_cb_t set_cb;
    // ...
} dm_node_t;
```

Java 版本应使用面向对象的继承体系：
```java
public abstract class DataModelNode {
    private String name;
    private DataModelNodeType type;
    private DataModelNode parent;
    private Map<String, DataModelNode> children;
    // ...
}

public class ParameterNode extends DataModelNode {
    public abstract String getValue(DmInstances instances);
    public abstract void setValue(DmInstances instances, String value);
}

public class ObjectNode extends DataModelNode {
    private boolean multiInstance;
    public abstract List<Integer> getInstances();
}
```

#### 5.5.2 条件编译替代方案

C 版本大量使用 `#ifdef` 进行功能裁剪：
```c
#ifdef ENABLE_MQTT
    // MQTT code
#endif
```

Java 版本应使用以下替代方案：
- **编译时：** Maven Profile 控制依赖引入
- **运行时：** 配置文件 + 条件 Bean 加载（类似 Spring 的 `@ConditionalOnProperty`）
- **接口隔离：** MTP 协议通过 SPI 机制按需加载

#### 5.5.3 线程模型

C 版本每个 MTP 协议一个 pthread 线程，使用 pipe 通信：

Java 版本建议：
- 使用 `ExecutorService` 管理线程池
- 考虑使用 JDK 21 Virtual Threads 提升并发效率
- 使用 `BlockingQueue` 替代 pipe 进行线程间通信
- 使用 `CompletableFuture` 处理异步操作结果

#### 5.5.4 内存管理

C 版本有自定义内存追踪 (`usp_mem.c`)：

Java 版本：
- 依赖 JVM GC，无需手动内存管理
- 对于大对象（如 Bulk Data 报告），使用对象池模式
- 使用 JMX 或 Micrometer 进行内存监控

### 5.6 风险与挑战

| 风险 | 说明 | 应对策略 |
|------|------|----------|
| **性能差异** | Java 在网络 I/O 密集场景下可能有额外开销 | 使用 NIO / Netty 优化，Virtual Threads 减少上下文切换 |
| **平台特定功能** | UDS (Unix Domain Socket) 在 Windows 不可用 | JDK 16+ 已支持 UDS；Windows 可降级为 TCP |
| **Protobuf 兼容性** | 需确保 Java/C 版本 Protobuf 编解码完全一致 | 使用同一 .proto 文件生成，交叉测试验证 |
| **C 语言习惯差异** | 大量全局变量、宏定义、函数指针需重新设计 | 使用 OOP 设计模式、依赖注入、策略模式等 |
| **代码体量巨大** | 核心 C 代码约 4.5MB，涉及 80+ 源文件 | 分阶段迁移，优先核心路径，逐步补齐 |
| **嵌入式场景** | C 版本面向嵌入式设备，Java 资源消耗更大 | GraalVM Native Image 可减小体积和启动时间 |
| **OpenSSL 特性** | 部分 OpenSSL 高级功能在 Java SSE 中无对应 | Bouncy Castle 库补充 |

### 5.7 依赖清单 (Maven)

```xml
<dependencies>
    <!-- Protobuf -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>3.25.x</version>
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>org.xerial</groupId>
        <artifactId>sqlite-jdbc</artifactId>
        <version>3.45.x</version>
    </dependency>

    <!-- MQTT -->
    <dependency>
        <groupId>org.eclipse.paho</groupId>
        <artifactId>org.eclipse.paho.mqttv5.client</artifactId>
        <version>1.2.x</version>
    </dependency>

    <!-- CoAP -->
    <dependency>
        <groupId>org.eclipse.californium</groupId>
        <artifactId>californium-core</artifactId>
        <version>3.x</version>
    </dependency>

    <!-- WebSocket -->
    <dependency>
        <groupId>org.java-websocket</groupId>
        <artifactId>Java-WebSocket</artifactId>
        <version>1.5.x</version>
    </dependency>

    <!-- JSON -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.17.x</version>
    </dependency>

    <!-- HTTP Client -->
    <!-- JDK 11+ 内置 java.net.http.HttpClient -->

    <!-- Logging -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.4.x</version>
    </dependency>

    <!-- CLI -->
    <dependency>
        <groupId>info.picocli</groupId>
        <artifactId>picocli</artifactId>
        <version>4.7.x</version>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.x</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.x</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 六、C 到 Java 核心映射表

| C 模块/文件 | Java 类 | 包路径 |
|-------------|---------|--------|
| `main.c` | `OBUSPAgent` | `org.broadbandforum.obuspa` |
| `data_model.c/h` | `DataModel`, `DataModelNode` | `datamodel` |
| `path_resolver.c/h` | `PathResolver` | `datamodel` |
| `dm_access.c/h` | `DataModelAccess` | `datamodel` |
| `dm_exec.c/h` | `DataModelExecutor` | `datamodel` |
| `dm_trans.c/h` | `TransactionManager` | `datamodel` |
| `msg_handler.c/h` | `MessageHandler` | `handler` |
| `handle_get.c` | `GetHandler` | `handler` |
| `handle_set.c` | `SetHandler` | `handler` |
| `handle_add.c` | `AddHandler` | `handler` |
| `handle_delete.c` | `DeleteHandler` | `handler` |
| `handle_operate.c` | `OperateHandler` | `handler` |
| `handle_notify.c` | `NotifyHandler` | `handler` |
| `mtp_exec.c/h` | `MtpExecutor` | `mtp` |
| `stomp.c/h` | `StompClient` | `mtp.stomp` |
| `mqtt.c/h` | `MqttClient` | `mtp.mqtt` |
| `coap_*.c` | `CoapClient`, `CoapServer` | `mtp.coap` |
| `wsclient.c/h`, `wsserver.c/h` | `WsClient`, `WsServer` | `mtp.websocket` |
| `uds.c/h` | `UdsTransport` | `mtp.uds` |
| `database.c/h` | `Database`, `SqliteDatabase` | `persistence` |
| `device_*.c` | `Device*` 各类 | `device` |
| `usp_broker.c/h` | `UspBroker` | `broker` |
| `usp_service.c/h` | `UspService` | `broker` |
| `usp_err.c/h` | `UspError`, `UspErrorCode` | `error` |
| `usp_log.c/h` | SLF4J (无需自实现) | - |
| `usp_mem.c/h` | JVM GC (无需自实现) | - |
| `text_utils.c/h` | `TextUtils` | `util` |
| `vendor_defs.h` | `AgentConfig` | `config` |
| `protobuf-c/*` | protobuf-java 自动生成 | `proto` |
| `libjson/*` | Jackson/Gson | - |

---

## 七、总结

OB-USP-AGENT 是一个功能完整、协议兼容性良好的 USP Agent C 语言参考实现，代码体量约 4.5MB（核心部分），涵盖 5 种 MTP 传输协议、完整的 TR-181 数据模型引擎、USP Broker/Service 架构、以及丰富的安全和管理功能。

Java 改造的核心挑战在于：
1. **代码体量大**：需要分阶段、模块化迁移
2. **C 到 OOP 的范式转换**：大量全局变量和函数指针需重新设计为面向对象架构
3. **协议兼容性**：必须确保 Protobuf 消息、TLS 行为与 C 版本完全一致
4. **性能目标**：在保持功能等价的前提下，利用 Java 生态优势提升可维护性

建议采用 **自底向上、核心优先** 的迁移策略：先完成 Protobuf 层和数据模型引擎，再逐步实现消息处理和 MTP 传输层，最后补齐 Broker/Service 等高级功能。
