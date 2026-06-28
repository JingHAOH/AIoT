# OOD 设计质量审查报告（b_v9_diag_v1）

> 审查对象：a_v9_design_v2.md（第9轮迭代的领域层OOD设计文档）
> 审查范围：需求响应充分度、事实错误与逻辑矛盾、深度与完备性（侧重内部审议未充分覆盖的维度）
> 审查视角：实际落地——设计是否可直接指导编码、接口是否足以支持下游消费者、边界条件是否已考虑

---

## 审查发现

### 问题 1【一般·接口完备性】VehicleRepository 仓储接口未在 §3.7 中形式化定义

**问题描述**：§3.7 仅对 TripRepository、DriverRepository、SystemAccountRepository、RoadRageVoiceRecordRepository 四个仓储定义了完整的接口契约（方法签名、返回类型、错误语义）。VehicleRepository 仅在聚合根-仓储对照表的注释中被提及——"方法契约细节在此不单独展开，编码阶段参照 TripRepository / DriverRepository 的接口契约风格实现"。然而，多处领域服务需要对 Vehicle 聚合执行读写操作：

- DS-12 EmergencyRescueService 需更新车门锁状态（远程解锁授权场景）
- DS-15 OTAUpdateService 需读取当前固件版本（VO-08）、读写 OTAUpgradeStatus（VO-19）、升级完成后更新固件版本号
- DS-14 SensorSelfCheckService 需更新传感器自检状态

缺少 VehicleRepository 的形式化契约（至少 `findById` / `save` 方法签名和乐观锁冲突语义），上述领域服务的协作描述将缺少可编码的持久化契约依据。

**所在位置**：§3.7 聚合根仓储接口契约（仅定义了4个仓储）、§3.1 聚合根-仓储对照表（VehicleRepository 以注释代替）

**严重程度**：一般

**改进建议**：在 §3.7 中补充 VehicleRepository 的方法契约，至少包含：
```
func findById(vehicleId: VehicleId): Result<Option<Vehicle>, PersistenceError>
func save(vehicle: Vehicle): Result<Unit, PersistenceError>
```
标注乐观锁冲突处理与其余仓储一致。

---

### 问题 2【一般·接口完备性】DS-06 EmergencyResponseService 缺少形式化方法签名

**问题描述**：a_v9_v1 的修订清单为 DS-02~DS-05、DS-07、DS-13~DS-14、DS-17 共8个领域服务补充了核心方法契约。DS-06 作为碰撞失能判定的核心服务——承担着"碰撞冲击信号触发 → 通过 PhysiologicalDataBuffer 回取生理数据 → 通过 VehicleStateBuffer 回取车辆状态 → 判定失能条件 → 产出 EmergencyActivatedEvent"的完整判定链路——却未获得任何方法签名。该服务的调用方（基础设施层碰撞信号检测组件）需要明确：以什么方法名、什么参数类型、什么返回类型调用本服务。缺少方法签名，编码阶段无法确定调用契约。

**所在位置**：DS-06 EmergencyResponseService（§3.4，约第548-560行）

**严重程度**：一般

**改进建议**：参照 DS-02~DS-05 的接口定义风格，为 DS-06 补充核心方法签名，如：
```
func evaluateEmergency(collisionSignal: CollisionImpactSignal, tripId: TripId): Result<Option<EmergencyActivatedEvent>, DeterminationError>
```
明确 `CollisionImpactSignal` 的载荷结构（碰撞时刻、加速度幅值等）、`DeterminationError` 至少表达输入不可用和缓冲不可用的语义。

---

### 问题 3【一般·接口完备性】DS-15 OTAUpdateService 缺少形式化方法签名

**问题描述**：与问题2同类。DS-15 OTAUpdateService 驱动了完整的 OTA 升级状态机（PENDING → TRANSMITTING → VERIFYING → READY → UPGRADING → COMPLETED / ROLLED_BACK），其协作关系涉及版本比对、传输进度处理、校验结果处理、刷写结果处理等多个独立操作。但该服务没有任何方法签名——下游消费者（OTA 管理调度器、IoTDA 传输回调）无法确定以什么契约调用这些操作。对比 DS-07（有两个方法签名）、DS-14（有三个方法签名），DS-15 的处理标准明显不一致。

**所在位置**：DS-15 OTAUpdateService（§3.4，约第722-731行）

**严重程度**：一般

