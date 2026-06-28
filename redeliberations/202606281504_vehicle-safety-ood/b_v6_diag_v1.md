# 质量审查报告（b_v6_diag_v1）

> 审查对象：D:\软件测试\redeliberations\202606281504_vehicle-safety-ood\a_v6_copy_from_v5.md
> 审查轮次：第 6 轮迭代 / 首轮审查
> 审查视角：需求响应充分度、事实错误/逻辑矛盾、深度与完整性；以及从实际落地视角评估：设计是否可直接指导编码实现、接口定义是否足以支持下游消费者、异常场景和边界条件是否已考虑。
> 审查范围侧重：内部审议（设计-验证循环）已覆盖技术可行性，本审查侧重审议未充分覆盖的维度。

---

## 一、发现的质量问题

### 问题 1：PhysiologicalDataBuffer 端口仅提及名称，未定义接口契约

- **问题描述**：DS-06 EmergencyResponseService（第 477 行）明确声明"通过领域层声明的依赖接口（PhysiologicalDataBuffer 端口）回取该缓冲窗内数据"；决策 13（第 1110–1112 行）也引用该端口。然而，整个设计文档中 **PhysiologicalDataBuffer 的方法签名、输入参数类型、返回类型、错误语义均未定义**。与此形成对比的是，同属基础设施层缓存的 VehicleStateBuffer 在决策 14（第 1125 行）有完整方法契约 `getSnapshots(tripId, window): Result<Array<VehicleStateSnapshot>, BufferError>`，DrivingBehaviorTrackingPort 在决策 20（第 1250–1253 行）有完整回调契约 `onHardBrakingDetected(event)`，四个外部系统端口在决策 19（第 1190–1237 行）均有完整定义。**PhysiologicalDataBuffer 是唯一一个被引用但无接口契约的端口**——实现者拿到设计文档后，对 DS-06 的生理数据回取接口无任何可编码的契约依据。
- **所在位置**：DS-06（第 477 行）、决策 13（第 1110–1112 行）；对照：决策 14 VehicleStateBuffer（第 1125 行）、决策 20 DrivingBehaviorTrackingPort（第 1250–1253 行）、决策 19 四个外部端口（第 1190–1237 行）
- **严重程度**：一般
- **改进建议**：参照决策 14 的 VehicleStateBuffer 建模方式，补充 PhysiologicalDataBuffer 端口契约。至少包含：方法名（如 `getRecentReadings(tripId, window)` 或 `getBuffer(window)`）、输入参数（TripId + TimeWindow）、返回类型（如 `Result<Array<PhysiologicalSnapshot>, BufferError>`）、错误语义（如 `WindowNotCovered` / `BufferUnavailable`，以及 DS-06 的降级策略）。

---

### 问题 2：TripScore 写入 Trip 聚合的职责归属存在内部表述不一致

- **问题描述**：AR-01 Trip 聚合（第 84 行）的协作关系中明确表述"领域服务 ScoringService 在行程结束时计算 TripScore，数值以值对象形式存入 Trip"——即 ScoringService 负责将 TripScore 写回 Trip 聚合。但 DS-09 ScoringService 的正式职责描述与协作小节（第 532–548 行）中仅描述查询/读取操作和事件产出（TripScoredEvent、DriverScoreUpdatedEvent、PerformanceWarningEvent），**未提及将 TripScore 写回 Trip 聚合的持久化操作**。场景 5（第 819–826 行）也只描述了计算与事件产出，第 823 行仅写"发出 TripScoredEvent"，未写"写入 Trip"。AR-01 与 DS-09/场景 5 之间对"谁写入 TripScore"的表述不一致：若 ScoringService 直接通过 TripRepository 写入（如 AR-01 所述），则 DS-09 协作说明遗漏此操作；若 TripScore 由 TripScoredEvent 的某个消费方写入（类似 DS-18 DriverScoreUpdateService 的写回模式），则该消费方未定义。无论哪种情况，实现者无法确定 TripScore 的持久化职责归属。
- **所在位置**：AR-01 第 84 行 vs DS-09 第 532–548 行 vs 场景 5 第 819–826 行
- **严重程度**：一般
- **改进建议**：二选一：(a) 在 DS-09 协作中显式补充"通过 TripRepository 将 TripScore 值对象写入 Trip 聚合"；(b) 新增轻量服务（如 TripScorePersistenceService）订阅 TripScoredEvent 执行写回，参照 DS-18 的 Driver 评分写回模式。无论取哪一方案，AR-01、DS-09、场景 5 三处必须表述一致。

---

### 问题 3：Scene 6 传感器故障场景未反映 DS-14 已分化的双路异常检测模型

