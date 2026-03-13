# Oktopus TR-369/USP 协议流程分析文档

## 1. 项目概述

Oktopus 是一个开源的 USP Controller (TR-369) 及 CWMP (TR-069) 兼容的多厂商 CPE/IoT 设备管理平台。项目采用 Go 语言实现，基于微服务架构，通过 NATS 消息总线进行服务间通信。

### 1.1 技术栈

| 组件 | 技术 |
|------|------|
| 后端语言 | Go |
| 消息总线 | NATS (JetStream) |
| 数据库 | MongoDB |
| 消息编码 | Protocol Buffers (USP Msg 1.3 / USP Record 1.3) |
| MTP 协议 | MQTT, STOMP, WebSocket |
| 前端 | Next.js |
| 部署 | Docker Compose |

### 1.2 微服务架构

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Frontend    │────>│  Controller  │────>│  NATS JetStream  │
│  (Next.js)  │     │  (REST API)  │     │  (消息总线)       │
└─────────────┘     └──────────────┘     └────────┬────────┘
                                                   │
                    ┌──────────────────────────────┤
                    │              │                │
              ┌─────▼─────┐ ┌─────▼─────┐  ┌──────▼──────┐
              │ MTP        │ │ MQTT      │  │ Adapter     │
              │ Adapter    │ │ Adapter   │  │ (设备管理)   │
              │ (事件处理) │ │ (MQTT桥接)│  │             │
              └─────┬─────┘ └─────┬─────┘  └──────┬──────┘
                    │             │                │
              ┌─────▼─────┐ ┌─────▼─────┐  ┌──────▼──────┐
              │ MongoDB   │ │ MQTT      │  │ STOMP/WS    │
              │ (设备存储) │ │ Broker    │  │ Adapter     │
              └───────────┘ └─────┬─────┘  └──────┬──────┘
                                  │                │
                           ┌──────▼────────────────▼──────┐
                           │         CPE/IoT 设备          │
                           └───────────────────────────────┘
```

### 1.3 核心服务说明

| 服务 | 路径 | 职责 |
|------|------|------|
| **Controller** | `backend/services/controller/` | REST API 网关，USP 消息构造与发送，用户认证 |
| **MTP Adapter** | `backend/services/mtp/adapter/` | 设备事件处理，设备信息采集，状态管理，MongoDB 操作 |
| **MQTT Broker** | `backend/services/mtp/mqtt/` | 自定义 MQTT Broker，设备上下线检测 Hook |
| **MQTT Adapter** | `backend/services/mtp/mqtt-adapter/` | MQTT <-> NATS 消息桥接 |
| **STOMP Adapter** | `backend/services/mtp/stomp-adapter/` | STOMP <-> NATS 消息桥接 |
| **WS Adapter** | `backend/services/mtp/ws-adapter/` | WebSocket <-> NATS 消息桥接 |
| **ACS** | `backend/services/acs/` | CWMP/TR-069 自动配置服务器 |

### 1.4 NATS Subject 命名规则

```
mqtt.usp.v1.{deviceSN}.status        # MQTT设备状态消息（JetStream）
mqtt.usp.v1.{deviceSN}.info          # MQTT设备信息响应（JetStream）
mqtt-adapter.usp.v1.{deviceSN}.api   # Controller -> MQTT Adapter -> 设备 API消息
mqtt-adapter.usp.v1.{deviceSN}.info  # MTP Adapter -> MQTT Adapter -> 设备 Info采集
device.usp.v1.{deviceSN}.api         # 设备 -> MQTT Adapter -> Controller API响应
device.v1.new                        # 新设备注册通知
adapter.usp.v1.*                     # Adapter 服务请求-应答
```

### 1.5 MQTT Topic 命名规则

```
oktopus/usp/v1/agent/{deviceId}       # Controller -> 设备（设备订阅）
oktopus/usp/v1/controller/{deviceId}  # 设备 -> Controller（设备信息响应）
oktopus/usp/v1/api/{deviceId}         # 设备 -> Controller（API消息响应）
oktopus/usp/v1/status/{deviceId}      # 设备上下线状态
```

---

## 2. USP 协议消息结构

### 2.1 USP Record (传输层封装)

```protobuf
message Record {
  string version = 1;           // 协议版本 "1.0"
  string to_id = 2;             // 目标端点ID
  string from_id = 3;           // 源端点ID（如 "oktopusController"）
  PayloadSecurity payload_security = 4;  // PLAINTEXT 或 TLS12
  oneof record_type {
    NoSessionContextRecord no_session_context = 7;  // 无会话上下文
    SessionContextRecord session_context = 8;       // 有会话上下文
    MQTTConnectRecord mqtt_connect = 10;             // MQTT连接记录
    STOMPConnectRecord stomp_connect = 11;           // STOMP连接记录
    WebSocketConnectRecord websocket_connect = 9;    // WS连接记录
  }
}
```

### 2.2 USP Msg (业务层消息)

```protobuf
message Msg {
  Header header = 1;  // 消息头（msg_id + msg_type）
  Body body = 2;      // 消息体（Request / Response / Error）
}

