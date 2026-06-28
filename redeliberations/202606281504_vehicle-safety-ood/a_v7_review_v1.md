# OOD 设计方案审查报告（v7）

## 审查结果

APPROVED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计方案中的类型形态选择与仓颉类型系统能力匹配。

- **聚合根/实体 → `class`**：Trip、Driver、Vehicle、SystemAccount、RoadRageVoiceRecord 五类聚合根，以及 SafetyAlertEvent、DriverHealthProfile 两类实体，均设计为 `class`（引用类型）。仓颉支持 `class` 作为引用类型，具备单继承能力，设计方案中无多重继承场景，符合仓颉类型系统约束。
- **值对象 → `struct`**：PhysiologicalSnapshot、GeoLocation、TripScore、OTAVersion、VehicleStateSnapshot、TimeRange、SensorReading、InterventionInstruction、RescueReport、Permission、DriverStatusSnapshot、DrivingBehaviorCounters、L3DurationTracker、NotificationPreference、OTAUpgradeStatus 共 15 个值对象均标注为 `struct`。仓颉中 `struct` 为值类型（值相等语义），天然契合值对象的不可变、由值定义相等性的语义。VO-03 中已标注"若因实现约束需用 `class` 则重写相等性方法"，设计决策 12 对此有完整论述。
- **枚举分类体系 → `enum`**：RiskLevel（L1_HINT / L2_WARNING / L3_CRITICAL）、AlertType（6 取值）、SensorStatus（3 取值）、AccountRole（FAMILY / MANAGER / RESCUE）、InterventionInstructionType（10 取值）、StatusColor（3 取值）、UpgradeStage（7 取值）、SessionType（2 取值）均使用 `enum`。仓颉 `enum` 支持编译期穷尽检查和模式匹配（match），设计方案中多处引用的"match 穷尽检查"语义在仓颉中是可行的。
- **端口/服务契约 → `interface`**：VehicleStateBuffer、PhysiologicalDataBuffer、DrivingBehaviorTrackingPort 及四个外部端口（NotificationPort、RescueReportPort、MediaSessionPort、OTADeliveryPort）均以 `interface` 声明。仓颉 `interface` 支持多实现和依赖注入，方案中的"基础设施层实现、装配阶段注入"模式可实现。
- **泛型使用**：领域事件总线契约中 `publish<E: DomainEvent>` 使用了泛型约束。仓颉支持泛型及类型边界约束，该用法在能力范围内。
- **集合类型**：`Array<T>`、`Set<RiskLevel>`、`Map<AlertType, RiskLevel>` 等均在仓颉标准集合类型覆盖范围内。
- **单继承约束**：聚合根和实体均为 `class`，端口均为 `interface`，不存在需要多重继承的场景。聚合间引用一律使用标识（`struct` 类型），符合 DDD 聚合设计原则且无类型系统冲突。

**[轻微]** VO-03 提到了 `class` 回退方案的相等性重写要求，此标注为良好实践，不构成问题。

### 2. 标准库与生态覆盖

**[通过]** 设计方案所需的能力在仓颉标准库或合理自定义范围内。

- **集合类型**：`Array`、`Set`、`Map` 为编程语言基本集合类型，仓颉标准库假设可覆盖。
- **Result / Option 模式**：设计方案中领域服务接口广泛使用 `Result<T, E>` 和 `Option<T>` 表达业务结果。"判定无法完成返回 None"、"业务规则违反返回 Result"等语义在 §5.2 错误表达方式选择表中有完整定义。仓颉支持 `Option<T>` 和 `Result<T, E>` 类型，方案使用方式匹配。
- **序列化**：Outbox 表的 `payload` 字段以 JSON 序列化存储，该需求属基础设施层职责，仓颉可通过标准或第三方 JSON 库覆盖。
- **领域事件总线**：§3.6 明确声明"仓颉标准库未原生提供领域事件发布/订阅机制，需自定义实现"，并提供了四项接口契约（publish / registerSyncHandler / registerAsyncHandler / outbox 表结构）。该自声明是诚实的，契约定义完整到足以指导自定义实现。
- **外部系统集成**：SMN 推送、SparkRTC 音视频、IoTDA OTA 下发、120 救援中心上报等外部系统依赖均通过领域端口（interface）抽象，基础设施层负责具体实现。设计未假设标准库提供这些能力，不存在不合理假设。
- **并发**：乐观锁（版本号）为通用模式，仓颉可通过数据库版本字段或 CAS 实现，不依赖语言级并发原语。

