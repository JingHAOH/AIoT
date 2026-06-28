# 车载安全监测系统 领域层 OOD 设计方案（a_v2 / v1）

> 本文档为「智能物联——基于多传感器融合的车载安全监测系统」的架构级 OOD 设计方案，采用 DDD（领域驱动设计）分层思想，聚焦领域层的实体、值对象、聚合根、领域服务和领域事件设计。设计目标语言为仓颉（Cangjie），方案在仓颉类型系统能力范围内做抽象，不涉及具体代码实现。本版（a_v2_v1）在前一轮产出（a_v1_design_v3）基础上，修复了诊断与质询确认的 2 个严重、5 个一般、5 个轻微共 12 个问题，详见文末「修订说明（a_v2_v1）」。

---

## 一、概述

### 设计目标

本系统领域层的核心使命是：**接收多维传感器感知输入，基于业务规则执行融合风险判定，按风险等级驱动分级干预与通知，同时支撑远程监护、车队管理及应急救援等协作场景**。

设计遵循以下目标：

- **安全优先**：边缘侧本地兜底的核心判定链路不可降级，断网时安全告警链路仍成立。
- **职责内聚**：感知、判定、干预、通知、管理各自归属清晰的领域模块，模块间通过领域事件解耦。
- **隐私内建**：原始敏感数据（人脸图像、语音存证）的存储边界与脱敏要求直接体现在领域对象的设计约束中。
- **可验收导向**：本期设计聚焦软件判定逻辑与控制指令的正确性，不依赖真实硬件/算法，所有领域逻辑在模拟数据下可复现验证。

### 核心抽象层次

系统在领域层分为四个抽象层次：

1. **聚合根** — 事务一致性边界，统领一组关联实体与值对象，外部只能通过聚合根访问其内部对象。
2. **实体** — 有唯一标识、有生命周期的领域对象，其相等性由标识决定。
3. **值对象** — 无独立标识、由属性值定义相等性的不可变领域概念。
4. **领域服务** — 无状态的操作抽象，封装不属于任何单一聚合根的业务规则与协调逻辑。

### 风险等级与干预映射的总览约定

为消除前一轮「通用 RiskLevel→干预映射」与「具体场景行为契约」之间的矛盾，本版确立如下总览约定（细节见 VO-01、DS-07、决策 15）：

- 干预策略**不是**由单一 RiskLevel 决定，而是由 **AlertType × RiskLevel 的二维组合**决定。同一 RiskLevel 在不同 AlertType 下对应不同的干预指令集合（如疲劳 L2 = 氛围灯变橙、路怒 L2 = 环境调节、分心 L2 = 告警）。
- **RiskLevel.L1 为预留等级**：本期 BR-01~BR-08 无任一规则映射到 L1，L1 无触发路径，保留以备未来扩展（见 VO-01、决策 15）。
- 风险**解除**不是 RiskLevel 的取值，而是独立的领域事件 **RiskResolvedEvent**（见 VO-02 说明、决策 16），用以表达「此前成立的某类风险已不再持续」这一状态转换。

---

## 二、模块划分

系统领域层按职责拆分为以下模块，模块间依赖方向为单向——上层模块可依赖下层模块，下层不可反向依赖上层。

### 2.1 模块一览

| 模块 | 职责 | 依赖方向 |
|------|------|----------|
| `domain.model` | 核心领域对象：实体、值对象、聚合根的纯数据结构定义，不含业务逻辑 | 无外部依赖 |
| `domain.risk` | 风险判定领域服务，含两类调用模型：①**流式融合判定门面**（RiskDeterminationService + 子服务 BR-01 疲劳、BR-03 路怒、分心检出）处理持续到达的 DMS/生理/语音流式感知，并在风险由成立转为解除时产出 RiskResolvedEvent；②**事件触发型独立判定服务**（BR-02 活体遗留、BR-06 碰撞失能），由各自的领域事件触发、独立产出领域事件，不经门面汇总；以及三级风险等级的统一映射 | 依赖 `domain.model`、`domain.event` |
| `domain.intervention` | 干预与反馈领域服务：基于 **AlertType × RiskLevel 二维映射**的分级指令生成、HMI 反馈控制、CAN 标准指令下发逻辑、驾驶员覆盖检测 | 依赖 `domain.model`、`domain.event` |
| `domain.family` | 远程监护领域服务：家属权限管理与常规撤销闭环（BR-07）、家属端自动激活接入（高危场景）、远程对讲/视频/车窗控制授权与执行、**家属端常态状态快照周期同步（≥1Hz 推送绿/黄/红状态）** | 依赖 `domain.model`、`domain.event` |
| `domain.fleet` | 车队运营管理领域服务：看板聚合查询、钻取查询、驾驶评分（BR-05）、报告生成与导出、绩效预警推送 | 依赖 `domain.model`、`domain.event` |
| `domain.emergency` | 应急救援领域服务：SOS 自动呼叫、车辆状态共享、远程解锁授权、健康档案调取 | 依赖 `domain.model`、`domain.event` |
| `domain.privacy` | 隐私保护领域服务：BR-04 脱敏边界约束、数据授权校验、路怒语音存证的生命周期管理 | 依赖 `domain.model`、`domain.event` |
| `domain.ota` | OTA 升级管理领域服务：版本管理、升级包下发、断点续传、完整性校验、失败回滚、静默升级状态机 | 依赖 `domain.model`、`domain.event` |
| `domain.monitor` | 系统自检领域服务：BR-08 传感器故障检测、监测脱线标记、失效告警 | 依赖 `domain.model`、`domain.event` |
| `domain.event` | 领域事件定义：系统中所有领域事件的类型与载体，供各模块间解耦通信 | 仅依赖 `domain.model` 中的标识类型 |

### 2.2 依赖原则

- `domain.model` 是最底层，不依赖任何其他模块。
- `domain.event` 仅依赖 `domain.model` 中的标识类型（如司机 ID、行程 ID），不依赖聚合根内部结构。
- 各领域服务模块之间**禁止直接调用**，跨模块协作统一通过领域事件完成。
- 各模块**允许同时依赖** `domain.model` 和 `domain.event`。

---

## 三、核心抽象

### 3.1 聚合根

#### AR-01：Trip（行驶行程）

**角色与职责**：系统的核心聚合根，代表一次从点火到熄火的完整行驶行程。Trip 是数据汇聚与告警生成的枢纽——它持有该行程中产生的所有生理体征快照，并负责在该行程生命期内产出的安全告警事件建立关联。Trip 不直接执行风险判定（由领域服务完成），但它是判定结果与干预指令的作用上下文。

**类型形态**：`class`（聚合根）。Trip 具有独立的生命周期（点火创建、熄火终结）和唯一标识（行程 ID），其内部一致性需要事务边界保护，因此是聚合根而非普通实体。

**协作关系**：
- 与 Driver、Vehicle 是一对多归属关系（一个 Trip 属于一个 Driver 和一个 Vehicle），Trip 通过标识引用它们。
- 内部持有 PhysiologicalSnapshot 值对象集合（快照不可变，新增即追加）。
- 产出 SafetyAlertEvent，事件归属 Trip 但可独立于 Trip 生命周期存在。
- 领域服务 RiskDeterminationService 以 Trip 为判定上下文，判定结果写入 Trip 关联的告警列表。
- 领域服务 ScoringService 在行程结束时计算 TripScore，数值以值对象形式存入 Trip。

---

#### AR-02：Driver（驾驶员）

**角色与职责**：代表被监测的驾驶员，是被监护的主体。Driver 聚合管理驾驶员的基础身份信息、脱敏人脸特征向量、生理健康基准、综合风险评分，以及与其 1:1 绑定的健康档案（DriverHealthProfile）。Driver 不持有行程列表（行程独立为聚合根），仅通过标识关联。

**类型形态**：`class`（聚合根）。Driver 拥有唯一标识（驾驶员 ID），其生命周期独立于行程与车辆（换车、换行程不影响 Driver 身份），且内部 DriverHealthProfile 需要通过 Driver 聚合根才能访问和修改，符合聚合根的事务一致性边界。

**协作关系**：
- 与 Vehicle 通过 Trip 间接关联。
- 内部持有 DriverHealthProfile 实体（1:1）。
- 与 SystemAccount（家属角色）是多对多监护关系，监护关系的变更通过领域事件通知。
- 接收 PerformanceWarningEvent（当评分 < 60 时），但不直接消费，由通知模块路由给管理员。
- 驾驶员注销/账号删除时，其监护关系清理与历史行程数据处理策略见 §5.4 边界条件 (2)。

---

#### AR-03：Vehicle（车辆）

**角色与职责**：代表搭载监测终端的物理车辆资产。Vehicle 聚合管理车辆标识（车牌号、VIN）、终端序列号、传感器自检状态（在线/离线/故障），以及当前监测脱线标记（BR-08）。Vehicle 不负责判定逻辑，但为判定引擎提供设备状态上下文。

**类型形态**：`class`（聚合根）。Vehicle 有唯一标识（VIN/终端序列号），其生命周期独立于行程和驾驶员，设备状态需要以事务一致性方式更新（如一次自检结果需原子性覆盖多个传感器的状态），因此适合作为聚合根。

**协作关系**：
- 与 Driver 通过 Trip 间接关联。
- 传感器自检状态由领域服务 SensorSelfCheckService 更新，故障时触发 SensorFailureEvent。
- 与管理员角色的 SystemAccount 是管理关系。
- 车门锁状态由 EmergencyRescueService（远程解锁授权）与远程车窗/车门控制流程（场景 11）经授权后更新。

---

#### AR-04：SystemAccount（系统账户）

**角色与职责**：代表外部监护与管理主体——家属或车队管理员。SystemAccount 聚合管理账户标识、联系方式、**角色（AccountRole，见 VO-14）**、通知权限和监护/管理范围。不同角色拥有不同的数据访问和操作权限。

**类型形态**：`class`（聚合根）。SystemAccount 拥有唯一标识（账号 ID）和独立生命周期，角色变更、权限授予等操作需要事务一致性，因此是聚合根。

**协作关系**：
- 家属角色（AccountRole.FAMILY）：与 Driver 存在监护关系，权限受 Permission 值对象约束（BR-07）。
- 管理员角色（AccountRole.MANAGER）：拥有全局统计权限，接收车队级告警与绩效预警。
- 接收 SafetyAlertEvent 并根据角色（AccountRole）与通知偏好决定是否推送。
- 权限路由与判定（DS-08）以 AccountRole 枚举进行类型安全分支，而非字符串/魔法值比较。

---

### 3.2 实体（非聚合根）

#### E-01：SafetyAlertEvent（安全告警事件）

**角色与职责**：满足判定规则时生成的异常记录，是系统告警链路的统一载体。它记录告警类型（疲劳/分心/路怒/活体遗留/碰撞失能/绩效预警）、风险等级（L1/L2/L3）、发生时间、GPS 位置和异常特征快照。告警事件一旦生成即不可修改（只追加），其生命周期独立于 Trip。

**类型形态**：`class`（实体）。SafetyAlertEvent 有唯一标识（告警 ID），可独立于 Trip 被查询、推送和归档，它在数据库中可能有自己的存储表，因此是实体而非值对象。它不设计为聚合根，因为：告警事件的生成总是以 Trip 为上下文（绩效预警除外，以评分计算为上下文），通过 Trip 聚合根的一致性边界来关联。

