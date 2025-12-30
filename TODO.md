# 源码开发 TODO 清单 (Target: IoT Gateway)

目前的 src/main.cpp 是一个优秀的串口工具，需要按照以下步骤升级为 IoT
网关。

## 1. 网络基础模块

- \[ \] **WiFi 连接**: 引入 WiFi.h，在 setup() 中连接路由器。建议使用
  WiFiManager 库实现配网，避免硬编码密码。

- \[ \] **HTTP 客户端**: 引入 HTTPClient.h，用于发送 POST 请求。

## 2. 数据上报逻辑 (Loop 改造)

- \[ \] **JSON 封装**: 引入 ArduinoJson 库。

  - 创建 serializeData() 函数：将 radarBuf 解析出的 3 个目标数据打包成
    JSON。

- \[ \] **发送与接收**:

  - 在 loop() 中替换 Serial.println。

  - 发送 POST 请求到 /api/v1/device/sync。

  - **解析响应**: 读取 JSON 响应中的 data.interval 字段，更新全局变量
    currentUploadInterval。

## 3. 指令执行回调

- \[ \] **指令解析**: 在 HTTP 响应中检查 data.cmd 字段。

- \[ \] **执行映射**:

  - 如果 cmd.type == \"SET_MODE\", 调用现有的 runCmd(\"Set Mode\",
    0x0080/90\...)。

  - 如果 cmd.type == \"REBOOT\", 调用现有的
    requestAction(\"Reboot\"\...) (注意去掉手动确认逻辑，改为直接执行)。

## 4. 稳定性优化

- \[ \] **断网重连**: 在 loop 中检测 WiFi.status()，断线时尝试重连。

- \[ \] **看门狗**: 启用 Task Watchdog，防止网络请求超时导致系统卡死。

1.  
