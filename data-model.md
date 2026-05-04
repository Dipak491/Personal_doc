# OnlineRTS — Data Model & Entities Reference

Entities live in the `OnlineRTS_Entities` module (`com.pmc.entities`). They are scanned by Hibernate from that module. This file documents the key entities and their relationships as used in `OnlineRTS_Controller`.

---

## Core Application Entities

### ApplicationDetail
The central entity. Every citizen application is one `ApplicationDetail` row.

| Field | Type | Notes |
|---|---|---|
| `appId` | Long | PK |
| `appTokenNo` | String | Citizen-facing application number |
| `appStatus` | String | `PENDING`, `APPROVED`, `REJECTED`, `QUERY`, `CHALLAN_GENERATED`, `CREATED`, `FORWARD_WITH_REJECTION` |
| `appPamId` | ParentAppMaster | The service being applied for |
| `appParentPamId` | Long | Parent service ID (denormalized) |
| `appCurrentUsmId` | Users | Currently assigned officer |
| `escalationDetails` | EscalationDetails | Current escalation stage |
| `appPaymentStatus` | String | Payment status |
| `appDogLicesneApprovalWardOffice` | Character | `'D'` = approved by ward office (dog license) |
| `pethId` | PethNameDetails | Peth/locality (used for water dept signature lookup) |
| `version` | Long | Document version |

Named queries used:
- `fetchApplicationDetailByTokenNo` — param: `appTokenNo=value=string`

---

### MasterStructure
Stores all custom form field values for an application. Uses generic key columns (`key1`–`key50`+).

| Field | Notes |
|---|---|
| `key1`–`key50` | String — mapped to `FieldConfigurationMaster.fcmColumnNo` |
| `key43` | Challan remarks (used by Fire NOC, Hoarding services) |

Named queries:
- `fetchMasterStructureByApplication` — param: `appId=value=long`

---

### ApplicationAttachments
Documents uploaded by citizen or officer for an application.

| Field | Type | Notes |
|---|---|---|
| `attachId` | Long | PK |
| `validFlag` | Character | `'Y'` = valid, `'N'` = invalid, null = not reviewed |
| `attachRemark` | String | Officer's remark on the document |
| `attachIsStructualStablityReport` | Character | `'Y'` if structural stability report |

Named queries:
- `fetchAttachedDocumentsByAppIdAndVersion` — param: `appId=value=long`
- `fetchAttachedDocumentsByAppId` — param: `appId=value=long&version=value=long`
- `fetchStablityReportByAppIdAndDocId` — param: `appId=value=long`
- `fetchInspectionReportDetails` / `fetchDrawingReportDetails` — by appId + flag
- `fetchNoDuesCertificate` — param: `appId=value=long`

---

### ApplicationTransaction
Audit trail of every action taken on an application.

| Field | Type | Notes |
|---|---|---|
| `tnAppId` | ApplicationDetail | FK to application |
| `tnEsdId` | EscalationDetails | FK to escalation |
| `tnActionTaken` | String | `APPROVED`, `REJECTED`, `QUERY`, `FORWARD_WITH_REJECTION` |

Used in inbox queries via `tbl_application_transaction` (native table name).

---

## Escalation Entities

### EscalationMaster
Defines the workflow stages for a service.

| Field | Type | Notes |
|---|---|---|
| `esmId` | Long | PK |
| `esmStage` | Long | Stage number (1, 2, 3...) |
| `serviceId` | ParentAppMaster | The service this escalation belongs to |

### EscalationDetails
Maps a specific officer to a stage for a specific application.

| Field | Type | Notes |
|---|---|---|
| `esdId` | Long | PK |
| `escalationMaster` | EscalationMaster | Stage definition |
| `esd_usm_id` (DB col) | Long | Officer user ID |

Named queries:
- `fetchActiveEscalationByUsersId` — param: `usmId=value=long`
- `fetchPamIdsByUserId` — param: `usmId=value=long` → returns `List<Long>` of pamIds

---

## Service Master Entities

### ParentAppMaster
Defines each service offered by the portal.

| Field | Type | Notes |
|---|---|---|
| `pamId` | Long | PK — the service ID |
| `pamParentId` | Long | Parent department/category ID |

### ServiceInfo
Service information for display in navigation menus.

Named queries:
- `getGroupedServicesByDepartment` / `getGroupedServicesByDepartmentMr` — via `CommonService`

### FieldConfigurationMaster
Defines custom form fields for a service.

| Field | Notes |
|---|---|
| `fcmPamId` | FK to ParentAppMaster |
| `fcmFieldLabel` | Display label |
| `fcmColumnNo` | Maps to `MasterStructure` field name (used with BeanUtils reflection) |
| `fcmIsConfigField` | `'Y'` if field uses SystemConfiguration options |
| `fcmConfigFieldName` | Config type to load options from |

Named queries:
- `fetchAllCustomFieldByService` — param: `pamId=value=long`

---

## User & Employee Entities

### Users
The logged-in user (officer or citizen).

