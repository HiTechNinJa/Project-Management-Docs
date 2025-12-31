# WatchYourBehind 系统架构设计文档 (V1.0.3)

## 1. 系统概述 (Overview)

本系统构建了一个从 毫米波感知 到 云端可视化 的完整物联网闭环。

系统采用 "状态驱动 (State-Driven)"
架构，核心目标不仅仅是数据的单向采集，更实现了端到端的 指令闭环控制 和
按需加速
机制。系统能够根据前端用户的活跃状态，动态调整边缘设备的采样频率，并支持基于区域的"守卫模式"预警。

## 2. 系统分层架构

系统分为四层架构：

1.  **感知层 (Sensing Layer)**: HLK-LD2450 毫米波雷达模块
    (负责原始信号处理)。

2.  **边缘层 (Edge Layer)**: ESP32 智能网关
    (负责协议转换、指令解析、边缘过滤)。

3.  **服务层 (Service Layer)**: Flask API Backend + MySQL
    (负责业务逻辑、状态管理、告警触发)。

4.  **应用层 (App Layer)**: Web 前端实时仪表盘
    (负责可视化渲染、指令下发)。

## 3. 边缘层：ESP32 智能网关逻辑

ESP32
在本架构中具备**指令缓存**和**状态执行**能力的智能节点。

### 3.1 核心固件逻辑 (Firmware Logic)

- **指令闭环控制 (Control Loop)**:

  - **机制**: ESP32 位于内网，无法被动接收公网请求。因此采用
    **"捎带响应 (Piggyback)"** 机制。

  - **流程**: ESP32 发起 POST /api/v1/device/sync 上报数据 -> 服务器在 HTTP Response
    中返回 pending_cmd -> ESP32
    解析并执行指令（如切换模式、重启、写入配置）。

  - **0x00C2 协议支持**: [未来版本 - 守卫模式相关] 针对"守卫模式"，固件支持 0x00C2
    (设置区域过滤) 指令的解析逻辑。当收到 API 下发的 JSON
    坐标数组时，ESP32 将其转换为 26 字节的 Hex
    补码流写入雷达，在硬件层过滤掉非关注区的干扰源。

- **动态频率控制 (Active Sync)**:

  - **待机模式 (Standby)**: 默认 **1Hz** (1000ms/次)。当 API 返回
    next_interval: 1000 时，系统处于低功耗、低流量状态。

  - **加速模式 (Turbo)**: 当 API 返回 next_interval: 100 时，ESP32
    立即切换至 **10Hz** (100ms/次) 上报频率，以支持前端的丝滑实时渲染。

### 3.2 ESP32 开机流程 (Boot Sequence)

ESP32 采用智能启动策略，针对热重启场景进行优化：

#### 阶段1: 雷达通信建立
```
graph TD
    A[ESP32启动] --> B[尝试256000波特率]
    B --> C{2秒内检测到雷达帧头?}
    C -->|是| D[发送重启命令0x00A3]
    C -->|否| E[执行自动波特率扫描]
    D --> F[等待2秒雷达重启]
    F --> G{重启后仍响应?}
    G -->|是| H[锁定256000波特率]
    G -->|否| E
    E --> I[波特率扫描完成]
```

- **热重启优化**: 优先尝试256000波特率（ESP32热重启最常见情况）
- **智能重启**: 仅在确认雷达响应时才发送重启命令，避免不必要的雷达重启
- **容错机制**: 如果256000失败，自动回退到完整的波特率扫描

#### 阶段2: 网络连接与系统报告
```
graph TD
    A[雷达通信建立] --> B[连接WiFi网络]
    B --> C{WiFi连接成功?}
    C -->|是| D[获取设备MAC地址]
    C -->|否| E[继续串口模式]
    D --> F[生成系统状态报告]
    F --> G[查询雷达信息]
    G --> H[保存初始配置]
    H --> I[进入主循环]
```

- **系统状态报告**: 自动显示雷达波特率、WiFi状态、MAC地址、雷达版本/MAC/模式/区域信息
- **初始配置保存**: 建立变更检测基准，用于后续的15秒自动巡检
- **用户友好**: 显示"System ready. Type '?' for help."提示

#### 阶段3: 运行时监控
- **自动巡检**: 每15秒后台查询雷达配置状态，检测模式变化
- **WiFi保活**: 每5秒检测网络连接，自动重连
- **动态频率**: 根据服务器指令调整数据上报频率（100ms/1000ms）

## 4. 服务层与通信协议

### 4.1 数据流向与控制流

graph TD\
%% 物理层\
Radar\[HLK-LD2450\] \-- UART(256000) \--\> ESP32\
\
%% 边缘层\
subgraph Edge_Layer\
ESP32 \-- 1. POST Data \--\> API\
ESP32 \-- 3. Execute Cmd \--\> Radar\
end\
\
%% 云端\
subgraph Cloud_Server\
API\[Flask Backend\] \-- Write \--\> DB\[(MySQL)\]\
API \-- Read Cmd \--\> CmdQueue\[指令缓存区\]\
API \-- 2. Response (Next_Interval + Cmd) \--\> ESP32\
API \-- WS Stream \--\> WebSocket\
end\
\
%% 前端\
User\[Web Dashboard\] \-- WebSocket (10Hz Stream) \--\> API\
User \-- REST API \--\> API\
API \-- Enqueue \--\> CmdQueue

