# 总体架构设计

## 架构模式
State-Driven IoT Architecture

## 四层结构
1. 感知层：HLK-LD2450
2. 边缘层：ESP32（协议解析 + 指令执行）
3. 服务层：API + MySQL
4. 应用层：Web Dashboard

## 核心设计思想
- 设备不被动接收命令
- 服务端通过“响应”控制设备行为
