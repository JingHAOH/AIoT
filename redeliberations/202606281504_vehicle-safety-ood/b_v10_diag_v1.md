# 质量审查报告 — a_v10_design_v1.md

> 审查对象：`D:\软件测试\redeliberations\202606281504_vehicle-safety-ood\a_v10_design_v1.md`
> 审查轮次：第 10 次迭代 / 首轮审查
> 审查视角：需求响应充分度、事实错误/逻辑矛盾、深度与完整性（实际落地视角）
> 审查日期：2026-06-28

---

## 总体评价

a_v10_v1 在 a_v9_v2 基础上修复了 8 个持续问题（VehicleRepository 契约补全、DS-06/DS-15 方法签名补充、碰撞失能告警持久化路径修正、救援授权模型形式化、DomainEvent aggregateId 类型安全性修正、评分周期边界语义定义、OTA 与传感器自检并发策略补充），BR-01~BR-08 全部业务规则均有对应领域服务，12 个行为契约场景覆盖全部 19 个领域服务，聚合根、实体、值对象已齐备。文档总体结构完整、模块边界清晰。

经审查，本轮发现 **3 个一般性问题** 和 **4 个轻微问题**，主要集中在：事件消费路径的逻辑矛盾、错误类型的建模一致性缺口、临时状态类型的形式化不足，以及部分生命周期/数据获取路径的细节遗漏。以下逐项描述。

---

## 问题清单

### 问题 1【一般·逻辑矛盾】FamilyManualRescueRequestedEvent 的消费路径存在设计矛盾

- **问题描述**：§3.5 领域事件表中 `FamilyManualRescueRequestedEvent` 的消费方列为"EmergencyRescueService（组装 RescueReport 并投递至救援中心）"，但 DS-12 EmergencyRescueService 同时是该事件的**产出者**（场景 4b 步骤 5 由 DS-12 产出该事件）。同一领域服务既是事件产出者又是消费者，构成设计层面的自循环——步骤 4 已由 DS-12 完成 RescueReport 数据组装，步骤 7 再由"救援链路"消费事件后投递 RescueReport，与步骤 4 的组装动作存在逻辑冗余，且"救援链路"指代不清（是 DS-12 自身还是其他服务？）。
- **所在位置**：§3.5 事件表 FamilyManualRescueRequestedEvent 条目（第 874 行）、§四 场景 4b 步骤 4-7（第 1088-1097 行）
- **严重程度**：一般
- **改进建议**：二选一：(a) DS-12 在 `triggerManualRescue` 方法内直接调用 RescueReportPort 投递 RescueReport，随后产出 FamilyManualRescueRequestedEvent 仅用于通知家属 APP（消费方移除 EmergencyRescueService）；(b) 新增轻量领域服务 RescueReportDeliveryService 作为独立消费方，DS-12 仅产出事件、RescueReportDeliveryService 消费后执行投递，消除自循环。

---

### 问题 2【一般·接口完备性】领域服务中使用的多个错误类型未在 §3.3 值对象目录中正式定义

- **问题描述**：DeterminationError（DS-02~DS-06）、DetectionError（DS-05）、BufferError（VehicleStateBuffer/PhysiologicalDataBuffer）、PersistenceError（§3.7）、QueryError（DS-10）、ReportError（DS-11）、NotificationError、RescueReportError、MediaSessionError、OTADeliveryError、OTAUpgradeError、PrivacyError、SelfCheckError、EventPublishError 等十余种错误类型均以**内联方式**描述于各领域服务接口契约或端口契约中（如"`DeterminationError` 至少表达 `InputInvalid`"），但 **§3.3 值对象目录无任何错误类型的独立 VO 条目**。这导致：(1) 错误类型的完整取值集合（enum 变体）分散在各处、无统一索引；(2) 不同领域服务间的 PersistenceError 是否共享同一类型定义不明；(3) 下游消费者在实现跨模块错误处理分支时缺少可引用的中心化类型契约。
- **所在位置**：§3.3 值对象目录（全章）、§3.4 各领域服务接口契约、§3.6 事件总线契约、§3.7 仓储接口契约、决策 14/19/20/21
- **严重程度**：一般
- **改进建议**：在 §3.3 新增 VO 条目或独立子节（如"错误类型枚举"），对全部错误类型做中心化定义。各领域服务接口契约中仅引用类型名，不再以内联方式重复描述。建议至少为 PersistenceError（跨多个仓储复用）、DeterminationError（跨多个判定服务复用）、BufferError（跨多个缓冲端口复用）三类高频复用错误类型提供正式 enum 取值定义。

