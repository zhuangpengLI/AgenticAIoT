# AgenticAIoT - 自进化智能物联网平台

<p align="center">
 <img src="https://img.shields.io/badge/Spring%20Boot-3.4.5-blue.svg" alt="Spring Boot">
 <img src="https://img.shields.io/badge/Vue-3.2-blue.svg" alt="Vue">
 <img src="https://img.shields.io/badge/Spring%20AI-1.1.2-green.svg" alt="Spring AI">
 <img src="https://img.shields.io/badge/IoT%20Protocols-13+-orange.svg" alt="IoT Protocols">
 <img src="https://img.shields.io/badge/Qoder-AI%20Coding-red.svg" alt="Qoder AI">
</p>

## 平台简介

**AgenticAIoT** 是一款**企业级自进化智能物联网平台**，深度融合 **AI 大模型**、**物联网（IoT）** 与 **AI 自主编程** 三大核心能力。平台以"智能设备接入 + 数据智能流转 + 规则引擎联动 + AI 决策运维 + 自主进化"为核心理念，提供设备全生命周期管理、多协议自主接入、智能规则引擎、AI 辅助运维决策、Qoder 自主编码等核心能力，打造**可感知、可分析、可决策、可进化、可自主开发**的新一代 AIoT 智能解决方案。

