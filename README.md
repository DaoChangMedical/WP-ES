**概述**

-   在 InterSystems Health Connect中实现一条"患者筛选→风险评分→结果输出"的端到端集成流程。

-   主要环节：①FHIR 解析与预处理；②按《华法林疗效路径患者入组筛选规则》执行纳排标准；③对入组患者调用《华法林患者风险分层模型接口》三类 API（出血、血栓、综合风险）；④统一输出和监控。

**华法林疗效路径患者入组筛选规则**

一、患者纳入标准

华法林治疗适应症：患者诊断代码符合以下任意主要或次要适应症：

主要适应症（首选）：

1.  机械心脏瓣膜置换术（此为华法林抗凝唯一选择）

2.  房颤伴有明显二尖瓣狭窄（此为华法林抗凝唯一选择）

3.  抗磷脂抗体综合征（首选抗凝剂）

4.  肥厚型心肌病（首选抗凝剂）

次要适应症（补充）：

1.  深静脉血栓和肺栓塞

2.  心腔内血栓形成

二、 患者排除标准

患者符合以下任一条件，系统将排除出潜在入组队列，确保研究的安全性与有效性。

2.1 严重出血风险

2.1.1 活动性出血或高风险出血病灶

A诊断代码：

-   胃溃疡

-   十二指肠溃疡

-   呕血

-   黑便

-   未特指的胃肠道出血

B病例报告关键词：

-   活动性出血

-   正在出血

-   血便

-   呕血

-   活动性胃肠道溃疡

-   消化道出血

-   咯血

-   血尿

2.1.2 3个月内严重出血史

**颅内出血史**

A诊断代码

-   非创伤性蛛网膜下腔出血

-   非创伤性脑内出血

-   其他非创伤性颅内出血

-   颅内出血伴有颅脑损伤

B病例报告关键词：

-   颅内出血

-   脑出血

-   硬膜下出血

-   蛛网膜下腔出血

**消化道大出血史**

A诊断代码

-   未特指的胃肠道出血（重度）

-   肛门直肠出血

-   腹腔出血

B病例报告关键词：

-   消化道大出血

-   上消化道出血

-   下消化道出血

-   因出血住院

2\. 严重功能障碍：

1.  肝功能障碍：肝硬化或肝功能Child-Pugh C级

2.  严重肾功能障碍： eGFR\<15 ml/min/1.73m²

3.  血小板减少症：血小板计数\<50×10⁹/L

4.  恶性肿瘤史

3\. 特殊人群：

1.  妊娠或哺乳期妇女。

2.  预期寿命\<12个月

3.  用药与试验冲突：

a)  DOACs被明确证明更适合作为一线选择的适应症

-   非瓣膜性房颤（NVAF）

b)  合并抗血小板治疗

c)  参与其他临床试验

**系统架构与组件**

-   **输入通道**：FHIR Inbound Adapter (HS.FHIRServer.Adapter.Inbound) 接收HTTPS POST的患者数组；使用Business Service BS.PatientIntake校验Bundle类型、Schema、必填段。

-   **预处理与数据映射**：Business Process (BPL/BPMN) BP.PatientScreening将FHIR资源映射到内部对象：诊断（ICD10/ICD9）、实验室数据、药物、人口学字段；利用 FHIR.Search 查询历史检验（INR、eGFR、血小板）并写入临时表。

-   **纳入标准引擎**：在BP中嵌入DTL或使用规则引擎（Business Rule Warfarin.PatientIn.JoinGroup.Identification）判断主要/次要诊断是否命中：

-   支持多诊断来源：FHIR Condition.code, Encounter.diagnosis, Procedure.code。

-   **排除标准引擎**：Business Rule Warfarin.PatientIn.JoinGroup.Identification维护多维条件：

-   活动性出血/高风险病灶：ICD+自由文本（经NLP或关键词匹配Observation.note、DocumentReference).

-   3个月内重大出血（颅内、消化道）通过时间窗过滤Encounter/Condition recordedDate。

-   严重功能障碍：

-   肝：Condition=肝硬化或Observation(Child-Pugh)；

-   肾：最近eGFR\<15(Observation.code=eGFR)；

-   血小板：Observation platelet\<50x10\^9/L；

-   恶性肿瘤史：Condition.category=oncology。

-   特殊人群：妊娠/哺乳（Condition/Observation）、预期寿命\<12 月（CarePlan.goal 或肿瘤分期）、用药冲突（MedicationStatement中DOACs、抗血小板）及临床试验参与标记。

<img width="656" height="658" alt="image" src="https://github.com/user-attachments/assets/d843ff02-f578-422c-8f24-c7423c07c167" />

-   **风险评分调用**：对通过筛选的患者，Business Operation BO.RiskModel以患者ID先后调用三个API：

1.  GetBleeding(patientId)→返回HAS-BLED 分值/等级；

2.  GetThrombosis(patientId)→返回CHA2DS2-VASc；

3.  GetRiskIndex(patientId, thrombosisScore, bleedScore)→结合INR波动、PGx、依从性，产出综合风险。

-   调用方式：HS.Gateway.HTTPOutboundAdapter或内嵌tkMakeServerCall；处理超时/重试、接口鉴权（Token/Session）。

-   **输出与存储**：

-   将筛选结果与风险等级写入Custom.Warfarin.PatientRisk表，字段包括PatientId,EligibilityStatus,ExclusionReason\[\],BleedScore,ThrombosisScore,RiskIndex.

-   对外可通过FHIR Observation批量返回，或生成 CSV/REST JSON。

**业务流程**

1.  **数据接收**：外部系统推送数据

2.  **结构校验**：校验结构是否正常完整。

3.  **纳排判定**：Warfarin.PatientIn.JoinGroup.Identification→true?；若false直接输出"不满足纳入"。若true，再进入Warfarin.PatientIn.JoinGroup.Identification；命中任一排除→输出"不合格"并记录理由。

<img width="865" height="487" alt="image" src="https://github.com/user-attachments/assets/f322a64b-6f58-4e7e-abeb-70e19a90b94c" />


4.  **风险评分**：对合格患者顺序调用GetBleeding、GetThrombosis、GetRiskIndex；若某接口失败，标记"待人工复核"并触发重试队列。

5.  **结果整合**：生成RiskAssessment对象，推送至下游（FHIR/数据库），并返回响应。

**数据治理与安全**

-   **数据质量**：

-   缺失关键字段（诊断、检验）时，返回校验错误，同时写入异常队列。

-   统一编码映射表（ICD10、LOINC）供规则使用。

-   **权限与隐私**：

-   基于InterSystems安全域配置OAuth或JWT鉴权；

-   API 调用记录 PHI，需启用审计日志与TLS；

-   对外导出的数据可脱敏（只保留内部ID）。

**后续优化方向**

-   引入NLP服务（IRIS NLP）提升病例关键词识别精度；

-   对INR波动、依从性等字段可直接在HealthShare Analytics中计算缓存，减少BO负载；

-   构建前端仪表盘展示筛选通过率、风险等级分布，辅助临床运营。