**协作关系**：
- 关联到一个 Trip（绩效预警关联到一次评分计算上下文）。
- 触发时生成 AlertTriggeredEvent 领域事件，由通知模块消费。
- 路怒类型的告警事件与 RoadRageVoiceRecord 存在 1:1 关联。

> **设计约束**：SafetyAlertEvent 作为聚合内实体与"独立查询"需求之间存在张力。设计层面采用 **CQRS 读模型投影**策略倾向——写侧仍通过 Trip 聚合根访问 SafetyAlertEvent 以保证事务一致性；读侧（车队管理员跨行程查询告警历史、看板聚合）通过独立的只读投影或物化视图完成，不穿透聚合根边界加载。实现时可将告警写入主存储后异步投射至读模型。

---

#### E-02：RoadRageVoiceRecord（路怒语音存证）

**角色与职责**：路怒告警成立时生成的语音留存记录，存储于车载边缘侧。它封装了录制起止时间、加密音频引用、脱敏标记和保留到期时间。其生命周期受保留策略约束——到期后自动清除。

**类型形态**：`class`（实体）。RoadRageVoiceRecord 有唯一标识（存证 ID），有独立的生命周期（创建→到期→清除），因此是实体。不设计为聚合根，因为它 1:1 关联到一个 SafetyAlertEvent（路怒类型），其创建和清除与告警事件紧密耦合，通过告警事件的一致性边界管理更自然。

**协作关系**：
- 1:1 关联路怒类型的 SafetyAlertEvent。
- 默认不上云、不外传，仅通过 PrivacyProtectionService 在授权审计场景下访问。
- 录制的起止：创建于 PrivacyProtectionService 消费 RiskDeterminedEvent（AlertType=ROAD_RAGE）时；封闭于 PrivacyProtectionService 消费 RiskResolvedEvent（AlertType=ROAD_RAGE）时（见 VO-02、决策 16、场景 3）。
- 到期清除由 PrivacyProtectionService 按预设周期执行。

---

#### E-03：DriverHealthProfile（驾驶员健康档案）

**角色与职责**：与实时生理快照区分的档案级对象，包含血型、过敏史、慢性病史、用药史、基础生命体征基线和紧急医疗联系人等。仅在 SOS/救援授权场景下、经权限校验后可被救援机构调取。

**类型形态**：`class`（实体）。DriverHealthProfile 有独立的存储模型和业务含义，与实时监测数据（PhysiologicalSnapshot）是不同粒度和访问约束的数据，但其生命周期完全依附于 Driver——一个 Driver 仅有一份档案，档案的修改须通过 Driver 聚合根操作。因此它是 Driver 聚合根内部的实体，而非独立聚合根。

**协作关系**：
- 与 Driver 是 1:1 聚合内包含关系。
- 由 EmergencyRescueService 在救援场景下经权限校验后调取，并作为摘要纳入 RescueReport（VO-13）。

---

### 3.3 值对象

#### VO-01：RiskLevel（风险等级）

**角色与职责**：统一的三级风险**严重度**标签，贯穿系统所有判定和干预逻辑。它表达"风险有多严重"，而非"风险是否解除"——后者由独立的 RiskResolvedEvent 表达（见 VO-02、决策 16）。

**类型形态**：`enum`。风险等级是有限、固定的三个取值（L1_HINT / L2_WARNING / L3_CRITICAL），且不同等级在 AlertType × RiskLevel 二维映射下对应不同的干预策略分支，使用枚举可确保类型安全和编译期穷尽检查。

> **L1 取值边界（预留等级声明）**：经核对 BR-01~BR-08，本期无任一业务规则映射到 L1——疲劳轻度为 L2、路怒为 L2、分心为 L2，重度疲劳/活体遗留/碰撞失能为 L3。因此 **L1_HINT 为预留等级，本期无触发路径**，保留于枚举中以备未来扩展（如新增"提示级"轻量风险），不构成需删除的死代码。各 match 分支应对 L1 给出"无干预/仅记录"的明确处理以满足穷尽检查（见 DS-07、决策 15）。

---

#### VO-02：AlertType（告警类型）

**角色与职责**：标识告警的种类，驱动通知路由和干预策略选择。

**类型形态**：`enum`。告警类型集合是有限的、稳定的（FATIGUE / DISTRACTION / ROAD_RAGE / LIFE_DETECTION / COLLISION_DISABILITY / PERFORMANCE_WARNING），使用枚举确保各模块对类型的引用保持一致。

> **取值来源边界（消除重复判定事件）**：AlertType 是 **SafetyAlertEvent（告警实体）** 的完整分类维度，覆盖以上全部取值。但各**判定事件**仅携带其中互不重叠的子集：`RiskDeterminedEvent` 仅携带流式融合判定子集 `{FATIGUE, DISTRACTION, ROAD_RAGE}`；`LIFE_DETECTION` 仅由 `LifeDetectedEvent`（DS-05 独立产出）承载；`COLLISION_DISABILITY` 仅由 `EmergencyActivatedEvent`（DS-06 独立产出）承载。三类判定事件的 AlertType 取值集合两两不相交，因此同一判定结果**不会同时产出两条判定事件**，从根上消除了重复事件歧义。`PERFORMANCE_WARNING` 则由 `PerformanceWarningEvent`（DS-09）承载，属离线评分触发，亦不与上述判定事件重叠。

> **风险解除的表达方式（避免污染 AlertType/RiskLevel 取值）**：风险"已解除"**不**作为 AlertType 或 RiskLevel 的新增枚举值，而是由独立领域事件 **RiskResolvedEvent** 承载（见 §3.5、决策 16）。RiskResolvedEvent 复用既有的 AlertType 取值（如 AlertType=ROAD_RAGE）来标识"哪一类风险已解除"，并以独立事件类型与 RiskDeterminedEvent 区分，使消费方能明确区分"新告警"与"解除信号"，避免出现路怒解除后录制不停止、空调持续低温等问题。如此 RiskLevel 保持纯粹的严重度阶梯，AlertType 保持纯粹的类别维度，二者语义不被状态转换标记污染。

---

#### VO-03：PhysiologicalSnapshot（生理体征快照）

**角色与职责**：传感器按固定频率采集的瞬时生理数据，是一个不可变的时间点数据切片。包含采集时间戳、实时心率、血氧饱和度、情绪指数。

**类型形态**：`struct`（值对象）。PhysiologicalSnapshot 没有独立标识，其相等性由属性值决定（同一时间点的相同读数为同一快照）。它被 Trip 聚合根持有为集合，新增即追加，不修改已有快照。设计为值对象可避免快照被修改导致的并发一致性问题。

> **仓颉类型约束**：仓颉中 `struct` 为值类型（值相等语义），天然契合值对象的相等性定义，是值对象的首选类型形态。若因具体实现约束必须使用 `class`（引用类型），则需重写相等性方法以满足值相等语义。

**协作关系**：
- 被 Trip 聚合根持有为不可变集合。
- 作为 RiskDeterminationService 的输入数据。
- 不独立持久化，随 Trip 聚合根统一存储。

---

#### VO-04：GeoLocation（地理位置）

**角色与职责**：GPS/北斗坐标信息，用于告警定位、热力图展示和救援定位。

**类型形态**：`struct`（值对象）。经纬度坐标对由值定义相等性，无独立标识，是不可变的值对象。

---

#### VO-05：TripScore（行程评分）

**角色与职责**：驾驶行为评分值，范围 [0, 100]，按 BR-05 公式计算。行程级评分和周期级评分使用同一值对象类型，区别仅在于计算上下文。

**类型形态**：`struct`（值对象）。评分是一个纯数值概念，由值定义相等性。设计为值对象而非裸数值的理由：评分需要携带 clamp 至 [0,100] 的不变式约束，值对象的构造器可确保非法值（如负数、>100）无法被创建。

---

#### VO-06：SensorStatus（传感器状态）

**角色与职责**：描述单个传感器或设备通道的健康状态（在线/离线/故障），供 BR-08 失效保护逻辑使用。

**类型形态**：`enum`。状态集合有限且互斥（ONLINE / OFFLINE / FAULT），使用枚举便于模式匹配和类型安全的状态机转换。

---

#### VO-07：Permission（访问权限）

**角色与职责**：描述家属账户对某驾驶员的授权级别，决定可执行哪些远程操作（查看状态、对讲、视频、车窗控制）。BR-07 的"60 秒持续 L3 后授权"和"高危场景自动激活"是权限授予的两种路径，而非权限类型本身。

**类型形态**：`struct`（值对象）。权限由一组可执行操作的集合定义，不同账户获得的权限实例不同，但权限本身由操作集合的值定义相等性。不可变，修改权限意味着创建新的 Permission 实例（包括撤销——撤销表现为以"空操作集合/已撤销标记"的新实例替换旧实例，见 DS-08、场景 7）。

---

#### VO-08：OTAVersion（固件版本）

**角色与职责**：描述车载终端固件的版本号、适用车型范围和升级包摘要信息。

**类型形态**：`struct`（值对象）。版本号由值定义相等性（同一版本号即同一版本），不可变。用于 OTA 升级管理中的版本比对和兼容性校验。

---

#### VO-09：VehicleStateSnapshot（车辆状态快照）

**角色与职责**：事故前 30 秒或特定时刻的车辆状态摘要，包含车速、加速度、车门锁状态、起火风险标志、燃油泄漏标志等。用于 BR-06 应急救援上报。

**类型形态**：`struct`（值对象）。快照是一个时间点的不可变状态切面，由属性值定义相等性，无独立标识。

> **生产者与缓存归属（消除"快照从何而来"的不确定性）**：BR-06 要求在碰撞时刻回取**事故前 30 秒**的车辆状态，意味着系统必须在碰撞发生**之前**就持续采集并缓存近期车辆状态。该"持续采样 + 滚动缓存"职责**不属于任何领域服务**（领域服务保持无状态、按需触发），而是一个**基础设施层职责**：由边缘侧的车辆状态采集组件按固定频率生成 VehicleStateSnapshot，并维护一个覆盖至少 30s 时间窗的**滚动缓冲（ring buffer）**。领域层仅声明一个依赖接口 **VehicleStateBuffer**（端口），暴露"按时间窗回取快照序列"的能力契约（端口方法契约见决策 14）；EmergencyResponseService（DS-06）在碰撞时刻通过该端口回取事故前 30s 窗内的快照。该端口由基础设施层实现，从而将"有状态的持续缓存"与"无状态的领域判定"清晰分离。

---

#### VO-10：TimeRange（时间范围）

**角色与职责**：表示一段连续的时间区间（起止时间戳），用于报告查询的周期约束以及 VehicleStateBuffer 的时间窗回取参数。

**类型形态**：`struct`（值对象）。时间区间由起止值定义相等性，不可变。需确保起始时间不晚于结束时间的不变式。

---

#### VO-11：SensorReading（传感器读数）

**角色与职责**：四大感知通道（DMS 视觉、生理体征、语音情绪、毫米波雷达）的**统一感知数据抽象**。封装感知通道类型、采集时间戳、通道级原始载荷引用和已提取的特征向量。为各判定服务提供统一的输入契约，避免各判定服务以松散的自然语言描述各自理解输入格式。

**类型形态**：`struct`（值对象）。SensorReading 是一个不可变的感知数据切片，无独立标识，由通道类型、时间戳和特征向量的组合值定义相等性。

**协作关系**：
- 由感知采集层（基础设施层）按固定频率生成并送入领域层。
- 作为 RiskDeterminationService 及其子判定服务的统一输入。
- 本身不持久化在领域模型中，由 Trip 聚合中的 PhysiologicalSnapshot 和 VehicleStateSnapshot 等提取所需维度后独立存储。

