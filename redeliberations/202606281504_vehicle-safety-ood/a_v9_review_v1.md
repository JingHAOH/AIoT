# OOD 设计方案审查报告（v9）

## 审查结果

REJECTED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计方案中的类型形态选择与仓颉类型系统能力匹配。聚合根使用 `class`（引用类型、有独立标识和生命周期）、值对象使用 `struct`（值类型、值相等语义）、分类枚举使用 `enum`（编译期穷尽检查）——均符合仓颉语言特性。Decision 12 明确阐述了 `struct` 作为值对象首选类型形态的理由并考虑了 `class` 的退路（重写相等性方法），类型选择有据可依。

**[通过]** 抽象之间的继承和实现关系在仓颉约束范围内：聚合根之间通过标识（`struct` 标识类型）引用而非对象引用（Decision 8），避免了跨聚合的深层对象图；领域服务通过 `interface` 定义契约并由基础设施层实现，遵循单继承多接口实现模式；未出现在仓颉类型系统中不可行的继承结构。

**[通过]** 泛型使用方式在仓颉泛型系统能力范围内：事件总线 `publish<E: DomainEvent>`、集合类型 `Array<T>`/`Set<T>`/`Map<K,V>`、`Result<T, E>` 和 `Option<T>` 均为仓颉泛型系统的标准使用模式，无超出能力边界的泛型约束。

**[通过]** 协作关系中描述的类型交互模式可行：跨聚合通过标识引用（`TripId`、`DriverId` 等 `struct` 类型）、领域服务通过 `Result`/`Option` 与调用方交互、领域事件通过事件总线（泛型接口）解耦发布与消费，均可在仓颉中实现。

### 2. 标准库与生态覆盖

**[通过]** 设计中需要的基础数据结构和集合能力均在仓颉标准库覆盖范围内：`Array`、`Set`、`Map`、`struct`、`enum`、`interface`、泛型。

**[通过]** 设计对库能力的假设合理且已有应对：设计明确承认"仓颉标准库未原生提供领域事件发布/订阅机制，需自定义实现"（§3.6），并自行定义了完整的事件总线契约（`publish`、`registerSyncHandler`、`registerAsyncHandler`、outbox 表结构约定）。Result/Option 等类型虽需自行定义或从扩展库引入，但设计层已明确声明其语义，不对标准库做过高假设。

**[通过]** 无标准库能力可显著简化当前自定义抽象的场景——事件总线、仓储模式等 DDD 基础设施均需自定义实现，设计已提供了充分契约。

### 3. 语言特性可行性

**[通过]** 错误处理策略与仓颉的错误处理能力匹配：§五明确采用 `Result<T, E>` + `Option<T>` 模式，区分 A 类（判定失败 → `None`）、B 类（业务规则违反 → `Result<T, E>`）、C 类（系统级故障 → 异常抛出），三层策略层次清晰且在仓颉中可完整实现。

**[通过]** 并发设计与仓颉的并发模型兼容：边缘侧单线程模型（§6.1）保证确定性实时响应，无并发竞争——在仓颉单线程运行时中天然成立；云端侧聚合根使用乐观锁（版本号）控制并发——与仓颉中通过仓储实现的乐观锁模式完全兼容。

**[通过]** 资源管理方案在仓颉资源管理模式内可行：边缘侧会话状态（EdgeSessionContext）的创建与销毁对应行程开始与结束——可在仓颉中以明确的生命周期管理（如构造/析构或显式 init/dispose 方法对）实现。

**[通过]** 模块/包结构设计（`domain.model`、`domain.risk`、`domain.intervention`、`domain.family`、`domain.fleet`、`domain.emergency`、`domain.privacy`、`domain.ota`、`domain.monitor`、`domain.event`）符合 cjpm 的项目组织方式——每个模块可映射为一个 cjpm package，模块间单向依赖（§2.2）在 cjpm 依赖管理中不作反向依赖。

### 4. 设计一致性

