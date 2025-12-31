# 源码开发 TODO 清单 (Target: IoT Gateway) V1.0.4

当前 src/main.cpp 已实现如下功能：
- WiFi连接（硬编码密码，未用WiFiManager）
- HTTP客户端，数据POST上传
- 雷达数据解析与JSON封装
- 自动获取MAC，仅开机一次
- 支持动态调整上传频率（uploadInterval）
- 热重启优化：优先尝试256000波特率，智能启动序列
- 系统状态报告：开机时显示完整系统信息
- 自动检测间隔优化：15秒后台配置检查
- 远程指令执行（REBOOT、SET_MODE）

---

## 已完成
- [x] WiFi连接（setup()已实现，硬编码密码）
- [x] HTTP客户端（已用HTTPClient.h上传数据）
- [x] 雷达数据解析与JSON封装（parseTargetsFromRadarBuf + ArduinoJson）
- [x] 自动获取MAC地址（仅开机一次）
- [x] 支持动态调整上传频率（uploadInterval）
- [x] 热重启优化：优先256000波特率检测，避免不必要的扫描
- [x] 智能启动序列：检测响应→重启清理→重新验证→回退扫描
- [x] 系统状态报告：WiFi连接后自动显示雷达波特率、MAC、版本等信息
- [x] 自动检测间隔调整：从10秒优化为15秒，减少雷达查询压力
- [x] 板端上电自动重启雷达（setup()已插入runCmd("Reboot Module", 0x00A3, NULL, 0）- [x] loop()内断网自动重连（WiFi.status()检测与重连逻辑）
- [x] HTTP响应解析指令（如data.next_interval，自动调整上传频率）
- [x] 远程下发指令映射（已实现REBOOT、SET_MODE；SET_ZONE标记为未来版本）
- [x] 坐标解析修复（parseCoordinate函数符合协议定义）
- [x] raw模式限流（避免串口阻塞）
- [x] 帮助信息优化（添加Target输出解释）
- [x] 指令执行功能测试通过（REBOOT和SET_MODE指令均正常工作）
---

## 待完成
- [ ] WiFiManager配网（替换硬编码密码，提升用户体验）
- [ ] 看门狗机制（防止网络请求卡死，提升系统健壮性）
- [ ] 代码结构优化与注释完善

---

### 近期进展
- [x] **热重启流程优化**：ESP32启动时优先尝试256000波特率，2秒内检测雷达响应。如果响应则发送重启命令清理状态，等待2秒重启完成后再验证连接；如果失败才执行完整的自动波特率扫描
- [x] **系统状态报告**：WiFi连接成功后，自动生成包含雷达波特率、WiFi状态、MAC地址、雷达版本/MAC/模式/区域信息的完整系统报告
- [x] **自动检测优化**：将后台配置检查间隔从10秒增加到15秒，减少对雷达的查询频率，提升系统稳定性
- [x] 板端ESP32上电自动重启雷达，setup()已插入runCmd("Reboot Module", 0x00A3, NULL, 0)
- [x] 数据上传与动态频率调整已联调通过
- [x] WiFi自动重连机制已实现，每5秒检测并重连
- [x] 坐标解析逻辑修复，Y坐标不再出现负数
- [x] raw模式限流修复，解决串口卡顿问题
- [x] 帮助信息添加Target输出解释
- [x] 远程指令执行功能实现并测试通过，支持REBOOT和SET_MODE指令
- [x] ArduinoJson兼容性修复，使用is<JsonObject>()替代containsKey()
---

> 如需补充其他功能或细化任务，请直接补充。