**[轻微]** 无。

### 3. 语言特性可行性

**[通过]** 设计方案中的错误处理、并发、资源管理和模块组织策略均与仓颉语言特性兼容。

- **错误处理策略**：§5 定义了完整的三类错误分类（A 类判定失败 → `Option/None` / B 类业务规则违反 → `Result<T, E>` / C 类系统级故障 → 异常抛出），并在 §5.2 中以表格逐场景给出了错误类型和理由。该策略完全可在仓颉中实现：`Option<T>` 和 `Result<T, E>` 为仓颉标准类型，异常抛出亦有支持。§5.3 整体原则对异常使用场景（仅限调用方错误使用 API）有明确边界约束。
- **并发设计**：§6 区分了边缘侧单线程确定性实时响应和云端侧多并发乐观锁两种模型。边缘侧依赖单线程顺序处理保证判定一致性（§6.1），云端侧依赖乐观锁处理聚合并发写入冲突（§6.2）。两种模型均与仓颉并发能力（无共享可变状态的 Actor 模型或协程等）无冲突——边缘侧单线程是部署约束而非语言特性要求，云端乐观锁是持久化层模式。
- **资源管理**：隐私数据生命周期管理（路怒语音存证录制→封闭→到期清除）、驾驶员注销的数据清理（DriverDeactivatedEvent 驱动的异步收尾）、历史数据匿名化保留等资源管理策略均以领域事件为核心机制设计，不依赖语言级 RAII 或其他特定资源管理原语。实现层面可通过仓储操作完成。
- **模块/包结构**：§2 定义了 10 个领域模块（`domain.model`、`domain.event`、`domain.risk` 等），依赖方向为单向——`domain.model` 为最底层，各领域服务模块同时依赖 `domain.model` 和 `domain.event`，模块间禁止直接调用。该结构与 cjpm 的包（package）组织方式兼容，无循环依赖。
- **值对象不可变性**：`struct` 类型选择天然支持不可变语义（创建后不可修改，状态推进以"创建新实例替换旧实例"方式完成），与仓颉 `struct` 的值语义一致。
- **枚举穷尽检查**：设计方案多处引用"编译期穷尽检查"（如 RiskLevel 的 match 分支、AlertType × RiskLevel 二维映射），仓颉 `enum` 的 match 表达式支持穷尽性检查。

**[轻微]** 无。

### 4. 设计一致性

**[通过]** 各抽象职责清晰，协作关系形成闭环，行为契约完整，依赖方向合理无循环。

- **职责清晰性**：
  - 5 个聚合根各有明确职责边界：Trip 为数据汇聚与告警枢纽、Driver 为被监护主体与健康档案持有者、Vehicle 为设备状态与固件版本管理、SystemAccount 为角色化外部主体、RoadRageVoiceRecord 为独立隐私数据载体。
  - 16 个领域服务各有唯一核心职责，无一服务承担多项不相关职责。
  - 15 个值对象各有明确定义的不可变领域概念。
- **协作闭环**：
  - §3.5 领域事件表覆盖所有 17 个领域事件，每个事件均有明确的触发时机、携带信息和消费方。
  - §4 关键行为契约以 12 个场景覆盖全部 16 个领域服务的核心交互，带时序约束的场景（分心 ≤0.5s、OTA 状态机、报告 ≤15s、常态同步 ≥1Hz/≤2s）均有可验证的契约描述。
  - 跨模块协作统一通过领域事件完成，各模块禁止直接调用，§2.2 依赖原则对此有明确约束。
