# HLK-LD.2450 智能雷达可视化前端设计文档 (V1.0.2)

## 1. 页面功能模块规划

本系统采用响应式 Web 设计，分为 **实时态势 (Live)**、**历史回溯 (History)** 和 **设备配置 (Settings)**
三大核心模块。[守卫预警 (Guard) 模块标记为未来版本，现阶段不用实现]

### 1.1 实时态势仪表盘 (Live Dashboard)

- **可视化扇形图**：使用 Canvas 绘制 \$\\pm 60\^\\circ\$、半径 6
  米的扇形区域 1-3。

- **多目标追踪**：以不同颜色的圆点实时渲染最多 3 个目标（Target 1-3），根据设备当前模式动态显示（单目标模式仅显示T1，多目标模式显示全部） 3,
  4。

- **动态浮窗**：点击目标点显示实时 X/Y 坐标、速度及距离 3, 5。

- **上传频率指示**：显示当前 ESP32 的上报状态（待机 1Hz / 活跃 10Hz） 6,
  7。

### 1.2 历史回溯与统计 (Analytics) ------ **新增核心逻辑**

- **轨迹重放 (Path Replay)**：

- 通过 API /api/v1/radar/history 获取指定时间段的坐标点位 12。

- 在扇形图上以"淡入淡出"的尾迹线展示目标的移动路线，支持 1x/2x/4x
  速度回放。

- **空间占用热力图 (Heatmap)**：[未来版本 - 需后端实现聚合API]

- 统计数据库内一段周期（如 24 小时）的所有位置点。

- 将 6
  米扇形区划分为若干网格，根据点位密度渲染颜色，直观显示哪个位置是人员高频活动区（如家中的办公椅、沙发区）。

### 1.3 设备高级控制 (Control Panel)

- **模式切换**：单目标/多目标追踪模式一键下发（指令 0x0080/0x0090）
  15-17。

- **蓝牙开关**：控制雷达蓝牙功能的开启/关闭，并提示用户需重启生效 10,
  18, 19。

- **系统诊断**：实时显示 ESP32 的固件版本、MAC
  地址、最后一次心跳时间及在线状态 14, 20。

- **区域设置**：[未来版本 - 守卫模式相关]

## 2. 核心算法与交互逻辑

### 2.1 坐标转换公式 (Radar to Canvas)

雷达上报的是以自身为原点的直角坐标 (X, Y)，单位 mm 21,
22。前端需按照以下比例尺映射至网页像素 17：

- **坐标含义**：X=0代表雷达视角正前方有人，X>0代表左侧，X<0代表右侧。Y代表的距离，恒为正，单位为毫米。

- **画布原点**：设 Canvas 宽度 \$W\$，高度 \$H\$。原点 \$(0, 0)\$ 对应
  Canvas 底部中心 \$(W/2, H)\$ 17。

- **转换公式**：

- screenX = (W / 2) + (x_mm \* Scale)

- screenY = H - (y_mm \* Scale)

- 其中 Scale = H / 6000 (基于 6 米最大量程) 17。

### 2.2 实时加速通信流 (Active Sync)

为保证前端查看时的"丝滑感"，系统遵循以下流程 6, 7：

1.  **用户进入**：前端打开 Web 页面，通过 WebSocket 连接服务器。

2.  **触发加速**：服务器检测到当前该 MAC 地址有活跃观察者，在 ESP32 下次
    HTTP Sync 时，通过响应体告知其 interval: 100ms 6。

3.  **高频渲染**：前端通过 WebSocket 接收 10Hz 的 JSON 数据包，使用
    requestAnimationFrame 确保 Canvas 渲染不掉帧 6。

4.  **用户离开**：关闭页面后，服务器自动调低 ESP32 上传频率至 1Hz 6。

## 3. 前端 API 调用规划 (供开发参考)

### 设备发现与选择机制

为简化用户操作，前端应实现自动设备发现：

1. **页面初始化**：加载时自动调用 `GET /api/v1/devices` 获取所有设备列表
2. **设备选择**：
   - 如果只有一个在线设备，自动选中
   - 如果有多个设备，显示下拉选择器让用户选择
   - 优先选择在线设备 (`online_status: true`)
3. **MAC获取**：使用选中的设备MAC调用后续API，无需用户手动输入

