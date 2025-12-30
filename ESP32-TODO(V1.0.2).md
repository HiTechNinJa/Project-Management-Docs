# 源码开发 TODO 清单 (Target: IoT Gateway)

当前 src/main.cpp 已实现如下功能：
- WiFi连接（硬编码密码，未用WiFiManager）
- HTTP客户端，数据POST上传
- 雷达数据解析与JSON封装
- 自动获取MAC，仅开机一次
- 支持动态调整上传频率（uploadInterval）
- 板端上电自动重启雷达

---

## 已完成
- [x] WiFi连接（setup()已实现，硬编码密码）
- [x] HTTP客户端（已用HTTPClient.h上传数据）
- [x] 雷达数据解析与JSON封装（parseTargetsFromRadarBuf + ArduinoJson）
- [x] 自动获取MAC地址（仅开机一次）
- [x] 支持动态调整上传频率（uploadInterval）
- [x] 板端上电自动重启雷达（setup()已插入runCmd("Reboot Module", 0x00A3, NULL, 0）

---

## 待完成
- [ ] WiFiManager配网（替换硬编码密码，提升用户体验）
- [ ] loop()内断网自动重连（WiFi.status()检测与重连逻辑）
- [ ] HTTP响应解析指令（如data.cmd，自动执行SET_MODE/REBOOT等）
- [ ] 远程下发指令映射（如cmd.type=="SET_MODE"自动runCmd，cmd.type=="REBOOT"自动重启）
- [ ] 看门狗机制（防止网络请求卡死，提升系统健壮性）
- [ ] 代码结构优化与注释完善

---

### 近期进展
- [x] 板端ESP32上电自动重启雷达，setup()已插入runCmd("Reboot Module", 0x00A3, NULL, 0)
- [x] 数据上传与动态频率调整已联调通过

---

> 如需补充其他功能或细化任务，请直接补充。
