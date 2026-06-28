# 质量审查诊断报告（b_v8_diag_v1）

> 审查对象：`a_v8_design_v1.md`（v8 迭代 OOD 设计文档）
> 审查范围：需求响应充分度、事实错误与逻辑矛盾、深度与完整性、可落地性
> 前 7 轮迭代已覆盖大量技术可行性维度，本报告侧重此前未充分覆盖的维度

---

## 一、诊断发现

### 问题 1：【轻微·需求响应缺失】家属手动应急救援联动路径未建模

- **问题描述**：req_v4 §3.4 第 87 行明确要求"紧急救援按钮：家属在视频中发现紧急情况（如昏迷）时可一键触发应急救援联动（手动升级路径）"，这是与 BR-06 自动激活路径**并存**的手动升级路径。当前设计完整覆盖了高危失能场景下的自动激活路径（EmergencyActivatedEvent → PermissionService 自动授予家属端接入），但**家属端手动触发应急救援联动的路径在设计层面完全缺失**——既无对应领域事件（如 `FamilyManualRescueRequestedEvent`），也无对应领域服务方法，亦无行为契约场景覆盖。
- **所在位置**：§3.4 领域服务、§3.5 领域事件表、§四 关键行为契约（三处均无对应内容）
- **严重程度**：轻微（自动化路径已覆盖高危场景的救援上报，手动路径缺失不影响核心安全链路，但属需求忠实性偏差）
- **改进建议**：二选一——(a) 在 DS-12 EmergencyRescueService 中新增 `triggerManualRescue(driverId: DriverId, requesterAccount: AccountId)` 方法契约，由家属端在视频巡视时手动触发，产出事件驱动救援上报且不依赖碰撞/失能判定；(b) 在决策中声明"家属手动救援按钮由应用层编排，复用已有 EmergencyRescueService 的组装与上报能力，不引入新的领域事件类型"。

---

### 问题 2：【一般·关键遗漏】聚合根仓储（Repository）接口契约未定义

- **问题描述**：领域服务多处依赖 TripRepository、DriverRepository、RoadRageVoiceRecordRepository 等仓储接口进行聚合根读写操作：
  - DS-09 ScoringService：通过 TripRepository 写入 TripScore（第 553 行）
  - DS-18 DriverScoreUpdateService：通过 DriverRepository 读写 Driver 聚合（第 675 行）
  - DS-13 PrivacyProtectionService：通过 RoadRageVoiceRecordRepository 管理 AR-05 生命周期（第 608–611 行）
  - DS-19 AlertPersistenceService：通过 TripRepository 创建 SafetyAlertEvent（第 685 行）
  - DS-08 PermissionService：通过 SystemAccount 聚合的仓储写回 Permission 值对象（第 530 行）
  - DS-17 DrivingBehaviorTrackingService：更新 Trip 聚合的计数器（第 663 行）
  
  然而，设计文档中**所有仓储接口的方法签名、返回类型、错误语义均未形式化定义**。对比之下，领域事件总线（§3.6）、外部系统端口（决策 14/17/19/20/21）均有完整接口契约，处理标准不一致。缺失仓储契约将导致以下阻塞：
  - 开发者无法确定 `TripRepository.save(trip)` 的返回类型（`Unit`？`Result<Trip, PersistenceError>`？）
  - 无法确定乐观锁冲突时的错误语义（冲突抛异常 vs 返回 `Result`？）
  - RoadRageVoiceRecordRepository 的按到期时间查询方法签名未定义
- **所在位置**：全文未定义——§3.4 各领域服务中引用仓储；§6.2 提及乐观锁但未定义仓储接口；§3.6 事件总线契约已完整但仓储契约缺位
- **严重程度**：一般（阻塞编码实现，但属 DDD 设计惯例层次，实现者可能自行推断）
- **改进建议**：(a) 在 §三 或独立决策中至少为 TripRepository、DriverRepository、SystemAccountRepository、RoadRageVoiceRecordRepository 定义核心方法契约——`findById`、`save` 的方法签名、返回类型（`Result<T, PersistenceError>` 或 `Option<T>`）、乐观锁冲突错误语义；(b) 或明确声明"仓储接口遵循 DDD 通用仓储模式，由基础设施层定义具体接口，领域层仅声明依赖"，以消除歧义并统一标准。