message Header {
  string msg_id = 1;        // UUID消息ID
  MsgType msg_type = 2;     // 消息类型枚举
  enum MsgType {
    GET = 1; GET_RESP = 2; NOTIFY = 3; SET = 4; SET_RESP = 5;
    OPERATE = 6; OPERATE_RESP = 7; ADD = 8; ADD_RESP = 9;
    DELETE = 10; DELETE_RESP = 11; GET_SUPPORTED_DM = 12;
    GET_INSTANCES = 14; REGISTER = 19; DEREGISTER = 21;
    // ...
  }
}
```

### 2.3 消息类型总览

| 消息类型 | 方向 | 说明 |
|---------|------|------|
| GET / GET_RESP | Controller -> Agent -> Controller | 获取参数值 |
| SET / SET_RESP | Controller -> Agent -> Controller | 设置参数值 |
| ADD / ADD_RESP | Controller -> Agent -> Controller | 创建对象实例 |
| DELETE / DELETE_RESP | Controller -> Agent -> Controller | 删除对象实例 |
| OPERATE / OPERATE_RESP | Controller -> Agent -> Controller | 执行操作命令 |
| NOTIFY / NOTIFY_RESP | Agent -> Controller / Controller -> Agent | 事件通知 |
| GET_SUPPORTED_DM | Controller -> Agent -> Controller | 获取支持的数据模型 |
| GET_INSTANCES | Controller -> Agent -> Controller | 获取对象实例 |
| REGISTER / REGISTER_RESP | Agent -> Controller | 注册数据模型路径 |
| DEREGISTER | Agent -> Controller | 注销数据模型路径 |

---

## 3. 设备注册/上线流程

设备注册是整个系统的核心流程，涉及 MQTT Broker、MQTT Adapter、MTP Adapter 三个微服务协同工作。

### 3.1 流程时序图

```
┌────────┐    ┌──────────┐    ┌──────────────┐    ┌───────┐    ┌────────────┐    ┌─────────┐
│ Device │    │MQTT Broker│    │MQTT Adapter  │    │ NATS  │    │MTP Adapter │    │MongoDB  │
└───┬────┘    └────┬─────┘    └──────┬───────┘    └───┬───┘    └─────┬──────┘    └────┬────┘
    │              │                 │                 │              │                │
    │ 1.MQTT Connect (username/pwd) │                 │              │                │
    │─────────────>│                 │                 │              │                │
    │              │ 认证检查(NatsAuthHook)             │              │                │
    │              │──── KV Get ────>│                 │              │                │
    │  CONNACK     │                 │                 │              │                │
    │<─────────────│ (subscribe-topic属性)              │              │                │
    │              │                 │                 │              │                │
    │ 2.Subscribe: oktopus/usp/v1/agent/{deviceId}    │              │                │
    │─────────────>│                 │                 │              │                │
    │              │ OnSubscribed Hook触发             │              │                │
    │              │ Publish status=1                  │              │                │
    │              │ 到 oktopus/usp/v1/status/{id}     │              │                │
    │              │────────────────>│                 │              │                │
    │              │                 │ 3.转发到NATS     │              │                │
    │              │                 │ mqtt.usp.v1.    │              │                │
    │              │                 │ {id}.status     │              │                │
    │              │                 │────────────────>│              │                │
    │              │                 │                 │ 4.JetStream  │                │
    │              │                 │                 │ 消费status   │                │
    │              │                 │                 │─────────────>│                │
    │              │                 │                 │              │                │
    │              │                 │                 │  5.HandleDeviceStatus          │
    │              │                 │                 │  → deviceOnline()              │
    │              │                 │                 │  构造Get请求:                   │
    │              │                 │                 │  Manufacturer,ModelName,       │
    │              │                 │                 │  SoftwareVersion,SerialNumber, │
    │              │                 │                 │  ProductClass                  │
    │              │                 │                 │<─────────────│                │
    │              │                 │                 │              │                │
    │              │                 │ 6.NATS Publish  │              │                │
    │              │                 │ mqtt-adapter... │              │                │
    │              │                 │ {id}.info       │              │                │
    │              │                 │<────────────────│              │                │
    │              │ 7.Publish MQTT  │                 │              │                │
    │              │ agent/{id}      │                 │              │                │
    │              │<────────────────│                 │              │                │
    │ 8.Get请求    │                 │                 │              │                │
    │<─────────────│                 │                 │              │                │
    │              │                 │                 │              │                │
    │ 9.Get响应    │                 │                 │              │                │
    │─────────────>│                 │                 │              │                │
    │              │────────────────>│                 │              │                │
    │              │                 │────────────────>│              │                │
    │              │                 │                 │─────────────>│                │
    │              │                 │                 │              │ 10.HandleDeviceInfo
    │              │                 │                 │              │ 解析设备信息     │
    │              │                 │                 │              │ 写入MongoDB     │
    │              │                 │                 │              │────────────────>│
    │              │                 │                 │              │                │
    │              │                 │                 │              │ 11.如果新设备    │
    │              │                 │                 │              │ Publish         │
    │              │                 │                 │              │ device.v1.new   │
