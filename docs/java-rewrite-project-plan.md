# Oktopus TR-369/USP Java 重写项目计划

## 1. 项目概述

将现有 Go 微服务架构的 Oktopus TR-369/USP Controller 平台用 Java 重写，保持相同的微服务架构和功能，采用 Spring Boot 生态体系。

### 1.1 技术选型

| 组件 | Go 原始技术 | Java 替代方案 | 说明 |
|------|------------|--------------|------|
| Web 框架 | gorilla/mux | Spring Boot + Spring MVC | REST API 框架 |
| 消息总线 | nats.go | jnats (io.nats:jnats) | NATS Java 客户端 |
| 消息编码 | protobuf-go | protobuf-java / protoc-gen-grpc-java | Protobuf 序列化 |
| 数据库 | mongo-driver | Spring Data MongoDB | MongoDB 访问层 |
| MQTT 客户端 | paho.golang | Eclipse Paho Java Client | MQTT 桥接 |
| MQTT Broker | mochi-co/mqtt | HiveMQ CE / Moquette / 自研 | 内嵌 MQTT Broker |
| STOMP | 自研 frame 协议 | Spring WebSocket STOMP | STOMP 支持 |
| WebSocket | 标准库 | Spring WebSocket | WebSocket 支持 |
| 认证/授权 | 自研 JWT | Spring Security + JWT | 用户认证 |
| 配置管理 | 环境变量 | Spring Boot Config + Profile | 配置管理 |
| 容器化 | Docker | Docker (Jib / Dockerfile) | 容器构建 |
| 构建工具 | go mod | Maven / Gradle | 依赖管理 |

### 1.2 项目结构规划

```
oktopus-java/
├── pom.xml                              # 父POM（多模块Maven项目）
├── oktopus-common/                      # 公共模块
│   ├── src/main/java/
│   │   └── com/oktopus/common/
│   │       ├── proto/                   # Protobuf生成的Java类
│   │       │   ├── usp_msg/             # USP Msg 1.3
│   │       │   └── usp_record/          # USP Record 1.3
│   │       ├── model/                   # 公共数据模型
│   │       │   ├── Device.java
│   │       │   ├── DeviceStatus.java
│   │       │   └── MtpType.java
│   │       ├── usp/                     # USP消息构造工具
│   │       │   ├── UspMessageBuilder.java
│   │       │   └── UspRecordBuilder.java
│   │       └── nats/                    # NATS公共配置
│   │           └── NatsSubjects.java
│   └── src/main/proto/                  # .proto源文件
│       ├── usp-msg-1-3.proto
│       └── usp-record-1-3.proto
│
├── oktopus-controller/                  # Controller服务（REST API）
│   ├── src/main/java/
│   │   └── com/oktopus/controller/
│   │       ├── ControllerApplication.java
│   │       ├── config/
│   │       │   ├── SecurityConfig.java
│   │       │   ├── NatsConfig.java
│   │       │   └── CorsConfig.java
│   │       ├── api/
│   │       │   ├── DeviceController.java       # 设备管理API
│   │       │   ├── UspController.java          # USP消息API
│   │       │   ├── FirmwareController.java     # 固件升级API
│   │       │   ├── AuthController.java         # 认证API
│   │       │   ├── InfoController.java         # 仪表盘API
│   │       │   └── TemplateController.java     # 消息模板API
│   │       ├── service/
│   │       │   ├── UspMessageService.java      # USP消息发送服务
│   │       │   ├── DeviceService.java          # 设备管理服务
│   │       │   ├── FirmwareService.java        # 固件升级服务
│   │       │   ├── NatsBridgeService.java      # NATS通信服务
│   │       │   └── AuthService.java            # JWT认证服务
│   │       ├── repository/
│   │       │   ├── DeviceRepository.java
│   │       │   ├── UserRepository.java
│   │       │   └── TemplateRepository.java
│   │       └── dto/
│   │           ├── DeviceDTO.java
│   │           ├── UspRequestDTO.java
│   │           └── FirmwareUpdateDTO.java
│   └── src/main/resources/
│       └── application.yml
│
├── oktopus-adapter/                     # MTP Adapter服务（设备事件处理）
│   ├── src/main/java/
│   │   └── com/oktopus/adapter/
│   │       ├── AdapterApplication.java
│   │       ├── config/
│   │       │   ├── NatsConfig.java
│   │       │   └── MongoConfig.java
│   │       ├── event/
│   │       │   ├── EventListener.java          # JetStream事件监听
│   │       │   ├── UspStatusHandler.java       # 设备上下线处理
│   │       │   ├── UspInfoHandler.java         # 设备信息处理
│   │       │   └── CwmpHandler.java            # CWMP事件处理
│   │       ├── service/
│   │       │   ├── DeviceService.java          # 设备CRUD
│   │       │   └── NatsRequestService.java     # NATS请求应答
│   │       └── repository/
│   │           └── DeviceRepository.java
│   └── src/main/resources/
│       └── application.yml
│
├── oktopus-mqtt-broker/                 # MQTT Broker服务
│   ├── src/main/java/
│   │   └── com/oktopus/mqtt/broker/
│   │       ├── MqttBrokerApplication.java
│   │       ├── config/
│   │       │   └── BrokerConfig.java
│   │       ├── hook/
│   │       │   ├── DeviceConnectionHook.java   # 设备连接/断开检测
│   │       │   └── DeviceAuthHook.java         # 设备认证Hook
│   │       └── bridge/
│   │           └── NatsBridge.java             # NATS桥接
│   └── src/main/resources/
│       └── application.yml
│
├── oktopus-mqtt-adapter/                # MQTT Adapter服务
│   ├── src/main/java/
│   │   └── com/oktopus/mqtt/adapter/
│   │       ├── MqttAdapterApplication.java
│   │       ├── config/
│   │       │   ├── MqttConfig.java
│   │       │   └── NatsConfig.java
│   │       └── bridge/
│   │           ├── MqttToNatsBridge.java       # MQTT->NATS
│   │           └── NatsToMqttBridge.java       # NATS->MQTT
│   └── src/main/resources/
│       └── application.yml
│
├── oktopus-ws-adapter/                  # WebSocket Adapter服务
│   └── (结构类似mqtt-adapter)
│
├── oktopus-stomp-adapter/               # STOMP Adapter服务
│   └── (结构类似mqtt-adapter)
│
└── deploy/
    ├── docker-compose.yml
    └── Dockerfile.*
```