**改进建议**：参照 DS-07/DS-14 的接口契约风格，为 DS-15 补充核心方法签名，至少包含：
- `initiateUpgrade(vehicleId: VehicleId, targetVersion: OTAVersion): Result<Unit, OTAError>` — 版本比对后启动升级
- `handleTransferProgress(vehicleId: VehicleId, progress: DeliveryProgress): Result<Unit, OTAError>` — 处理传输进度并驱动状态机
- `handleVerificationResult(vehicleId: VehicleId, checksumMatch: Bool): Result<Unit, OTAError>` — 校验结果判定

---

### 问题 4【一般·逻辑矛盾】DS-06 EmergencyResponseService 在 DS-19 AlertPersistenceService 中的消费路径存在歧义

**问题描述**：DS-19 AlertPersistenceService 的协作描述（第779行）明确写道"对于 EmergencyActivatedEvent（碰撞失能）：创建 SafetyAlertEvent...不产生独立的 AlertTriggeredEvent（碰撞失能的推送由 EmergencyRescueService 路径处理，不走常规告警通知链路）"。但 §3.5 领域事件表中 EmergencyActivatedEvent 的消费方列有 AlertPersistenceService，且场景 4 步骤 3 描述为"AlertPersistenceService 消费 EmergencyActivatedEvent，创建 SafetyAlertEvent（AlertType=COLLISION_DISABILITY）"。

问题在于：BR-06 触发后，碰撞失能告警是否应出现在车队管理员的"安全告警事件"查询结果中？若 DS-19 仅创建 SafetyAlertEvent 但不发出 AlertTriggeredEvent，则车队管理者无法通过常规告警查询/统计看到碰撞失能记录——但 BR-05 评分需要 AlertType=COLLISION_DISABILITY 的告警统计吗？设计文档中 DS-09 的统计维度未包含 COLLISION_DISABILITY，但碰撞失能事件是否应被管理员事后审计/查询，设计未明确表态。DS-19 的"不产生 AlertTriggeredEvent"与"通过 TripRepository 创建 SafetyAlertEvent 并持久化"的并存，形成了 SafetyAlertEvent 实体存在但无法通过常规告警查询路径检索到的半可见状态。

**所在位置**：DS-19 协作（第779行）vs §3.5 事件表 EmergencyActivatedEvent 条目

**严重程度**：一般

**改进建议**：二选一：(a) 碰撞失能告警仍需发出 AlertTriggeredEvent（至少标记为"仅管理端可见"），使管理员可在看板/钻取中审计；(b) 在决策中明确声明碰撞失能告警不走常规告警通知链路但走独立的审计查询投影，避免 SafetyAlertEvent 持久化后成为不可检索的"暗数据"。

---

### 问题 5【轻微·需求响应缺失】救援机构的远程解锁授权模型未形式化

**问题描述**：req_v4 §3.6 明确"救援机构核实险情后，由云端授权开启车门锁"，§3.6 还要求"救援机构在授权后可调取驾驶员的健康档案"。DS-12 EmergencyRescueService 和场景 11 均提及"校验救援机构远程解锁授权（云端授权开启车门锁）"。但整份设计文档中，"救援机构授权"的模型完全以自然语言存在——授权的生命周期（何时授予、何时撤销、有效期）、与 AccountRole.RESCUE 的关系、授权凭证的建模方式均无形式化描述。对比之下，家属的常规权限拥有完整的状态机模型（授予→临时撤销→常规自动撤销→遮挡恢复），救援机构的权限模型严重缺失，编码阶段无法确定授权的数据结构和校验逻辑。

**所在位置**：DS-12 EmergencyRescueService（§3.4）、场景 11（§四）、AR-04 SystemAccount（§3.1）

**严重程度**：轻微

**改进建议**：二选一：(a) 在 DS-12 或决策中补充救援机构授权的建模——至少明确授权是作为 SystemAccount 的 Permission 值对象（与家属共享模型）还是独立的一次性授权令牌，以及授权的生命周期和校验流程；(b) 在决策中明确声明救援授权为"外部救援协调平台的一次性授权令牌，领域层仅校验令牌有效性而不维护其生命周期"，以消除建模歧义。

---

### 问题 6【轻微·接口设计】DomainEvent 接口的 aggregateId 字段使用 String 类型丢失类型安全性

**问题描述**：§3.5.0 将 DomainEvent 定义为 `interface DomainEvent { let aggregateId: String ... }`，aggregateId 被选型为 `String`。但整份设计文档中，所有聚合根标识均使用强类型——TripId、DriverId、VehicleId、AccountId、RecordId（决策 8 明确"聚合标识使用仓颉的 `struct` 类型"）。使用 String 会导致：(a) 事件消费方需手动将 aggregateId 字符串解析为具体标识类型才能调用仓储（如 DriverRepository.findById 需要 DriverId 而非 String），引入类型不安全的字符串操作；(b) 无法在编译期区分"这是哪个聚合根标识"——TripId 和 DriverId 同为 String 时，事件总线消费方可能误将 TripId 传入 DriverRepository 而编译期无警告。

