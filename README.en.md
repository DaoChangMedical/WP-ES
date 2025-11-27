**Overview**

- Build an end-to-end integration flow in InterSystems Health Connect that covers “patient screening → risk scoring → result output.”
- Core steps: (1) FHIR parsing and preprocessing; (2) applying the inclusion/exclusion criteria from “Warfarin Therapeutic Pathway Patient Enrollment Rules”; (3) invoking the three APIs (bleeding, thrombosis, composite risk) defined in “Warfarin Patient Risk Stratification Model Interface” for all eligible patients; (4) unified output and monitoring.

**Warfarin Therapeutic Pathway Patient Enrollment Rules**

1. **Inclusion Criteria**

   Warfarin indications: the patient’s diagnosis codes must match any of the following primary or secondary indications.

   - Primary indications (preferred):
     1. Mechanical heart valve replacement (warfarin is the only anticoagulant choice).
     2. Atrial fibrillation with significant mitral stenosis (warfarin is the only anticoagulant choice).
     3. Antiphospholipid antibody syndrome (first-line anticoagulant).
     4. Hypertrophic cardiomyopathy (first-line anticoagulant).
   - Secondary indications (supplemental):
     1. Deep vein thrombosis and pulmonary embolism.
     2. Intracardiac thrombus formation.

2. **Exclusion Criteria**

   A patient is removed from the potential cohort if they meet any of the following, to ensure safety and validity.

   2.1 Serious bleeding risk  
   2.1.1 Active bleeding or high-risk bleeding lesions  
   - Diagnosis codes: gastric ulcer, duodenal ulcer, hematemesis, melena, unspecified gastrointestinal hemorrhage.  
   - Case report keywords: active bleeding, ongoing bleeding, hematochezia, hematemesis, active GI ulcer, GI bleeding, hemoptysis, hematuria.

   2.1.2 History of severe bleeding within 3 months  

   **Intracranial bleeding history**  
   - Diagnosis codes: non-traumatic subarachnoid hemorrhage, non-traumatic intracerebral hemorrhage, other non-traumatic intracranial hemorrhage, intracranial bleeding with head injury.  
   - Case report keywords: intracranial hemorrhage, cerebral hemorrhage, subdural hemorrhage, subarachnoid hemorrhage.

   **Gastrointestinal major bleeding history**  
   - Diagnosis codes: severe unspecified GI bleeding, anorectal bleeding, intra-abdominal bleeding.  
   - Case report keywords: major GI bleeding, upper GI bleeding, lower GI bleeding, hospitalization due to bleeding.

   2.2 Severe organ dysfunction  
   - Hepatic impairment: cirrhosis or Child-Pugh class C.  
   - Severe renal impairment: eGFR < 15 ml/min/1.73 m².  
   - Thrombocytopenia: platelet count < 50 × 10⁹/L.  
   - Malignancy history.

   2.3 Special populations  
   - Pregnant or breastfeeding women.  
   - Life expectancy < 12 months.  
   - Medication or trial conflicts:
     - Indications where DOACs are clearly preferred as first-line (e.g., non-valvular atrial fibrillation).  
     - Concomitant antiplatelet therapy.  
     - Participation in other clinical trials.

**System Architecture and Components**

- **Input channel**: FHIR Inbound Adapter (`HS.FHIRServer.Adapter.Inbound`) receives HTTPS POST bundles; Business Service `BS.PatientIntake` validates bundle type, schema, and required fields.
- **Preprocessing and data mapping**: Business Process (`BPL/BPMN`) `BP.PatientScreening` maps FHIR resources to internal objects (diagnoses, labs, meds, demographics) and uses FHIR Search to gather historical labs (INR, eGFR, platelets) into temp tables.
- **Inclusion rule engine**: Within the BP, use DTL or Business Rule `Warfarin.PatientIn.JoinGroup.Identification` to detect primary/secondary diagnoses, supporting multiple sources (`Condition.code`, `Encounter.diagnosis`, `Procedure.code`).
- **Exclusion rule engine**: The same Business Rule maintains multidimensional conditions:
  - Active bleeding/high-risk lesions via ICD + free text (NLP or keyword match in `Observation.note`/`DocumentReference`).
  - Major bleeding within 3 months (intracranial/GI) by filtering `Encounter`/`Condition.recordedDate`.
  - Severe dysfunction: liver (`Condition`=cirrhosis or `Observation` Child-Pugh), renal (latest eGFR <15), platelet count <50×10⁹/L, cancer history (`Condition.category`=oncology).
  - Special populations: pregnancy/lactation, life expectancy <12 months (`CarePlan.goal` or oncology staging), medication conflicts (`MedicationStatement` entries for DOACs/antiplatelets) or clinical trial flags.
<img width="656" height="658" alt="image" src="https://github.com/user-attachments/assets/e978a446-1290-46da-aa0e-a344bd2ff106" />

- **Risk scoring calls**: For eligible patients, Business Operation `BO.RiskModel` sequentially calls:
  1. `GetBleeding(patientId)` → HAS-BLED score/grade.
  2. `GetThrombosis(patientId)` → CHA2DS2-VASc score.
  3. `GetRiskIndex(patientId, thrombosisScore, bleedScore)` → composite risk using INR variability, PGx, adherence.
  - Invocation via `HS.Gateway.HTTPOutboundAdapter` or embedded `tkMakeServerCall`; handle timeout/retry and API security (token/session).

- **Output and storage**: Persist results in `Custom.Warfarin.PatientRisk` (fields: `PatientId`, `EligibilityStatus`, `ExclusionReason[]`, `BleedScore`, `ThrombosisScore`, `RiskIndex`). Expose externally via FHIR `Observation`, CSV, or REST JSON.

**Business Process**

1. **Data ingestion**: external systems push bundles.
2. **Structural validation**: ensure schema completeness.
3. **Inclusion/exclusion**: run `Warfarin.PatientIn.JoinGroup.Identification`; if false → return “Not eligible.” If true, continue to exclusion check; any hit → “Disqualified” with reasons.
<img width="865" height="487" alt="image" src="https://github.com/user-attachments/assets/e7b28d13-b341-4345-8ccc-ebffb123d3c3" />

4. **Risk scoring**: call `GetBleeding`, `GetThrombosis`, `GetRiskIndex`. On API failure, flag “Pending manual review” and enqueue retry.
5. **Result consolidation**: build `RiskAssessment`, send downstream (FHIR/database), and return response.

**Data Governance and Security**

- **Data quality**:
  - If key fields (diagnoses, labs) are missing, return validation errors and log to an exception queue.
  - Maintain unified code maps (ICD-10, LOINC) for rule evaluation.

- **Security & privacy**:
  - Configure OAuth/JWT within InterSystems security domains.
  - Enable audit logging and TLS for PHI-bearing API calls.
  - Mask outbound data when necessary (retain only internal IDs).

**Future Enhancements**

- Integrate NLP services (e.g., IRIS NLP) to improve keyword detection in clinical narratives.
- Precompute INR variability and adherence metrics in HealthShare Analytics to reduce BO workload.
- Build dashboarding to track enrollment pass rates and risk distribution to aid clinical operations.
