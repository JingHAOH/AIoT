# OOD 设计方案审查报告（v2）

## 审查结果

REJECTED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计中各抽象的类型形态选择与仓颉类型系统能力匹配：聚合根和实体选用 `class`（引用类型、支持单继承和多接口实现），值对象统一选用 `struct`（值类型、值相等语义），有限分类体系选用 `enum`（编译期穷尽检查），子判定服务接口选用 `interface`（多态替换）。决策 12 明确了值对象的首选类型形态为 `struct`，并说明若因约束使用 `class` 需重写相等性方法，这一设计约束是合理的。

**[通过]** 抽象间的继承与实现关系在仓颉约束范围内。聚合根之间通过标识（`struct` 类型的 ID）引用而非对象引用（决策 8），符合仓颉的单继承约束和聚合独立加载原则。

**[通过]** 泛型使用方式在仓颉泛型系统能力范围内。设计中涉及的 `Option<T>`、`Result<T, E>` 及泛型集合（PhysiologicalSnapshot 集合）均为仓颉类型系统标准能力。

**[通过]** 协作关系中描述的类型交互模式（领域事件驱动、门面委托、值对象传递）均可在仓颉中实现。

### 2. 标准库与生态覆盖

**[通过]** `Option<T>` 和 `Result<T, E>` 在仓颉标准库中有原生支持，可直接用于 §5 错误处理策略中定义的 B 类业务规则违反和判定失败场景。

**[通过]** 集合类型（PhysiologicalSnapshot 的不可变集合存储、通用集合操作）在仓颉标准库覆盖范围内。

**[通过]** 设计中涉及的加解密能力（路怒语音存证加密）、文件 I/O（OTA 升级包、报告导出）、时间戳处理等均在仓颉标准库或常用库覆盖范围内。

**[通过]** 设计在 §6.2 已明确识别仓颉标准库未原生提供领域事件发布/订阅机制，并给出了两类消费链路的实现策略（边缘侧进程内同步回调 + 云端侧 outbox+消息队列异步投递），属于恰当的自定义实现规划。

### 3. 语言特性可行性

**[通过]** 错误处理策略（§5）与仓颉的错误处理能力匹配：`Option<T>` 表达"可能无结果"，`Result<T, E>` 表达"可能失败"，异常用于系统级故障，分类清晰且策略合理。

**[通过]** 并发设计（§6）与仓颉的并发模型兼容。边缘侧单线程串行处理保证确定性实时响应，云端侧使用乐观锁和 outbox 模式处理并发，均为仓颉并发模型下可行的设计。

**[通过]** 模块/包结构设计（domain.model / domain.risk / domain.intervention 等）符合 cjpm 项目组织方式，依赖方向单向（底层 domain.model → domain.event → 各领域服务模块），无循环依赖。

**[通过]** 资源管理方案：设计中领域服务均为无状态（除 DS-05 标注例外），聚合根的持久化通过乐观锁和事务边界管理，在仓颉资源管理模式内可行。

### 4. 设计一致性

**[通过]** 各抽象的职责描述清晰。Trip 作为核心聚合根汇聚感知数据与告警事件，Driver 管理驾驶员身份与健康档案，Vehicle 管理设备状态，SystemAccount 管理外部角色与权限，职责边界明确。

**[通过]** §5.2 表格将错误处理策略与具体业务场景一一对应（感知缺失→`Option<None>`、权限拒绝→`Result<PermissionDenied>` 等），映射清晰无歧义。

**[通过]** 模块间依赖方向合理，各领域服务模块之间禁止直接调用，跨模块协作统一通过领域事件完成（§2.2），不存在循环依赖。

**[一般]** **LifeDetectionService 与 EmergencyResponseService 绕过 RiskDeterminationService 门面直接产出领域事件，与决策 2 约束矛盾。**

- DS-01 协作描述中声明"内部委托：将各通道数据分派给 FatigueDeterminationService、DistractionDetectionService、RoadRageDeterminationService、LifeDetectionService 和 EmergencyResponseService 各自判定。汇总各子判定服务的判定结果，产出 RiskDeterminedEvent"，且决策 2 明确"子判定服务不得直接调用其他模块的领域服务或直接产出领域事件"。
- 但 DS-05 输出直接声明为 `LifeDetectedEvent`——该事件未被 RiskDeterminationService 汇总为中转，且场景 2（BR-02）的行为契约全程未提及 RiskDeterminationService 参与，从触发（熄火落锁→毫米波扫描→60s 窗口判定）到输出（LifeDetectedEvent）均由 LifeDetectionService 独立完成。
- 同理，DS-06 输出直接声明为 `EmergencyActivatedEvent`，场景 4（BR-06）从未经过 RiskDeterminationService，且 EmergencyActivatedEvent 也不在 RiskDeterminationService 的汇总产出之列。
- 这构成设计内部矛盾：DS-01 将两者纳入委托子服务列表，但两者的触发时机（熄火事件、碰撞事件）和产出事件类型均不同于 FatigueDeterminationService 等"先回传结果再统一产出 RiskDeterminedEvent"的子服务模式。两者实质上是以独立领域服务而非门面子服务的形态运作，事件产出路径与决策 2 的约束不一致。