- **问题描述**：DS-14 SensorSelfCheckService（第 600–610 行）已在 v4 迭代中重构为区分两类异常：①传感器硬件/链路故障→产出 SensorFailureEvent（全系统失效告警）；②摄像头物理遮挡→产出 CameraOcclusionDetectedEvent / CameraOcclusionRemovedEvent（仅影响依赖该摄像头的功能，不触发全系统失效）。然而，Scene 6（第 830–837 行）的标题为"传感器故障失效保护（BR-08）"，其内容仅覆盖 SensorFailureEvent 的出产与消费（HMI 语音提示 + 车队看板标记脱线），**完全没有涉及 CameraOcclusionDetectedEvent / CameraOcclusionRemovedEvent 的处理路径**。物理遮挡→权限临时撤销的关键行为链路在 Scene 6 中缺席（仅出现在 Scene 7 的权限视角），导致 Scene 6 作为"传感器自检的端到端行为契约"实际上遗漏了 DS-14 设计的一半产出路径。
- **所在位置**：Scene 6（第 830–837 行）vs DS-14（第 600–610 行）
- **严重程度**：轻微
- **改进建议**：(a) 在 Scene 6 中补充物理遮挡检出与解除的处理分叉——CameraOcclusionDetectedEvent → HMI 提示（区别于全系统失效告警的文字/级别）+ PermissionService 权限临时撤销；CameraOcclusionRemovedEvent → PermissionService 权限恢复判定；(b) 或将物理遮挡的端到端契约独立为 Scene 6b，与 Scene 6a（故障）并列，并在 Scene 6 标题下添加总述说明两条路径的差异。

---

### 问题 4：驾驶员注销/账号删除的触发领域事件未列入事件表

- **问题描述**：§5.4 边界条件 (2)（第 963 行）描述了驾驶员注销/账号删除时的清理动作——清理监护关系（发出 FamilyAccessRevokedEvent）、匿名化保留历史行程数据、删除/脱敏敏感数据，并声明"清理动作以领域事件驱动各模块异步收尾"。然而，§3.5 领域事件一览表（第 676–693 行）中**没有类似 DriverDeactivatedEvent 或 DriverAccountDeletedEvent 的事件类型**。导致"以领域事件驱动"的承诺在设计层面悬空——各模块无法订阅一个尚未定义的事件来触发各自的清理逻辑。
- **所在位置**：§5.4 边界条件 (2)（第 963 行）vs §3.5 领域事件一览表（第 676–693 行）
- **严重程度**：轻微
- **改进建议**：在 §3.5 事件表中新增类似 `DriverDeactivatedEvent`（携带 Driver 标识、时间戳、清理策略标识），并为其指定消费方（如 SystemAccount 聚合清理监护关系、PrivacyProtectionService 清理 RoadRageVoiceRecord、匿名化处理服务等）。或若认为 Driver 注销是基础设施层操作、不建模为领域事件，则在 §5.4 中明确说明清理由"应用层服务编排"驱动而非领域事件，以消除矛盾表述。

---

### 问题 5：NAVIGATE_TO_SHOULDER 指令类型承载了两个语义不同阶段的干预意图

- **问题描述**：VO-12 InterventionInstructionType 枚举（第 294 行）定义了 NAVIGATE_TO_SHOULDER 指令类型。在 DS-07 二维映射 FATIGUE×L3 条目（第 495 行）中，渐进升级链使用 NAVIGATE_TO_SHOULDER 同时表达"建议/请求减速"和"引导靠边"两个阶段——二者干预强度、HMI 表现和驾驶员覆盖检测的触发阈值均不相同（"建议减速"阶段驾驶员轻微干预即可覆盖，"引导靠边"阶段可能需要更明确的覆盖动作）。当前设计以同一枚举值 + 不同参数区分两个阶段，语义上可行，但缺乏对两阶段差异的明确建模（如 VOICE_BROADCAST 与 NAVIGATE_TO_SHOULDER 之间再加入 NAVIGATE_DECELERATION 或分阶段参数结构），基础设施层收到 NAVIGATE_TO_SHOULDER 指令时无法仅凭指令类型区分当前处于建议减速还是强制引导靠边。
- **所在位置**：VO-12 第 294 行、DS-07 二维映射表第 495 行
- **严重程度**：轻微
- **改进建议**：二选一：(a) 拆分 NAVIGATE_TO_SHOULDER 为 NAVIGATE_DECELERATION（建议减速）和 NAVIGATE_TO_SHOULDER（引导靠边），各含明确的参数结构与驾驶员覆盖策略；(b) 维持当前设计，在 InterventionInstruction 的参数映射中显式定义阶段标记参数（如 `phase: "deceleration" | "shoulder"`），使基础设施层可按阶段做差异化处理。

---

## 二、整体评价

本产出（a_v6_copy_from_v5.md）在第 5 轮基础上修复了 8 个已确认问题，领域模型覆盖完整（5 个聚合根、19 个值对象、19 个领域服务、16 个领域事件），行为契约覆盖全部 16 个领域服务，错误处理策略含 A/B/C 三类分类及边界条件（断网去重、驾驶员注销、雷达中断），并发设计区分边缘侧单线程与云端乐观锁。领域事件总线契约（§3.6）和外部系统端口契约（决策 19）为下游编码提供了正式的接口约束。整体上产出已达到较高的设计完成度。

以上 5 个问题中，2 个为"一般"严重度（问题 1 阻塞 DS-06 的编码实施，问题 2 造成 TripScore 持久化职责悬空），3 个为"轻微"（问题 3-5 不阻塞主线编码但在细节上留有歧义或遗漏）。无事实性严重错误或根本性逻辑矛盾。

---

DIAG_WRITTEN:D:\软件测试\redeliberations\202606281504_vehicle-safety-ood\b_v6_diag_v1.md
主Agent请勿阅读产出文件内容，直接将路径转发给相关方。