```

### 3.2 关键代码路径

**Step 1-2: MQTT 连接认证与订阅检测**

- 文件: `backend/services/mtp/mqtt/internal/listeners/mqtt/hook.go`
- `NatsAuthHook.OnConnectAuthenticate()` - 通过 NATS KV Store 验证设备用户名/密码
- `MyHook.OnSubscribed()` - 检测设备订阅 `oktopus/usp/v1/agent` 前缀的 topic，发送 status=1
- `MyHook.OnDisconnect()` - 设备断开时发送 status=0

**Step 3: MQTT -> NATS 桥接**

- 文件: `backend/services/mtp/mqtt-adapter/internal/bridge/bridge.go`
- `mqttMessageHandler()` 将 MQTT status topic 消息转发到 NATS `mqtt.usp.v1.{deviceId}.status`

**Step 4-5: 事件消费与设备上线处理**

- 文件: `backend/services/mtp/adapter/internal/events/events.go`
- `StartEventsListener()` 从 JetStream 消费 mqtt/ws/stomp 等 stream
- 文件: `backend/services/mtp/adapter/internal/events/usp_handler/status.go`
- `HandleDeviceStatus()` -> `deviceOnline()` 构造 USP Get 请求获取设备基础信息

**Step 10-11: 设备信息处理与持久化**

- 文件: `backend/services/mtp/adapter/internal/events/usp_handler/info.go`
- `HandleDeviceInfo()` -> `parseDeviceInfoMsg()` 解析 Protobuf 响应，提取设备信息
- 文件: `backend/services/mtp/adapter/internal/db/device.go`
- `CreateDevice()` 使用 MongoDB 事务进行 upsert 操作

### 3.3 设备下线流程

```
Device断开MQTT -> OnDisconnect Hook -> Publish status=0 ->
MQTT Adapter -> NATS mqtt.usp.v1.{id}.status ->
MTP Adapter HandleDeviceStatus -> deviceOffline() ->
MongoDB UpdateStatus(Offline)
```

设备下线时会更新各 MTP 层的独立状态（mqtt/stomp/websockets），并计算全局 status：
- 如果所有 MTP 层都 Offline，则全局 status = Offline
- 如果任一 MTP 层 Online，则全局 status = Online

---

## 4. 消息上报（Notify）流程

### 4.1 设备主动上报 (Agent -> Controller)

Notify 消息支持以下通知类型：

| 通知类型 | 说明 |
|---------|------|
| `Event` | 事件通知（含 obj_path, event_name, params） |
| `ValueChange` | 参数值变化通知 |
| `ObjectCreation` | 对象创建通知 |
| `ObjectDeletion` | 对象删除通知 |
| `OperationComplete` | 异步操作完成通知 |
| `OnBoardRequest` | 设备入网请求（含 OUI, ProductClass, SerialNumber） |

```
┌────────┐        ┌──────────┐      ┌──────────────┐      ┌───────┐
│ Device │        │MQTT Broker│      │MQTT Adapter  │      │ NATS  │
└───┬────┘        └────┬─────┘      └──────┬───────┘      └───┬───┘
    │                  │                   │                   │
    │ Publish到         │                   │                   │
    │ controller/topic │                   │                   │
    │ (USP Record +    │                   │                   │
    │  Notify Msg)     │                   │                   │
    │─────────────────>│                   │                   │
    │                  │ controller chan   │                   │
    │                  │──────────────────>│                   │
    │                  │                   │ Publish到NATS      │
    │                  │                   │ mqtt.usp.v1.      │
    │                  │                   │ {id}.info         │
    │                  │                   │──────────────────>│
    │                  │                   │                   │
