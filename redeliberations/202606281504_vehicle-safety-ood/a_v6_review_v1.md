# OOD 设计方案审查报告（v6）

## 审查结果

**APPROVED**

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计方案中类型形态选择合理：聚合根（AR-01~05）与实体（E-01、E-03）使用 `class` 承载引用语义与独立标识；值对象（VO-03~05、VO-07~13、VO-15~19）统一采用 `struct` 以获取值相等语义；有限分类体系（VO-01 RiskLevel、VO-02 AlertType、VO-06 SensorStatus、VO-14 AccountRole、VO-12 内 InterventionInstructionType、VO-15 内 StatusColor、VO-19 内 UpgradeStage、决策 19 内 SessionType）使用 `enum` 提供编译期穷尽检查。继承/实现关系遵循单继承、多接口实现的约束（各领域端口和子判定服务均以 `interface` 声明契约）。跨聚合引用统一采用标识引用（决策 8），标识以 `struct` 承载不可变性——均与仓颉类型系统能力匹配。

**[通过]** 泛型使用方式合理：`Array<T>`、`Set<RiskLevel>`、`Map<AlertType, RiskLevel>`、`Result<T, E>`、`Option<T>` 均为标准化泛型构造，无协变/逆变等复杂变形。

**[轻微]** §3.6 领域事件总线契约中 `func publish<E: DomainEvent>(event: E)` 使用了泛型上界约束 `E: DomainEvent`，且 `registerSyncHandler` 使用了 `Type<E>`（需 reified 泛型支持）。方案未正式定义 `DomainEvent` 的类型形态（`interface` 还是 `abstract class`）。这些是仓颉泛型系统的高级特性，具体语法与支持程度需在详细设计阶段对照仓颉语言规范确认。当前不阻塞架构设计通过。

**[轻微]** §3.6 与第六章中多处使用 `Unit` 类型（如 `Result<Unit, EventPublishError>`），设计未确认仓颉标准库是否提供该类型或等价类型。不影响架构可行性，属实现阶段需确认的细节。

### 2. 标准库与生态覆盖

**[通过]** 设计中所需的集合类型（Array、Set、Map）、数值类型、字符串等基础能力均在仓颉标准库覆盖范围内。模块组织方式（§2.2 依赖方向）符合 cjpm 项目组织规范。

**[通过]** §3.6 明确声明仓颉标准库未原生提供领域事件发布/订阅机制，设计方案以四项契约自建事件总线，并给出 outbox 表结构约定与 at-least-once 投递保证，假设合理。

**[通过]** 设计方案使用的 `Result<T, E>`、`Option<T>` 错误处理模式与仓颉语言的错误处理能力匹配（§五整体原则）。外部系统端口（NotificationPort、RescueReportPort、MediaSessionPort、OTADeliveryPort）均以 `interface` 声明于领域层，由基础设施层实现并注入，符合依赖反转原则。基础设施层所需的网络、MQTT、SparkRTC、IoTDA 等属于部署环境提供的平台能力，领域层不直接依赖，假设合理。

### 3. 语言特性可行性

**[通过]** 错误处理策略（§五）统一采用 `Option<T>` 表达"可能无结果"语义、`Result<T, E>` 表达"可能成功也可能失败"语义、异常仅用于调用方违反前置条件的场景，与仓颉语言错误处理能力匹配。错误分类（A/B/C 三级）边界清晰。

**[通过]** 并发设计（第六章）区分边缘侧单线程串行处理与云端侧乐观并发控制两层策略。边缘侧会话上下文（EdgeSessionContext）无并发竞争，聚合根级别的乐观锁解决云端并发写冲突，与仓颉并发模型兼容。

**[通过]** 模块/包结构（§2.2）以 `domain.model` 为最底层、`domain.event` 仅依赖标识类型、各领域服务模块间禁止直接调用仅通过领域事件协作，依赖方向单向无循环，符合 cjpm 项目组织方式。

**[通过]** 资源管理方面，RoadRageVoiceRecord（AR-05）的到期自动清除、OTAUpgradeStatus 的状态机生命周期均在领域模型中声明管理策略，具体资源操作委托给基础设施层，领域层保持无状态。