- **行为契约完整性**：
  - v7 迭代修复的 5 个问题均在设计中完整闭环：PhysiologicalDataBuffer 端口契约（决策 21）、TripScore 持久化职责（DS-09 + 场景 5 + VO-05 + AR-01 七处一致）、Scene 6 双路径覆盖（6a 故障 + 6b 遮挡）、DriverDeactivatedEvent 入表（§3.5 + AR-02 + DS-08 + DS-13 + §5.4 一致更新）、NAVIGATE_DECELERATION/NAVIGATE_TO_SHOULDER 拆分（VO-12 + DS-07 + 场景 1 一致）。
  - 风险解除语义由独立 RiskResolvedEvent 承载，不污染 RiskLevel/AlertType 枚举（决策 16）。
  - 家属权限管理覆盖授予→临时撤销→常规自动撤销→遮挡恢复的完整状态机（DS-08、场景 7）。
- **依赖方向**：
  - `domain.model`（最底层）→ `domain.event`（仅依赖 model 标识）→ 各领域服务模块（依赖 model + event），无循环依赖。
  - 领域服务模块之间通过事件解耦，无编译期直接依赖。
  - EdgeSessionContext 归属基础设施层、不归属领域模型，L3DurationTracker 和 DrivingBehaviorCounters 归属 Trip 聚合（持久化状态），边界清晰（§6.2）。

**[轻微]** 无。

### 5. 设计质量

**[通过]** 职责划分合理，抽象层次恰当，便于后续详细设计和单元测试。

- **单一职责原则**：
  - RiskDeterminationService 作为门面委托三个子判定服务（Fatigue / Distraction / RoadRage），各子服务专注于单一风险类型的判定规则。
  - LifeDetectionService 和 EmergencyResponseService 作为独立事件触发型服务，不与流式融合门面耦合。
  - AlertPersistenceService 专注于"判定结果→告警实体→持久化→事件发出"编排，将判定与持久化解耦。
  - DriverScoreUpdateService 作为轻量服务仅负责领域事件到仓储操作的桥接。
- **抽象层次恰当**：
  - 本方案定位为架构级 OOD 设计，聚焦于职责划分、类型形态选择、协作模式和关键设计决策，未过度深入到具体字段、方法签名等实现细节。抽象度与 verifier 指令中"这是设计级别的抽象，缺少实现细节是正常的"期望一致。
  - 端口契约（决策 14、17、19、21）提供了方法签名和错误语义，程度适中——既足以指导上下游并行开发，又不涉及具体实现。
  - 2.2 模块依赖原则、3.6 事件总线契约、5.3 整体错误处理原则等提供了跨模块的设计约束，足以指导实现阶段保持架构一致性。
- **可测试性**：
  - 领域服务设计为无状态或纯函数式（DS-01 门面为纯函数、DS-05 以 DetectionWindow 为输入输出的纯函数、DS-08 以 L3DurationTracker 为输入输出的无状态服务），输入输出明确，便于单元测试。
  - 外部系统依赖通过端口（interface）抽象，可 mock，便于隔离测试。
  - 值对象的不可变性（struct）消除了测试中的状态泄漏风险。
  - 风险判定阈值（急刹/急加速、分心窗口）建模为可配置参数（决策 20），便于验收测试注入固定值。
- **设计决策文档化**：21 个设计决策均有明确的理由阐述和仓颉语言考量，为后续实现阶段的设计意图溯源提供了完整依据。
- **迭代修订可追溯**：v2~v7 六轮修订说明以表格记录每轮问题和修改措施，完整追溯到问题来源。

**[轻微]** 无。

## 修改要求

无。本轮审查未发现严重或一般问题。