**所在位置**：§3.5.0 DomainEvent 公共抽象（第792-803行）

**严重程度**：轻微

**改进建议**：考虑将 `aggregateId` 声明为泛型或专用值对象。例如：(a) 定义 `struct AggregateId { let value: String; let aggregateType: AggregateType }` 其中 AggregateType 为枚举（TRIP/DRIVER/VEHICLE/ACCOUNT/RECORD），保留字符串携带但附加类型区分；或 (b) 将 DomainEvent 声明为泛型接口 `interface DomainEvent<Id> { let aggregateId: Id ... }`，由各事件类型指定具体标识类型。

---

### 问题 7【轻微·边界条件缺失】DS-09 ScoringService 评分周期边界语义未定义

**问题描述**：DS-09 描述"周期级评分（周/月/季）由 ScoringService 按该周期内所有行程的 TripScore 按时长加权平均计算"，但以下边界条件未明确：(a) 当一次行程跨越周期边界时（如从周五23:00跨到周六01:00），TripScore 归属哪个周期？按行程开始时刻还是结束时刻归属？(b) "按行程时长加权平均"的具体计算公式未给出；(c) 周期评分在周期结束后即时计算还是按管理员的查询请求按需计算？这些缺失使 BR-05 的周期级评分在编码时缺乏确定性口径。

**所在位置**：DS-09 ScoringService（§3.4，约第613-633行）

**严重程度**：轻微

**改进建议**：在 DS-09 职责或决策中明确周期归属规则（如"按行程结束时刻归属周期"）、加权公式（如 `sum(TripScore × 行程时长) / sum(行程时长)`）、计算时机（如"周期结束后即时预计算"），并标注可配置或需与需求方确认的参数。

---

### 问题 8【轻微·并发策略遗漏】OTAUpgradeStatus 与传感器自检的并发一致性场景在 §6.3 未覆盖

**问题描述**：VO-19 OTAUpgradeStatus 被 Vehicle 聚合持有，DS-15 以无状态方式读写更新（通过 VehicleRepository）。同时 DS-14 SensorSelfCheckService 也通过 VehicleRepository 更新 Vehicle 聚合的传感器状态。§6.3 并发策略表覆盖了"评分计算与告警事件并发写入同一 Trip"、"家属权限授予/撤销的并发"等场景，但未覆盖"OTA 升级状态更新与传感器自检状态更新同时操作 Vehicle 聚合"的并发场景。虽然 §6.1 声明边缘侧单线程（无真正并发），但云端侧（OTA 升级管理可能部署于云端）与边缘侧的 Vehicle 聚合操作可能产生跨环境竞争。

**所在位置**：§6.3 并发策略表、VO-19 OTAUpgradeStatus（§3.3）、DS-15/DS-14（§3.4）

**严重程度**：轻微

**改进建议**：在 §6.3 并发策略表中补充 OTA 升级与传感器自检对 Vehicle 聚合的并发策略条目——明确乐观锁冲突时的处理语义（OTA 升级冲突时重试机制、传感器自检冲突时的降级策略），或明确声明"OTA 升级管理云端侧操作与边缘侧传感器自检通过单向事件通信隔离，不产生对 Vehicle 聚合的并发写"。

---

## 整体评价

经过9轮迭代，该设计文档已在需求覆盖（BR-01~BR-08 全部有对应领域服务）、模块划分、聚合根与实体建模、值对象形式化、领域事件体系、错误处理策略、边界条件覆盖等方面达到了较高的完备度。本轮审查在内部审议已覆盖的维度之外，主要发现以下可改进点：(1) Vehicle/DS-06/DS-15 三处接口契约存在不完备，会直接阻塞编码阶段；(2) 碰撞失能告警的持久化与可见性存在逻辑歧义；(3) 救援授权模型、评分周期边界、DomainEvent 类型安全性、OTA 并发策略四处细节尚有提升空间。8个问题中4个为一般严重度、4个为轻微，均为可修复的具体质量问题，不构成对整体架构正确性的否定。

---

## 与迭代历史的关系

本轮发现的8个问题均未在历史9轮迭代的诊断/质询中明确识别和修复。其中问题1~3（接口契约不完备）与第8轮问题1（仓储契约未定义）和第9轮问题4（部分领域服务缺方法签名）属同类但未被覆盖的具体实例；问题4（碰撞失能告警持久化歧义）未被任何前序轮次提出；问题5~8均为此次审查首次识别的细节遗漏。
