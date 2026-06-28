# OOD 设计方案审查报告（v5）

## 审查结果

APPROVED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计方案中的类型形态选择与仓颉类型系统能力完全匹配。

- 聚合根（AR-01~AR-04）和实体（E-01~E-03）选用 `class`：仓颉 `class` 为引用类型，支持单继承，有唯一标识和生命周期，契合聚合根的语义需求。实体与聚合根的包含关系（如 E-03 DriverHealthProfile 内聚于 AR-02 Driver）通过 class 间组合/嵌套表达，可行。
- 值对象（VO-01~VO-18）选用 `struct`：仓颉 `struct` 为值类型，天然提供值相等语义，是值对象的首选类型形态。设计已显式标注"若因实现约束需用 class，须重写相等性方法"，该约束合理。
- 有限集合（RiskLevel、AlertType、SensorStatus、AccountRole、StatusColor 等）选用 `enum`：仓颉 `enum` 支持编译期穷尽检查，与设计中"类型安全分支而非字符串/魔法值比较"的诉求一致。
- 领域服务抽象和外部端口（VehicleStateBuffer、DrivingBehaviorTrackingPort、NotificationPort 等）选用 `interface`：仓颉支持多接口实现，领域服务作为无状态操作封装以 interface 声明行为契约完全可行。
- 泛型使用（`Result<T,E>`、`Option<T>`、`Array<T>`、`Set<RiskLevel>`、`Map<AlertType,RiskLevel>`）：均为仓颉标准库具备的泛型类型构造，无新增语言能力诉求。
- 继承关系：聚合根之间无继承（各自独立），实体内聚于聚合根内（组合而非继承），接口实现关系符合仓颉单继承+多接口实现约束。

**[轻微]** 第1094行声明"`Result`/`Array`/`TimeRange` 均为前序已确认可行的标准类型构造"，但实际上本文档是第一版 OOD 设计（a_v5_v1），此处"前序已确认"的引用指向不清。建议改为"均为仓颉标准库已支持的类型构造"以明确结论来源。

### 2. 标准库与生态覆盖

**[通过]** 设计所依赖的能力均在仓颉标准库或合理自定义实现的覆盖范围内。

- 集合类型（`Array`、`Set`、`Map`）：仓颉标准库 native 支持，无额外依赖。
- `Result<T,E>` 与 `Option<T>`：仓颉标准库内置类型，设计中错误处理策略（§五）完全基于此构建。
- 领域事件总线：设计在 §3.6 明确声明"仓颉标准库未原生提供领域事件发布/订阅机制，需自定义实现"，并定义了完整的接口契约（publish/registerSyncHandler/registerAsyncHandler/outbox 表结构），该依赖假设合理。
- 外部系统（SMN/SparkRTC/IoTDA/120救援中心）：通过独立端口接口（NotificationPort、RescueReportPort、MediaSessionPort、OTADeliveryPort，决策19）抽象，领域层不直接依赖外部库，假设合理。
- 并发控制（乐观锁 on 云端侧）：为基础设施层职责，仓颉标准库提供基础原语支撑该模式，无额外生态假设。

**[轻微]** 领域事件总线的 `func publish<E: DomainEvent>(event: E): Result<Unit, EventPublishError>` 中，`DomainEvent` 作为泛型上界约束的一个类型（或 interface），但在设计中未明确定义 `DomainEvent` 接口本身应包含哪些公共字段（如 `occurredAt` 时间戳、`aggregateId` 聚合标识等）。建议在 §3.6 契约开头补充一个 `DomainEvent` 的最小接口定义（如含 `eventId` 和 `occurredAt` 两个公共字段），使各具体事件类型在实现时有一致基类。

### 3. 语言特性可行性

**[通过]** 设计的错误处理策略、并发模型、资源管理方案和模块结构均与仓颉语言特性兼容。

- **错误处理策略**（§五）：A 类（判定失败）→ `Option<T>` 返回 None，B 类（业务规则违反）→ `Result<T,E>`，C 类（系统级故障）→ 异常上抛。此三级分层与仓颉的 `Option` / `Result` / `throw` 机制完全对应，领域服务对外接口的 `Option`/`Result` 选择均有明确理由。
- **并发模型**（§六）：边缘侧单线程串行处理——不依赖并发语言特性，确定性实时响应可度量。云端侧乐观锁——基础设施层职责，仓颉标准库可支撑。共享状态（EdgeSessionContext）在单线程环境下无竞争，无需加锁。该设计合理利用部署拓扑避免了领域中引入复杂并发控制。
- **资源管理**：路怒语音存证（E-02）的创建→录制→封闭→到期清除生命周期清晰，由 PrivacyProtectionService（DS-13）通过领域事件驱动管理。OTA 升级（DS-15）的状态机各阶段转换条件明确，失败回滚保证终端可退至旧固件。EdgeSessionContext 随行程创建/销毁的生命周期边界在 §6.2 明确定义。
- **模块/包结构**（§二）：`domain.model` → `domain.event` → 各领域服务模块的单向依赖，符合 cjpm 包组织方式，无循环依赖。