**[轻微]** DS-05 LifeDetectionService 设计约束段（§3.4）承认该服务"存在'有状态'特性（维护判定窗口计时）"，并建议后续将窗口计时状态封装为独立的 `DetectionWindow` 值对象使服务回归纯函数形态。当前描述中判定窗口的倒计时状态管理边界较模糊——是服务内部的 mutable 字段、还是作为输入参数的一部分——缺少明确交代，可能影响后续详细设计中的可测试性判断。

**[轻微]** 领域事件表（§3.5）中 `LifeDetectedEvent` 的触发时机描述为"LifeDetectionService 在 60s 窗口内判定成立"，与其在 DS-01 门面汇总逻辑中的位置形成模糊地带：若该事件由 DS-05 直接产出，则与 DS-01 汇总 RiskDeterminedEvent（含 AlertType=LIFE_DETECTION）的关系不清晰——是否会出现同一判定结果产出两条领域事件（LifeDetectedEvent + RiskDeterminedEvent）的情况。

**[轻微]** VehicleStateSnapshot（VO-09）的生产者不明确。BR-06 要求事故触发时携带"事故前 30 秒车辆状态快照"，意味着系统需在碰撞前持续缓存车辆状态数据。当前设计仅定义该值对象的结构，未说明由哪个领域服务或基础设施层负责持续采集、缓存并供事故时刻回取。

### 5. 设计质量

**[通过]** 职责划分遵循单一职责原则。感知、判定、干预、通知、隐私、车队管理、应急救援、OTA、自检各自归属独立模块，每个领域服务职责聚焦。

**[通过]** 抽象层次恰当。RiskDeterminationService 的门面+委托模式（决策 2）将融合判定入口与具体判定逻辑分层，既避免了巨型服务，又保持了调用方的统一接口。

**[通过]** SafetyAlertEvent 采用 CQRS 读模型投影策略（§3.2 E-01 设计约束），在聚合一致性边界与独立查询需求之间取得合理平衡，属于设计级别的恰当抽象。

**[通过]** 聚合间一律使用标识引用（决策 8），保持事务边界独立，为未来的微服务拆分预留清晰边界。

**[通过]** 设计便于单元测试：领域服务均为无状态（除 DS-05 标注特例），子判定服务的接口抽象为 `interface` 支持 mock 替换，值对象不可变消除了测试中的副作用风险，领域事件驱动的模块解耦使各模块可独立测试。

## 修改要求（REJECTED 时存在）

### 问题 1：LifeDetectionService 和 EmergencyResponseService 的事件产出路径与决策 2 门面约束矛盾

- **问题**：DS-05（LifeDetectionService）直接产出 LifeDetectedEvent，DS-06（EmergencyResponseService）直接产出 EmergencyActivatedEvent，均不经过 RiskDeterminationService 门面汇总。而 DS-01 的协作描述将两者列为委托子服务，"子判定服务不直接产出领域事件"是决策 2 的核心约束。场景 2 和场景 4 的行为契约也未体现门面参与。

- **原因**：BR-02（熄火落锁后 60s 判定窗口）和 BR-06（碰撞冲击+失能判定）的触发时机与 BR-01/BR-03/分心检出的"持续感知管道输入"模式本质不同——前者是事件触发型判定，后者是流式数据驱动判定。强行将两者纳入同一门面模式是架构层面的不适配。如果维持当前 DS-01 将两者列为子服务、同时两者独立产出自身事件的双重描述，会导致实现阶段产生歧义（是否走门面？是否产生重复事件？）。

- **建议方向**：两种可选路径——

  **路径 A（推荐）**：将 LifeDetectionService 和 EmergencyResponseService 从 RiskDeterminationService 的门面委托列表中移除，明确其作为**独立领域服务**的地位——它们有独立的触发条件、独立的判定逻辑和独立的领域事件产出，不走 RiskDeterminationService 门面。同步更新 DS-01 协作描述中的委托列表（仅保留 FatigueDeterminationService、DistractionDetectionService、RoadRageDeterminationService），更新决策 2 中的子服务枚举范围。

  **路径 B**：保留门面委托关系，但 LifeDetectionService 和 EmergencyResponseService 的判定结果一律回传给 RiskDeterminationService，由门面统一产出 RiskDeterminedEvent（扩展 AlertType 枚举包含 LIFE_DETECTION 和 EMERGENCY 类型），废除 LifeDetectedEvent 和 EmergencyActivatedEvent，各消费方统一订阅 RiskDeterminedEvent 根据 AlertType 分发。此路径需改造场景 2/4 的行为契约以体现门面参与，并处理判定窗口计时由谁维护的归属问题。