**前端实现示例**:
```javascript
// 设备发现与选择
async function loadDevices() {
  try {
    const response = await fetch('/api/v1/devices');
    const result = await response.json();
    
    if (result.code === 200 && result.data.length > 0) {
      // 过滤在线设备
      const onlineDevices = result.data.filter(d => d.online_status);
      
      if (onlineDevices.length === 1) {
        // 自动选择唯一在线设备
        setSelectedDevice(onlineDevices[0].device_mac);
      } else if (onlineDevices.length > 1) {
        // 显示设备选择器
        showDeviceSelector(onlineDevices);
      } else {
        // 无在线设备，显示提示
        showNoDeviceMessage();
      }
    }
  } catch (error) {
    console.error('Failed to load devices:', error);
  }
}

// 使用选中的MAC调用其他API
function loadDeviceStatus(mac) {
  fetch(`/api/v1/device/status?device_mac=${mac}`)
    .then(res => res.json())
    .then(data => updateDeviceStatus(data));
}
```

这样用户无需手动输入MAC，前端自动获取并使用，提高用户体验。

### API 调用表

功能模块,API 路径,方法,说明

设备列表,/api/v1/devices,GET,获取所有设备MAC列表，用于自动设备发现

实时流,/ws/radar/live?mac={id},WS,获取 10Hz 实时坐标推送（后端已实现WebSocket支持） 6

历史回溯,/api/v1/radar/history,GET,\"参数：device_mac, start_time, end_time。获取历史轨迹点（后端已实现） 12\"

守卫日志,/api/v1/guard/events,GET,\"[未来版本] 获取入侵报警的历史列表与坐标节点（后端需实现） 13\"

设备状态,/api/v1/device/status,GET,获取ESP32固件版本、MAC、最后心跳时间及在线状态（后端已实现） 14, 20

下发指令,/api/v1/device/command,POST,\"存储待执行指令（重启、切模式，后端已实现pending_commands处理） 14, 20\"


---

## 4. 详细API使用指南

### 4.1 设备列表API (GET /api/v1/devices)

**用途**: 获取系统中所有已注册的雷达设备信息，用于前端设备选择和自动发现。

**请求示例**:
```javascript
fetch('/api/v1/devices')
  .then(response => response.json())
  .then(result => {
    if (result.code === 200) {
      console.log('设备列表:', result.data);
    }
  });
```

**成功响应**:
```json
{
  "code": 200,
  "data": [
    {
      "device_mac": "E4:B0:63:B4:C0:E0",
      "online_status": true,
      "firmware_ver": "V2.04.23101915",
      "last_heartbeat": "2025-12-31T08:30:00Z",
      "active_viewers": 1
    }
  ]
}
```

**前端处理逻辑**:
- 过滤在线设备 (`online_status: true`)
- 如果只有一个设备，自动选中
- 如果有多个设备，显示选择器
- 显示设备状态指示器（在线/离线）

### 4.2 实时数据流 (WebSocket /ws/radar/live)

**用途**: 建立WebSocket连接，接收10Hz实时雷达数据推送。

**连接建立**:
```javascript
const ws = new WebSocket(`ws://your-server:5000/ws/radar/live?mac=${selectedMac}`);

ws.onopen = () => {
  console.log('WebSocket连接已建立');
  // 触发ESP32加速上传 (10Hz)
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'radar_data') {
    updateRadarDisplay(data.targets);
  }
};

ws.onclose = () => {
  console.log('WebSocket连接已关闭');
  // ESP32恢复正常上传频率 (1Hz)
};
```

**数据格式**:
```json
{
  "type": "radar_data",
  "device_mac": "E4:B0:63:B4:C0:E0",
  "targets": [
    {
      "x": 1456,
      "y": 1017,
      "speed": 0,
      "resolution": 360
    }
  ],
  "timestamp": 1735631400.123
}
```

**前端渲染逻辑**:
- 使用Canvas绘制扇形区域 (±60°, 6米半径)
- 坐标转换: `screenX = canvas.width/2 + (x_mm * scale)`
- 不同目标用不同颜色标识
- 支持点击显示详细信息浮窗

### 4.3 历史轨迹API (GET /api/v1/radar/history)

**用途**: 获取指定时间段的历史雷达数据，用于轨迹回放和分析。

**请求示例**:
```javascript
const params = new URLSearchParams({
  device_mac: selectedMac,
  start_time: '2025-12-31T08:00:00',
  end_time: '2025-12-31T09:00:00'
});

fetch(`/api/v1/radar/history?${params}`)
  .then(response => response.json())
  .then(result => {
    if (result.code === 200) {
      renderTrajectory(result.data);
    }
  });