---

#### VO-12：InterventionInstruction（干预指令）

**角色与职责**：DS-07 InterventionService 依据 **AlertType × RiskLevel 二维映射**生成的单条分级干预指令，是领域层向基础设施层（HMI/CAN）下发干预意图的统一载体。它表达"对哪个目标设备、以什么参数、什么优先级执行何种干预"，由基础设施层翻译为具体硬件指令。一次判定可产出一个 InterventionInstruction 的有序集合（如重度疲劳 L3 同时含语音播报、座椅震动、双闪、CAN 减速指令）。

**类型形态**：`struct`（值对象）。指令由其内容（指令类型 + 目标设备 + 参数映射 + 优先级）定义相等性，无独立标识、不可变。其概念维度包括：
- **指令类型**（建议以 `enum` 表达有限指令集合，如氛围灯调色/语音播报/座椅震动/双闪/空调调节/音频播放/CAN 减速请求/引导靠边等）。
- **目标设备标识**：指令作用的车载设备/通道。
- **参数映射**：指令的可变参数（如颜色、温度增量、音量、震动强度），以键值映射表达以适配异构设备。
- **优先级**：用于多指令并存时的下发/抢占次序（如 L3 安全指令优先于 L2 提示）。

**协作关系**：
- 由 InterventionService（DS-07）按二维映射生成。
- 集合形式输出，由基础设施层 HMI/CAN 适配器消费并翻译为硬件指令。

---

#### VO-13：RescueReport（救援报告）

**角色与职责**：EmergencyRescueService（DS-12）在 SOS 上报时产出的救援信息聚合载体，向救援中心/120 一次性提供定位、体征与车辆状态的完整快照集。

**类型形态**：`struct`（值对象）。报告由其内容快照定义相等性，无独立标识、不可变（一次上报即定格一份）。其概念维度包括：
- **GeoLocation**：事故精准定位（VO-04）。
- **生命体征摘要**：从 Driver 实时生理监测提取的关键体征摘要（如心率/血氧状态）。
- **VehicleStateSnapshot 集合**：经 VehicleStateBuffer 回取的事故前 30s 车辆状态快照序列（VO-09）。
- **健康档案摘要**：经授权后从 DriverHealthProfile（E-03）提取的关键医疗信息摘要（血型、过敏史、紧急联系人等）。

**协作关系**：
- 由 EmergencyRescueService 在消费 EmergencyActivatedEvent 后组装。
- 由基础设施层投递至救援中心；健康档案摘要部分受授权校验约束（见 §五）。

---

#### VO-14：AccountRole（账户角色）

**角色与职责**：标识 SystemAccount（AR-04）的角色类别，驱动 DS-08 PermissionService 的权限判定与通知路由分支。

**类型形态**：`enum`。角色集合有限且稳定（FAMILY / MANAGER），使用枚举可在权限路由与通知分发中进行类型安全、编译期可穷尽校验的分支判断，避免以字符串大写标识/魔法值做比较。

**协作关系**：
- 作为 SystemAccount 聚合的角色字段（AR-04）。
- DS-08 依据 AccountRole 决定权限授予/撤销与可执行远程操作集合；通知模块依据 AccountRole 决定推送路由（家属端 vs 管理端）。

---

#### VO-15：DriverStatusSnapshot（驾驶员状态快照）

**角色与职责**：用于**家属端常态状态同步**的轻量周期快照，表达驾驶员当前的总体安全状态色（绿/黄/红），与离散的告警事件互补——告警事件仅在判定成立时触发，而本快照按 ≥1Hz 周期持续生成，使家属端在无告警时也能看到实时的"绿色平稳"状态（见 DS-16、场景 10）。

**类型形态**：`struct`（值对象）。快照由其内容（Driver 标识 + 状态色 + 时间戳）定义相等性，无独立标识、不可变。其概念维度包括：
- **Driver 标识**：所属驾驶员。
- **状态色（StatusColor）**：建议以 `enum`（GREEN / YELLOW / RED）表达，由当前风险状态派生——无风险/L1 → 绿、L2 → 黄、L3 → 红。
- **时间戳**：采样时刻。

**协作关系**：
- 由 DriverStatusBroadcastService（DS-16）按 ≥1Hz 周期采样当前 Driver 风险状态生成。
- 经推送通道下发至家属端（端到端 ≤2s 上报约束），不持久化为领域核心数据（属常态遥测）。

---

### 3.4 领域服务

#### DS-01：RiskDeterminationService（风险判定服务）

**职责**：作为**流式感知融合判定的门面**，接收持续到达的流式感知通道统一输入（SensorReading），按 BR-01 疲劳、BR-03 路怒、分心检出规则执行融合判定，输出统一的 RiskLevel 和 AlertType（取值限定于 `{FATIGUE, DISTRACTION, ROAD_RAGE}` 子集）。它是边缘侧流式判定链路的入口——在边缘本地完成轻量判定以保证 500ms 端到端时延和断网可用性。此外，当某类此前成立的流式风险不再持续（条件不再满足）时，门面产出 **RiskResolvedEvent**（携带对应 AlertType）以表达风险解除（见决策 16）。云端承担更重的多维融合行为风控建模，但那是云侧服务而非本领域服务范畴。

> **判定模型边界（与 DS-05/DS-06 的关系）**：本门面只负责**持续流式数据驱动**的判定（DMS 视觉、生理、语音三条持续到达的感知流）。BR-02 活体遗留与 BR-06 碰撞失能属于**事件触发型判定**——其触发源是离散的领域事件（熄火落锁、碰撞冲击），判定逻辑、生命周期与产出事件均不同于流式融合，故由 LifeDetectionService（DS-05）与 EmergencyResponseService（DS-06）作为**独立领域服务**承担，**不纳入本门面的委托列表**（见决策 2、决策 13）。

> **解除判定的状态边界**：判定"风险由成立转为解除"需对比当前判定与此前状态，该"当前活跃风险集"为**会话级状态**，由边缘侧流式会话上下文持有并随每次 SensorReading 作为输入参数传入门面，门面返回更新后的会话状态与（可选的）RiskDeterminedEvent / RiskResolvedEvent。门面本身对外仍是纯函数，可变状态归会话上下文，与 DS-05 的 DetectionWindow 处理方式一致，不引入服务内部 mutable 字段，边缘侧单线程环境（§6.1）保证无并发竞争。

**协作**：
- 输入：多条流式 SensorReading（分别来自 DMS 视觉、生理体征、语音情绪通道）。生理通道同时也被 EmergencyResponseService 用于失能判定，毫米波雷达通道被 LifeDetectionService 使用，加速度通道被 EmergencyResponseService 使用——这些通道由感知分发层按消费方路由，不流经本门面。
- 内部委托：将各通道数据分派给 FatigueDeterminationService、DistractionDetectionService、RoadRageDeterminationService 各自判定。
- 汇总各子判定服务的判定结果，产出 RiskDeterminedEvent（携带 RiskLevel 和 AlertType ∈ `{FATIGUE, DISTRACTION, ROAD_RAGE}`），驱动干预模块和通知模块；在风险解除时产出 RiskResolvedEvent。
- 门面职责：子判定服务不直接产出领域事件，也不直接调用其他模块的领域服务。各子判定服务仅将判定结果返回给 RiskDeterminationService，由门面统一产出 RiskDeterminedEvent / RiskResolvedEvent。跨模块协作（如路怒成立触发语音存证、环境调节）由消费方订阅事件完成，而非在子判定服务中直接调用。

**为何是领域服务**：风险判定操作跨越多个聚合（Trip、Driver、Vehicle），不属于任何单一聚合的行为，且判定过程是无状态的——给定相同的输入（含会话状态），判定结果确定。

---

#### DS-02：FatigueDeterminationService（疲劳判定服务 — BR-01）

**职责**：依据 DMS 视觉数据执行疲劳分级：轻度疲劳（L2）与重度疲劳（L3）。判定条件如 BR-01 所述——连续 3 分钟内频繁眨眼或视线偏离累计 >15s → L2；眼睑闭合 >1.5s 或点头频率 >2 次/10s → L3。

**协作**：被 RiskDeterminationService 委托调用。输出疲劳等级和判定依据快照，**仅返回给门面，不自行产出领域事件或调用其他模块**。

---

#### DS-03：DistractionDetectionService（分心检出服务）

**职责**：依据 DMS 视觉数据判定分心行为——视线偏离前方持续/累计达 3 秒即判定为分心成立（L2），判定成立后须在 0.5 秒内发出告警。

**协作**：被 RiskDeterminationService 委托调用。输出分心标志和判定时间戳，**仅返回给门面，不自行产出领域事件或调用其他模块**。完整端到端链路与 0.5s 时延约束见场景 8。

---

#### DS-04：RoadRageDeterminationService（路怒判定服务 — BR-03）

**职责**：融合语音情绪特征（声压级 >85dB + 谩骂关键词）和生理特征（心率较静息上升 20%+）判定路怒状态（L2）。

**协作**：
- 被 RiskDeterminationService 委托调用。
- 判定成立时，将判定结果（含 AlertType=ROAD_RAGE 及判定依据快照）**返回给 RiskDeterminationService 门面**，不直接调用任何其他模块的领域服务。
- RiskDeterminationService 汇总后产出 RiskDeterminedEvent（携带 AlertType=ROAD_RAGE），由以下消费方各自订阅处理：
  - **PrivacyProtectionService**（`domain.privacy`）消费 RiskDeterminedEvent（AlertType=ROAD_RAGE），触发路怒语音存证录制。
  - **InterventionService**（`domain.intervention`）消费 RiskDeterminedEvent（AlertType=ROAD_RAGE），触发车内环境调节指令（空调调低 2°C + 白噪音/舒缓音乐）。
- 路怒解除（条件不再满足）时，门面产出 RiskResolvedEvent（AlertType=ROAD_RAGE），驱动 PrivacyProtectionService 封闭存证、InterventionService 恢复环境（见决策 16、场景 3）。

---

#### DS-05：LifeDetectionService（活体遗留检测服务 — BR-02，**独立事件触发型领域服务**）

**职责**：本服务是**独立的事件触发型判定服务**，**不属于 RiskDeterminationService 门面的委托子服务**（见决策 2、决策 13）。其触发源是离散的"熄火且车门落锁"事件（而非持续到达的流式感知），触发后接收毫米波雷达扫描信号，在 60 秒判定窗口内持续监测呼吸/移动微动。若判定窗口内持续感应到微动，判定为遗留生命风险（L3，AlertType=LIFE_DETECTION）。判定成立后，须在 10 秒内**直接产出** LifeDetectedEvent 驱动告警推送（家属 APP 红色高频振动报警 + 车辆双闪 + 短促鸣笛），其产出路径**不经过 RiskDeterminationService 汇总**，因此不会与门面产生重复的判定事件。

**协作**：
- 触发：订阅"熄火落锁"事件（由车辆状态变更产生），据此启动一次判定窗口。
- 输入：毫米波雷达活体信号（按消费方路由直送本服务，不流经融合门面）。
- 输出：LifeDetectedEvent（携带 Vehicle 标识、判定置信度、时间戳）；告警落库时生成 AlertType=LIFE_DETECTION 的 SafetyAlertEvent。

