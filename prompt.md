# prompt

## 需求澄清与获取

启动需求澄清或者审议框架，对 `D:\软件测试\requirements\202606242158_vehicle-safety-monitoring\req_v3.md` 需求文档进行审议和需求澄清。

---

## OOD设计

### 第一阶段：领域层 OOD

启动再审议框架，执行OOD流程，完成如下任务：

需求文档：
D:\软件测试\requirements\202606242158_vehicle-safety-monitoring\req_v4.md

这是一个「智能物联——基于多传感器融合的车载安全监测系统」软件项目。系统包含多维感知、AI 风险判定引擎（边缘—云协同）、闭环干预与反馈、远程监护（家属 APP）、车队运营管理（大屏/报表）、应急救援联动、OTA 固件升级管理等功能域。

技术栈：前端（家属 APP / 车队大屏）采用 ArkTS（HarmonyOS），后端（云端服务 + 边缘车载端服务）统一 Java Spring Boot，部署于华为云。设备接入基于华为云 IoTDA（MQTT 协议），存储使用 GaussDB/RDS，推送使用 SMN，音视频对讲使用 SparkRTC。

已完成的需求澄清与评审文档在：
D:\软件测试\requirements\202606242158_vehicle-safety-monitoring\
（含 req_v1~v4 四轮迭代及 review_v1~v4 四轮审查，v4 审查结论为 APPROVED。）

请你为本系统做领域层 OOD 设计，产出：实体 / 值对象 / 聚合根 / 领域服务 / 领域事件。需要覆盖需求文档第四~五节的全部业务规则（BR-01~BR-08）和核心业务对象。不要实现。
在流程启动阶段不要读取文件，直接启动流程。

---

### 第二阶段：应用层 OOD

启动再审议框架，执行OOD流程，完成如下任务：

承接第一阶段领域层 OOD 产出，为本系统做应用层 OOD 设计。

领域层 OOD 产出在：
{第一阶段最终产出路径}

需求文档：
D:\软件测试\requirements\202606242158_vehicle-safety-monitoring\req_v4.md

（项目概述与技术栈同第一阶段，不再重复。）

请你为本系统做应用层 OOD 设计，产出：各功能域的应用服务定义（RiskMonitoringService / InterventionService / RemoteGuardianshipService / FleetManagementService / EmergencyRescueService / OTAManagementService）及其接口契约、服务间协作关系、核心时序图（至少覆盖疲劳判定→告警→干预链路、活体遗留→报警链路、碰撞失能→SOS+家属自动激活链路三条关键路径）。不要实现。
在流程启动阶段不要读取文件，直接启动流程。

---

### 第三阶段：基础设施/适配层

（留待技术方案阶段）