### 4.2 边缘-云端通信接口 (HTTP Interface)

- **上报与心跳接口**: POST /api/v1/device/sync

- **请求体 (Request)**: 包含雷达解析后的 3 个目标坐标 (X, Y, Speed,
  Resolution)。

- **响应体 (Response)**: 状态驱动的核心载体。\
  {\
  \"code\": 200,\
  \"data\": {\
  \"next_interval\": 100, // 核心控制：告诉 ESP32 下次多久后联系我
  (100ms 或 1000ms)\
  \"server_time\": 1715000000,\
  \"pending_cmd\": { // 指令下发通道 (可选)\
  \"command_type\": \"SET_ZONE\", // 对应 0x00C2\
  \"payload\": [ // 预警区坐标 (将被固件转换为 Hex)\
  {\"x1\": 100, \"y1\": 200, \"x2\": 500, \"y2\": 600},\
  ...\
  ]\
  }\
  }\
  }

### 4.3 数据库设计与优化

- **核心表结构**:
  - radar_tracking_logs: 存储轨迹数据，索引 idx_mac_time 加速时间查询
  - device_shadow: 存储设备状态，包含 active_viewers 字段控制频率
  - guard_events: 存储报警事件，支持区域和时间过滤
  - pending_commands: 缓存待执行指令，索引 idx_mac_status 快速查找

- **性能优化**:
  - 使用联合索引加速历史查询
  - 自动过滤无效数据减少存储压力
  - WebSocket 连接计数实现按需加速

### 4.4 WebSocket 实时流

- **连接URL**: ws://server/ws/radar/live?mac={device_mac}
- **推送频率**: 10Hz (加速模式) 或 1Hz (待机模式)
- **消息格式**: JSON 包含 targets 数组和时间戳
- **连接管理**: 自动增加/减少 active_viewers 计数

### 4.5 API 接口列表

系统实现以下REST API接口：

- GET /api/v1/devices - 获取设备列表
- POST /api/v1/device/sync - 数据上报
- GET /api/v1/device/status - [未来版本] 设备状态查询
- GET /api/v1/radar/history - 历史轨迹查询
- GET /api/v1/guard/events - [未来版本] 守卫事件查询
- POST /api/v1/device/command - 指令下发
- WS /ws/radar/live - 实时数据流

所有API支持CORS跨域，统一错误响应格式。

## 5. 守卫模式 (Guardian Mode) 逻辑架构 [未来版本]

"守卫模式"的实现采取 **边缘-云端协同** 的策略：

1.  **边缘层 (过滤)**:

    - **职责**: 执行雷达内部指令 0x00C2 (Set Zone Filtering)。

    - **作用**:
      在硬件源头屏蔽掉"预警区"以外的信号（如窗帘摆动、宠物活动），确保上报的数据聚焦于核心区域。

2.  **服务层 (判定与告警)**:

    - **职责**: 维护"警戒时间轴"和"报警记录"。

    - **逻辑**:

      - 当数据上报时，API 检查当前是否处于用户设定的 Alert Time Window
        (如 22:00-06:00)。

      - 如果数据点落入 device_shadow 中存储的 zone_config
        区域，且速度/停留时间符合阈值，则触发 **Alarm Event**。

      - 生成一条记录写入 guard_events 表，并通过 WebSocket
        向前端推送红色警报状态。

3.  **应用层 (展示)**:

    - **职责**: 实时显示报警事件和历史记录。

    - **功能**: 通过 GET /api/v1/guard/events 查询历史报警，支持时间和设备过滤。

## 6. 技术实现细节

### 6.1 数据库优化

- **索引策略**: 
  - radar_tracking_logs: idx_mac_time (device_mac, created_at)
  - pending_commands: idx_mac_status (device_mac, status)

- **数据清理**: 自动过滤无效目标 (x=y=speed=resolution=0)

### 6.2 实时通信

- **WebSocket**: 使用 Flask-SocketIO 支持跨域连接
- **频率控制**: 通过 active_viewers 计数动态调整 ESP32 上报间隔
- **异步处理**: 使用 eventlet 实现非阻塞 I/O

### 6.3 错误处理

- **统一响应格式**: {"code": 200|400|404, "message": "...", "data": ...}
- **参数验证**: 所有API接口验证必需参数
- **异常捕获**: 数据库操作和网络请求的错误处理

### 6.4 安全考虑

- **MAC验证**: 所有查询需要设备MAC地址
- **CORS支持**: 允许跨域前端访问
- **输入过滤**: 防止SQL注入和恶意输入