> **设计约束（判定窗口状态边界澄清）**：判定窗口（60s 倒计时）是本服务在一次活体监测会话内的**会话级临时状态**，而非领域服务的持久成员状态。设计层面明确其归属为：将窗口的剩余时长、起始时刻、累计微动观测等封装为独立的 **`DetectionWindow` 值对象**，由调用上下文（边缘侧的活体监测会话）持有并在每次雷达信号到达时作为**输入参数**传入本服务、由本服务返回**更新后的 `DetectionWindow` 与判定结论**。如此本服务对外仍是**纯函数**（输入 = 当前 DetectionWindow + 新雷达信号，输出 = 新 DetectionWindow + 可选 LifeDetectedEvent），窗口的可变状态由会话上下文管理而非服务内部 mutable 字段，消除了对可测试性的影响。边缘侧单线程运行环境（见 §6.1）进一步保证该会话状态无并发竞争。雷达信号在窗口内短暂中断又恢复的窗口处理策略见 §5.4 边界条件 (3)。

---

#### DS-06：EmergencyResponseService（应急响应服务 — BR-06，**独立事件触发型领域服务**）

**职责**：本服务是**独立的事件触发型判定服务**，**不属于 RiskDeterminationService 门面的委托子服务**（见决策 2、决策 13）。其触发源是离散的"碰撞特征冲击"信号（而非持续流式感知）。当加速度传感器检测到碰撞冲击，且生理监测显示心率骤停或意识丧失 >10 秒时，跳过人工确认，立即**直接产出** EmergencyActivatedEvent（L3，AlertType=COLLISION_DISABILITY），驱动救援上报与家属端自动激活；其产出路径**不经过 RiskDeterminationService 汇总**，不会与门面产生重复判定事件。

**协作**：
- 触发：碰撞冲击信号（加速度通道按消费方路由直送本服务，不流经融合门面）+ 生理特征（共享生理通道）。
- 事故前 30 秒 VehicleStateSnapshot 的获取：本服务**不负责持续采集与缓存**车辆状态；它在碰撞时刻通过一个由基础设施层实现的**车辆状态滚动缓冲端口**（VehicleStateBuffer，领域层声明的依赖接口，方法契约见决策 14）回取覆盖事故前 30s 时间窗的 VehicleStateSnapshot（见 VO-09、决策 14）。
- 输出：EmergencyActivatedEvent（携带 GeoLocation、回取的 VehicleStateSnapshot、Driver 标识、时间戳）；告警落库时生成 AlertType=COLLISION_DISABILITY 的 SafetyAlertEvent。
- 消费方：EmergencyRescueService（救援上报）、PermissionService（家属端自动激活）。

---

#### DS-07：InterventionService（干预执行服务）

**职责**：接收 RiskDeterminedEvent，按 **AlertType × RiskLevel 二维映射**生成对应的分级干预指令集合（InterventionInstruction，VO-12）。干预策略由"告警类型 + 风险等级"共同决定，而非单一 RiskLevel，从而与各场景行为契约保持一致。负责"语音唤醒 → 建议/请求减速 → 引导靠边"的渐进式指令升级逻辑（疲劳 L3）。同时监控驾驶员覆盖（override）信号，一旦检测到转向/踩踏板等有效操作即停止升级并归还控制权。当收到 RiskResolvedEvent 时，停止对应类型的干预升级并恢复车内环境（如路怒解除恢复空调）。

**二维干预映射（AlertType × RiskLevel，本期取值）**：

| AlertType ＼ RiskLevel | L2 | L3 |
|---|---|---|
| FATIGUE（疲劳） | 氛围灯变橙提醒 | 语音播报 + 座椅强力震动 + 渐进升级（语音唤醒→建议/请求减速→引导靠边）+ 双闪 + CAN 减速指令下发 |
| ROAD_RAGE（路怒） | 环境调节：空调调低 2°C + 白噪音/舒缓音乐 | 本期无（路怒仅 L2） |
| DISTRACTION（分心） | 告警（判定成立后 0.5s 内发出，如语音/视觉告警） | 本期无（分心仅 L2） |

> **L1 处理**：L1 为预留等级（VO-01），本期 RiskDeterminedEvent 不会携带 L1（无规则触发）。二维映射对 L1 给出"无干预/仅记录"分支以满足穷尽检查，构成显式占位而非死代码——一旦未来新增映射到 L1 的规则，仅需在此补充对应指令。LIFE_DETECTION / COLLISION_DISABILITY 的响应不经 InterventionService 的 RiskDeterminedEvent 路径，分别由 LifeDetectedEvent / EmergencyActivatedEvent 的专属消费方（HMI 双闪鸣笛、救援上报）处理。

**协作**：
- 订阅 RiskDeterminedEvent（生成干预指令集合）、RiskResolvedEvent（停止升级/恢复环境）和 AlertTriggeredEvent。
- 输出 InterventionInstruction（VO-12）有序集合，由基础设施层翻译为具体硬件指令。
- 与驾驶员覆盖检测（override detection）交互——覆盖信号由感知层提供，本服务决定是否中止干预升级。

---

#### DS-08：PermissionService（权限管理服务 — BR-07）

**职责**：管理家属账户对驾驶员的访问权限，依据 SystemAccount 的 AccountRole（VO-14）判定授权范围。维护家属常规权限的**完整状态机**：授予、临时撤销、常规自动撤销。
- **常规授予**：当驾驶员持续处于 L3 风险 >60 秒时授予"远程对讲 + 视频监控"权限。
- **高危自动激活**：BR-06 触发的高危失能场景下自动激活接入，不受 60 秒约束。
- **临时撤销**：驾驶员物理遮挡摄像头时触发权限临时撤销。
- **常规自动撤销（新增闭环）**：当驾驶员的风险等级由 L3 下降——后续 RiskDeterminedEvent 携带 < L3 的等级，或收到该类型的 RiskResolvedEvent，即"L3 不再持续"——自动撤销此前因常规路径授予的家属权限，并发出 **FamilyAccessRevokedEvent**，使家属端对讲/视频接入闭环结束。高危自动激活路径的接入由救援流程另行管理，不走此常规撤销。

**协作**：
- 订阅 RiskDeterminedEvent（跟踪 L3 持续时长、识别 L3 下降）与 RiskResolvedEvent（识别风险解除）。
- 订阅 EmergencyActivatedEvent（触发自动激活）。
- 输出 FamilyAccessGrantedEvent（携带被授权账户、驾驶员、权限范围、授权原因）与 FamilyAccessRevokedEvent（携带被撤销账户、驾驶员、撤销原因：常规风险下降/物理遮挡）。
- 权限授予/撤销结果写回 SystemAccount 聚合中的 Permission 值对象（撤销表现为以已撤销的新 Permission 实例替换旧实例）。

---

#### DS-09：ScoringService（驾驶评分服务 — BR-05）

**职责**：按 BR-05 公式计算行程级评分：`max(0, 100 − 重度疲劳×10 − 分心×5 − 路怒×8 − 急刹/急加速×2)`，clamp 至 [0,100]。周期级评分按行程时长加权平均。当评分 < 60 时，触发绩效预警通知推送。

**协作**：
- 输入：Trip 聚合中的告警统计（重度疲劳次数、分心次数、路怒触发次数、急刹/急加速次数）。
- 输出：TripScore 值对象（行程级）/ TripScore 值对象（周期级）。
- 评分 < 60 时发出 PerformanceWarningEvent。

---

#### DS-10：FleetAnalyticsService（车队分析服务）

**职责**：按车队维度聚合疲劳指数分布——正常/轻度/重度占比、风险热力图（地理位置 + 风险等级）。支持默认每 5 分钟周期刷新和手动即时刷新。支持钻取：点击某风险等级板块下钻至高风险司机明细列表。

**协作**：
- 查询多个 Trip 和 Driver 聚合的数据。
- 输出聚合结果（纯数据，不产生副作用）。
- 看板刷新和钻取查询由本服务提供接口，缓存和刷新策略由基础设施层支持。钻取交互的行为契约见场景 12。

---

#### DS-11：ReportGenerationService（报告生成服务）

**职责**：按指定司机 + 时间范围（周/月/季）生成驾驶行为分析报告，含疲劳、分心、急加速、急刹车等指标的统计与趋势。报告支持导出为 PDF/Excel。须在 15 秒内完成生成。

**协作**：
- 输入：Driver 标识 + TimeRange 值对象。
- 查询 Trip 聚合中的告警和快照数据。
- 调用 ScoringService 获取周期评分。
- 输出报告数据结构（由基础设施层负责 PDF/Excel 渲染）。15s SLA 行为契约见场景 10。

---

#### DS-12：EmergencyRescueService（应急救援服务）

**职责**：SOS 自动呼叫与上报——向救援中心/120 发送精准定位、事故前 30 秒车辆状态快照、驾驶员实时生命体征。管理远程解锁授权：救援机构核实险情后云端授权开启车门锁。管理驾驶员健康档案调取：救援机构授权后可调取 DriverHealthProfile。

**协作**：
- 输入：EmergencyActivatedEvent。
- 查询 Driver 聚合（DriverHealthProfile）、Vehicle 聚合（车门锁状态）。
- 输出：RescueReport（VO-13，包含 GeoLocation、生命体征摘要、VehicleStateSnapshot 集合、健康档案摘要）。
- 远程解锁/车窗控制的授权校验与执行链路见场景 11。

---

#### DS-13：PrivacyProtectionService（隐私保护服务 — BR-04）

**职责**：确保 BR-04 的隐私数据边界被遵守——DMS 原始图像在边缘侧完成脱敏（人脸关键点提取/模糊化），云端仅接收脱敏后的数值特征向量；未经授权严禁原始高清视频上云。管理路怒语音存证的全生命周期：录制→加密→留存→到期清除。

**协作**：
- 在数据上云路径中作为**守门人**校验数据是否已脱敏。
- **订阅 RiskDeterminedEvent**：当 AlertType=ROAD_RAGE 时触发路怒语音存证录制（RoadRageVoiceRecord 的创建、加密存储和保留到期时间设置）。
- **订阅 RiskResolvedEvent**：当 AlertType=ROAD_RAGE 解除时停止录制并封闭存证文件（见决策 16、场景 3）。
- 管理 RoadRageVoiceRecord 的到期检查和自动清除。
- 处理审计场景下的授权访问请求。

---

#### DS-14：SensorSelfCheckService（传感器自检服务 — BR-08）

**职责**：对关键传感器（摄像头/雷达等）执行周期性自检。若发现遮挡或链路故障，在 3 秒内通过 HMI 持续语音提示"安全监测系统已失效"，并在车队大屏同步标记该车辆为"监测脱线"。

**协作**：
- 输入：传感器自检信号。
- 输出：SensorFailureEvent（携带车辆标识、故障传感器列表、时间戳）。
- 更新 Vehicle 聚合中的 SensorStatus。

---

#### DS-15：OTAUpdateService（OTA 升级管理服务）

**职责**：管理车载终端固件的版本管理、升级包下发与校验、断点续传、完整性校验、失败回滚、静默升级状态机。

**协作**：
- 管理 OTAVersion 值对象的新旧版本比对。
- 升级包的状态机（待下发 → 传输中 → 校验中 → 已就绪 → 升级中 → 完成/回滚）由本服务维护，状态转换触发条件与失败回滚/断点续传语义见场景 9。
- 升级完成时发出 OTAUpgradeCompletedEvent；升级失败回滚时发出 OTAUpgradeFailedEvent。

---

#### DS-16：DriverStatusBroadcastService（驾驶员状态广播服务 — 家属常态同步）

