# 再审议判定报告（v10）

## 判定结果

RETRY

## 判定理由

组件B诊断报告（b_v10_diag_v1.md）识别出 7 个问题：3 个一般等级（问题1：FamilyManualRescueRequestedEvent 自循环逻辑矛盾、问题2：错误类型未在值对象目录中形式化定义、问题3：ActiveRiskSet 未形式化为值对象类型）和 4 个轻微等级（问题4~7）。质询报告（b_v10_challenge_v1.md）确认为 LOCATED，全部 7 个问题证据充分、逻辑一致、覆盖完备，审查结论可信。

组件B内部循环实际轮次 1 < 最大轮次 12，提前终止且审查被确认（LOCATED）。根据判定标准，审查报告包含一般等级问题（3 个），不满足 PASS 条件（需不含严重或一般等级问题，或均为轻微等级），应判定为 RETRY。

## 需要解决的问题

- **问题描述**：FamilyManualRescueRequestedEvent 的消费路径存在设计矛盾——DS-12 EmergencyRescueService 既是该事件的产出者（场景 4b 步骤 5）又被列为消费方（"组装 RescueReport 并投递至救援中心"），构成自循环，且与步骤 4 已完成的 RescueReport 组装动作逻辑冗余
- **所在位置**：§3.5 事件表 FamilyManualRescueRequestedEvent 条目（第 874 行）、§四 场景 4b 步骤 4-7（第 1088-1097 行）
- **严重程度**：一般
- **改进建议**：二选一：(a) DS-12 在 triggerManualRescue 方法内直接调用 RescueReportPort 投递 RescueReport，随后产出 FamilyManualRescueRequestedEvent 仅用于通知家属 APP（消费方移除 EmergencyRescueService）；(b) 新增轻量领域服务 RescueReportDeliveryService 作为独立消费方，DS-12 仅产出事件、RescueReportDeliveryService 消费后执行投递

- **问题描述**：DeterminationError、DetectionError、BufferError、PersistenceError、QueryError、ReportError、NotificationError、RescueReportError、MediaSessionError、OTADeliveryError、OTAUpgradeError、PrivacyError、SelfCheckError、EventPublishError 等十余种错误类型均以內联方式描述于各领域服务接口契约或端口契约中，§3.3 值对象目录无任何错误类型的独立 VO 条目，导致无统一索引、共享类型定义不明、下游消费者缺中心化引用
- **所在位置**：§3.3 值对象目录（全章）、§3.4 各领域服务接口契约、§3.6 事件总线契约、§3.7 仓储接口契约、决策 14/19/20/21
- **严重程度**：一般
- **改进建议**：在 §3.3 新增 VO 条目或独立子节（如"错误类型枚举"），对全部错误类型做中心化定义。各领域服务接口契约中仅引用类型名，不再以内联方式重复描述。建议至少为 PersistenceError、DeterminationError、BufferError 三类高频复用错误类型提供正式 enum 取值定义

- **问题描述**：EdgeSessionContext 中 ActiveRiskSet（当前活跃风险集）以 `Map<AlertType, RiskLevel>` 松散标注，未如 DetectionWindow（VO-20）等类似会话级状态一样形式化为独立值对象类型，缺少不变量约束和类型安全的状态操作契约
- **所在位置**：§6.2 边缘侧会话上下文（第 1285 行）、DS-01 解除判定说明（第 491 行）、DS-16 职责描述（第 770-772 行）
- **严重程度**：一般
- **改进建议**：参照 VO-20 DetectionWindow 的建模方式，在 §3.3 新增 VO-24 ActiveRiskSet 值对象（struct），定义其不可变更新契约和不变量约束，明确由 EdgeSessionContext 持有，作为 DS-01 判定门面的输入/输出参数和 DS-16 的读取来源

- **问题描述**：DS-17 DrivingBehaviorTrackingService 的 `onHardBrakingDetected` / `onHardAccelerationDetected` 回调参数不含 Trip 标识，设计也未描述 DS-17 如何获取当前活跃 Trip，阻塞编码实现
- **所在位置**：DS-17 协作说明（第 786-789 行）、决策 20 端口方法签名（第 1546-1547 行）
- **严重程度**：轻微
- **改进建议**：二选一：(a) 在决策 20 的回调方法签名中增加 `tripId: TripId` 参数；(b) 在 DS-17 协作说明中明确通过 EdgeSessionContext 获取当前活跃 TripId

- **问题描述**：E-03 DriverHealthProfile 作为关键医疗数据实体，缺少生命周期管理描述（创建时机、更新机制、负责创建/更新的领域服务）
- **所在位置**：E-03 DriverHealthProfile（§3.2，第 184-193 行）、AR-02 Driver 协作关系（第 97-104 行）
- **严重程度**：轻微
- **改进建议**：在 E-03 协作关系中补充档案的创建时机（建议 Driver 聚合创建时同步创建空档案）和更新路径，或在 AR-02 协作关系中新增相关说明

- **问题描述**：OTAUpgradeStatus 状态机构成要素分散于五处（VO-19 + 四个方法契约 + 场景 9），缺少统览性的合法状态转换表或状态机图，存在遗漏非法转换校验的风险
- **所在位置**：VO-19 OTAUpgradeStatus（§3.3，第 408-422 行）、DS-15 方法契约（第 761-766 行）、场景 9（第 1164-1176 行）
- **严重程度**：轻微
- **改进建议**：在 VO-19 或 DS-15 中新增合法状态转换矩阵，或明确声明除四个方法描述的状态转换路径外任何其他跃迁均为非法

- **问题描述**：边缘侧与云端侧仓储实现差异未在仓储接口契约中体现——单一契约掩盖了边缘侧单线程无锁与云端乐观锁的本质差异
- **所在位置**：§3.7.1 TripRepository 接口契约（第 944-951 行）、§5.4(1) 边界条件（第 1256 行）、§6.2 边缘侧状态管理（第 1278 行）
- **严重程度**：轻微
- **改进建议**：在 §3.7 仓储设计原则或 §5.4(1) 中补充说明边缘侧与云端侧可各自提供独立的 TripRepository 实现，两实现遵循相同接口契约但并发策略不同
