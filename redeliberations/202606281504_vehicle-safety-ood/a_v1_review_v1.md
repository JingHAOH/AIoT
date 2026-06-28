# OOD 设计方案审查报告（v1）

## 审查结果

REJECTED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计中使用的类型形态（class / enum / interface / struct）均在仓颉类型系统能力范围内。聚合根和实体使用 class 正确地表达了唯一标识和独立生命周期的语义；RiskLevel、AlertType、SensorStatus 使用 enum 准确反映了有限稳定的分类体系；聚合间引用使用 struct 契合栈分配、不可变的轻量标识需求。

**[轻微]** 值对象（PhysiologicalSnapshot、GeoLocation、TripScore、Permission、OTAVersion、VehicleStateSnapshot、TimeRange）统一标注为 class，但值对象核心语义是"由属性值定义相等性"。仓颉中 class 为引用类型（默认引用相等），struct 为值类型（值相等）。建议设计层面明确值对象在仓颉中的首选类型形态（struct），或注明 class 需重写相等性的设计约束，避免后续实现时产生混淆。

### 2. 标准库与生态覆盖

**[通过]** 设计中依赖的核心能力——泛型集合（PhysiologicalSnapshot 集合、告警列表）、`Option<T>` / `Result<T, E>` 错误处理类型、乐观并发控制的基础设施——均在仓颉标准库覆盖范围内。cjpm 包管理可支持 `domain.model`、`domain.risk` 等层级化模块组织。

**[轻微]** 领域事件发布/订阅机制未在仓颉标准库中原生提供，需自定义实现或引入第三方库。建议在设计层面简要标注事件总线的实现策略（如进程内同步回调 vs 消息队列异步投递），尤其是边缘侧同步消费与云端异步消费的机制差异。

### 3. 语言特性可行性

**[通过]** 错误处理策略使用 `Option<T>` 表达"可能无结果"、`Result<T, E>` 表达"可能因业务原因失败"，与仓颉标准库的错误处理模型一致。边缘侧单线程流水线模型与云端乐观并发控制的线程模型均在仓颉并发能力范围内。资源管理（边缘侧语音存证文件 I/O）可行。

**[轻微]** §5.1 将"领域事件发布失败"归类为 C 类（向上抛异常），与 §5.3 的"事件消费方自己负责错误处理和重试"存在张力——发布失败时发布方事务是否回滚、outbox 模式如何兜底，建议在设计层面给出更清晰的一致性边界。此外，§6.2 提到 outbox 模式确保"发布与聚合根状态更新在同一事务中"，但 §5.1 又说 C 类错误"上抛给基础设施层统一处理"，两处的异常传播路径需要统一。

### 4. 设计一致性

**[一般]** **RoadRageDeterminationService 跨模块通信违反依赖原则**。§2.2 明确约定"各领域服务模块之间禁止直接调用，跨模块协作统一通过领域事件完成"，但 DS-04（`domain.risk`）的协作描述中直接向 PrivacyProtectionService（`domain.privacy`）和 InterventionService（`domain.intervention`）发送指令。正确的解耦路径应为：RoadRageDeterminationService 将判定结果返回 RiskDeterminationService 门面 → 门面产出 RiskDeterminedEvent（AlertType=ROAD_RAGE）→ PrivacyProtectionService 和 InterventionService 各自消费该事件触发语音存证录制与环境调节。当前写法将导致实现者错误地在 `domain.risk` 中直接依赖 `domain.privacy` 和 `domain.intervention`，破坏模块边界。

**[通过]** 其他模块间的依赖方向符合单向约束（领域服务依赖 domain.model + domain.event，domain.event 仅依赖 domain.model）。BR-01~BR-08 及分心检出规则均有对应的领域服务覆盖。需求第五节全部核心业务对象均已建模（Driver、Vehicle、Trip、PhysiologicalSnapshot、SafetyAlertEvent、RoadRageVoiceRecord、SystemAccount、DriverHealthProfile）。

