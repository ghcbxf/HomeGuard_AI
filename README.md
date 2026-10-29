# HomeGuard AI

微缩智能家居模型 + 家庭安全机器人助手“小卫”。

## 项目简介
固定安防节点采集门磁、人体、温湿度、漏水等数据，机器人底盘执行巡检；设备通过 MQTT 上报，Python 后端汇总入库，中文网页控制台查看状态，后期接入 AI 语音管家。

## 目录结构
- firmware/house_node/  固定安防节点（ESP32：门磁、PIR、温湿度、漏水、蜂鸣器、LED）
- firmware/robot/       机器人底盘（两轮电机、测距、急停、状态机）
- backend/app/          FastAPI 后端接口
- backend/db/          SQLite 数据库与初始化脚本
- frontend/             中文网页控制台
- docs/                 项目计划、文档、测试记录
- config/               配置文件（secrets.env 不入库）

## 技术栈
ESP32、MQTT、Python FastAPI、SQLite、网页前端、AI 语音管家

## 开发阶段标签
- v0.1-sensors  本地传感器闭环
- v0.2-robot    机器人底盘
- v0.3-mqtt     设备通信
- v0.4-backend  后端服务
- v0.5-web      网页控制台
- v1.0-release  交付版本