**职责**：为 BR-07 远程监护提供**常态状态同步**机制——离散的告警事件仅在判定成立时触发，无法满足家属端持续展示驾驶员实时状态（绿/黄/红）的需求。本服务按 **≥1Hz** 周期采样驾驶员当前风险状态，生成 DriverStatusSnapshot（VO-15）并经推送通道下发至家属端，端到端 ≤2s 上报。状态色由当前风险派生：无风险/L1 → 绿、L2 → 黄、L3 → 红。

**协作**：
- 输入：驾驶员当前风险状态（由边缘侧会话上下文维护的当前活跃风险集派生，与 DS-01 解除判定共用同一会话状态来源，避免重复维护）。
- 输出：DriverStatusSnapshot（VO-15），经基础设施层推送通道下发家属端。
- 与离散告警链路互补：告警链路负责"事件性"高优先级推送，本服务负责"常态性"低优先级遥测，二者不互相阻塞。
- 作为高频遥测，其推送走非安全攸关的异步链路（§6.2），不占用 ≤500ms 安全判定链路资源。

---

### 3.5 领域事件

| 事件 | 触发时机 | 携带关键信息 | 主要消费方 |
|------|----------|-------------|-----------|
| `RiskDeterminedEvent` | RiskDeterminationService 完成一次**流式融合判定**（仅 AlertType ∈ `{FATIGUE, DISTRACTION, ROAD_RAGE}`） | Trip 标识、RiskLevel、AlertType、判定时间戳、异常特征快照 | InterventionService（按 AlertType×RiskLevel 生成干预）、PermissionService（跟踪 L3 持续时长/识别 L3 下降）、PrivacyProtectionService（AlertType=ROAD_RAGE 时触发语音存证录制） |
| `RiskResolvedEvent` | RiskDeterminationService 判定某类此前成立的流式风险**不再持续**（条件不再满足） | Trip 标识、已解除的 AlertType、解除时间戳 | PrivacyProtectionService（AlertType=ROAD_RAGE 时停止录制、封闭存证）、InterventionService（停止对应类型干预升级、恢复车内环境）、PermissionService（风险解除参与常规撤销判定） |
| `AlertTriggeredEvent` | SafetyAlertEvent 被创建 | 告警 ID、AlertType、RiskLevel、GPS、Timestamp | 通知推送模块、车队看板刷新 |
| `LifeDetectedEvent` | **LifeDetectionService（独立事件触发型服务，不经门面汇总）** 在 60s 窗口内判定成立（AlertType=LIFE_DETECTION，独占此取值） | Vehicle 标识、判定置信度、Timestamp | 家属推送模块、车辆 HMI 控制（双闪/鸣笛） |
| `EmergencyActivatedEvent` | **EmergencyResponseService（独立事件触发型服务，不经门面汇总）** 判定 BR-06 碰撞+失能条件满足（AlertType=COLLISION_DISABILITY，独占此取值） | Driver 标识、GeoLocation、VehicleStateSnapshot、Timestamp | EmergencyRescueService、PermissionService（自动激活家属端） |
| `SensorFailureEvent` | SensorSelfCheckService 检测到关键传感器故障 | Vehicle 标识、故障传感器列表、Timestamp | 车队看板（标记脱线）、HMI 语音提示 |
| `FamilyAccessGrantedEvent` | 家属获得对某 Driver 的访问权限（常规 60s 或自动激活） | SystemAccount 标识、Driver 标识、Permission、授权原因（常规/高危） | 家属 APP 推送、远程对讲/视频通道建立 |
| `FamilyAccessRevokedEvent` | 家属对某 Driver 的常规权限被撤销（L3 风险下降不再持续，或驾驶员物理遮挡） | SystemAccount 标识、Driver 标识、撤销原因（风险下降/物理遮挡） | 家属 APP（断开对讲/视频接入）、SystemAccount 聚合（清除 Permission） |
| `TripScoredEvent` | ScoringService 完成一次行程级评分 | Trip 标识、TripScore、扣分项明细 | 报告生成、趋势统计 |
| `PerformanceWarningEvent` | 行程级或周期级评分 < 60 | Driver 标识、Score、评估周期、主要扣分项 | 管理员通知推送 |
| `OTAUpgradeCompletedEvent` | 车载终端固件升级成功 | Vehicle 标识、旧版本、新版本、升级耗时 | 车队管理日志、版本追踪 |
| `OTAUpgradeFailedEvent` | OTA 升级因校验/传输失败回滚 | Vehicle 标识、目标版本、失败阶段、回滚结果 | 车队管理日志、升级重试调度 |

> **常态状态同步说明**：DriverStatusSnapshot（VO-15）的 ≥1Hz 周期推送属高频常态遥测，不建模为离散领域事件（避免污染事件总线与 outbox），由 DS-16 经独立的异步推送通道下发（见 §6.2）。

---

## 四、关键行为契约

> 本节覆盖全部 16 个领域服务的核心交互场景。本版新增场景 8（分心检出）、场景 9（OTA 升级）、场景 10（报告生成与状态同步）、场景 11（远程车窗/车门控制）、场景 12（看板钻取），并修订场景 3、场景 7 以反映风险解除语义与权限撤销闭环。

### 场景 1：疲劳驾驶判定与干预（BR-01 + 干预链）

1. 感知通道持续送入 DMS 视觉 SensorReading。
2. RiskDeterminationService 委托 FatigueDeterminationService 按 BR-01 条件判定。
3. 判定为轻度疲劳（L2）时：
   - RiskDeterminationService 产出 RiskDeterminedEvent（AlertType=FATIGUE，RiskLevel=L2）。
   - 生成 SafetyAlertEvent（AlertType=FATIGUE，RiskLevel=L2）。
   - 发出 AlertTriggeredEvent。
   - InterventionService 按二维映射（FATIGUE × L2）生成"氛围灯变橙提醒"InterventionInstruction。
4. 判定为重度疲劳（L3）时：
   - RiskDeterminationService 产出 RiskDeterminedEvent（AlertType=FATIGUE，RiskLevel=L3）。
   - 生成 SafetyAlertEvent（AlertType=FATIGUE，RiskLevel=L3）。
   - InterventionService 按二维映射（FATIGUE × L3）生成"语音播报 + 座椅强力震动 + 渐进升级 + 双闪 + CAN 减速指令"InterventionInstruction 集合。
   - 若 L3 持续超过 60 秒，PermissionService 则向关联家属授予远程对讲/视频权限（BR-07 常规路径）。
5. 整个过程在边缘侧完成，端到端时延 ≤ 500ms（判定到指令下发）。

---

### 场景 2：车内活体遗留检测与报警（BR-02）

1. 车辆熄火、车门落锁后，"熄火落锁"事件触发活体监测会话；毫米波雷达自动开始扫描。
2. LifeDetectionService（独立事件触发型服务，不经 RiskDeterminationService 门面）以会话级 DetectionWindow 值对象承载 60 秒判定窗口状态，随每次雷达信号到达迭代更新窗口与判定结论（见 DS-05 设计约束）。
3. 若窗口内持续感应到微动：
   - 判定为"遗留生命风险"（L3，AlertType=LIFE_DETECTION）。
   - 直接产出 LifeDetectedEvent（不与门面产生重复判定事件）。
   - 10 秒内：家属 APP 收到红色高频振动报警；车载双闪开启、短促鸣笛。
4. 若窗口内微动消失（存在时间 < 60s），判定取消，不触发告警（抑制误报）。雷达信号短暂中断又恢复时的窗口处理见 §5.4 边界条件 (3)。

---

### 场景 3：路怒检测、存证与解除（BR-03）

1. 语音情绪通道检测到声压级 >85dB 且含谩骂关键词。
2. 生理通道显示心率较静息上升 20%+。
3. RoadRageDeterminationService 两条件同时满足时判定路怒成立（L2），**将判定结果返回给 RiskDeterminationService 门面**。
4. RiskDeterminationService 汇总后产出 RiskDeterminedEvent（AlertType=ROAD_RAGE，RiskLevel=L2）。
5. 领域事件驱动的后续行为：
   - **InterventionService** 消费 RiskDeterminedEvent（ROAD_RAGE × L2）→ 按二维映射触发环境调节指令：空调温度调低 2°C、播放白噪音/舒缓音乐。
   - **PrivacyProtectionService** 消费 RiskDeterminedEvent（AlertType=ROAD_RAGE）→ 创建 RoadRageVoiceRecord，开始录制当前时段语音片段，存储于边缘侧，标记脱敏/加密，设置保留到期时间。
6. **路怒解除**：当路怒判定条件不再满足，RiskDeterminationService 产出 **RiskResolvedEvent（AlertType=ROAD_RAGE）**（而非携带某 RiskLevel 的 RiskDeterminedEvent）：
   - PrivacyProtectionService 消费后**停止录制并封闭存证文件**。
   - InterventionService 消费后**恢复车内环境**（解除空调调低/停止白噪音）。
   - 消费方据事件类型即可明确区分"新告警"与"解除信号"，不会出现录制不停止、空调持续低温（见决策 16）。

---

### 场景 4：碰撞事故应急响应（BR-06）

1. 加速度传感器检测到碰撞特征冲击。
2. 生理监测显示心率骤停或意识丧失（无操作反馈）>10 秒。
3. EmergencyResponseService（独立事件触发型服务，不经 RiskDeterminationService 门面）判定 BR-06 条件成立（L3，AlertType=COLLISION_DISABILITY）：
   - 经 VehicleStateBuffer 端口（决策 14 方法契约）回取事故前 30 秒窗内的 VehicleStateSnapshot 序列。
   - 直接产出 EmergencyActivatedEvent（不与门面产生重复判定事件）。
   - 跳过人工确认，立即驱动救援链路。
4. PermissionService 收到 EmergencyActivatedEvent，自动激活关联家属端对讲/视频接入（无需家属手动发起）。
5. EmergencyRescueService 消费 EmergencyActivatedEvent，组装 RescueReport（VO-13：GPS + 事故前 30s VehicleStateSnapshot 集合 + 实时生命体征摘要 + 授权后的健康档案摘要），向救援中心/120 上报；并处理远程解锁授权和健康档案调取（授权链路见场景 11）。

---

### 场景 5：驾驶评分与绩效预警（BR-05）

1. 行程结束时，Trip 聚合被标记为完成。
2. ScoringService 查询 Trip 关联的 SafetyAlertEvent 统计：重度疲劳次数、分心次数、路怒次数、急刹/急加速次数。
3. 按公式计算 TripScore：`max(0, 100 − A×10 − B×5 − C×8 − D×2)`，结果 clamp 至 [0,100]。
4. 发出 TripScoredEvent。
5. 若 TripScore < 60，发出 PerformanceWarningEvent，通知模块将绩效预警推送给关联车队管理员。
6. 周期评分（周/月/季）由 ScoringService 按该周期内所有行程的 TripScore 按时长加权平均计算，同样 clamp 至 [0,100] 并在 < 60 时触发绩效预警。

---

### 场景 6：传感器故障失效保护（BR-08）

1. SensorSelfCheckService 周期性对关键传感器执行自检。
2. 若发现遮挡或链路故障：
   - 在 3 秒内发出 SensorFailureEvent。
   - HMI 层消费事件：持续语音提示"安全监测系统已失效，请注意驾驶安全"。
   - 车队看板消费事件：标记该 Vehicle 为"监测脱线"。