---

## 2. 实施阶段规划

### 阶段一：基础框架搭建

**目标**: 搭建 Maven 多模块项目骨架，完成基础设施组件

| 任务 | 说明 | 产出 |
|------|------|------|
| 1.1 创建 Maven 多模块项目 | 父 POM + 各子模块 | 项目骨架 |
| 1.2 Protobuf 代码生成 | 配置 protobuf-maven-plugin，生成 Java 类 | USP Msg/Record Java 类 |
| 1.3 NATS 连接管理 | 封装 NATS 连接、JetStream、KV Store | NatsConfig, NatsTemplate |
| 1.4 MongoDB 配置 | Spring Data MongoDB 集成 | Repository 层 |
| 1.5 USP 消息构造工具 | UspMessageBuilder, UspRecordBuilder | common 模块工具类 |
| 1.6 公共模型定义 | Device, Status, MTP 枚举等 | common 模块实体类 |

**关键依赖:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>io.nats</groupId>
        <artifactId>jnats</artifactId>
        <version>2.20.5</version>
    </dependency>
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>4.28.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
</dependencies>
```

---

### 阶段二：MQTT Broker 与 Adapter

**目标**: 实现 MQTT Broker 以及 MQTT <-> NATS 桥接

| 任务 | 说明 | 对应Go代码 |
|------|------|-----------|
| 2.1 MQTT Broker 搭建 | 集成 HiveMQ CE 或 Moquette 作为嵌入式 Broker | `mtp/mqtt/` |
| 2.2 设备认证 Hook | 实现基于 NATS KV 的 MQTT 认证插件 | `hook.go` NatsAuthHook |
| 2.3 连接事件 Hook | 设备订阅/断开检测，发布 status 消息 | `hook.go` MyHook |
| 2.4 CONNACK 属性注入 | 返回 subscribe-topic 属性 | `OnPacketEncode` |
| 2.5 MQTT Adapter 开发 | MQTT <-> NATS 双向桥接 | `mqtt-adapter/bridge/` |
| 2.6 MQTT 消息路由 | status/controller/api 三类消息分发 | `mqttMessageHandler` |
| 2.7 NATS -> MQTT 转发 | 订阅 NATS subject，发布到 MQTT topic | `natsMessageHandler` |

**MQTT Broker 技术选型对比:**

| 方案 | 优点 | 缺点 |
|------|------|------|
| HiveMQ CE | 成熟稳定，扩展API丰富 | 社区版功能有限 |
| Moquette | 轻量级，嵌入式友好 | 插件生态较小 |
| Eclipse Mosquitto + Java Client | 标准实现 | 非嵌入式，需独立部署 |

**推荐方案**: HiveMQ CE 嵌入式，或 Moquette，便于自定义 Hook。

---

### 阶段三：MTP Adapter (设备事件处理)

**目标**: 实现设备上下线检测、信息采集、状态管理

| 任务 | 说明 | 对应Go代码 |
|------|------|-----------|
| 3.1 JetStream Stream/Consumer 创建 | 初始化 mqtt/ws/stomp/cwmp 等 stream | `nats.go` createStreams |
| 3.2 事件监听器 | 消费 JetStream 消息，按类型分发 | `events.go` StartEventsListener |
| 3.3 设备上线处理 | 发送 Get 请求获取设备信息 | `status.go` deviceOnline |
| 3.4 设备下线处理 | 更新 MongoDB 设备状态 | `status.go` deviceOffline |
| 3.5 设备信息解析 | 解析 Protobuf GetResp，提取设备属性 | `info.go` HandleDeviceInfo |
| 3.6 设备持久化 | MongoDB upsert 操作，MTP 状态合并 | `device.go` CreateDevice |
| 3.7 NATS Request 服务 | 响应 Controller 的设备查询请求 | `reqs.go` |
| 3.8 设备过滤/分页查询 | MongoDB 聚合查询 | `reqs.go` devices.retrieve |

**Java 实现示例 - 设备上线处理:**

```java
@Service
public class UspStatusHandler {