| Field | Type | Notes |
|---|---|---|
| `usmId` | Long | PK |
| `usmUserName` | String | Login username |
| `usmDepartmentId` | DepartmentMaster | Officer's department |
| `usmEpmId` | EmployeeMaster | Employee details |
| `usmRomId` | RoleMaster | Role |
| `activeFlag` | Character | `'Y'` = active |

### EmployeeMaster
| Field | Notes |
|---|---|
| `epmId` | PK |
| `epmFirstName` / `epmLastName` | Name |
| `epmWardId` | WardMaster — officer's ward |
| `designation` | Job title |

### DepartmentMaster
| Field | Notes |
|---|---|
| `deptId` | PK — see department ID reference in controllers.md |

### RoleMaster
| Field | Notes |
|---|---|
| `romId` | PK |

---

## Payment & Challan Entities

### Challan
| Field | Type | Notes |
|---|---|---|
| `clnId` | Long | PK |
| `clnNo` | String | `"CLN" + timestamp` |
| `clnDate` | Date | |
| `clnAmount` | Double | Total amount |
| `clnAmountFlag` | Character | `'S'` = system/manual set |
| `clnAppId` | ApplicationDetail | FK |
| `activeFlag` | Character | `'Y'` |
| `cln_department_generated_challan` (DB col) | | Used in Dept 3 filter |

Named queries:
- `fetchChallanByAppIdAndStatus` — param: `appId=value=long`
- `fetchChallanByAppId` — param: `appId=value=long`

### ChallanDetails
One row per budget head line item on a challan.

| Field | Type | Notes |
|---|---|---|
| `clndtClnId` | Challan | FK |
| `budgetHeadId` | BudgetHead | FK |
| `clndtBudgetHeadName` | String | Budget head name (denormalized) |
| `clndtBudgetHeadAmount` | Double | Amount for this line |
| `clndtPaid` | Character | `'Y'` / `'N'` |
| `clndtShcId` | ServiceWiseAndHeadWiseCalculations | Rate calculation reference |

Named queries:
- `fetchChallanDetailsByChallanId` — param: `clnId=value=long`

### BudgetHead
| Field | Notes |
|---|---|
| `bhdId` (PK) | Budget head ID |
| `headName` | Display name |

Key budget head IDs:
- `99` — Advertisement / Sky Sign base
- `100` — Development Cost
- `101` — Additional Development Cost
- `102` — Regularisation Cost

### OnlinePayment
Tracks online payment transactions.

---

## Certificate & NOC Entities

### CertificateData
Stores generated certificate data for an application.

### CertificateNOCConditions
NOC conditions attached to an approved application.

| Field | Notes |
|---|---|
| `cncAppId` | FK to ApplicationDetail |
| `cncCondition` | Condition text (English) |
| `cncConditionMr` | Condition text (Marathi) |
| `arrangement` / `requirement` / `provision` | Fire NOC specific fields |
| `remarks` | Officer remarks |
| `activeFlag` | `'Y'` |

---

## Location Entities

### WardMaster
| Field | Notes |
|---|---|
| `wamId` | PK |
| Ward name fields | |

Named queries:
- `findWardName` — all wards (no params)
- `findWardNamdetails` — param: `wamId=value=long`

### PethNameDetails
Locality/peth within a ward.

Named queries:
- `fetPethByParentPmaId` — param: `mpPamParentId=value=long`

### Taluka / Village
Geographic hierarchy entities.

---

## Service-Specific Entities

### Tree Services
- `TreeDetails` — individual tree records for cutting/replantation applications
- `TreeNameDetails` — tree species master

Named queries:
- `fetchTreeDetailsByAppId` — param: `appId=value=long`
- `fetchTreeReportBranchCuttingByAppId` — native, returns `Object[]`

### Fire NOC Services
- `BuildingInformation` — building sides data (PamId 237)
- `ConstructionDetail` — construction details (PamId 237)
- `PumpDetail` — pump details (PamId 238)
- `PumpInformation` — pump information (PamId 238)
- `AreaInspectionReport` — area inspection (PamId 238)
- `FireNocCondition` — selectable fire NOC conditions

### Sky Sign / Hoarding
- `SkysignBoardDetails` — board type master
- `SkysignBoardMeasurmentDetails` — measurement details per application
- `SkysignHordingDetails` — hoarding details
- `SkysignTemporaryHording` — temporary hoarding
- `AreaStatement` — area statement for hoarding (PamId 267)

Named queries:
- `fetchSkysignDetailsByAppId` — param: `appId=value=long`
- `fetchSkysignBoardMeasurment` — no params
- `fetchAreaStatementByAppId` — param: `appId=value=long`

### Animal Rescue
- `AnimalRescueRehabilitationCenter` — animal rescue records

Named queries:
- `fetchAnimalRescueDetailsByAppId` — param: `appId=value=long`

### Dog License
- `DogLicenseHousingDetails` — housing details for dog license

### Health Services
- `HealthService` — health service records
- `HealthNursingBedDetails` — nursing bed details