> gitee: [AgenticAIoT](https://gitee.com/zhuangpengli/AgenticAIoT.git)

> gitcode: [AgenticAIoT](https://gitcode.com/lizhuangpeng/AgenticAIoT)

> github: [AgenticAIoT](https://github.com/zhuangpengLI/AgenticAIoT)

### 技术架构

* **后端框架**：JDK 17/21 + Spring Boot 3.x + Spring AI 1.1.2
* **前端框架**：Vue3（Element Plus / Vben）+ uni-app 移动端
* **数据存储**：MySQL / PostgreSQL / 达梦 + TDEngine（时序数据）+ Redis + 向量数据库（Qdrant / Milvus）
* **消息总线**：支持 Local Event、Redis Stream、RocketMQ、Kafka、RabbitMQ
* **IoT 网关**：基于 Vert.x 高性能异步框架 + Californium（CoAP）+ j2mod（Modbus）
* **权限认证**：Spring Security + Token + Redis，支持 SaaS 多租户、SSO 单点登录
* **工作流引擎**：Flowable（仿钉钉 / 飞书 + BPMN 双设计器）
* **AI 编程**：Qoder 自主编码助手 + Specs/Plans 规范化开发

### Qoder 自主编码

平台集成 **Qoder AI 编码助手**，实现 AI 驱动的自主开发能力：

| 能力 | 描述 |
|------|------|
| 智能代码生成 | 基于自然语言描述自动生成业务代码、API 接口、数据库表结构 |
| 协议扩展开发 | 通过 AI 对话快速实现新协议接入（如自定义 TCP/UDP 协议），自动生成编解码器 |
| 物模型生成 | 根据设备描述自动生成物模型定义（属性、服务、事件），减少手动配置 |
| 规则引擎配置 | 自然语言描述业务场景，自动生成场景联动规则和告警配置 |
| 代码审查与优化 | AI 自动审查代码质量，提供优化建议和安全漏洞检测 |
| 文档自动生成 | 自动生成 API 文档、数据库设计文档、协议说明文档 |

### 基于 Specs/Plans 的规范化 AI 编程

平台创新性地引入**规范化 AI 编程工作流**，通过 `.qoder` 目录下的规范文件实现高质量自主编码：

| 文件类型 | 路径 | 作用 |
|---------|------|------|
| **Specs（规范）** | `.qoder/specs/` | 定义编码规范、技术标准、架构约束、代码风格 |
| **Plans（计划）** | `.qoder/plans/` | 定义任务分解、实施步骤、验收标准、交付物清单 |
| **Agents（代理）** | `.qoder/agents/` | 定义 AI 代理角色、职责边界、协作流程 |
| **Skills（技能）** | `.qoder/skills/` | 定义可复用技能、代码模板、最佳实践 |

#### 标准化 AI 编程流程

1. **需求对齐阶段**
   - AI 自动读取 Specs 规范，理解技术栈、编码标准、架构模式
   - AI 解析 Plans 计划，明确需求范围、功能边界、验收标准
   - 生成实施方案，与用户确认后再执行

2. **方案设计阶段**
   - AI 根据规范自动设计技术方案（类图、流程图、数据模型）
   - 生成详细的实施计划（任务分解、依赖关系、优先级）
   - 输出验收标准（功能测试、性能指标、代码质量要求）

3. **自主编码阶段**
   - **无手写代码**：AI 根据方案自主生成完整代码
   - **纯 AI 编程**：从业务逻辑到单元测试，全部由 AI 生成
   - **规范遵循**：自动遵循 Specs 定义的编码规范和架构约束
   - **质量保障**：自动生成单元测试、集成测试、性能测试

4. **验收交付阶段**
   - AI 自动执行测试用例，验证功能是否符合 Plans 中的验收标准
   - 生成验收报告（测试覆盖率、性能指标、代码质量评分）
   - 输出完整文档（API 文档、使用说明、部署指南）

#### 核心优势

| 优势 | 说明 |
|------|------|
| 🎯 **需求精准对齐** | 通过 Specs/Plans 确保 AI 理解无偏差，避免"AI 乱写代码" |
| 📋 **方案先行** | 先设计方案和验收标准，经用户确认后再编码，降低返工风险 |
| 🤖 **纯 AI 自主编程** | 从需求到代码全流程 AI 化，无需手写代码，提升开发效率 10x+ |
| ✅ **质量可保障** | 自动测试 + 规范约束 + 验收标准，确保代码质量可控 |
| 🔄 **持续自进化** | 根据项目反馈自动优化 Specs/Plans，形成正向循环 |

---

## AI 大模型能力

平台深度集成 Spring AI 框架，提供开箱即用的多模态 AI 能力：

### 支持的 AI 模型平台（18+）

| 国内平台 | 国外平台 |
|---------|---------|
| 通义千问（阿里巴巴） | OpenAI（GPT 系列） |
| 文心一言（百度） | Azure OpenAI（微软） |
| DeepSeek | Anthropic（Claude） |
| 智谱 GLM | Google Gemini |
| 讯飞星火 | Ollama（本地开源模型） |
| 豆包（字节跳动） | Grok（X 公司） |
| 混元（腾讯） | Midjourney（AI 绘图） |
| 硅基流动 | Stable Diffusion |
| 百川智能 | Suno（AI 音乐） |

### AI 功能矩阵

| 功能 | 描述 |
|------|------|
| AI 对话 | 多轮对话、流式响应、上下文记忆、对话角色自定义、Web 搜索增强 |
| AI 绘图 | 文生图（DALL-E / Stable Diffusion / Midjourney），支持放大、变换等操作 |
| AI 音乐 | 描述模式 / 歌词模式生成音乐，输出音频、视频、配图 |
| AI 写作 | 撰写 / 回复模式，可配置格式、语气、语言、长度 |
| AI 思维导图 | 智能结构化输出，Markdown 格式，流式返回 |
| 知识库 / RAG | 文档上传与切片、向量检索、语义搜索，支持 Qdrant / Redis / Milvus |
| AI 工作流 | 基于 TinyFlow 的可视化工作流编排，节点拖拽、条件分支 |
| MCP 协议 | Model Context Protocol 集成，AI 可直接查询设备状态、属性历史、下发控制指令 |
| 工具调用 | Spring AI Function Calling，内置天气查询、用户信息等工具，可自定义扩展 |

### AI + IoT 融合（MCP 工具）

平台通过 MCP 协议实现 AI 与 IoT 的深度融合，AI 可直接操作物联网设备：

| MCP 工具 | 能力 |
|---------|------|
| 产品管理工具 | 查询产品列表、获取产品信息与物模型 |
| 设备管理工具 | 查询设备列表 / 状态 / 属性历史、下发控制指令 |
| 告警管理工具 | 查询告警配置与记录、告警统计分析 |
| 物模型工具 | 查询物模型定义、属性 / 事件 / 服务元数据 |

---

## IoT 物联网能力

### 支持的通信协议（13 种）

| 协议 | 说明 |
|------|------|
| MQTT | 轻量级发布 / 订阅消息协议，IoT 首选协议 |
| EMQX | EMQX 内置协议适配 |
| HTTP | RESTful API 方式接入设备 |
| WebSocket | 全双工实时通信协议 |
| CoAP | 受限应用协议，适合资源受限设备（基于 Californium） |
| TCP | 自定义 TCP 报文接入，支持定长 / 分隔符 / 长度字段编码 |
| UDP | 用户数据报协议接入 |
| Modbus TCP Client | 主动轮询 Modbus 设备数据 |
| Modbus TCP Server | 作为 Modbus 服务端接收设备上报 |
| **TR-069 (CWMP)** | CPE 广域网管理协议，用于运营商终端设备远程管理（ACS 自动配置服务器） |
| **TR-369 (USP)** | 用户服务平台协议，TR-069 演进版，支持 CRUD-ON 消息模型，多 MTP 传输（MQTT/WebSocket/STOMP） |
| **GB/T 28181** | 公安视频监控联网国家标准，实现视频设备统一接入与信令控制 |
| **ONVIF** | 开放型网络视频接口论坛标准，支持 IP 摄像头发现、设备管理、流媒体传输 |

### 协议自主扩展机制

平台采用**插件化协议架构**，支持自主扩展新协议：

* **IotProtocol 接口规范**：定义协议的 `start()`、`stop()`、`isRunning()` 完整生命周期
* **IotProtocolManager 管理器**：根据配置动态创建和管理协议实例
* **上下行处理器分离**：每个协议独立的 upstream / downstream Handler
* **编解码可配置**：TCP / UDP 支持定长、分隔符、长度字段三种编解码策略
* **扩展步骤**：实现 IotProtocol 接口 → 注册到 ProtocolManager → 启用配置即可接入新协议

---

## 规则引擎

平台提供**数据流转规则**和**场景联动规则**双引擎，实现设备数据的智能处理与自动化联动。

### 数据流转规则

将设备消息按规则路由到外部系统，支持 **9 种数据目的地（Data Sink）**：

| 数据目的地 | 说明 |
|-----------|------|
| HTTP | 推送到 HTTP 端点 |
| TCP | 推送到 TCP 连接 |
| WebSocket | 推送到 WebSocket 端点 |
| MQTT | 推送到 MQTT Broker |
| Redis | 存储到 Redis（支持 Stream / Hash / List / Set / ZSet / String 六种数据结构） |
| RocketMQ | 推送到 RocketMQ |
| RabbitMQ | 推送到 RabbitMQ |
| Kafka | 推送到 Kafka |
| Database | 存储到数据库 |

### 场景联动规则

基于**触发 - 条件 - 动作（TCA）**模型的场景联动引擎：

**触发类型：**

| 触发类型 | 说明 |
|---------|------|
| 设备状态变更 | 设备上下线状态变更触发 |
| 属性上报 | 物模型属性上报触发 |
| 事件上报 | 设备事件上报触发 |
| 服务调用 | 设备服务调用触发 |
| 定时触发 | 基于 Cron 表达式的定时触发 |

**条件判断：**
* 支持设备状态条件、设备属性条件、当前时间条件
* 丰富的操作符：`=`、`!=`、`>`、`>=`、`<`、`<=`、`in`、`not in`、`between`、`like`、日期时间比较等
* 基于 Spring Expression Language（SpEL）的灵活条件表达式

**执行动作：**

| 动作类型 | 说明 |
|---------|------|
| 设备属性设置 | 远程设置设备属性值（控制设备） |
| 设备服务调用 | 远程调用设备服务 |
| 触发告警 | 触发告警通知 |
| 恢复告警 | 自动恢复告警状态 |

---

## 告警系统

| 功能 | 描述 |
|------|------|
| 告警配置 | 自定义告警规则、告警级别、触发条件 |
| 告警记录 | 完整的告警历史记录查询与管理 |
| 告警通知 | 支持短信（SMS）、邮件（Mail）、站内信（Notify）多渠道通知 |
| 多接收人 | 支持配置多个告警接收人 |
| 告警联动 | 与场景规则引擎深度集成，支持自动触发和自动恢复 |
| AI 告警分析 | 通过 MCP 工具支持 AI 查询告警配置与记录，智能分析告警趋势 |

---

## 设备管理

### 设备全生命周期管理

| 功能 | 描述 |
|------|------|
| 产品管理 | 产品分类、产品密钥、设备类型（直连 / 网关 / 网关子设备）、联网方式（Wi-Fi / 蜂窝 / 以太网） |
| 设备管理 | 设备注册与激活、状态监控（在线 / 离线 / 未激活）、位置信息（地图展示）、批量导入导出 |
| 设备分组 | 灵活的设备分组管理 |
| 物模型 | 属性（Property）、服务（Service）、事件（Event）三要素模型，支持多种数据类型 |
| 设备消息 | 完整的设备消息历史记录与查询 |
| 设备属性 | 实时属性值查询、属性历史数据（TDEngine 时序存储） |
| 网关与子设备 | 网关拓扑管理、子设备动态注册、绑定 / 解绑 |
| Modbus 采集 | Modbus 设备配置、采集点管理 |

### OTA 固件升级

| 功能 | 描述 |
|------|------|
| 固件管理 | 固件版本管理、固件文件上传与发布 |
| 升级任务 | 创建升级任务，支持全部设备 / 指定设备升级范围 |
| 进度追踪 | 单设备升级状态跟踪（待升级 / 升级中 / 成功 / 失败 / 超时） |
| 任务管理 | 升级任务状态管理（进行中 / 已结束 / 已取消） |

---

## 数据统计与可视化

| 功能 | 描述 |
|------|------|
| 全局概览 | 产品总数、设备总数、消息总数、今日新增统计 |
| 设备状态分布 | 在线 / 离线 / 未激活设备分布 |
| 品类设备统计 | 各产品分类下的设备数量 |
| 消息趋势 | 设备消息按日期的时间序列统计 |
| 报表与大屏 | 集成报表设计器、大屏设计器，拖拽生成数据报表与大屏 |

---

## 内置通用功能

系统内置多种业务功能，可用于快速搭建物联网平台系统：

* 通用模块（必选）：系统功能、基础设施
* 通用模块（可选）：工作流程、支付系统、数据报表、会员中心
* 业务系统（按需）：微信公众号、AI 大模型、IoT 物联网

### 系统功能

|     | 功能    | 描述                              |
|-----|-------|---------------------------------|
|     | 用户管理  | 用户是系统操作者，该功能主要完成系统用户配置          |
| ⭐️  | 在线用户  | 当前系统中活跃用户状态监控，支持手动踢下线           |
|     | 角色管理  | 角色菜单权限分配、设置角色按机构进行数据范围权限划分      |
|     | 菜单管理  | 配置系统菜单、操作权限、按钮权限标识等，本地缓存提供性能    |
|     | 部门管理  | 配置系统组织机构（公司、部门、小组），树结构展现支持数据权限  |
|     | 岗位管理  | 配置系统用户所属担任职务                    |
| 🚀  | 租户管理  | 配置系统租户，支持 SaaS 场景下的多租户功能        |
| 🚀  | 租户套餐  | 配置租户套餐，自定每个租户的菜单、操作、按钮的权限       |
|     | 字典管理  | 对系统中经常使用的一些较为固定的数据进行维护          |
| 🚀  | 短信管理  | 短信渠道、短信模板、短信日志，对接阿里云、腾讯云等主流短信平台 |
| 🚀  | 邮件管理  | 邮箱账号、邮件模版、邮件发送日志，支持所有邮件平台       |
| 🚀  | 站内信   | 系统内的消息通知，提供站内信模版、站内信消息          |
| 🚀  | 操作日志  | 系统正常操作日志记录和查询，集成 Swagger 生成日志内容 |
| ⭐️  | 登录日志  | 系统登录日志记录查询，包含登录异常               |
| 🚀  | 错误码管理 | 系统所有错误码的管理，可在线修改错误提示，无需重启服务     |
|     | 通知公告  | 系统通知公告信息发布维护                    |
| 🚀  | 敏感词   | 配置系统敏感词，支持标签分组                  |
| 🚀  | 应用管理  | 管理 SSO 单点登录的应用，支持多种 OAuth2 授权方式 |
| 🚀  | 地区管理  | 展示省份、城市、区镇等城市信息，支持 IP 对应城市      |

### 基础设施

|     | 功能        | 描述                                           |
|-----|-----------|----------------------------------------------|
| 🚀  | 代码生成      | 前后端代码的生成（Java、Vue、SQL、单元测试），支持 CRUD 下载       |
| 🚀  | 系统接口      | 基于 Swagger 自动生成相关的 RESTful API 接口文档          |
| 🚀  | 数据库文档     | 基于 Screw 自动生成数据库文档，支持导出 Word、HTML、MD 格式      |
|     | 表单构建      | 拖动表单元素生成相应的 HTML 代码，支持导出 JSON、Vue 文件         |
| 🚀  | 配置管理      | 对系统动态配置常用参数，支持 SpringBoot 加载                 |
| ⭐️  | 定时任务      | 在线（添加、修改、删除）任务调度包含执行结果日志                     |
| 🚀  | 文件服务      | 支持将文件存储到 S3（MinIO、阿里云、腾讯云、七牛云）、本地、FTP、数据库等   |
| 🚀  | WebSocket | 提供 WebSocket 接入示例，支持一对一、一对多发送方式              |
| 🚀  | API 日志    | 包括 RESTful API 访问日志、异常日志两部分，方便排查 API 相关的问题   |
|     | MySQL 监控  | 监视当前系统数据库连接池状态，可进行分析 SQL 找出系统性能瓶颈            |
|     | Redis 监控  | 监控 Redis 数据库的使用情况，使用的 Redis Key 管理           |
| 🚀  | 消息队列      | 基于 Redis 实现消息队列，Stream 提供集群消费，Pub/Sub 提供广播消费 |
| 🚀  | Java 监控   | 基于 Spring Boot Admin 实现 Java 应用的监控           |
| 🚀  | 链路追踪      | 接入 SkyWalking 组件，实现链路追踪                      |
| 🚀  | 日志中心      | 接入 SkyWalking 组件，实现日志中心                      |
| 🚀  | 服务保障      | 基于 Redis 实现分布式锁、幂等、限流功能，满足高并发场景              |
| 🚀  | 日志服务      | 轻量级日志中心，查看远程服务器的日志                           |
| 🚀  | 单元测试      | 基于 JUnit + Mockito 实现单元测试，保证功能的正确性、代码的质量等    |

### 工作流程

基于 Flowable 构建，支持仿钉钉 / 飞书 + BPMN 双设计器，满足从简单审批到复杂流程编排的全场景需求。支持会签、或签、依次审批、转办、委派、加签、条件分支、并行分支、父子流程、超时审批、自动提醒等完整流程能力。

---

## 技术栈

### 模块

| 项目                    | 说明                    |
|-----------------------|-----------------------|
| `yudao-dependencies`  | Maven 依赖版本管理          |
| `yudao-framework`     | Java 框架拓展             |
| `yudao-server`        | 管理后台 + 用户 APP 的服务端    |
| `yudao-module-system` | 系统功能的 Module 模块       |
| `yudao-module-infra`  | 基础设施的 Module 模块       |
| `yudao-module-iot`    | IoT 物联网的 Module 模块    |
| `yudao-module-ai`     | AI 大模型的 Module 模块     |
| `yudao-module-pay`    | 支付系统的 Module 模块       |
| `yudao-module-mp`     | 微信公众号的 Module 模块      |
| `yudao-module-report` | 大屏报表 Module 模块        |

### 框架

| 框架                                                                                          | 说明               | 版本             | 学习指南                                                           |
|---------------------------------------------------------------------------------------------|------------------|----------------|----------------------------------------------------------------|
| [Spring Boot](https://spring.io/projects/spring-boot)                                       | 应用开发框架           | 3.5.5          | [文档](https://github.com/YunaiV/SpringBoot-Labs)                |
| [MySQL](https://www.mysql.com/cn/)                                                          | 数据库服务器           | 5.7 / 8.0+     |                                                                |
| [Druid](https://github.com/alibaba/druid)                                                   | JDBC 连接池、监控组件    | 1.2.27         | [文档](http://www.iocoder.cn/Spring-Boot/datasource-pool/?yudao) |
| [MyBatis Plus](https://mp.baomidou.com/)                                                    | MyBatis 增强工具包    | 3.5.12         | [文档](http://www.iocoder.cn/Spring-Boot/MyBatis/?yudao)         |
| [Dynamic Datasource](https://dynamic-datasource.com/)                                       | 动态数据源            | 4.3.1          | [文档](http://www.iocoder.cn/Spring-Boot/datasource-pool/?yudao) |
| [Redis](https://redis.io/)                                                                  | key-value 数据库    | 5.0 / 6.0 /7.0 |                                                                |
| [Redisson](https://github.com/redisson/redisson)                                            | Redis 客户端        | 3.35.0         | [文档](http://www.iocoder.cn/Spring-Boot/Redis/?yudao)           |
| [Spring MVC](https://github.com/spring-projects/spring-framework/tree/master/spring-webmvc) | MVC 框架           | 6.2.9          | [文档](http://www.iocoder.cn/SpringMVC/MVC/?yudao)               |
| [Spring Security](https://github.com/spring-projects/spring-security)                       | Spring 安全框架      | 6.5.2          | [文档](http://www.iocoder.cn/Spring-Boot/Spring-Security/?yudao) |
| [Hibernate Validator](https://github.com/hibernate/hibernate-validator)                     | 参数校验组件           | 8.0.2          | [文档](http://www.iocoder.cn/Spring-Boot/Validation/?yudao)      |
| [Flowable](https://github.com/flowable/flowable-engine)                                     | 工作流引擎            | 7.0.0          | [文档](https://doc.iocoder.cn/bpm/)                              |
| [Quartz](https://github.com/quartz-scheduler)                                               | 任务调度组件           | 2.5.0          | [文档](http://www.iocoder.cn/Spring-Boot/Job/?yudao)             |
| [Springdoc](https://springdoc.org/)                                                         | Swagger 文档       | 2.8.9          | [文档](http://www.iocoder.cn/Spring-Boot/Swagger/?yudao)         |
| [SkyWalking](https://skywalking.apache.org/)                                                | 分布式应用追踪系统        | 9.5.0          | [文档](http://www.iocoder.cn/Spring-Boot/SkyWalking/?yudao)      |
| [Spring Boot Admin](https://github.com/codecentric/spring-boot-admin)                       | Spring Boot 监控平台 | 3.5.2          | [文档](http://www.iocoder.cn/Spring-Boot/Admin/?yudao)           |
| [Jackson](https://github.com/FasterXML/jackson)                                             | JSON 工具库         | 2.30.14        |                                                                |
| [MapStruct](https://mapstruct.org/)                                                         | Java Bean 转换     | 1.6.3          | [文档](http://www.iocoder.cn/Spring-Boot/MapStruct/?yudao)       |
| [Lombok](https://projectlombok.org/)                                                        | 消除冗长的 Java 代码    | 1.18.38        | [文档](http://www.iocoder.cn/Spring-Boot/Lombok/?yudao)          |
| [JUnit](https://junit.org/junit5/)                                                          | Java 单元测试框架      | 5.12.2         | -                                                              |
| [Mockito](https://github.com/mockito/mockito)                                               | Java Mock 框架     | 5.17.0         | -                                                              |

---

## 交流社区

欢迎加入 AgenticCPS 交流社区，与志同道合的开发者、创业者一起探索 Vibe Coding 与 CPS 赚钱的无限可能！

| 渠道 | 二维码 |
|:---------:|:--------:|
| 加入知识星球，获取深度教程、源码解析、运营经验分享| ![知识星球](/.image/知识星球.jpg) |
| 添加群主，备注：进技术交流群 |  ![微信](/.image/微信.png)  |
| 扫码加入微信群，获取最新动态、技术答疑、部署支持 |  ![微信群](/.image/微信群.png)  |



> **微信群**：技术交流、Bug 反馈、功能建议，欢迎扫码加入。
>
> **知识星球**：付费精品社区，内含完整部署教程、Vibe Coding 实战案例、一人公司 CPS 创业经验分享，以及专属答疑服务。

---

## 开源协议

本项目采用 **GNU Affero General Public License v3.0 (AGPL-3.0)** 开源协议。

### 协议要点

| 要点 | 说明 |
|------|------|
| 📜 开源义务 | 任何使用、修改、分发本项目的代码必须以 AGPL-3.0 协议开源 |
| 🌐 网络服务条款 | 通过网络提供服务（如 SaaS）时，必须向用户提供完整源代码 |
| 🔄 著作权归属 | 修改后的代码需保留原作者著作权声明和协议声明 |
| ⚠️ 无担保声明 | 本软件按"原样"提供，不提供任何形式的担保 |

### 使用场景说明

* ✅ **允许**：个人学习、研究、商业二次开发（需开源）
* ✅ **允许**：内部企业使用（无需开源，除非对外提供服务）
* ⚠️ **需开源**：基于本项目提供 SaaS 服务或对外网络服务
* ❌ **禁止**：闭源商业化分发、移除版权声明

### 完整协议文本

详见 [LICENSE](./LICENSE) 文件或访问 [GNU AGPL-3.0 官方协议文本](https://www.gnu.org/licenses/agpl-3.0.html)

---

## 💝 赞助与支持

开源项目的发展离不开社区的支持。如果您觉得本项目对您有帮助，欢迎赞助支持持续开发！

### 为什么需要赞助？

| 用途 | 说明 |
|------|------|
| 🖥️ 服务器部署 | 测试环境、演示环境、CI/CD 服务器租赁费用 |
| 🤖 AI Token 费用 | 大模型 API 调用费用（通义千问、DeepSeek、OpenAI 等） |
| 🔧 持续开发 | 新协议开发（TR-069/TR-369/GB28181/ONVIF）、功能迭代、Bug 修复 |
| 📚 文档完善 | 技术文档、API 文档、视频教程制作 |

### 赞助方式

#### 微信支付 / 支付宝

<p align="center">
  <img src="./.image/微信收款.jpg" alt="微信支付">
  <img src="./.image/支付宝收款.jpg" alt="支付宝">
</p>

> 💡 请在备注中留下您的 GitHub ID 或联系方式，我们将列在赞助者名单中（如愿意公开）


### 企业赞助

欢迎企业用户进行商业赞助，我们将提供以下回报：

| 赞助等级 | 回报 |
|---------|------|
| 🥉 青铜赞助商 | README 中显示企业 Logo |
| 🥈 白银赞助商 | 优先 Issue 处理 + Logo 展示 |
| 🥇 黄金赞助商 | 专属技术支持 + 优先功能开发 + Logo 展示 |
| 💎 钻石赞助商 | 定制开发支持 + 专属技术顾问 + 首页显著展示 |

---

## 🎁 功能悬赏

为了加速项目功能开发，我们推出**功能悬赏计划**。您可以悬赏特定功能的开发，开发者完成后可获得赏金。

### 当前悬赏列表

| 功能 | 悬赏金额   | 状态 | 说明 |
|------|--------|------|------|
| TR-069 完整实现 | ¥1,000 | 🟡 进行中 | 包含所有 RPC 方法、文件传输、诊断测试 |
| TR-369 完整实现 | ¥1,000 | 🟡 进行中 | Protobuf 消息、多 MTP 支持、安全机制 |
| GB/T 28181 视频接入 | ¥1,000 | 🔴 待开发 | SIP 信令、SDP 协商、 RTP/PS 流媒体 |
| ONVIF 设备发现 | ¥1,000 | 🔴 待开发 | WS-Discovery、设备管理、流媒体传输 |
| Modbus RTU 支持 | ¥1,000 | 🔴 待开发 | 串口通信、RTU 协议栈 |

### 悬赏规则

1. **认领任务**：在 Issue 中评论认领，确认后开始开发
2. **开发周期**：根据功能复杂度协商，一般 2-4 周
3. **代码审核**：提交 PR 后进行代码审核，通过后合并
4. **发放赏金**：合并后 3 个工作日内发放至指定账户

### 如何发起悬赏？

如果您需要特定功能但不在列表中，可以：

1. 在 [Issues](../../issues) 中创建功能请求，标注 `💰 悬赏` 标签
2. 说明功能需求和悬赏金额
3. 等待开发者认领或我们评估后添加到悬赏列表

---

### 🙏 感谢所有赞助者

感谢以下赞助者的慷慨支持（按时间排序）：

<!-- 赞助者名单将在此处更新 -->

> 成为第一个赞助者，让开源走得更远！
