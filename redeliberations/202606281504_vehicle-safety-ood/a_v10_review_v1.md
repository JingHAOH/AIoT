# OOD 设计方案审查报告（v10）

## 审查结果

APPROVED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计方案中使用的类型形态均在仓颉类型系统能力范围内：
- 聚合根（AR-01~AR-05）使用 `class`——仓颉 `class` 为引用类型，适合需要唯一标识、有生命周期的聚合根。五个聚合根之间不存在继承关系，不触及单继承约束；各聚合根通过标识引用（`struct` 类型的 ID）实现跨聚合关联，符合 DDD 聚合设计原则。
- 实体（E-01、E-03）使用 `class`——仓颉 `class` 支持实体所需的标识相等性语义（需重写 equals/hashCode 或在语言层面通过标识字段判别）。
- 值对象（VO-01~VO-23）使用 `struct` 或 `enum`——仓颉 `struct` 为值类型，天然提供值相等语义，是值对象的首选类型形态。`enum` 用于 RiskLevel、AlertType、SensorStatus、AccountRole、InterventionInstructionType、UpgradeStage、OverrideType、AggregateType、SessionType、StatusColor 等有限穷举集合，编译期可进行穷尽性检查。
- DomainEvent 以 `interface` 定义公共抽象——仓颉 `interface` 可声明抽象属性，各具体事件以 `struct` 定义并 `implements` 该接口，`struct` 保证事件载荷的不可变性。
- 泛型使用方式（`Result<T, E>`、`Option<T>`、`Array<T>`、`Map<K,V>`、`Set<T>`、`func publish<E: DomainEvent>(...)` 等）在仓颉泛型系统能力范围内——仓颉支持有上界约束的泛型参数。

**[轻微]** `AggregateId` struct 的 `aggregateType` 字段使用 `AggregateType` 枚举（TRIP / DRIVER / VEHICLE / SYSTEM_ACCOUNT / ROAD_RAGE_VOICE_RECORD），该枚举取值直接对应五个聚合根类型。当未来新增聚合根时，需同步扩展此枚举——但这属于正常的设计演进，不构成当前设计的缺陷。建议在 `AggregateType` 枚举的说明中标注"新增聚合根时需同步扩展"。

### 2. 标准库与生态覆盖

**[通过]** 设计所依赖的基础类型构造在仓颉标准库或可轻量自定义范围内：
- `Result<T, E>` 与 `Option<T>`：设计全文以这两种类型作为错误处理和可空语义的统一载体。仓颉若原生提供这两个类型则直接可用；若未原生提供，它们均为简单的和类型（sum type），可在项目内部以 `enum` 或 `sealed class` 轻量定义，不构成阻塞性依赖。
- `Array<T>`、`Map<K,V>`、`Set<T>`：基础集合类型，属标准库常规覆盖范围。
- `String`、`Bool`、`Int`：基础原生类型，必然可用。
- 领域事件总线（§3.6）：设计明确声明"仓颉标准库未原生提供领域事件发布/订阅机制，需自定义实现"，并给出四项接口契约。该判断合理——事件总线属架构模式而非标准库职责，自定义实现无语言能力障碍。
- `Timestamp` 与 UUID 生成：若标准库未直接提供对应类型，可分别以 `Int64`（Unix 毫秒时间戳）和自定义 `UUID` struct（封装 128 位值）替代，设计层面不依赖具体实现细节。
- cjpm 模块组织：`domain.model`、`domain.risk`、`domain.event` 等模块划分符合 cjpm 的子包（subpackage）组织方式，无循环依赖。

**[轻微]** `Result<T, E>` 与 `Option<T>` 是贯穿全部接口契约的核心类型。若仓颉标准库未原生提供，需在项目早期（基础设施搭建阶段）完成这两个类型的自定义实现并稳定其 API（`Result.ok()` / `Result.err()`、`Option.Some()` / `Option.None()`、`map` / `flatMap` 等组合子），否则所有依赖模块的编译会受阻。建议在项目启动阶段将此列为前置任务。

### 3. 语言特性可行性

**[通过]** 各语言特性维度的设计决策在仓颉中均可落地：
- **错误处理策略**（§五）：A 类判定失败以 `Option<T>` 表达"无结果"，B 类业务规则违反以 `Result<T, E>` 表达可恢复失败，C 类系统级故障向上抛异常。该三层策略在仓颉中均可实现——`Option`/`Result` 为和类型模式，异常机制为语言级特性。
- **并发设计**（§六）：边缘侧单线程串行处理（确定性实时响应），云端侧并发由基础设施层（数据库连接池、缓存、消息队列）承担，领域服务保持无状态。该设计未对仓颉的并发原语（协程/线程）提出特殊要求——边缘侧的单线程模型甚至回避了并发竞争，云端侧并发控制在基础设施层（Spring Boot）而非仓颉领域层。
- **Outbox 模式**（§3.6、§5.3、§6.2）：事件持久化与聚合根状态更新在同一事务中提交，post-transaction 阶段的异步投递由基础设施层 outbox 投递器负责。该模式为架构模式，不对仓颉语言特性提出额外要求。
- **资源管理**：设计中的资源边界清晰——聚合根通过仓储（`interface` 声明于领域层）管理持久化生命周期，RoadRageVoiceRecord 的边缘侧存储、VehicleStateBuffer 和 PhysiologicalDataBuffer 的 ring buffer 均归属基础设施层。领域层不直接管理操作系统资源（文件句柄、网络连接），资源泄漏风险由基础设施层承担。
- **模块/包结构**：`domain.model` → `domain.event` → 各 `domain.*` 服务模块的单向依赖链在 cjpm 项目中可自然表达（各子包在 `cjpm.toml` 中声明依赖关系）。