**[一般]** **DetectionWindow 值对象未在 §3.3 值对象目录中正式定义**。DS-05 接口契约 `evaluateLifeDetection(radarSignal: SensorReading, window: DetectionWindow): Result<DetectionResult, DetectionError>` 将 `DetectionWindow` 作为形式化方法参数，§6.2 EdgeSessionContext 也将其列为持有内容之一，DS-05 设计约束中称"将窗口的剩余时长、起始时刻、累计微动观测等封装为独立的 **`DetectionWindow` 值对象**"。但该值对象未被列入 §3.3 值对象目录（当前目录仅含 VO-01~VO-19），缺少其类型形态（`struct`）、概念维度（各字段含义）和协作关系的正式定义。它出现在领域服务的形式化方法签名中，缺少定义将阻塞编码阶段对该参数类型的实现。

**[一般]** **`DomainEvent` 类型/接口在设计文档中未定义**。§3.6 事件总线契约的四个核心方法均依赖 `DomainEvent` 作为泛型约束：`publish<E: DomainEvent>(event: E)`、`registerSyncHandler<E: DomainEvent>`、`registerAsyncHandler<E: DomainEvent>`。该类型决定了"一个类型如何才能被视为领域事件"——是需要实现一个 marker interface、继承某个基类，还是仅作为约定（任何携带特定字段的类型都可作为领域事件）。此外，不同事件共有的元数据（如 `occurredAt` 时间戳、`eventId` 唯一标识）是否由 `DomainEvent` 定义还是由各事件自行携带也未澄清。该类型是事件总线实现的前提，未定义将直接阻塞事件总线的编码。

**[通过]** 各抽象的职责描述清晰无歧义。19 个领域服务、5 个聚合根、19 个值对象均有完整的角色与职责、类型形态、协作关系三栏说明。

**[通过]** 协作关系形成闭环。跨模块协作统一通过领域事件完成（§2.2），各事件的触发时机、携带信息和消费方在 §3.5 事件表中完整列出；仓储操作（§3.7）闭合了领域服务对聚合根的读写路径；外部系统交互通过端口契约（Decision 14/17/19/21）闭合。

**[通过]** 行为契约完整。§四 覆盖 12+ 个场景（含新增的场景 4b 手动救援、场景 8 分心检出、场景 10 报告生成与状态同步等），每个领域服务均有场景覆盖，带性能约束者（0.5s、15s、≥1Hz/≤2s）有可验证契约。

**[通过]** 模块间依赖方向合理。`domain.model` → `domain.event` → 各领域服务模块的依赖方向单向不循环；跨模块协作通过领域事件解耦，不存在编译期循环依赖。

### 5. 设计质量

**[通过]** 职责划分遵循单一职责原则。RiskDeterminationService 门面与子判定服务（DS-02/03/04）的委托关系清晰，AlertPersistenceService 将告警实体创建与判定逻辑解耦，DriverScoreUpdateService 单一负责评分写回——各服务职责聚焦。

**[通过]** 抽象层次恰当。聚合根（5 个）、实体（3 个含预留 E-02）、值对象（19 个）、领域服务（19 个）的划分粒度适中，既未过度设计（如将每个判定条件都独立建模），也未设计不足（如将判定、干预、通知混在一个服务中）。

**[通过]** 设计便于后续详细设计和实现。仓储接口（§3.7）、事件总线（§3.6）、外部端口（Decision 14/17/19/20/21）均有形式化方法契约，各领域服务（DS-02/03/04/05/07/13/14/17）本轮已补充完整方法签名（a_v9 修订），编码阶段可直接基于这些契约展开实现。

**[通过]** 设计便于单元测试。领域服务对外保持无状态或通过 `DetectionWindow`/`L3DurationTracker` 等值对象作为显式输入输出（纯函数语义），仓储和端口均可通过 mock 实现隔离测试；边缘侧单线程模型进一步降低并发测试的复杂性。

**[轻微]** **`OverrideSignal` 和 `DriverComprehensiveScore` 类型未在设计中正式定义**。DS-07 接口契约 `handleOverride(overrideSignal: OverrideSignal): InterventionResult` 使用 `OverrideSignal` 作为形式化参数类型——该类型在 DS-07 协作关系中简短提及"OverrideSignal 携带操作类型和时间戳"但无完整的概念维度定义（如操作类型是否以 enum 穷举、是否含加速度幅值）。DriverRepository §3.7.2 的 `updateScore` 方法使用 `DriverComprehensiveScore` 参数类型，但该类型与 VO-05 TripScore 的关系（是否共用同一值对象类型还是独立类型）未予说明。