    @Autowired
    private NatsTemplate natsTemplate;

    public void handleDeviceOnline(String deviceSn, String mtp) {
        Usp.Get getMsg = Usp.Get.newBuilder()
            .addParamPaths("Device.DeviceInfo.Manufacturer")
            .addParamPaths("Device.DeviceInfo.ModelName")
            .addParamPaths("Device.DeviceInfo.SoftwareVersion")
            .addParamPaths("Device.DeviceInfo.SerialNumber")
            .addParamPaths("Device.DeviceInfo.ProductClass")
            .setMaxDepth(1)
            .build();

        Usp.Msg uspMsg = UspMessageBuilder.newGetMsg(getMsg);
        byte[] payload = uspMsg.toByteArray();
        UspRecord.Record record = UspRecordBuilder.newRecord(payload, deviceSn, controllerId);
        byte[] tr369Message = record.toByteArray();

        natsTemplate.publish(mtp + "-adapter.usp.v1." + deviceSn + ".info", tr369Message);
    }
}
```

---

### 阶段四：Controller (REST API)

**目标**: 实现完整的 REST API 层

| 任务 | 说明 | 对应Go代码 |
|------|------|-----------|
| 4.1 用户认证 API | 注册/登录/JWT 生成 | `user.go`, `auth.go` |
| 4.2 Spring Security 配置 | JWT 过滤器，路由权限 | `middleware.go` |
| 4.3 USP 消息发送服务 | 构造+发送+等待响应 | `usp.go` sendUspMsg |
| 4.4 Get 参数 API | GET 消息发送 | `usp.go` deviceGetMsg |
| 4.5 Set 参数 API | SET 消息发送 | `usp.go` deviceUpdateMsg |
| 4.6 Add/Delete API | ADD/DELETE 消息发送 | `usp.go` deviceCreateMsg/DeleteMsg |
| 4.7 Operate API | OPERATE 消息发送 | `usp.go` deviceOperateMsg |
| 4.8 GetSupportedDM API | 数据模型查询 | `usp.go` deviceGetSupportedParametersMsg |
| 4.9 GetInstances API | 实例查询 | `usp.go` deviceGetParameterInstances |
| 4.10 Notify API | 通知消息发送 | `usp.go` deviceNotifyMsg |
| 4.11 Generic API | 通用消息发送 | `usp.go` deviceGenericMessage |
| 4.12 固件升级 API | 双步骤固件升级 | `fwupdate.go` |
| 4.13 设备列表/查询 API | 分页+过滤查询 | `device.go` |
| 4.14 设备认证管理 API | KV Store CRUD | `device.go` deviceAuth |
| 4.15 WiFi 管理 API | WiFi 配置获取/设置 | `wifi.go` |
| 4.16 仪表盘信息 API | 厂商/状态/类型统计 | `info.go` |
| 4.17 消息模板 API | CRUD 操作 | `device.go` Template 相关 |
| 4.18 NATS 桥接服务 | Pub/Sub 交互封装 | `bridge.go` |

**API 端点清单:**

```
POST   /api/auth/register          # 用户注册
PUT    /api/auth/login              # 用户登录
POST   /api/auth/admin/register     # 管理员注册
GET    /api/auth/admin/exists       # 检查管理员
DELETE /api/auth/delete/{user}      # 删除用户
PUT    /api/auth/password/{user}    # 修改密码