```

### 4.2 Controller 推送 Notify (Controller -> Agent)

- REST API: `PUT /api/device/{sn}/{mtp}/notify`
- 文件: `backend/services/controller/internal/api/usp.go` - `deviceNotifyMsg()`

```go
// Controller构造Notify消息示例
notify := usp_msg.Notify{
    SubscriptionId: uuid.NewString(),
    SendResp:       true,
    Notification: &usp_msg.Notify_Event_{
        Event: &usp_msg.Notify_Event{
            EventName: "Push!",
            ObjPath:   "Device.BulkData.Profile.1.",
        },
    },
}
```

流程:
```
REST API请求 -> Controller构造Notify USP Msg -> Protobuf序列化 ->
封装USP Record -> NATS Publish -> MTP Adapter -> MQTT/STOMP/WS -> Device
```

---

## 5. 参数获取（Get）流程

### 5.1 时序图

```
┌────────┐   ┌────────────┐   ┌───────┐   ┌──────────────┐   ┌──────────┐   ┌────────┐
│ 前端/   │   │ Controller │   │ NATS  │   │MQTT Adapter  │   │MQTT Broker│   │ Device │
│ 客户端  │   │ REST API   │   │       │   │              │   │          │   │        │
└───┬────┘   └─────┬──────┘   └───┬───┘   └──────┬───────┘   └────┬─────┘   └───┬────┘
    │              │              │               │                │              │
    │ PUT /api/    │              │               │                │              │
    │ device/{sn}/ │              │               │                │              │
    │ {mtp}/get    │              │               │                │              │
    │ Body: {      │              │               │                │              │
    │  param_paths │              │               │                │              │
    │ }            │              │               │                │              │
    │─────────────>│              │               │                │              │
    │              │ 1.构造Get Msg│               │                │              │
    │              │ (UUID+GET)  │               │                │              │
    │              │              │               │                │              │
    │              │ 2.Proto序列化│               │                │              │
    │              │ Msg -> Record│              │                │              │
    │              │              │               │                │              │
    │              │ 3.ChanSubscribe              │                │              │
    │              │ device.usp.v1│               │                │              │
    │              │ .{sn}.api   │               │                │              │
    │              │─────────────>│               │                │              │
    │              │              │               │                │              │
    │              │ 4.Publish    │               │                │              │
    │              │ mqtt-adapter │               │                │              │
    │              │ .usp.v1.    │               │                │              │
    │              │ {sn}.api    │               │                │              │
    │              │─────────────>│               │                │              │
    │              │              │ 5.转发        │                │              │
    │              │              │──────────────>│                │              │
    │              │              │               │ 6.Publish MQTT │              │
    │              │              │               │ agent/{sn}     │              │
    │              │              │               │───────────────>│              │
    │              │              │               │                │ 7.投递到设备  │
    │              │              │               │                │─────────────>│
    │              │              │               │                │              │
    │              │              │               │                │ 8.设备处理   │
    │              │              │               │                │ 返回GetResp  │
    │              │              │               │                │<─────────────│
    │              │              │               │ 9.api/{sn}    │              │
    │              │              │               │<───────────────│              │
    │              │              │ 10.转发       │                │              │
    │              │              │<──────────────│                │              │
    │              │ 11.Chan接收  │               │                │              │
    │              │ USP Record   │               │                │              │
    │              │<─────────────│               │                │              │
    │              │              │               │                │              │
    │              │ 12.反序列化   │               │                │              │
    │              │ Record->Msg  │               │                │              │
    │              │ 提取GetResp  │               │                │              │
    │ 13.返回JSON  │              │               │                │              │
    │<─────────────│              │               │                │              │