3. 故障恢复后再次自检通过，清除脱线标记并停止语音提示。

---

### 场景 7：家属常规权限授予与撤销（BR-07 常规路径，授予—撤销闭环）

1. PermissionService 订阅 RiskDeterminedEvent，跟踪每个 Driver 的 L3 风险持续时长。
2. **授予**：当某 Driver 的 L3 连续超过 60 秒时：
   - 授予关联家属账户"远程对讲 + 视频监控"权限（创建新 Permission 实例，更新 SystemAccount 聚合）。
   - 发出 FamilyAccessGrantedEvent（授权原因=常规）。
   - 家属 APP 收到事件后可发起对讲/视频接入。
3. **常规自动撤销**：当该 Driver 的风险由 L3 下降不再持续——后续 RiskDeterminedEvent 携带 < L3 等级，或收到 RiskResolvedEvent——PermissionService：
   - 撤销此前常规授予的权限（以已撤销的新 Permission 实例替换旧实例）。
   - 发出 **FamilyAccessRevokedEvent**（撤销原因=风险下降）。
   - 家属端据此断开对讲/视频接入，闭环结束。
4. **临时撤销**：驾驶员可在任何时候物理遮挡摄像头，该操作通过感知通道检测，触发权限临时撤销并发出 FamilyAccessRevokedEvent（撤销原因=物理遮挡）。
5. 高危失能场景（BR-06）的家属端自动激活接入不走本常规撤销路径，由救援流程结束后另行收束。

---

### 场景 8：分心检出与告警（DS-03，端到端时延约束）

1. DMS 视觉通道持续送入 SensorReading（视觉特征：视线方向）。
2. RiskDeterminationService 委托 DistractionDetectionService 判定：视线偏离前方持续/累计达 3 秒，判定分心成立（L2）。
3. 子服务将判定结果（分心标志、判定时间戳）**仅返回门面**，不自行产出事件。
4. RiskDeterminationService 汇总后产出 RiskDeterminedEvent（AlertType=DISTRACTION，RiskLevel=L2），并生成 SafetyAlertEvent + AlertTriggeredEvent。
5. InterventionService 消费 RiskDeterminedEvent（DISTRACTION × L2），按二维映射生成"告警"InterventionInstruction（如语音/视觉提醒）。
6. **时延约束**：从判定成立到告警下发须在 **0.5 秒**内完成；整链路在边缘侧同步消费完成（§6.2 边缘侧同步链路），满足该约束。

---

### 场景 9：OTA 固件升级全流程（DS-15，状态机与失败处理）

1. **待下发**：OTAUpdateService 比对 OTAVersion，确认目标车载终端需升级，进入待下发态。
2. **传输中**：下发升级包；支持**断点续传**——传输中断后，按已传输偏移量从断点继续，不重传已完成分片。
   - **下发失败重试**：传输失败时按退避策略重试；超过重试上限则转入失败处理（回滚态，发出 OTAUpgradeFailedEvent，失败阶段=传输）。
3. **校验中**：传输完成后执行完整性校验（摘要比对）。
   - **校验失败回滚**：校验不通过则丢弃升级包、保持原固件不变，转入回滚态，发出 OTAUpgradeFailedEvent（失败阶段=校验）。
4. **已就绪 → 升级中**：校验通过后进入已就绪态，按静默升级时机进入升级中态执行刷写。
5. **完成 / 回滚**：
   - 刷写成功 → 完成态，发出 OTAUpgradeCompletedEvent（含旧/新版本、耗时）。
   - 刷写失败 → 回滚至旧固件，发出 OTAUpgradeFailedEvent（失败阶段=刷写）。
6. 状态转换均由 OTAUpdateService 的静默升级状态机驱动；各失败路径均保证终端可回退至可用旧固件（安全回滚不变式）。

---

### 场景 10：报告生成与导出 + 家属常态状态同步

**10a 报告生成与导出（DS-11，15s SLA）**
1. 管理员（AccountRole=MANAGER）请求生成报告，输入 Driver 标识 + TimeRange（周/月/季）。
2. ReportGenerationService 查询该范围内 Trip 聚合的告警与快照统计（疲劳、分心、急加速、急刹车指标与趋势）。
3. 调用 ScoringService 获取周期评分。
4. 组装报告数据结构，交由基础设施层渲染导出为 PDF/Excel。
5. **SLA**：从请求到生成完成须在 **15 秒**内；超时或范围内无数据按 §五 Result 语义返回业务结果（无数据 → 空报告/明确提示，而非异常）。

**10b 家属常态状态快照同步（DS-16，≥1Hz/≤2s）**
1. DriverStatusBroadcastService 按 ≥1Hz 周期采样当前 Driver 风险状态（取自边缘侧会话上下文的当前活跃风险集）。
2. 派生状态色：无风险/L1 → 绿、L2 → 黄、L3 → 红，生成 DriverStatusSnapshot（VO-15）。
3. 经异步推送通道下发家属端，端到端 ≤2s 上报。
4. 该常态同步与离散告警链路互补：即便当前无告警，家属端也能持续看到"绿色平稳"状态。

---

### 场景 11：远程车窗/车门控制授权与执行（DS-08 / DS-12）

1. 请求方发起远程车窗/车门控制：家属端（AccountRole=FAMILY，依 Permission 含车窗控制权限）或救援机构（经 EmergencyRescueService 险情核实授权）。
2. **授权校验**：
   - 家属请求：PermissionService 依 AccountRole 与 Permission 校验是否具备车窗控制授权；不具备则按 §五 Result 语义返回 `PermissionDenied`（携带拒绝原因供前端提示）。
   - 救援请求：EmergencyRescueService 校验救援机构远程解锁授权（云端授权开启车门锁）；未授权返回 `AccessDenied`。
3. **执行**：授权通过后，生成控制指令经基础设施层 CAN/HMI 下发执行，更新 Vehicle 聚合的车门锁/车窗状态。
4. 执行失败（设备无响应/链路故障）作为 C 类系统级故障向上抛给基础设施层重试/降级（§5.3）。

---

### 场景 12：车队看板钻取交互（DS-10）

1. 管理员看板默认每 5 分钟周期刷新疲劳指数分布（正常/轻度/重度占比）与风险热力图（GeoLocation × RiskLevel）。
2. 管理员可手动触发即时刷新。
3. **钻取**：点击某风险等级板块，FleetAnalyticsService 以该风险等级为条件下钻，返回高风险司机明细列表（只读聚合查询，经 CQRS 读模型投影，不穿透聚合根）。
4. 钻取结果即时返回；缓存与刷新频率由基础设施层控制（§6.3）。

---

## 五、错误处理策略

### 5.1 错误分类

系统领域层的错误分为三类：

**A 类——判定失败**：风险判定无法完成（如感知数据缺失、传感器故障导致输入不可用）。此类错误应由 SensorSelfCheckService 提前检测并发出 SensorFailureEvent，判定服务在输入不可用时返回 `None` 而非抛出异常——"无法判定"本身是一种合法的系统状态，不应打断其他正常运行的判定链路。

**B 类——业务规则违反**：操作违反了业务约束（如未经授权的家属尝试调取原始视频、评分时行程尚未结束）。此类错误使用 `Result<T, Error>` 模式，调用方可显式处理业务拒绝，不抛异常。

**C 类——系统级故障**：数据持久化失败、基础设施层面的异常。此类错误在领域服务中不捕获，向上抛给基础设施层统一处理（重试、降级、熔断等）。

### 5.2 错误表达方式选择

| 场景 | 策略 | 理由 |
|------|------|------|
| 感知数据缺失导致无法完成判定 | `Option<RiskDeterminedEvent>` — 返回 None | "无风险"与"无法判定"是不同的概念，None 明确表达后者，消费方据此决定是否降级处理 |
| 家属在无授权状态下请求对讲/视频/车窗控制 | `Result<Unit, PermissionDenied>` | 调用方需要知道拒绝原因以生成适当的用户提示 |
| 救援机构远程解锁/调取健康档案但未获授权 | `Result<T, AccessDenied>` | 隐私合规要求显式处理未授权访问 |
| 评分时行程尚未结束（状态错误） | `Result<TripScore, ScoringError>` | 属于逻辑错误，应阻止操作并向上反馈 |
| 报告生成时间范围内无数据 | `Result<ReportData, ReportError>` 或空报告 | 范围内无数据是正常业务结果，应返回明确提示而非异常 |
| VehicleStateBuffer 回取窗口超出缓冲覆盖范围 | `Result<Array<VehicleStateSnapshot>, BufferError>` | 缓冲未覆盖请求窗口是可预期的业务情形，调用方（DS-06）需据此降级上报（见决策 14） |
| 活体检测 60 秒窗口内微动消失 | `Option<LifeDetectedEvent>` — 返回 None | 正常业务结果，非错误 |
| 领域事件发布失败（outbox 模式下持久化失败） | 抛异常，由基础设施层兜底（重试 + 死信） | outbox 模式将事件持久化与聚合根状态更新置于同一事务——若事务提交失败（含事件持久化失败），则聚合根状态更新也不生效，事务回滚后由调用方重试。此为 C 类系统级故障，领域层不捕获 |

### 5.3 整体原则

- 领域服务对外接口优先使用 `Option<T>` 表达"可能没有结果"的语义，使用 `Result<T, E>` 表达"可能成功也可能因业务原因失败"的语义。
- 仅在调用方错误使用 API（如传入非法参数、违反前置条件）时使用异常。
- 领域事件的发布采用 outbox 模式：事件持久化与聚合根状态更新在同一事务中提交，事务提交成功后事件才对消费方可见。若持久化失败，事务回滚，聚合根状态不变，事件不发布——保证"状态变更"与"事件已发布"的一致性。消费方自行负责事件处理的错误与重试。
- post-transaction 阶段的异步投递失败（outbox 表已持久化但消息代理投递失败）由基础设施层的 outbox 投递器负责重试，不属于领域层职责。

### 5.4 边界条件处理策略

**(1) 边缘侧断网时 Trip 本地持久化与云端同步一致性**：边缘侧采用"本地优先"——感知判定结果与 Trip 状态变更先在边缘侧本地持久化（保证断网时安全告警链路成立），再异步上报云端。云端同步采用**重试 + 幂等去重**：每条上报记录携带稳定的业务幂等键（如 Trip 标识 + 告警标识/事件序号），云端按幂等键去重，使断网恢复后的批量重传不产生重复行程/重复告警；上报失败不回滚本地状态，仅排队重试。该机制属基础设施层职责，领域层通过为聚合根/事件提供稳定标识予以支撑。

**(2) 驾驶员注销/账号删除时的监护关系清理与历史数据处理**：Driver 注销触发——① 清理其与 SystemAccount（家属）的监护关系，对在途的常规家属权限发出 FamilyAccessRevokedEvent 收束接入；② 历史行程数据（Trip、SafetyAlertEvent、评分）按合规策略处理：默认对统计/审计所需数据做**匿名化保留**（解除与 Driver 身份的关联但保留聚合统计价值），对隐私敏感数据（DriverHealthProfile、RoadRageVoiceRecord 等）按隐私规则**删除或脱敏**；③ 清理动作以领域事件驱动各模块异步收尾，避免跨聚合同步级联删除。