---

### 问题 3【一般·深度不足】EdgeSessionContext 中 ActiveRiskSet 未形式化为值对象类型

- **问题描述**：§6.2 描述 EdgeSessionContext 持有的"当前活跃风险集"以 `Map<AlertType, RiskLevel>` 松散标注，DS-01 RiskDeterminationService 的"风险解除判定"依赖该集合对比当前判定与历史状态以决定是否产出 RiskResolvedEvent，DS-16 DriverStatusBroadcastService 依赖该集合派生状态色（绿/黄/红）。但 ActiveRiskSet 未如 DetectionWindow（VO-20）、L3DurationTracker（VO-17）等类似会话级状态一样形式化为独立值对象类型，缺少：(1) 不变量约束（如同一 AlertType 不可同时存在于活跃集和已解除状态）；(2) 类型安全的状态操作契约（如 `add(alertType, riskLevel): ActiveRiskSet`、`remove(alertType): ActiveRiskSet` 等不可变更新方法）。实现者需自行从自然语言描述推断类型形态。
- **所在位置**：§6.2 边缘侧会话上下文（第 1285 行）、DS-01 解除判定说明（第 491 行）、DS-16 职责描述（第 770-772 行）
- **严重程度**：一般
- **改进建议**：参照 VO-20 DetectionWindow 的建模方式，在 §3.3 新增 VO-24 ActiveRiskSet 值对象（`struct`），定义其不可变更新契约和不变量约束，明确由 EdgeSessionContext 持有，作为 DS-01 判定门面的输入/输出参数和 DS-16 的读取来源。

---

### 问题 4【轻微·接口完备性】DS-17 DrivingBehaviorTrackingService 无法确定当前活跃 Trip

- **问题描述**：DS-17 的 `onHardBrakingDetected` / `onHardAccelerationDetected` 回调（决策 20 端口契约）携带 `HardBrakingEvent` / `HardAccelerationEvent`，DS-17 需据此递增当前活跃 Trip 聚合的 DrivingBehaviorCounters。但回调参数中不含 Trip 标识，设计也未描述 DS-17 如何获取当前活跃 Trip——是通过 EdgeSessionContext 持有活跃 TripId、通过 TripRepository 查询（无查询条件）、还是由基础设施层回调携带 TripId？该缺失将阻塞 DS-17 的编码实现。
- **所在位置**：DS-17 协作说明（第 786-789 行）、决策 20 端口方法签名（第 1546-1547 行）
- **严重程度**：轻微
- **改进建议**：二选一：(a) 在决策 20 的回调方法签名中增加 `tripId: TripId` 参数（与 `onHardBrakingDetected` / `onHardAccelerationDetected` 已有风格一致），由基础设施层在检出事件时携带当前活跃行程标识；(b) 在 DS-17 协作说明中明确通过 EdgeSessionContext 获取当前活跃 TripId。

---

### 问题 5【轻微·深度不足】DriverHealthProfile（E-03）缺少生命周期管理描述

- **问题描述**：E-03 DriverHealthProfile 作为关键医疗数据实体，设计描述了其被查询的路径（DS-12 救援场景经授权调取、VO-13 RescueReport 纳入健康档案摘要），但未描述档案的初始创建时机（驾驶员注册时创建空档案？首次行程时创建？）、更新机制（驾驶员本人通过 APP 更新？管理员维护？系统自动采集？）以及负责创建/更新的领域服务。档案的生命周期管理是驾驶员管理模块的必要设计输入，当前缺失将阻塞下游实现。
- **所在位置**：E-03 DriverHealthProfile（§3.2，第 184-193 行）、AR-02 Driver 协作关系（第 97-104 行）
- **严重程度**：轻微
- **改进建议**：在 E-03 协作关系中补充档案的创建时机（建议为 Driver 聚合创建时同步创建空档案）和更新路径（建议明确为 Driver 聚合的应用层服务通过 DriverRepository 操作），或在 AR-02 协作关系中新增"Driver 聚合负责 DriverHealthProfile 的创建与更新，外部通过 DriverRepository 加载 Driver 聚合后操作该实体"的说明。