```

### 5.2 关键代码

**Controller 端 - 构造并发送 Get 消息:**

```go
// backend/services/controller/internal/api/usp.go
func (a *Api) deviceGetMsg(w http.ResponseWriter, r *http.Request) {
    sn := getSerialNumberFromRequest(r)
    mtp, _ := getMtpFromRequest(r, w)

    var get usp_msg.Get
    utils.MarshallDecoder(&get, r.Body)     // 从请求体解码
    msg := usp_utils.NewGetMsg(get)          // 构造USP Get消息
    sendUspMsg(msg, sn, w, a.nc, mtp)       // 发送并等待响应
}
```

**消息发送核心逻辑:**

```go
// backend/services/controller/internal/api/usp.go
func sendUspMsg(msg usp_msg.Msg, sn string, ...) error {
    protoMsg, _ := proto.Marshal(&msg)                    // 1. 序列化Msg
    record := usp_utils.NewUspRecord(protoMsg, sn)        // 2. 封装Record
    protoRecord, _ := proto.Marshal(&record)               // 3. 序列化Record
    data, _ := bridge.NatsUspInteraction(                  // 4. 通过NATS发送并等待
        "device.usp.v1."+sn+".api",                       //    订阅响应subject
        mtp+"-adapter.usp.v1."+sn+".api",                 //    发布请求subject
        protoRecord, w, nc,
    )
    // 5. 反序列化响应 Record -> Msg -> Response -> GetResp
    // 6. JSON编码返回前端
}
```

### 5.3 NATS 交互模式 (Pub/Sub with Channel)

```go
// backend/services/controller/internal/bridge/bridge.go
func NatsUspInteraction(subSubj, pubSubj string, body []byte, ...) {
    ch := make(chan *nats.Msg, 64)
    nc.ChanSubscribe(subSubj, ch)   // 先订阅响应subject
    nc.Publish(pubSubj, body)        // 再发布请求
    select {
    case msg := <-ch:                // 等待响应
        return msg.Data, nil
    case <-time.After(30s):          // 超时处理
        return nil, errTimeout
    }
}
```

---

## 6. 参数设置（Set）流程

### 6.1 流程概述

- REST API: `PUT /api/device/{sn}/{mtp}/set`
- 文件: `backend/services/controller/internal/api/usp.go` - `deviceUpdateMsg()`

```
前端请求(Set参数列表) -> Controller构造Set USP Msg ->
封装USP Record -> NATS -> MTP Adapter -> MQTT -> Device ->
Device处理Set请求 -> 返回SetResp -> 原路返回
```

### 6.2 Set 消息结构

```protobuf
message Set {
  bool allow_partial = 1;              // 是否允许部分成功
  repeated UpdateObject update_objs = 2;

  message UpdateObject {
    string obj_path = 1;               // 对象路径 如 "Device.WiFi.Radio.1."
    repeated UpdateParamSetting param_settings = 2;
  }
  message UpdateParamSetting {
    string param = 1;                  // 参数名
    string value = 2;                  // 参数值
    bool required = 3;                 // 是否必须成功
  }
}
```

### 6.3 其他 CRUD 操作

| 操作 | API | USP消息类型 | 说明 |
|------|-----|------------|------|
| 获取参数 | `PUT /{sn}/{mtp}/get` | GET | 获取指定路径的参数值 |
| 设置参数 | `PUT /{sn}/{mtp}/set` | SET | 设置指定路径的参数值 |
| 创建对象 | `PUT /{sn}/{mtp}/add` | ADD | 创建新的对象实例 |
| 删除对象 | `PUT /{sn}/{mtp}/del` | DELETE | 删除对象实例 |
| 执行操作 | `PUT /{sn}/{mtp}/operate` | OPERATE | 执行设备命令 |
| 获取数据模型 | `PUT /{sn}/{mtp}/parameters` | GET_SUPPORTED_DM | 获取设备支持的数据模型 |
| 获取实例 | `PUT /{sn}/{mtp}/instances` | GET_INSTANCES | 获取对象实例列表 |
| 通用消息 | `PUT /{sn}/{mtp}/generic` | 任意 | 发送自定义USP消息 |

---

## 7. 固件升级（Firmware Update）流程

### 7.1 流程时序图

```
┌────────┐   ┌────────────┐   ┌───────┐   ┌──────────┐   ┌────────┐   ┌───────────┐
│ 前端   │   │ Controller │   │ NATS  │   │MTP Layer │   │ Device │   │File Server│
└───┬────┘   └─────┬──────┘   └───┬───┘   └────┬─────┘   └───┬────┘   └─────┬─────┘
    │              │              │              │              │              │
    │ PUT /api/    │              │              │              │              │
    │ device/{sn}/ │              │              │              │              │
    │ {mtp}/       │              │              │              │              │
    │ fw_update    │              │              │              │              │
    │ {url: "..."}│              │              │              │              │
    │─────────────>│              │              │              │              │
    │              │              │              │              │              │
    │              │ Step1: 查询FirmwareImage状态                │              │
    │              │ Get: Device.DeviceInfo.FirmwareImage.*.Status               │
    │              │─────────────>│─────────────>│─────────────>│              │
    │              │              │              │              │              │
    │              │ Step2: 设备返回FirmwareImage分区状态         │              │
    │              │<────────────────────────────────────────────│              │
    │              │              │              │              │              │
    │              │ Step3: 检查可用分区                         │              │
    │              │ checkAvaiableFwPartition()                 │              │
    │              │ (找Status=="Available"的分区)               │              │
    │              │              │              │              │              │
    │              │ Step4: 发送Operate命令                     │              │
    │              │ Command: Device.DeviceInfo.                │              │
    │              │ FirmwareImage.{partition}.Download()       │              │
    │              │ InputArgs:                                 │              │
    │              │   URL: {firmware_url}                      │              │
    │              │   AutoActivate: "true"                     │              │
    │              │   FileSize: "0"                            │              │
    │              │─────────────>│─────────────>│─────────────>│              │
    │              │              │              │              │              │
    │              │              │              │              │ Step5: 下载固件│
    │              │              │              │              │─────────────>│
    │              │              │              │              │<─────────────│
    │              │              │              │              │              │
    │              │              │              │              │ Step6: 安装激活│
    │              │              │              │              │ (AutoActivate)│
    │              │              │              │              │              │
    │              │ Step7: 返回OperateResp                    │              │
    │              │<────────────────────────────────────────────│              │
    │ 返回结果     │              │              │              │              │
    │<─────────────│              │              │              │              │