**[轻微]** **§3.6 事件总线契约中的 `Type<E>` 类型标记能力未在仓颉语言层面确认**。`registerSyncHandler<E: DomainEvent>(eventType: Type<E>, handler: (E) -> Unit)` 和 `registerAsyncHandler<E: DomainEvent>(eventType: Type<E>, handler: (E) -> Unit)` 依赖 `Type<E>` 表达运行时类型信息。若仓颉不原生支持运行时类型反射/类型标记，事件总线的消费方路由需要替代方案（如基于字符串事件类型名注册）。虽不阻塞设计层面（可用字符串替代），但应在设计决策中明确依托的语言能力。

## 修改要求（REJECTED 时存在）

- **问题**：`DetectionWindow` 值对象未在 §3.3 值对象目录中正式定义
- **原因**：该类型出现在 DS-05 的形式化方法签名（`evaluateLifeDetection`）和 §6.2 EdgeSessionContext 的持有内容中，属于领域层核心值对象，缺少正式定义将阻塞编码阶段对该类型的实现
- **建议方向**：在 §3.3 值对象目录中新增 VO-XX DetectionWindow 条目，仿照现有值对象的三栏格式（角色与职责/类型形态为 `struct`/协作关系），补充其概念维度（剩余时长、起始时刻、累计微动观测次数、容差阈值）和协作关系（由 EdgeSessionContext 持有、作为 DS-05 `evaluateLifeDetection` 的输入输出参数）

- **问题**：`DomainEvent` 类型/接口在设计文档中未定义
- **原因**：§3.6 事件总线契约的全部四个方法均依赖 `DomainEvent` 作为泛型约束（`E: DomainEvent`），该类型是所有领域事件的公共抽象，未定义将直接阻塞事件总线接口的实现
- **建议方向**：在 §3.5 领域事件表之前或 §3.6 事件总线契约之前，定义 `DomainEvent` 的形态——建议以仓颉 `interface`（marker interface）声明，并注明是否包含公共字段（如 `eventId: String`、`occurredAt: Timestamp`、`aggregateId: String`）供事件总线提取元数据；各具体事件类型（RiskDeterminedEvent 等）的 struct 定义在实现时实现该 interface

- **问题**：`OverrideSignal` 和 `DriverComprehensiveScore` 类型未正式定义
- **原因**：`OverrideSignal` 出现在 DS-07 的形式化方法签名中，仅以自然语言在其协作关系中简短提及；`DriverComprehensiveScore` 出现在 DriverRepository 的形式化方法签名中，未定义其与 TripScore 的关系——两者均为形式化接口中的参数类型，缺少定义会造成编码阶段的类型设计歧义
- **建议方向**：(a) 在 §3.3 值对象目录中分别新增 VO-XX OverrideSignal（`struct`，含操作类型 enum——TURNING/BRAKING/ACCELERATING——和时间戳）和 VO-XX DriverComprehensiveScore（`struct`，范围 [0,100]，与 TripScore 共用同一数值约束但作为独立值对象类型以区分语义）；(b) 或明确声明 OverrideSignal 与 DriverComprehensiveScore 为简单别名类型，由编码阶段按需要定义

- **问题**：事件总线契约中的 `Type<E>` 运行时类型标记能力未确认
- **原因**：`registerSyncHandler` 和 `registerAsyncHandler` 的 `eventType: Type<E>` 参数依赖运行时类型反射能力，若仓颉不原生支持则事件总线实现需要替代方案
- **建议方向**：(a) 确认仓颉语言支持运行时类型反射后再沿用 `Type<E>` 设计；或 (b) 在 §3.6 契约中采用字符串事件类型名注册（如 `registerSyncHandler(eventTypeName: String, handler: (DomainEvent) -> Unit)`）作为语言无关的替代方案，并注明"实现阶段若语言支持类型反射则可优化为类型安全注册"