---

### 问题 6【轻微·深度不足】OTAUpgradeStatus 状态机缺少统一的合法状态转换矩阵

- **问题描述**：VO-19 定义 UpgradeStage 枚举（PENDING / TRANSMITTING / VERIFYING / READY / UPGRADING / COMPLETED / ROLLED_BACK），DS-15 四个方法描述了各自驱动的状态转换：`initiateUpgrade`→PENDING/TRANSMITTING、`handleTransferProgress`→VERIFYING/ROLLED_BACK、`handleVerificationResult`→READY/ROLLED_BACK、`handleFirmwareFlashResult`→COMPLETED/ROLLED_BACK。场景 9 描述了完整流程。但状态机构成要素分散于五处（VO-19 + 四个方法契约 + 场景 9），**缺少统览性的合法状态转换表或状态机图**，实现者需自行从分散描述中推断完整状态机，存在遗漏非法转换校验（如直接从 PENDING 跳转 UPGRADING 的防护）的风险。
- **所在位置**：VO-19 OTAUpgradeStatus（§3.3，第 408-422 行）、DS-15 方法契约（第 761-766 行）、场景 9（第 1164-1176 行）
- **严重程度**：轻微
- **改进建议**：在 VO-19 或 DS-15 中新增合法状态转换矩阵（如表格列出 from→to 及触发条件），或明确声明"除上述四个方法描述的状态转换路径外，任何其他跃迁均为非法、DS-15 在写回 Vehicle 聚合前须校验当前阶段与目标阶段的合法性"。

---

### 问题 7【轻微·边界条件】边缘侧与云端侧仓储实现差异未在仓储接口契约中体现

- **问题描述**：§5.4(1) 描述边缘侧断网时 Trip 本地优先持久化以保证安全告警链路成立，§6.2 描述边缘侧 Trip 聚合"单线程写入、无需加锁"。但 TripRepository 接口契约（§3.7.1）仅定义 `findById` + `save` 的通用签名，乐观锁通过 `PersistenceError.OptimisticLockConflict` 表达。边缘侧本地存储（如 SQLite/文件存储，单线程无并发冲突）与云端 GaussDB/RDS 的乐观锁策略是本质不同的实现——边缘侧 save 永不会返回 OptimisticLockConflict（单线程保证），而云端可能并发冲突。当前单一契约掩盖了这一差异，基础设施层实现者需推断是提供两个独立实现还是通过配置切换乐观锁行为。
- **所在位置**：§3.7.1 TripRepository 接口契约（第 944-951 行）、§5.4(1) 边界条件（第 1256 行）、§6.2 边缘侧状态管理（第 1278 行）
- **严重程度**：轻微
- **改进建议**：在 §3.7 仓储设计原则（§3.7.6）或 §5.4(1) 中补充说明——边缘侧与云端侧可各自提供独立的 TripRepository 实现（边缘实现乐观锁为 no-op，云端实现执行版本号比对），两实现遵循相同接口契约但并发策略不同；或在 TripRepository 契约中注明"乐观锁检测为基础设施层可选实现，边缘侧单线程环境下实现可为空操作"。

---

## 审查结论

产出 a_v10_v1 已覆盖全部 BR-01~BR-08 业务规则与核心业务对象，12 个行为契约场景覆盖 19 个领域服务。本轮发现 7 个问题（3 一般 / 4 轻微），无严重（阻塞级）问题。其中：

- **问题 1**（事件自循环）和 **问题 2**（错误类型未形式化）建议在下一轮优先处理，因其影响下游消费者的接口依赖和实现确定性。
- **问题 3**（ActiveRiskSet）和 **问题 4**（DS-17 Trip 确定路径）影响编码完整性。
- **问题 5~7** 为深度补充项，可在后续迭代中逐步完善。