```

### 7.2 关键代码

```go
// backend/services/controller/internal/api/fwupdate.go
func (a *Api) deviceFwUpdate(w http.ResponseWriter, r *http.Request) {
    // 1. 获取固件下载URL
    var payload fwUpdate
    utils.MarshallDecoder(&payload, r.Body)

    // 2. 查询设备固件分区状态
    msg := usp_utils.NewGetMsg(usp_msg.Get{
        ParamPaths: []string{"Device.DeviceInfo.FirmwareImage.*.Status"},
        MaxDepth:   1,
    })

    // 3. 发送Get请求并等待响应
    data, _ := bridge.NatsUspInteraction(...)

    // 4. 检查可用分区
    partition := checkAvaiableFwPartition(getMsgAnswer.ReqPathResults)

    // 5. 构造Operate命令执行下载
    receiver := usp_msg.Operate{
        Command:    "Device.DeviceInfo.FirmwareImage." + partition + "Download()",
        CommandKey: "Download()",
        SendResp:   true,
        InputArgs: map[string]string{
            "URL":          payload.Url,
            "AutoActivate": "true",
            "FileSize":     "0",
        },
    }

    // 6. 发送Operate消息
    msg = usp_utils.NewOperateMsg(receiver)
    sendUspMsg(msg, sn, w, a.nc, mtp)
}
```

### 7.3 分区检查逻辑

```go
func checkAvaiableFwPartition(reqPathResult) string {
    // 遍历FirmwareImage实例
    // 查找Status为"Available"或"ValidationFailed"的分区
    // 双分区设备：当前激活分区和备用分区
    // 单分区设备：返回空（目前不支持单分区升级）
}
```

### 7.4 升级流程要点

1. **双分区机制**: 设备通常有2个固件分区，一个活跃一个备用
2. **自动激活**: `AutoActivate=true` 表示下载完成后自动切换到新固件
3. **异步操作**: Download() 是异步命令，设备完成后会通过 `OperationComplete` Notify 上报
4. **局限**: 当前代码不支持单分区设备升级、校验和验证等

---

## 8. 设备认证流程

### 8.1 认证架构

设备认证使用 NATS JetStream KeyValue Store:

```
┌────────────┐        ┌─────────────────┐        ┌──────────┐
│ Controller │───────>│ NATS KV Store   │<───────│MQTT Broker│
│ (管理凭证) │        │ Bucket:         │        │(认证检查) │
│            │        │ devices-auth    │        │          │
└────────────┘        │ Key: deviceId   │        └──────────┘
                      │ Val: password   │
                      └─────────────────┘