---

### 问题 3：【一般·逻辑遗漏】远程对讲/视频介入时的驾驶员声光提示未建模

- **问题描述**：req_v4 §3.4 明确要求"远程对讲/低码率视频介入...启动时必须声光提示驾驶员"。当前设计在 DS-08 PermissionService 授予权限并发出 FamilyAccessGrantedEvent 后，消费方仅包含"家属 APP 推送、远程对讲/视频通道建立"（§3.5 事件表第 706 行），**驾驶员侧的"HMI 声光提示"消费动作在设计层未体现**——既无领域服务声明此职责，亦无场景契约描述该提示何时、以何种形式触发。该声光提示是需求明确的安全设计（让驾驶员知晓有人在远程监控），缺失意味着此行为可能在实现中被遗漏或由前端随意处理。
- **所在位置**：§3.5 领域事件表 FamilyAccessGrantedEvent 消费方（第 706 行）；DS-08 协作（第 529 行）；§四 场景 7 步骤 2（第 876 行，仅描述"家属 APP 收到事件后可发起对讲/视频接入"，未提驾驶员声光提示）
- **严重程度**：一般（影响需求忠实性且属安全设计要点）
- **改进建议**：(a) 在 FamilyAccessGrantedEvent 消费方中新增"HMI 层（向驾驶员发出声光提示：对讲/视频已接通）"；(b) 在场景 7 步骤 2 中补充"同时通过 HMI 向驾驶员发出声光提示——告知远程对讲/视频已建立，并保留驾驶员物理遮挡权"。

---

### 问题 4：【轻微·接口一致性】部分领域服务仍缺少形式化方法签名

- **问题描述**：本轮已为 DS-10 FleetAnalyticsService 和 DS-11 ReportGenerationService 补充形式化接口（参照决策 19 风格）。但以下领域服务的核心行为契约仍仅以自然语言描述，缺少方法签名（参数类型、返回类型、错误语义）：
  - **DS-02 FatigueDeterminationService**（第 432–436 行）：仅描述"输出疲劳等级和判定依据快照"，无方法签名
  - **DS-03 DistractionDetectionService**（第 440–445 行）：输出分心标志和时间戳，无方法签名
  - **DS-04 RoadRageDeterminationService**（第 450–459 行）：输出含 AlertType 的判定结果，无方法签名
  - **DS-05 LifeDetectionService**（第 464–472 行）：输出 LifeDetectedEvent 或 None，输入 DetectionWindow + 雷达信号，无方法签名
  - **DS-07 InterventionService**（第 492–510 行）：核心的二维映射逻辑以表格表达，但服务方法签名未定义
  - **DS-13 PrivacyProtectionService**（第 603–612 行）：事件驱动的多个行为，无统一方法签名
  - **DS-14 SensorSelfCheckService**（第 617–628 行）：两类事件的产出逻辑，无方法签名
  - **DS-17 DrivingBehaviorTrackingService**（第 658–665 行）：输入增量事件更新计数器，无方法签名
  这些服务属于 RiskDeterminationService 的门面委托子服务或独立判定服务，其方法契约应被下游的流式判定门面引用。缺少签名将阻塞门面的编码实现。
- **所在位置**：DS-02（第 432 行）、DS-03（第 440 行）、DS-04（第 450 行）、DS-05（第 464 行）、DS-07（第 492 行）、DS-13（第 603 行）、DS-14（第 617 行）、DS-17（第 658 行）
- **严重程度**：轻微（门面已定义统一的 SensorReading 输入契约，子服务签名可由实现者从描述中推断，但缺少统一的形式化标准）
- **改进建议**：为上述服务至少定义核心方法签名。如 DS-02 可定义为 `determine(sensorData: SensorReading): Result<FatigueLevel, DeterminationError>`；DS-03 可定义为 `detect(sensorData: SensorReading, window: DetectionWindow): Result<DistractionResult, DeterminationError>` 等。也可声明"各子判定服务的方法签名由其调用方 RiskDeterminationService 在实现阶段按需定义"，以消除当前的不一致感。

---

### 问题 5：【轻微·语义重叠】BR-01 疲劳 L2 与分心 L2 共用"视线偏离"信号未显式区分