GET    /api/device                  # 获取设备列表(分页/过滤)
DELETE /api/device?id=xxx           # 删除设备
PUT    /api/device/alias?id=xxx     # 设置设备别名
GET/POST/DELETE /api/device/auth    # 设备凭证管理

PUT    /api/device/{sn}/{mtp}/get          # USP Get
PUT    /api/device/{sn}/{mtp}/set          # USP Set
PUT    /api/device/{sn}/{mtp}/add          # USP Add
PUT    /api/device/{sn}/{mtp}/del          # USP Delete
PUT    /api/device/{sn}/{mtp}/operate      # USP Operate
PUT    /api/device/{sn}/{mtp}/notify       # USP Notify
PUT    /api/device/{sn}/{mtp}/parameters   # USP GetSupportedDM
PUT    /api/device/{sn}/{mtp}/instances    # USP GetInstances
PUT    /api/device/{sn}/{mtp}/generic      # 通用USP消息
PUT    /api/device/{sn}/{mtp}/fw_update    # 固件升级

PUT/GET /api/device/{sn}/wifi              # WiFi管理
POST   /api/device/message/{type}          # 添加模板
PUT    /api/device/message                 # 更新模板
GET    /api/device/message                 # 获取模板
DELETE /api/device/message                 # 删除模板

GET    /api/info/vendors                   # 厂商统计
GET    /api/info/status                    # 状态统计
GET    /api/info/device_class              # 设备类型统计
GET    /api/info/general                   # 综合统计
GET    /api/users                          # 用户列表
```

---

### 阶段五：WebSocket / STOMP Adapter

**目标**: 实现 WebSocket 和 STOMP 传输协议适配

| 任务 | 说明 | 对应Go代码 |
|------|------|-----------|
| 5.1 WebSocket Server | Spring WebSocket 服务 | `mtp/ws/` |
| 5.2 WS Adapter | WebSocket <-> NATS 桥接 | `mtp/ws-adapter/` |
| 5.3 STOMP Adapter | STOMP <-> NATS 桥接 | `mtp/stomp-adapter/` |
| 5.4 STOMP 帧协议实现 | STOMP frame 编解码 | `stomp/frame/` |

---

### 阶段六：CWMP/TR-069 兼容层

**目标**: 实现 ACS (Auto Configuration Server) 兼容

| 任务 | 说明 | 对应Go代码 |
|------|------|-----------|
| 6.1 ACS HTTP Server | CWMP SOAP over HTTP | `services/acs/` |
| 6.2 CWMP 消息解析 | SOAP XML 解析 | `acs/internal/cwmp/` |
| 6.3 NATS 桥接 | ACS <-> NATS 通信 | `acs/internal/nats/` |

---

### 阶段七：附加功能

| 任务 | 说明 | 对应Go代码 |
|------|------|-----------|
| 7.1 BulkData Collector | HTTP 批量数据采集 | `services/bulkdata/` |
| 7.2 SocketIO 实时通信 | 前端实时推送 | `socketio` service |
| 7.3 File Server | 固件文件服务 | `file-server` service |

---

### 阶段八：测试与部署

| 任务 | 说明 |
|------|------|
| 8.1 单元测试 | USP 消息构造、Protobuf 序列化、业务逻辑 |
| 8.2 集成测试 | NATS 通信、MongoDB 操作、MQTT 桥接 |
| 8.3 端到端测试 | 设备注册→信息采集→参数读写→固件升级 |
| 8.4 Docker 镜像构建 | 各服务 Dockerfile (Jib 或多阶段构建) |
| 8.5 Docker Compose 编排 | 多服务编排配置 |
| 8.6 性能测试 | 大量设备并发连接、消息吞吐量 |
| 8.7 文档编写 | API 文档 (Swagger)、部署文档、开发指南 |

---

## 3. 核心类设计

### 3.1 UspMessageBuilder

```java
public class UspMessageBuilder {