**[轻微]** §6.2 将 EdgeSessionContext 明确定义为"基础设施层的运行时容器，非领域模型"，但 DS-01（RiskDeterminationService）、DS-05（LifeDetectionService）、DS-16（DriverStatusBroadcastService）均直接与其交互——当前交互路径中的会话状态以输入参数方式传入服务、由服务返回更新后的状态，这在一定程度上保持了服务的纯函数语义。但若在实现时 EdgeSessionContext 以具体类型而非接口形式被领域服务引用，将违反 DDD 依赖反转原则。建议在详细设计阶段为 EdgeSessionContext 定义领域层接口，使领域服务仅依赖该接口。

### 4. 设计一致性

**[通过]** 各抽象的职责描述清晰：19 个值对象、5 个聚合根、2 个实体（非聚合根）、19 个领域服务、16 个领域事件，均有明确的角色与职责说明。行为契约覆盖全部领域服务（§四，场景 1~12），关键场景带有显式性能约束（分心 0.5s、报告 15s、状态同步 ≤2s）供验证。

**[通过]** 协作关系形成闭环：
- 判定→持久化→通知链路闭合：DS-01/DS-05/DS-06 产出判定事件 → DS-19（AlertPersistenceService）创建 SafetyAlertEvent → AlertTriggeredEvent 驱动通知
- 风险解除链路闭合：RiskResolvedEvent → InterventionService 停止干预/恢复环境、PrivacyProtectionService 封闭存证、PermissionService 常规撤销
- 评分→更新链路闭合：DS-09 计算 TripScore → TripScoredEvent + DriverScoreUpdatedEvent → DS-18 写回 Driver 聚合
- 家属权限授予−撤销闭环：授予（60s L3 或自动激活）→ 常规撤销（L3 下降）→ 临时撤销（物理遮挡）→ 遮挡恢复后的权限判定（场景 7 步骤 5）

**[通过]** 模块间依赖方向单向且合理：各领域服务模块仅依赖 `domain.model` 和 `domain.event`，模块间无直接调用，通过领域事件解耦。

**[通过]** 风险判定事件的 AlertType 取值集合两两不相交（VO-02 取值来源边界说明），从结构上杜绝重复判定事件。二维干预映射（AlertType × RiskLevel）与各场景行为契约严格一致（决策 15）。

**[轻微]** LifeDetectedEvent（DS-05 产出）携带 Vehicle 标识而非 Trip 标识。DS-19 AlertPersistenceService 在消费 LifeDetectedEvent 创建 SafetyAlertEvent 时，需要将该告警关联到一个 Trip 聚合。当车辆处于熄火落锁状态（无活跃行程）时，存在无 Trip 可关联的边界情况。当前设计未说明此场景下的处理策略（创建独立告警记录/关联最近一次 Trip/丢弃），建议在详细设计中补充。

### 5. 设计质量

**[通过]** 职责划分遵循单一职责原则：判定逻辑（DS-01~06）、干预生成（DS-07）、权限管理（DS-08）、评分（DS-09）、分析（DS-10）、报告（DS-11）、救援（DS-12）、隐私（DS-13）、自检（DS-14）、OTA（DS-15）、状态广播（DS-16）、行为追踪（DS-17）、评分更新（DS-18）、告警持久化（DS-19）各司其职，无职责重叠。

**[通过]** 抽象层次恰当：既不过度设计（如二次身份验证未建模为独立值对象，而是 DS-08 的门控约束，决策 18；DriverStatusSnapshot 高频遥测不建模为领域事件，避免污染事件总线），也不设计不足（如 SensorReading 统一感知数据抽象、InterventionInstruction 穷举指令类型、各外部系统端口契约均已补齐）。

**[通过]** 设计便于单元测试：领域服务保持无状态纯函数语义（输入→输出），外部依赖均以 `interface` 端口声明（可 mock），聚合根通过仓储操作（可注入测试仓储）。检测窗口（DetectionWindow）、L3 时长追踪器（L3DurationTracker）等临时状态以值对象封装、随输入参数传入，服务不持有可变状态。

**[通过]** 架构级设计留有合理的实现自由度：具体字段、方法签名、持久化细节、缓存策略、网关配置等留给详细设计阶段；同时在关键接口（事件总线、外部端口）给出了方法签名级契约，为下游并行开发提供明确边界。

## 修改要求（REJECTED 时存在）

无。本轮审查未发现严重或一般级别问题，设计方案可进入下一阶段。