**(3) 毫米波雷达在 60s 判定窗口内短暂中断又恢复**：LifeDetectionService 的 DetectionWindow（DS-05）对信号中断采取"**保持窗口、暂停累计**"策略——短暂中断（小于约定的容差时长）不重置窗口、不清零已累计微动观测，信号恢复后在原窗口内继续累计；仅当中断超过容差时长（视为判定不可靠）才重置窗口并重新计时，避免因瞬时丢帧导致误判取消或误判成立。容差阈值作为 DetectionWindow 的判定参数由会话上下文携带，服务保持纯函数语义。

---

## 六、并发设计

### 6.1 整体线程模型

系统部署拓扑分为两个主要运行时环境：

**边缘侧（车载终端）**：单机部署，核心判定链路运行在有限 CPU 资源上。感知数据采集、风险判定、HMI 干预指令生成在同一个进程中按流水线方式串行处理——保证从判定到指令下发的 500ms 端到端时延在确定的计算资源上可度量。边缘侧不承受高并发压力，重点在于**确定性实时响应**而非并发吞吐。

**云端侧（华为云）**：Spring Boot 服务部署，承受车队级多车辆并发数据上报、多家属并发查询、多管理员并发看板操作。云端侧需要处理并发请求，但并发控制的重点在基础设施层（数据库连接池、缓存、消息队列），领域层本身保持无状态。

### 6.2 共享状态管理策略

**聚合根级别的乐观并发控制**：Trip、Driver、Vehicle 等聚合根的持久化更新采用乐观锁（版本号），避免多请求并发修改同一聚合根时产生的写冲突。冲突发生时由基础设施层重试或向调用方返回冲突错误。

**边缘侧状态管理**：边缘侧的"当前行程"（活跃 Trip 聚合）是单线程写入的——一次只有一次行程，感知数据按时间序列顺序到达。因此边缘侧的 Trip 聚合不存在并发写冲突，无需加锁。流式会话上下文（当前活跃风险集、DetectionWindow）亦由该单线程持有，无并发竞争。

**领域事件的发布与消费**：领域事件采用"先持久化事件、再异步投递"的 outbox 模式，确保事件不丢失且发布与聚合根状态更新在同一事务中。事件的消费方（如通知推送、看板刷新）异步处理，不阻塞核心判定链路。

**领域事件总线的实现策略**：仓颉标准库未原生提供领域事件发布/订阅机制，需自定义实现。设计层面区分两种消费链路：

- **边缘侧同步消费**：安全攸关的判定→干预链路（如 InterventionService 消费 RiskDeterminedEvent/RiskResolvedEvent、分心 0.5s 告警），在边缘侧采用**进程内同步回调**——门面产出事件后直接同步调用注册的消费方，确保 ≤500ms（分心 ≤0.5s）端到端时延和断网可用性（见决策 10）。
- **云端侧异步消费**：通知推送、看板刷新、报告生成、家属常态状态快照推送（DS-16）等非实时路径，采用**outbox + 消息队列异步投递**（DriverStatusSnapshot 高频遥测走独立轻量推送通道）模式，消费方独立部署，允许秒级延迟。

### 6.3 并发场景的具体策略

| 场景 | 策略 |
|------|------|
| 家属查询驾驶员当前状态 | 只读查询，无需锁，读已提交即可 |
| 家属端常态状态快照推送 | 单向高频遥测，无共享写状态，走异步推送通道，不与安全链路争用资源 |
| 多个管理员同时刷新车队看板/钻取 | 看板数据聚合为只读查询，缓存层面控制刷新频率（5 分钟），避免重复计算 |
| 评分计算与告警事件并发写入同一 Trip | Trip 聚合根使用乐观锁，冲突时评分计算重试 |
| 边缘侧判定写入与云端同步上报 | 边缘侧先本地持久化再异步上报云端，上报失败不影响本地判定；云端按幂等键去重（§5.4） |
| 家属权限授予/撤销的并发 | 对 SystemAccount 聚合使用乐观锁，权限变更以最后一次成功写入为准 |

---

## 七、设计决策

### 决策 1：以 Trip 而非 Driver/Vehicle 为核心聚合根

**理由**：系统的核心业务——风险判定、干预执行、评分计算——全部以"一次行驶行程"为上下文发生。Trip 是感知数据、告警事件、生理快照的自然汇聚点。以 Trip 为核心聚合根使得一次行程的所有关联数据在同一事务边界内保持一致，查询也最为自然。

**仓颉语言考量**：Trip 内部持有 PhysiologicalSnapshot 集合和 SafetyAlertEvent 标识集合，在仓颉中通过泛型集合类型表达。聚合根的标识使用仓颉的 `struct` 或值类型确保标识的不可变性。

---

### 决策 2：RiskDeterminationService 作为门面，内部委托子判定服务

**理由**：疲劳、分心、路怒三类判定均由**持续到达的流式感知**（DMS 视觉、生理、语音）驱动，各有独立的判定条件和阈值（BR-01、BR-03、分心规则），若将它们全部塞入一个判定服务将导致职责过重。采用门面+委托模式——RiskDeterminationService 负责流式感知数据的路由与融合判定结果的汇总，具体的判定逻辑委托给各自的子领域服务。这既保持了对调用方（流式感知通道）的统一入口，又保证了各判定规则的独立演进。

**门面委托范围（明确边界）**：本门面的委托子服务**仅限**流式融合判定的三个服务——**FatigueDeterminationService、DistractionDetectionService、RoadRageDeterminationService**。BR-02 活体遗留与 BR-06 碰撞失能**不在委托范围内**：二者是**事件触发型判定**，被设计为独立领域服务（DS-05、DS-06），自行产出 LifeDetectedEvent / EmergencyActivatedEvent，理由详见决策 13。

**仓颉语言考量**：各子判定服务的接口定义为仓颉 `interface`——这使得不同判定算法可互替换（如边缘侧使用轻量规则判定，云端使用 AI 模型判定），符合接口隔离原则。

**补充约束**：作为门面委托子服务的（仅）三个流式判定服务**不得直接调用其他模块的领域服务或直接产出领域事件**——判定结果一律返回给 RiskDeterminationService 门面，由门面统一产出 RiskDeterminedEvent / RiskResolvedEvent。跨模块协作由消费方订阅事件完成。该约束**不适用于** DS-05/DS-06 这两个独立事件触发型服务。

---

### 决策 3：SafetyAlertEvent 为实体而非值对象，但非聚合根

**理由**：告警事件有独立标识（告警 ID），可在 Trip 之外被独立查询（如车队管理员查询全队告警历史），且告警有独立生命周期（可被归档、统计）。但它不是聚合根，因为每个告警都产生于一个具体的 Trip 上下文——告警的创建与 Trip 的告警列表变更应在同一事务内完成。

**补充设计约束（CQRS 投影）**：写侧仍通过 Trip 聚合根访问 SafetyAlertEvent 以保持事务一致性；读侧（跨行程告警查询、看板聚合、钻取）使用独立只读投影，避免穿透聚合根加载大量数据。

---

### 决策 4：Permission 为值对象而非实体

**理由**：权限由一组可执行操作定义，修改权限意味着创建新的权限集合，而非修改原有权限的"状态"。将 Permission 设计为值对象（不可变）避免了"修改权限时可能影响正在进行的对讲会话"这类并发问题——每次授予或撤销权限都创建新的 Permission 实例，旧实例在会话结束后自然过期。

---

### 决策 5：BR-04 隐私保护不设计为聚合约束，而设计为领域服务

**理由**：隐私保护是一个横切关注点——它跨越数据采集（DMS 脱敏）、数据上云（过滤原始图像）、语音存证（加密留存与到期清除）、数据调取（授权审计）。将它建模为单一聚合根的约束既不合理，也容易遗漏。独立为 PrivacyProtectionService 领域服务，在各数据流动路径上作为守门人校验，更符合横切关注点的本质。

**补充**：PrivacyProtectionService 通过消费 RiskDeterminedEvent（AlertType=ROAD_RAGE，触发录制）与 RiskResolvedEvent（AlertType=ROAD_RAGE，停止录制）获知路怒判定的成立与解除，遵循模块间解耦原则，不产生对 `domain.risk` 模块的直接编译期依赖。

---

### 决策 6：路怒语音存证的生命周期归属

**理由**：RoadRageVoiceRecord 的创建时机与路怒判定（BR-03）紧密耦合，其清除策略又受隐私规则约束。它设计为与 SafetyAlertEvent（路怒类型）关联的实体——但不属于 Trip 聚合（因为存证存储于边缘侧、不上云，与 Trip 的存储策略不同）。其生命周期管理由 PrivacyProtectionService 负责，录制起于 RiskDeterminedEvent、止于 RiskResolvedEvent，通过领域事件与告警事件保持松耦合。

---

### 决策 7：评分模块与通知模块的耦合方式

**理由**：BR-05 要求"评分 <60 时自动向管理员推送绩效预警"。设计上，ScoringService 在评分完成后发出 TripScoredEvent；若评分 <60，额外发出 PerformanceWarningEvent。通知推送模块消费 PerformanceWarningEvent 完成预警推送。评分服务不直接依赖通知服务——它只产出事件、不关心谁消费，解耦程度最大化。

---

### 决策 8：聚合间引用一律使用标识而非对象引用

**理由**：Trip 引用 Driver 和 Vehicle 时通过标识（ID）而非对象引用。这是 DDD 聚合设计的核心原则。使用标识引用使聚合可以独立加载和持久化，避免将多个聚合锁在同一事务中，也为未来的微服务拆分预留了清晰的边界，并为断网重传去重（§5.4）提供稳定幂等键。

**仓颉语言考量**：聚合标识使用仓颉的 `struct` 类型——轻量、栈分配、不可变，适合作为跨聚合引用。

---

### 决策 9：使用 enum 表达有限的、稳定的分类体系

**理由**：RiskLevel（L1/L2/L3）、AlertType、SensorStatus、AccountRole（FAMILY/MANAGER）、StatusColor（绿/黄/红）等取值集合固定且极少变化。使用仓颉 `enum` 可确保：编译期穷尽检查、类型安全（不会误将字符串常量当作状态值/角色值）、代码可读性。AccountRole 的引入使 DS-08 权限路由以类型安全分支替代字符串/魔法值比较。

---

### 决策 10：领域事件的异步消费与边缘侧的顺序性

**理由**：边缘侧核心判定的端到端时延要求 ≤500ms（分心 ≤0.5s），判定→干预链路上的事件消费不能引入不可控的异步延迟。设计上，边缘侧的 RiskDeterminedEvent/RiskResolvedEvent → InterventionService 的消费是**同步的**（同一进程内直接调用）。云端侧和通知推送、家属常态状态推送的事件消费才采用异步模式。这体现了"安全链路优先"的原则。

---

### 决策 11：统一感知数据抽象 SensorReading

**理由**：四条感知通道的数据以自然语言分散描述于各判定服务，缺少统一契约。新增 SensorReading 值对象作为统一的感知数据抽象，为各判定服务提供一致的输入接口，使感知通道扩展不影响判定服务接口定义。

---

### 决策 12：值对象在仓颉中的首选类型形态为 struct

**理由**：仓颉中 `class` 为引用类型（默认引用相等），`struct` 为值类型（值相等）。值对象的核心语义是"由属性值定义相等性"，`struct` 天然提供该保证。将值对象（含本版新增的 VO-12~VO-15）标注为 `struct` 可避免误用引用相等导致的业务逻辑错误。若因实现约束需用 `class`，须显式标注"需重写相等性方法使 class 具备值相等语义"。

---

### 决策 13：区分"流式融合判定门面"与"事件触发型独立判定服务"两类判定模型

**理由**：系统的风险判定本质上存在两种截然不同的输入与触发模型：