```

### 8.2 REST API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/device/auth?id={deviceId}` | 查询设备凭证 |
| GET | `/api/device/auth` | 列出所有设备凭证 |
| POST | `/api/device/auth` | 创建设备凭证 `{id, password}` |
| DELETE | `/api/device/auth?id={deviceId}` | 删除设备凭证 |

### 8.3 MQTT Broker 认证 Hook

```go
// hook.go - NatsAuthHook
func OnConnectAuthenticate(cl *mqtt.Client, pk packets.Packet) bool {
    username := string(pk.Connect.Username)
    entry, err := h.kv.Get(context.TODO(), username)  // 从NATS KV获取密码
    return bytes.Equal(entry.Value(), pk.Connect.Password)  // 比较
}

func OnACLCheck(cl *mqtt.Client, topic string, write bool) bool {
    // oktopusController 拥有所有权限
    // 设备只能订阅 oktopus/usp/v1/agent/{自己的id}
    // 设备只能发布到 oktopus/usp/v1/controller 和 api/{自己的id}
}
```

---

## 9. 完整消息通信路径总结

### 9.1 Controller -> Device (下行)

```
REST API Request
    ↓
Controller: 构造USP Msg (Protobuf)
    ↓
Controller: 封装USP Record (Protobuf)
    ↓
Controller: NATS Publish → {mtp}-adapter.usp.v1.{sn}.api
    ↓
MTP Adapter (MQTT/STOMP/WS): NATS Subscribe
    ↓
MTP Adapter: 转发到对应MTP协议
    ↓ (MQTT示例)
MQTT Adapter: Publish → oktopus/usp/v1/agent/{deviceId}
    ↓
MQTT Broker → Device
```

### 9.2 Device -> Controller (上行响应)

```
Device: 构造USP Response Msg + Record
    ↓
MQTT Publish → oktopus/usp/v1/api/{deviceId}
    ↓
MQTT Broker → MQTT Adapter
    ↓
MQTT Adapter: NATS Publish → device.usp.v1.{sn}.api
    ↓
Controller: NATS ChanSubscribe 接收
    ↓
Controller: 反序列化 Record → Msg → Response
    ↓
REST API Response (JSON)
```

### 9.3 Device -> Controller (上行主动上报)

```
Device: 构造USP Notify Msg + Record
    ↓
MQTT Publish → oktopus/usp/v1/controller/{deviceId}
    ↓
MQTT Broker → MQTT Adapter
    ↓
MQTT Adapter: NATS JetStream Publish → mqtt.usp.v1.{sn}.info
    ↓
MTP Adapter: JetStream Consumer 消费
    ↓
HandleDeviceInfo() → 处理/存储
```

---

## 10. 数据库模型

### 10.1 Device 模型

```go
type Device struct {
    SN           string    // 序列号（主键）
    Model        string    // 设备型号
    Customer     string    // 客户
    Vendor       string    // 厂商
    Version      string    // 软件版本
    ProductClass string    // 产品类型
    Alias        string    // 别名
    Status       Status    // 全局状态 (Offline=0, Associating=1, Online=2)
    Mqtt         Status    // MQTT连接状态
    Stomp        Status    // STOMP连接状态
    Websockets   Status    // WebSocket连接状态
    Cwmp         Status    // CWMP连接状态
}
```

### 10.2 MTP 类型枚举

```
UNDEFINED = 0, MQTT = 1, STOMP = 2, WEBSOCKETS = 3, CWMP = 4
```
