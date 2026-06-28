# 智能物联——基于多传感器融合的车载安全监测系统

## 项目概述

这是一个「智能物联——基于多传感器融合的车载安全监测系统」软件项目。系统包含多维感知、AI 风险判定引擎（边缘—云协同）、闭环干预与反馈、远程监护（家属 APP）、车队运营管理（大屏/报表）、应急救援联动、OTA 固件升级管理等功能域。

## 技术栈

- 前端（家属 APP / 车队大屏）：ArkTS（HarmonyOS）
- 后端（云端服务 + 边缘车载端服务）：统一 Java Spring Boot，部署于华为云
- 设备接入：华为云 IoTDA（MQTT 协议）
- 存储：GaussDB/RDS
- 推送：SMN
- 音视频对讲：SparkRTC

## 需求文档

已完成的需求澄清与评审文档位于：
- D:\软件测试\requirements\202606242158_vehicle-safety-monitoring\
- 含 req_v1~v4 四轮迭代及 review_v1~v4 四轮审查
- v4 审查结论为 APPROVED

核心需求文档：D:\软件测试\requirements\202606242158_vehicle-safety-monitoring\req_v4.md

## 任务要求

为本系统做领域层 OOD（面向对象设计），产出：
- 实体（Entity）
- 值对象（Value Object）
- 聚合根（Aggregate Root）
- 领域服务（Domain Service）
- 领域事件（Domain Event）

需要覆盖需求文档第四~五节的全部业务规则（BR-01~BR-08）和核心业务对象。
不要实现。