**[轻微]** §3.6 事件总线契约以 `eventTypeName: String` 作为事件路由键，设计已说明此选择是因"仓颉对运行时类型反射的支持程度需在实现阶段确认"。若实现阶段确认仓颉不支持运行时类型反射，该设计不受影响；若支持，可优化为类型安全注册。当前以 String 为路由键的设计是务实且安全的，不构成阻塞。

### 4. 设计一致性

**[通过]** 各抽象的职责描述清晰，协作关系形成闭环，行为契约完整：
- **模块依赖方向**（§2.1/§2.2）：`domain.model`（零依赖）→ `domain.event`（仅依赖 `domain.model`）→ 各服务模块（依赖 `domain.model` + `domain.event`）。服务模块之间禁止直接调用，跨模块协作用领域事件解耦。无循环依赖。
- **聚合根-仓储对照**（§3.1 末尾）：AR-01~AR-05 五个聚合根均有对应仓储（TripRepository、DriverRepository、VehicleRepository、SystemAccountRepository、RoadRageVoiceRecordRepository），且仓储接口契约已在 §3.7.1~§3.7.5 中完整定义。
- **领域服务覆盖率**：16 个领域服务（DS-01~DS-19）在 §四（关键行为契约）的 12 个场景中均有对应的交互路径覆盖，且有形式化方法签名的服务比例较 v9 显著提升——DS-06（新增 `determineDisability`）、DS-15（新增四个方法签名）补齐了 v9 的契约缺口。
- **v10 迭代修复的一致性验证**：迭代需求所列 8 个持续问题均已在本版中修复，修复措施与问题描述一一对应：
  - 问题 1 → §3.7.5 VehicleRepository 已定义 findById + save
  - 问题 2 → DS-06 已补充 `determineDisability(collisionSignal)` 方法签名
  - 问题 3 → DS-15 已补充 `initiateUpgrade` / `handleTransferProgress` / `handleVerificationResult` / `handleFirmwareFlashResult` 四个方法签名
  - 问题 4 → DS-19 已修正为"EmergencyActivatedEvent 同样发出 AlertTriggeredEvent"，闭合暗数据问题
  - 问题 5 → VO-23 RescueAuthorizationToken 已形式化，DS-12 已补充完整授权生命周期描述
  - 问题 6 → §3.5.0 aggregateId 已从 String 改为 AggregateId struct（value + aggregateType 枚举）
  - 问题 7 → DS-09 已补充周期评分边界语义（归属规则、加权公式、计算时机）
  - 问题 8 → §6.3 并发策略表已补充 OTA + 传感器自检乐观锁冲突处理条目

### 5. 设计质量

**[通过]** 设计整体遵循 SOLID 原则，抽象层次恰当，便于测试与后续演进：
- **单一职责**：各领域服务职责明确无重叠——DS-01（流式融合门面）不承担活体/碰撞判定、DS-19（告警持久化）不承担判定逻辑、DS-18（评分写回）仅为轻量仓储操作桥梁。值对象各司其职（VO-16 DrivingBehaviorCounters 专注计数、VO-17 L3DurationTracker 专注计时、VO-19 OTAUpgradeStatus 专注升级状态机）。
- **抽象层次**：设计为架构级 OOD，聚合根/实体/值对象/领域服务的划分粒度恰当——未过度设计（如二次身份验证凭据未升格为值对象，决策 18 明确其为事务级技术细节），也未设计不足（v10 补齐了此前缺失的 VehicleRepository 契约、DS-06/DS-15 方法签名、RescueAuthorizationToken 建模等）。
- **可测试性**：领域服务均设计为无状态纯函数（DS-01/DS-05 的会话级临时状态归属 EdgeSessionContext 或值对象参数传入，DS-08 的 L3 计时归 VO-17 持久化于 Trip 聚合），仓储接口以 `interface` 声明可供 mock，事件总线以 `interface` 契约隔离具体实现。各接口契约的输入/输出类型明确（`Result<T, E>` / `Option<T>`），便于编写单元测试的 given-when-then 断言。
- **扩展性**：决策 2 的门面+委托模式允许子判定服务独立演进（如边缘侧用规则判定、云端用 AI 模型），各端口（VehicleStateBuffer、PhysiologicalDataBuffer、NotificationPort、RescueReportPort、MediaSessionPort、OTADeliveryPort、DrivingBehaviorTrackingPort、CameraOcclusionDetectionPort）以 `interface` 声明支持实现替换。

**[轻微]** VO-23 RescueAuthorizationToken 的生命周期中"过期凭证不自动清理，由审计模块定期归档"——审计归档的具体调度策略（定时任务 vs 事件驱动）未在设计层面明确，但此属基础设施层职责细节，不阻塞架构级设计通过。

## 修改要求

无。本轮审查未发现严重或一般问题，设计在仓颉语言可行性维度上通过验证。