```

**成功响应**:
```json
{
  "code": 200,
  "data": [
    {
      "target_id": 1,
      "pos_x": 1456,
      "pos_y": 1017,
      "speed": 0,
      "resolution": 360,
      "created_at": "2025-12-31T08:00:00.000Z"
    }
  ]
}
```

**前端处理逻辑**:
- 支持时间范围选择器
- 轨迹回放控制 (播放/暂停/倍速)
- 淡入淡出效果显示历史路径
- 可选择显示特定目标的轨迹

### 4.4 设备状态API (GET /api/v1/device/status)

**用途**: 获取设备的详细状态信息，包括模式、版本、连接状态等。

**请求示例**:
```javascript
fetch(`/api/v1/device/status?device_mac=${selectedMac}`)
  .then(response => response.json())
  .then(result => {
    if (result.code === 200) {
      updateDevicePanel(result.data);
    }
  });
```

**成功响应**:
```json
{
  "code": 200,
  "data": {
    "device_mac": "E4:B0:63:B4:C0:E0",
    "online_status": true,
    "firmware_ver": "V2.04.23101915",
    "track_mode": "multi",
    "bluetooth_state": false,
    "zone_config": [],
    "active_viewers": 1,
    "last_heartbeat": "2025-12-31T08:30:00Z"
  }
}
```

**前端显示逻辑**:
- 实时更新设备状态指示器
- 显示当前跟踪模式 (single/multi)
- 蓝牙状态开关
- 最后心跳时间和在线状态

### 4.5 指令下发API (POST /api/v1/device/command)

**用途**: 向ESP32设备发送控制指令，支持重启和模式切换。

**重启指令**:
```javascript
fetch('/api/v1/device/command', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    device_mac: selectedMac,
    command_type: 'REBOOT',
    payload: {}
  })
})
.then(response => response.json())
.then(result => {
  if (result.code === 200) {
    showMessage('重启指令已发送，设备将在下次上报时执行');
  }
});
```

**模式切换指令**:
```javascript
// 切换到单目标模式
fetch('/api/v1/device/command', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    device_mac: selectedMac,
    command_type: 'SET_MODE',
    payload: { mode: 'single' }
  })
});

// 切换到多目标模式
fetch('/api/v1/device/command', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    device_mac: selectedMac,
    command_type: 'SET_MODE',
    payload: { mode: 'multi' }
  })
});
```

**成功响应**:
```json
{
  "code": 200,
  "message": "Command queued successfully",
  "command_id": 3
}
```

**前端交互逻辑**:
- 提供模式切换按钮 (单目标/多目标)
- 重启设备按钮
- 显示指令发送状态和确认
- 指令可能延迟执行（需等待ESP32下次数据上报）

### 4.6 错误处理

**统一错误响应格式**:
```json
{
  "code": 400,
  "message": "Invalid device MAC address"
}
```

**常见错误码**:
- `400`: 请求参数错误
- `404`: 设备未找到
- `500`: 服务器内部错误

**前端错误处理**:
```javascript
function handleApiError(error) {
  switch(error.code) {
    case 404:
      showMessage('设备未找到，请检查设备是否在线');
      break;
    case 400:
      showMessage('请求参数错误');
      break;
    default:
      showMessage('网络错误，请稍后重试');
  }
}
```

### 4.7 实时更新机制

**WebSocket重连逻辑**:
```javascript
function connectWebSocket() {
  const ws = new WebSocket(wsUrl);

  ws.onclose = () => {
    setTimeout(connectWebSocket, 3000); // 3秒后重连
  };

  ws.onerror = () => {
    showMessage('实时连接断开，正在重连...');
  };
}
```

**定期状态更新**:
```javascript
// 每30秒更新一次设备状态
setInterval(() => {
  if (selectedMac) {
    loadDeviceStatus(selectedMac);
  }
}, 30000);
```

---

## 更新日志 (V1.0.1)

- **坐标解析修复**：后端parseCoordinate函数已修正，确保Y坐标为正值，符合坐标转换公式。
- **数据过滤优化**：后端device_sync已过滤无效目标（x=y=speed=resolution=0），前端无需额外处理无效数据。
- **单/多目标模式**：前端需根据设备模式动态显示目标数量（单目标仅T1，多目标最多3个）。
- **API规划更新**：统一历史回溯API路径为/api/v1/radar/history，添加设备状态API /api/v1/device/status。
- **后端实现需求**：需实现WebSocket实时流、历史查询、守卫事件、设备状态和指令下发API。
- **守卫模式调整**：守卫预警模块标记为[未来版本]，暂不开发；指令下发API移除SET_ZONE支持。
- **指令执行功能**：ESP32远程指令执行功能已实现并测试通过，支持REBOOT和SET_MODE指令。前端可通过POST /api/v1/device/command API发送指令，ESP32在下次数据上报时执行。
- **API文档完善**：添加详细的API使用指南，包括请求示例、响应格式、前端处理逻辑和错误处理。