### 4. 设计一致性

**[通过]** 各抽象的职责描述清晰，协作关系形成闭环，无缺失环节。

- **章节结构**：§3.5 领域事件标题仅出现一次（第638行），DS-18 DriverScoreUpdateService 已移至 §3.4 领域服务末尾，编号唯一性已恢复，下游引用歧义已消除。
- **Trip:Driver=1:1 约束**：AR-01（第81行）、VO-17（第366行）、DS-08（第504行）三处表述现已一致——均以"一个Trip内至多存在一个活跃L3DurationTracker，Trip:Driver=1:1"为约束，不再包含 per-Driver key 映射或多 Driver 表述。确认四处一致。
- **会话上下文（EdgeSessionContext）**：§6.2 已正式定义其生命周期、与 Trip 聚合的关系（协作而非归属）、持有内容边界（ActiveRiskSet + DetectionWindow），并明确 L3DurationTracker 和 DrivingBehaviorCounters 不归其持有（归属 Trip 聚合）。L3DurationTracker 的双重归属张力已消除。
- **外部端口契约**：决策19定义了四个完整端口（NotificationPort / RescueReportPort / MediaSessionPort / OTADeliveryPort），包含方法签名、参数类型、返回值类型和错误语义，与决策14/17的 VehicleStateBuffer / DrivingBehaviorTrackingPort 保持一致的契约标准。DS-12/DS-15/DS-16 的外部依赖已有领域层端口可依。
- **协作闭环**：16 个领域服务（DS-01~DS-18 减去无 DS-编号占位）均有行为契约场景覆盖（场景1~12），告警链路（判定→生成 SafetyAlertEvent→AlertTriggeredEvent→推送）、干预链路（RiskDeterminedEvent→InterventionService→InterventionInstruction）、权限链路（L3持续60s授予→L3下降撤销→FamilyAccessRevokedEvent）、评分链路（TripScoredEvent→DriverScoreUpdatedEvent→DriverScoreUpdateService→Driver聚合写回）均形成完整闭环。
- **模块依赖方向**：`domain.model`（底层）← `domain.event` ← 各领域服务模块，无反向依赖，无循环依赖。领域服务模块之间禁止直接调用，跨模块协作统一通过领域事件，该约束在 §2.2 明确声明。

### 5. 设计质量

**[通过]** 职责划分清晰，抽象层次恰当，便于后续实现和测试。

- **单一职责**：每个领域服务封装一个独立的业务规则或协作逻辑（如 DS-01 流式融合判定门面、DS-07 干预指令生成、DS-08 权限管理状态机），无过重服务。
- **值对象建模恰当**：VO-16 DrivingBehaviorCounters、VO-17 L3DurationTracker、VO-18 NotificationPreference 等新增值对象填补了前一轮缺失的领域概念，使各自职责内聚于独立的值类型中，避免裸数据暴露于聚合根。
- **抽象层次**：设计聚焦架构级 OOD——描述类型形态选择理由、协作关系和行为契约，未陷入字段/方法签名级别的实现细节。各个抽象（实体/值对象/领域服务）的"为何是当前类型形态"均有说明。
- **可测试性**：领域服务设计为无状态（DS-01/DS-05/DS-08 等将可变状态归 EdgeSessionContext 或 Trip 聚合，服务本身为纯函数），外部依赖通过 interface 端口注入，均可 mock 和隔离测试。L1 预留等级在枚举中存在，各 match 分支需穷尽处理，编译期即可保证分支完整性。
- **边界条件**：§5.4 覆盖断网本地持久化+幂等去重、驾驶员注销监护关系清理、毫米波雷达信号中断窗口策略三条，为后续实现提供了明确的容错设计方向。

**[轻微]** DS-17 DrivingBehaviorTrackingService 的描述（§3.4）注明其"在边缘侧持续运行，维持 trip 级会话内的增量累加语义"，但 DS-17 本身未声明对 Trip 仓储的依赖接口（通过什么机制读取和写回 Trip 聚合中的 VO-16）。虽然仓储操作属基础设施层职责，但作为领域服务设计，建议像 DS-09 ScoringService 那样补充一句"通过 TripRepository 读取和更新 Trip 聚合"以闭合数据访问路径描述。

## 修改要求

无。审查结论为 APPROVED，不存在严重或一般问题。