    public static Usp.Msg newGetMsg(Usp.Get get) {
        return Usp.Msg.newBuilder()
            .setHeader(Usp.Header.newBuilder()
                .setMsgId(UUID.randomUUID().toString())
                .setMsgType(Usp.Header.MsgType.GET))
            .setBody(Usp.Body.newBuilder()
                .setRequest(Usp.Request.newBuilder()
                    .setGet(get)))
            .build();
    }

    public static Usp.Msg newSetMsg(Usp.Set set) { /* ... */ }
    public static Usp.Msg newAddMsg(Usp.Add add) { /* ... */ }
    public static Usp.Msg newDeleteMsg(Usp.Delete delete) { /* ... */ }
    public static Usp.Msg newOperateMsg(Usp.Operate operate) { /* ... */ }
    public static Usp.Msg newNotifyMsg(Usp.Notify notify) { /* ... */ }
    // ...
}
```

### 3.2 UspRecordBuilder

```java
public class UspRecordBuilder {

    private static final String VERSION = "1.0";

    public static UspRecord.Record newRecord(byte[] payload, String toId, String fromId) {
        return UspRecord.Record.newBuilder()
            .setVersion(VERSION)
            .setToId(toId)
            .setFromId(fromId)
            .setPayloadSecurity(UspRecord.Record.PayloadSecurity.PLAINTEXT)
            .setNoSessionContext(
                UspRecord.NoSessionContextRecord.newBuilder()
                    .setPayload(ByteString.copyFrom(payload)))
            .build();
    }
}
```

### 3.3 NatsBridgeService

```java
@Service
public class NatsBridgeService {

    @Autowired
    private Connection natsConnection;

    private static final Duration TIMEOUT = Duration.ofSeconds(30);

    /**
     * 发送USP消息到设备并等待响应
     * 对应 Go 版 bridge.NatsUspInteraction()
     */
    public byte[] sendUspAndWaitResponse(String subscribeSub, String publishSub, byte[] data) {
        CompletableFuture<Message> future = new CompletableFuture<>();
        Dispatcher dispatcher = natsConnection.createDispatcher(msg -> {
            future.complete(msg);
        });
        dispatcher.subscribe(subscribeSub);

        natsConnection.publish(publishSub, data);

        try {
            Message response = future.get(TIMEOUT.toMillis(), TimeUnit.MILLISECONDS);
            return response.getData();
        } catch (TimeoutException e) {
            throw new GatewayTimeoutException("USP message response timeout");
        } finally {
            dispatcher.unsubscribe(subscribeSub);
        }
    }
}
```

### 3.4 Device 实体

```java
@Document(collection = "devices")
public class Device {
    @Id
    private String sn;               // 序列号
    private String model;            // 型号
    private String vendor;           // 厂商
    private String version;          // 软件版本
    private String productClass;     // 产品类型
    private String alias;            // 别名
    private DeviceStatus status;     // 全局状态
    private DeviceStatus mqtt;       // MQTT状态
    private DeviceStatus stomp;      // STOMP状态
    private DeviceStatus websockets; // WebSocket状态
    private DeviceStatus cwmp;       // CWMP状态
}

public enum DeviceStatus {
    OFFLINE(0), ASSOCIATING(1), ONLINE(2);
}