**[轻微]** 领域事件表中缺少"判定→路怒语音存证/环境调节"的显式事件条目。当前 RiskDeterminedEvent 在描述上可承载 AlertType=ROAD_RAGE 来触发这些行为，但事件表中未提及其可被 PrivacyProtectionService 消费以触发语音存证录制，建议补充消费方条目以形成完整的事件-消费闭环。

**[轻微]** SafetyAlertEvent 被描述为"Trip 聚合内的实体"（Decision 3），但又强调其"可在 Trip 之外被独立查询"且"在数据库中可能有自己的存储表"。聚合内实体与独立查询需求之间的张力（走聚合根加载 vs 独立读模型）建议在设计层面给出策略倾向（如 CQRS 读模型、或明确实体仍通过聚合根访问但提供只读投影）。

### 5. 设计质量

**[通过]** 各领域服务职责划分清晰：RiskDeterminationService 作为门面统一路由感知数据、内部委托子判定服务，符合单一职责与接口隔离原则。模块间通过领域事件解耦的整体策略（除上述 DS-04 违规外）使各业务域可独立演进。聚合根、实体、值对象的职责边界明确，聚合间一律使用标识引用，为后续实现和测试提供了良好的隔离基础。

**[轻微]** LifeDetectionService（DS-05）声明为"有状态"（判定窗口计时），与领域服务通常无状态的惯例有偏差。虽然设计明确指出"生命周期短于一次判定"且边缘侧单线程运行可规避并发问题，但在后续详细设计阶段应考虑将计时状态封装为独立的值对象或交由聚合根管理，以保持领域服务纯函数的可测试性。

**[轻微]** 隐私保护模块（`domain.privacy`）仅依赖 `domain.model` 而不依赖 `domain.event`（见模块表），但 DS-13 PrivacyProtectionService 需要感知路怒判定成立才能触发语音存证录制。若按事件解耦路径（见维度4），该模块需消费 RiskDeterminedEvent 或专用事件，则需增加对 `domain.event` 的依赖。建议同步修正模块依赖声明。

**[轻微]** 设计中提到的"感知数据"（DMS 视觉特征、语音情绪特征、毫米波雷达信号等）以自然语言分散描述于各服务的"输入"中，缺少统一的感知数据抽象。虽属架构级设计可暂不展开，但建议在领域事件或领域模型层面预留 SensorReading 类抽象，以便各判定服务有统一的输入契约。

## 修改要求（REJECTED 时存在）

### 一般问题 #1：RoadRageDeterminationService 跨模块通信违反依赖原则

- **问题**：DS-04 RoadRageDeterminationService 的协作描述中直接向 PrivacyProtectionService（`domain.privacy`）和 InterventionService（`domain.intervention`）发送指令，违反了 §2.2 中"各领域服务模块之间禁止直接调用，跨模块协作统一通过领域事件完成"的核心设计原则。

- **原因**：该写法会导致 `domain.risk` 模块对 `domain.privacy` 和 `domain.intervention` 产生编译期直接依赖，破坏模块间的松耦合设计意图。后续若隐私保护策略变更或干预逻辑调整，将迫使风险判定模块联动修改。

- **建议方向**：统一通过 RiskDeterminationService 门面 + 领域事件完成解耦——
  1. RoadRageDeterminationService 仅返回判定结果（含 AlertType=ROAD_RAGE 及判定依据快照）给 RiskDeterminationService。
  2. RiskDeterminationService 产出 RiskDeterminedEvent（携带 AlertType），由 PrivacyProtectionService 和 InterventionService 各自订阅消费。
  3. 在领域事件表（§3.5）中补全 RiskDeterminedEvent 的消费方列表，或在事件描述中明确标注"路怒类型触发语音存证录制"、"路怒类型触发环境调节指令"等消费语义。
  4. 若确需在路怒判定成立后立即启动存证录制（时延原因），可在边缘侧的同步事件消费链路中保证（Decision 10 已为此类场景留出同步消费通道），仍无需跨模块直接调用。