### Estate
- `License` / `LicenseDetails` — estate license records
- `AdvanceBookingDetail` — advance booking

### Building / Property
- `AdjacentDetail` — adjacent property details
- `MaterialAreaDetails` — material area
- `OwnerInformation` / `BuyerInformation` / `WarasInformation` — ownership details

---

## Master / Configuration Entities

### SystemConfiguration
Key-value configuration store. Used for filters, actions, field types, etc.

| Field | Notes |
|---|---|
| `configId` | PK |
| `configType` | Category: `FILTER`, `ACTION`, `Foward`, `TRACK`, `FIELDTYPE`, `FIELDUSE`, `FIELDVALIDATION`, `HEIRSHIP_CONDITIONS`, `FIRE_NOC`, etc. |
| `configValue` | The value (e.g. `PENDING`, `APPROVED`) |
| `config_pam_id` (DB col) | Optional service-specific config |

Named queries:
- `findByConfigType` — via `commonService.findByConfigType(String type)`
- `fetchConfigByConfigTypeAndPamId` — param: `configType=value=string&pamId=value=long`

Key config IDs:
- `164` = ALL filter (no status filter)
- `167` = default filter for inbox load
- `336` = default action configuration

### HolidayMaster
Public holidays for TAT calculation.

| Field | Notes |
|---|---|
| `holidayDate` | Date |

### ReasonMaster
Rejection/query reasons per service.

Named queries:
- `fetchReasonByPamId` — param: `mrmPamId=value=long`

### YesNoDetailsMaster
Yes/No options for form fields.

Named queries:
- `fetchYesNoName` — no params

### DocumentMaster
Document type master for required documents per service.

### ServiceRateSlabMaster
Fee rate slabs per service.

Named queries:
- `fetchRateByPamId` — param: `pamId=value=long`

### ServiceWiseAndHeadWiseCalculations
Maps service + budget head to calculation rules.

Named queries:
- `fetchCalByPamIdAndHeadId` — param: `pamId=value=long&bhdId=value=long`

### ServiceWiseOfficerName
Officer name display per service.

Named queries:
- `fetchOfficernameByPamId` — param: `pamId=value=long`

### DepartmentSignature
Officer signature image file per ward/department.

| Field | Notes |
|---|---|
| `signFileName` | Filename in the signatures directory |

Named queries:
- `fetchSignDocumentsByWamAndPamId` — param: `wamId=value=long&pamParentId=value=long`

### EscalationMaster / ServiceBudgetHeadMapping
Service-level escalation and budget head mapping configuration.

### BusinessPartners
Business partner details for commercial applications.

Named queries:
- `fetchPartnersByAppId` — param: `appId=value=long`

---

## Named Query Quick Reference

| Query Name | Entity | Params |
|---|---|---|
| `fetchApplicationDetailByTokenNo` | ApplicationDetail | `appTokenNo=x=string` |
| `fetchMasterStructureByApplication` | MasterStructure | `appId=x=long` |
| `fetchAttachedDocumentsByAppIdAndVersion` | ApplicationAttachments | `appId=x=long` |
| `fetchActiveEscalationByUsersId` | EscalationDetails | `usmId=x=long` |
| `fetchPamIdsByUserId` | — | `usmId=x=long` |
| `fetchChallanByAppId` | Challan | `appId=x=long` |
| `fetchChallanDetailsByChallanId` | ChallanDetails | `clnId=x=long` |
| `fetchReasonByPamId` | ReasonMaster | `mrmPamId=x=long` |
| `fetchYesNoName` | YesNoDetailsMaster | (none) |
| `fetchRateByPamId` | ServiceRateSlabMaster | `pamId=x=long` |
| `fetchOfficernameByPamId` | ServiceWiseOfficerName | `pamId=x=long` |
| `fetchAllCustomFieldByService` | FieldConfigurationMaster | `pamId=x=long` |
| `fetchSignDocumentsByWamAndPamId` | DepartmentSignature | `wamId=x=long&pamParentId=x=long` |
| `fetchTreeDetailsByAppId` | TreeDetails | `appId=x=long` |
| `fetchSkysignDetailsByAppId` | SkysignBoardMeasurmentDetails | `appId=x=long` |
| `fetchAreaStatementByAppId` | AreaStatement | `appId=x=long` |
| `fetchAnimalRescueDetailsByAppId` | AnimalRescueRehabilitationCenter | `appId=x=long` |
| `fetchPartnersByAppId` | BusinessPartners | `appId=x=long` |
| `fetPethByParentPmaId` | PethNameDetails | `mpPamParentId=x=long` |
| `findWardName` | WardMaster | (none) |
| `findWardNamdetails` | WardMaster | `wamId=x=long` |
| `fetchConfigByConfigTypeAndPamId` | SystemConfiguration | `configType=x=string&pamId=x=long` |
| `fetchCalByPamIdAndHeadId` | ServiceWiseAndHeadWiseCalculations | `pamId=x=long&bhdId=x=long` |