public enum MtpType {
    UNDEFINED, MQTT, STOMP, WEBSOCKETS, CWMP;
}
```

---

## 4. 关键技术要点

### 4.1 Protobuf 集成

使用 `protobuf-maven-plugin` 自动生成 Java 类:

```xml
<plugin>
    <groupId>org.xolstice.maven.plugins</groupId>
    <artifactId>protobuf-maven-plugin</artifactId>
    <version>0.6.1</version>
    <configuration>
        <protocArtifact>com.google.protobuf:protoc:4.28.0:exe:${os.detected.classifier}</protocArtifact>
        <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>
    </configuration>
    <executions>
        <execution>
            <goals><goal>compile</goal></goals>
        </execution>
    </executions>
</plugin>
```

### 4.2 NATS JetStream 配置

```java
@Configuration
public class NatsConfig {

    @Bean
    public Connection natsConnection(@Value("${nats.url}") String url) {
        Options options = new Options.Builder()
            .server(url)
            .maxReconnects(-1)
            .reconnectWait(Duration.ofSeconds(5))
            .build();
        return Nats.connect(options);
    }

    @Bean
    public JetStream jetStream(Connection nc) throws Exception {
        return nc.jetStream();
    }

    @Bean
    public JetStreamManagement jetStreamManagement(Connection nc) throws Exception {
        JetStreamManagement jsm = nc.jetStreamManagement();
        // 创建 streams: mqtt, ws, stomp, cwmp, lora, opc
        String[] streams = {"mqtt", "ws", "stomp", "cwmp", "lora", "opc"};
        for (String stream : streams) {
            StreamConfiguration sc = StreamConfiguration.builder()
                .name(stream)
                .subjects(stream + ".>")
                .retentionPolicy(RetentionPolicy.Interest)
                .build();
            jsm.addStream(sc);
        }
        return jsm;
    }
}
```

### 4.3 Go -> Java 核心差异处理

| Go 特性 | Java 对应方案 |
|---------|--------------|
| goroutine | `@Async` + `CompletableFuture` / `ExecutorService` |
| channel | `BlockingQueue` / `CompletableFuture` |
| select | `CompletableFuture.anyOf()` |
| context.Context | Spring `@Scope` + 请求上下文 |
| interface{} | 泛型 `<T>` |
| proto.Marshal/Unmarshal | `msg.toByteArray()` / `Msg.parseFrom(bytes)` |
| sync.Mutex | `synchronized` / `ReentrantLock` |
| 环境变量配置 | `application.yml` + `@Value` |

### 4.4 Docker 构建

```dockerfile
# 多阶段构建示例
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY . .
RUN ./mvnw package -pl oktopus-controller -am -DskipTests

FROM eclipse-temurin:21-jre-alpine
COPY --from=build /app/oktopus-controller/target/*.jar app.jar
EXPOSE 8000
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 5. 风险与注意事项

| 风险项 | 说明 | 缓解措施 |
|--------|------|---------|
| MQTT Broker 嵌入式 Hook | Go 版使用 mochi-co 内置 Hook，Java 需选合适的嵌入式 Broker | 评估 HiveMQ CE / Moquette 扩展能力 |
| Protobuf 版本兼容 | 确保生成的 Java 类与 proto 文件版本一致 | 在 common 模块统一管理 proto 文件 |
| NATS JetStream 语义 | Go 和 Java NATS 客户端 API 有差异 | 参考 jnats 官方文档，封装统一的 NatsTemplate |
| STOMP 帧协议 | Go 版自研了 STOMP 帧解析 | 使用 Spring STOMP 或移植帧解析代码 |
| 单分区固件升级 | Go 版未实现，Java 版可补充 | 阶段四中增加单分区升级逻辑 |
| 并发模型差异 | Go goroutine 轻量级 vs Java 线程 | 使用 Virtual Threads (Java 21+) 或响应式编程 |
| 性能对比 | Go 在网络IO场景通常更高效 | 使用 Java 21 Virtual Threads + 异步IO 优化 |

---

## 6. 推荐 Java 版本与框架版本

| 组件 | 推荐版本 |
|------|---------|
| Java | 21 LTS (Virtual Threads 支持) |
| Spring Boot | 3.3.x |
| Spring Security | 6.x |
| jnats | 2.20.x |
| protobuf-java | 4.28.x |
| Eclipse Paho MQTT | 1.2.5 (v3) 或 paho.mqttv5 |
| Spring Data MongoDB | 4.x (随 Spring Boot) |
| Maven | 3.9.x |