- **问题描述**：BR-01 将"连续 3 分钟内视线偏离前方累计超过 15 秒"归入**疲劳 L2**（氛围灯变橙），而分心规则将"最近 60 秒滑动窗口内视线偏离前方累计达到 3 秒"归入**分心 L2**（告警）。两者共用同一行为信号"视线偏离"，仅窗口时长和阈值不同（3min/15s vs 60s/3s）。当前设计将两者分别分配给 DS-02 FatigueDeterminationService 和 DS-03 DistractionDetectionService，但**未阐述二者同时满足时会发生什么**——同一驾驶员的视线偏离行为可能同时触发疲劳 L2 和分心 L2 两个 RiskDeterminedEvent，导致双重干预（氛围灯变橙 + 分心告警同时下发）。虽然 DS-01 门面的会话上下文 ActiveRiskSet 可以允许两个不同类型的风险共存，但设计未说明这种双重触发是否为预期行为，亦未说明是否有去重/优先级逻辑。
- **所在位置**：DS-02（第 432–436 行）、DS-03（第 440–445 行）、req_v4 BR-01（第 110 行）与分心规则（第 132 行）
- **严重程度**：轻微（属边界条件澄清，不影响核心判定链路的正确性，但可能影响实际干预体验）
- **改进建议**：(a) 在 DS-01 门面或决策章节中补充说明——"视线偏离"信号在不同时间窗口/阈值下可被两个子服务独立判定，同时满足时两个 RiskDeterminedEvent 均会产出，干预模块按各自二维映射独立执行（氛围灯变橙 + 分心告警互不冲突）；或(b) 明确分心 L2 优先级高于疲劳 L2，当两类判定同时成立时仅产出分心 L2 告警以抑制重复提醒。标注为"需与需求方确认"。

---

### 问题 6：【轻微·完整性】缺少聚合根-仓储对照表

- **问题描述**：设计文档定义了 5 个聚合根（AR-01 Trip、AR-02 Driver、AR-03 Vehicle、AR-04 SystemAccount、AR-05 RoadRageVoiceRecord），但未以结构化形式汇总每个聚合根对应哪个仓储以及仓储的基本职责。开发者在实现阶段需要自行梳理这一映射关系，增加了理解成本。
- **所在位置**：§三 全文，尤其是 §3.1 聚合根各条目
- **严重程度**：轻微（可由开发者从协作关系中推断，但缺少汇总表降低设计文档的组织可用性）
- **改进建议**：在 §3.1 末尾或 §三结尾处新增一张汇总表：

| 聚合根 | 对应仓储 | 仓储基本能力 |
|--------|---------|-------------|
| AR-01 Trip | TripRepository | 按 TripId 查找、保存（含乐观锁）、按 Driver/Vehicle/时间范围查询 |
| AR-02 Driver | DriverRepository | 按 DriverId 查找、保存（含乐观锁）、按家属账户查监护列表 |
| AR-03 Vehicle | VehicleRepository | 按 VehicleId 查找、保存 |
| AR-04 SystemAccount | SystemAccountRepository | 按 AccountId 查找、按角色查询、保存 |
| AR-05 RoadRageVoiceRecord | RoadRageVoiceRecordRepository | 按告警 ID 查找、按到期时间查询待清除记录、创建/保存 |

---

## 二、整体评价

经审查，a_v8_design_v1.md 作为第 8 轮迭代产出，已具备较高的成熟度：
- **BR-01~BR-08 全部业务规则**均有对应领域服务与行为契约覆盖；
- 前 7 轮诊断发现的 50 余项问题均已修复，设计内部一致性显著提升；
- 领域事件总线契约（§3.6）、外部系统端口（决策 19）、维映射体系（决策 15/16）均已达可编码粒度；
- AR-05 独立聚合根、DS-19 AlertPersistenceService 等架构决策解决了 DDD 分层惯例的核心张力。

本轮发现的 6 个问题中，**未发现严重问题**。1 个一般问题（仓储接口缺失）和 1 个需求忠实性偏差（家属手动救援路径缺失）建议优先修复；其余 4 个轻微问题为锦上添花性质。

---

## 修订说明（v1）

首轮审查，无前序质询或修订记录。