- **流式数据驱动判定**（疲劳 BR-01、分心、路怒 BR-03）：输入是持续到达的感知流，判定是对当前时间窗内流式特征的连续评估，多通道结果可被有意义地**融合**为单一的 RiskDeterminedEvent。适合门面+委托模式（决策 2）。
- **事件触发型判定**（活体遗留 BR-02、碰撞失能 BR-06）：触发源是离散领域事件，判定逻辑、生命周期和产出事件都与流式融合无关，无法被"融合"进 RiskDeterminedEvent。

**因此本设计将 BR-02、BR-06 明确为独立领域服务**：它们有独立触发条件、独立判定逻辑、独立领域事件，不经 RiskDeterminationService 门面。三类判定事件的 AlertType 取值两两不相交（见 VO-02），从结构上杜绝重复判定事件。

**仓颉语言考量**：独立判定服务与门面子服务同样以仓颉 `interface` 定义行为契约，与门面同处 `domain.risk` 模块，依赖方向仍单向。

---

### 决策 14：事故前车辆状态的滚动缓存归属基础设施层，领域层以端口依赖

**理由**：BR-06 需要在碰撞时刻回取"事故前 30 秒"的 VehicleStateSnapshot，这要求系统在碰撞**之前**就持续采集并缓存近期车辆状态。该"持续采样 + 时间窗滚动缓存"是一种**有状态的、与领域判定正交的技术职责**——交给领域服务会破坏其无状态基线，塞进 Trip 聚合又会让聚合背负高频写入。因此归属**基础设施层**：边缘侧采集组件维护覆盖 ≥30s 的滚动缓冲（ring buffer），领域层只声明依赖接口（端口）**VehicleStateBuffer**。

**端口能力契约（核心方法签名）**：为解除上下游并行开发阻塞，明确 VehicleStateBuffer 端口的核心方法契约：

```
func getSnapshots(tripId: TripId, window: TimeRange): Result<Array<VehicleStateSnapshot>, BufferError>
```

- **输入**：`tripId`（当前活跃行程标识，定位边缘侧对应缓冲）、`window`（TimeRange，事故前 30s 的回取时间窗）。
- **返回**：成功返回该时间窗内按时序排列的 `Array<VehicleStateSnapshot>`。
- **错误语义**：`BufferError` 表达可预期的业务情形，至少含：`WindowNotCovered`（请求窗口超出缓冲保留范围，如行程刚开始不足 30s）、`BufferUnavailable`（缓冲未初始化/采集组件异常）。DS-06 据此降级（如以可得的部分快照上报、或在报告中标注快照不完整），不抛异常。

**仓颉语言考量**：VehicleStateBuffer 以仓颉 `interface` 声明于领域层，基础设施层提供实现并在装配阶段注入 DS-06；`Result`/`Array`/`TimeRange` 均为前序已确认可行的标准类型构造，无新增语言能力诉求。

---

### 决策 15：干预策略采用 AlertType × RiskLevel 二维映射

**理由**：前一轮以单一 RiskLevel→干预的通用映射与各场景行为契约矛盾——同为 L2，疲劳应氛围灯、路怒应环境调节、分心应告警，三者不同，无法用一维 RiskLevel 表达。本版将 DS-07 的干预决策改为 **AlertType × RiskLevel 二维映射**：干预指令集合由"告警类型 + 风险等级"共同确定，与场景 1/3/8 的具体契约严格一致。L1 为预留维度（无规则触发，给出"无干预/仅记录"占位分支以满足穷尽检查），避免死代码同时保留扩展位。

**仓颉语言考量**：二维映射以 AlertType（`enum`）× RiskLevel（`enum`）的组合在 match/查表中分派，编译期可对（AlertType, RiskLevel）组合做穷尽性检查；输出统一为 InterventionInstruction（VO-12，`struct`）集合。

---

### 决策 16：风险解除建模为独立领域事件 RiskResolvedEvent，而非扩展 RiskLevel/AlertType 取值

**理由**：场景 3 需表达"路怒已解除"以驱动停止录制、恢复空调，但 RiskLevel（L1/L2/L3 严重度阶梯）与 AlertType（类别维度）均无"已解除"语义。审查给出两条路径：(a) 为 RiskLevel 增加 RESOLVED 值；(b) 单独定义 RiskResolvedEvent。**本版采纳 (b)**：

- RESOLVED 并非一个"严重度"，将其塞入 RiskLevel 会污染该枚举的语义纯粹性（审查自身亦指出 RESOLVED "不属于 L1/L2/L3 体系"），且每个消费 RiskLevel 的 match 分支都须额外处理一个非严重度取值。
- 独立的 RiskResolvedEvent 以**事件类型本身**区分"新告警"与"解除信号"，消费方订阅语义清晰，无需在事件载荷里做特例判断，从根上杜绝"录制不停止、空调持续低温"。
- RiskResolvedEvent 复用既有 AlertType 标识"哪类风险解除"，对 FATIGUE/DISTRACTION/ROAD_RAGE 通用，可自然支撑 InterventionService 恢复干预、PermissionService 参与常规撤销（决策见 DS-08、场景 7）。

**仓颉语言考量**：RiskResolvedEvent 为新增领域事件类型，复用 §6.2 既定事件总线（边缘侧同步、云端异步），无新增语言能力诉求；解除判定所需的"当前活跃风险集"为会话级状态，由会话上下文持有（DS-01 解除判定的状态边界说明），门面保持纯函数。

---

## 修订说明（a_v2_v1）

> 本轮针对诊断（b_v1_diag_v1）+ 质询（LOCATED，b_v1_challenge_v1）确认的 12 个问题（2 严重 / 5 一般 / 5 轻微）逐项修复。其中问题 1、2 为阻塞下游编码的严重逻辑矛盾，已确保 DS-07 / Scene 1 / VO-01 / VO-02 与新增的 RiskResolvedEvent 之间内部一致。本轮全部采纳，无分歧。

| 审查意见 | 修改措施 |
|---------|---------|
| **1【严重】DS-07 通用 RiskLevel→干预映射与场景行为契约矛盾** | 采纳。DS-07 改为 **AlertType × RiskLevel 二维映射**，以矩阵明确"疲劳 L2=氛围灯橙 / 路怒 L2=环境调节 / 分心 L2=告警 / 疲劳 L3=语音+震动+双闪+CAN"，与场景 1/3/8 严格一致；新增决策 15 阐述二维化理由；概述新增"风险等级与干预映射总览约定"。消除单一映射与场景契约的矛盾。 |
| **2【严重】路怒解除事件语义在现有类型系统中无法表达** | 采纳路径 (b)。新增独立领域事件 **RiskResolvedEvent**（携带 AlertType 标识解除类别），不污染 RiskLevel/AlertType 枚举；§3.5 事件表新增该事件；DS-01/DS-04/DS-13 增加解除产出与消费；场景 3 步骤 6 改为产出/消费 RiskResolvedEvent 以停止录制、恢复空调；VO-01/VO-02 增加解除语义说明；新增决策 16 阐述选型理由（含对路径 a 的取舍）。 |
| **3【一般】分心检出缺少独立行为契约场景** | 采纳。§四新增**场景 8（分心检出与告警）**：SensorReading(DMS 视觉)→DS-03 判定(L2)→门面汇总 RiskDeterminedEvent(DISTRACTION,L2)→InterventionService 告警，显式标注 0.5s 时延约束与边缘侧同步链路。 |
| **4【一般】OTA 升级管理缺少行为契约场景** | 采纳。§四新增**场景 9（OTA 固件升级全流程）**：明确状态机阶段（待下发→传输中→校验中→已就绪→升级中→完成/回滚）、状态转换触发条件（下发失败→重试、校验失败→回滚、刷写失败→回滚）、断点续传重试语义、安全回滚不变式；DS-15 补充并新增 OTAUpgradeFailedEvent。 |
| **5【一般】VehicleStateBuffer 端口接口契约未定义** | 采纳。决策 14 补充端口核心方法签名 `getSnapshots(tripId: TripId, window: TimeRange): Result<Array<VehicleStateSnapshot>, BufferError>`，明确输入参数、返回类型与错误语义（WindowNotCovered / BufferUnavailable 及 DS-06 降级策略）；VO-09、DS-06、场景 4、§5.2 同步引用。 |
| **6【一般】值对象目录遗漏 InterventionInstruction 与 RescueReport** | 采纳。§3.3 新增 **VO-12 InterventionInstruction**（指令类型枚举、目标设备标识、参数映射、优先级）与 **VO-13 RescueReport**（GeoLocation、生命体征摘要、VehicleStateSnapshot 集合、健康档案摘要）；DS-07/DS-12 引用之。 |
| **7【一般】行为契约场景覆盖不完整（约 67%）** | 采纳并超额覆盖。除场景 8（分心）、场景 9（OTA）外，新增**场景 10（报告生成 15s SLA + 家属常态同步）、场景 11（远程车窗/车门控制授权与执行）、场景 12（看板钻取交互）**，使 16 个领域服务均有场景覆盖，带显式性能/时序约束者（分心 0.5s、报告 15s、OTA 状态机、常态同步 ≥1Hz/≤2s）均有可验证契约。 |
| **8【轻微】DS-07 中 L1 级别干预缺少触发源** | 采纳。VO-01 增加"L1 预留等级声明"、DS-07 二维映射对 L1 给出"无干预/仅记录"占位分支、概述与决策 15 注明 L1 本期无触发路径但保留以备扩展，明确其为显式占位而非死代码。 |
| **9【轻微】SystemAccount 角色（FAMILY/MANAGER）缺少类型定义** | 采纳。§3.3 新增 **VO-14 AccountRole** `enum`（FAMILY \| MANAGER）；AR-04 关联角色字段；DS-08 权限判定与通知路由改为基于 AccountRole 的类型安全分支；决策 9 补充说明。 |
| **10【轻微】家属常规状态推送机制缺少设计描述** | 采纳。新增 **VO-15 DriverStatusSnapshot**（Driver 标识、状态色绿/黄/红、时间戳）与 **DS-16 DriverStatusBroadcastService**（≥1Hz 周期采样、≤2s 上报）；§四场景 10b 描述常态同步契约；§2.1 domain.family、§6.2/§6.3 补充其异步遥测链路与并发策略。 |
| **11【轻微】家属权限常规撤销路径不完整** | 采纳。DS-08 补充常规自动撤销状态机——L3 风险下降不再持续（后续 < L3 等级或 RiskResolvedEvent）时自动撤销并发出 **FamilyAccessRevokedEvent**（§3.5 新增该事件）；场景 7 改为"授予—撤销闭环"，区分常规撤销/物理遮挡撤销/高危激活的收束路径。 |
| **12【轻微】异常场景与边界条件覆盖不足** | 采纳。§五新增 **5.4 边界条件处理策略**：(1) 边缘侧断网 Trip 本地优先持久化 + 幂等键去重的云端同步一致性；(2) 驾驶员注销/账号删除的监护关系清理（FamilyAccessRevokedEvent）与历史数据匿名化保留/敏感数据删除脱敏处理；(3) 毫米波雷达短暂中断的"保持窗口、暂停累计、超容差重置"窗口策略。§6.3 同步补充断网去重并发条目。 |

---

DESIGN_WRITTEN:D:\软件测试\redeliberations\202606281504_vehicle-safety-ood\a_v2_design_v1.md
主Agent请勿阅读产出文件内容，直接将路径转发给相关方。
