# OnlineRTS — Department Inbox Logic

This file documents the core officer workflow in `DepartmentInboxController`.

---

## Application Lifecycle (Status Flow)

```
PENDING → (officer acts) → APPROVED / REJECTED / QUERY
                         → CHALLAN_GENERATED (if payment required)
                         → FORWARD_WITH_REJECTION
CHALLAN_GENERATED → (citizen pays) → APPROVED / next stage
```

Applications move through escalation stages (`EscalationMaster.esmStage`). Each stage maps to a specific officer (`EscalationDetails.esd_usm_id`).

---

## Key Database Views

| View | Purpose |
|---|---|
| `vw_department_inbox` | Applications currently pending/query for an officer |
| `vw_department_inbox_approved` | All applications an officer has ever acted on (approved/rejected/query) |
| `vw_department_inbox_after_payment` | Dept 2 — applications after payment received |
| `vw_department_inbox_for_ward_officer` | Dog license ward officer queue |
| `vw_department_inbox_esign` | E-sign officer queue (Dept 11, stage 2) |
| `vw_department_inbox_forward_rejection` | Applications forwarded with rejection |
| `vw_appeal_department_inbox` | Appeal applications (user 11 only) |

---

## Card Count Logic (Dashboard Counters)

Counts shown on the inbox dashboard cards:

```
PENDING  = COUNT(*) FROM vw_department_inbox WHERE app_status='PENDING' AND esd_usm_id = :usmId
APPROVED = COUNT(DISTINCT app_id) FROM vw_department_inbox_approved
           JOIN tbl_application_transaction ON tn_action_taken='APPROVED' AND esd_usm_id = :usmId
REJECTED = COUNT(DISTINCT app_id) FROM vw_department_inbox_approved
           JOIN tbl_application_transaction ON tn_action_taken='REJECTED' AND esd_usm_id = :usmId
QUERY    = COUNT(*) FROM vw_department_inbox_approved WHERE app_status='QUERY' AND esd_usm_id = :usmId
TOTAL    = APPROVED + REJECTED + PENDING + QUERY
```

Model attributes: `total`, `approved`, `rejected`, `pending`, `queryCount`

---

## Inbox List Query Selection Matrix

The query used to populate `lstInbox` depends on the officer's department, user ID, escalation stage, and selected filter.

### Filter Values (from `SystemConfiguration` type=`FILTER`)
- `164` = ALL (no filter)
- Config value `PENDING`, `APPROVED`, `REJECTED`, `QUERY`, `FORWARD_WITH_REJECTION`

### Decision Tree

```
IF dept ≠ 1 AND dept ≠ 5 AND usmId = 215
    → vw_department_inbox_for_ward_officer (dog license ward officer)

ELSE IF dept = 1 OR dept = 5 (Fire / Water)
    APPROVED  → vw_department_inbox_approved JOIN tbl_application_transaction (tn_action_taken='APPROVED')
    REJECTED  → vw_department_inbox_approved JOIN tbl_application_transaction (tn_action_taken='REJECTED')
    PENDING/QUERY → vw_department_inbox WHERE app_status = filter
    ALL       → UNION (approved/rejected transactions) + (pending/query from vw_department_inbox)

ELSE IF dept = 2 (after-payment)
    → vw_department_inbox_after_payment (all filters)

ELSE IF dept = 3 (challan dept)
    APPROVED  → vw_department_inbox_approved JOIN transaction, cln_department_generated_challan IS NULL
    REJECTED  → same with REJECTED
    PENDING/QUERY → vw_department_inbox WHERE cln_department_generated_challan IS NULL
    ALL       → UNION with cln_department_generated_challan IS NULL condition

ELSE IF dept = 11 AND stage = 2 AND pamId contains 113 (building e-sign)
    OR dept = 11 AND stage = 1 AND pamId IN (267, 268, 269, 270) (hoarding)
    APPROVED/REJECTED → vw_department_inbox_approved JOIN transaction
    PENDING   → vw_department_inbox
    QUERY     → vw_department_inbox_approved WHERE app_status='QUERY'
    FORWARD_WITH_REJECTION → vw_department_inbox_forward_rejection
    ALL       → UNION + TAT calculation appended

ELSE IF usmId IN (10, 11, 22) (senior e-sign officers)
    APPROVED/REJECTED → vw_department_inbox_approved JOIN transaction
    PENDING/QUERY → vw_department_inbox
    ALL       → UNION + TAT calculation appended

ELSE (all other officers)
    APPROVED  → vw_department_inbox_approved JOIN transaction
    REJECTED  → vw_department_inbox_approved JOIN transaction
    ALL/other → UNION query + TAT calculation appended
                also loads stage and pamId per row from EscalationDetails
```

---

## TAT (Turnaround Time) Calculation

TAT is appended as an **extra column** at the end of each inbox row array.

```java
// row[26] = expected_date from the view
tatDays = calculateWorkingDays(today, expectedDate, holidays) + 1;
newRow[newRow.length - 1] = tatDays;
```

- Holidays loaded from `HolidayMaster` entity
- `tatIndex` model attribute = `row.length - 1` (tells JSP which column index to read)
- Only applied for Dept 11, users 10/11/22, and general officers (not Dept 1/2/3/5)

---

## Status Label Cleanup

Applied to `row[5]` (app_status column) before sending to view:

```java
"CITIZENT_LOGIN"  → "CITIZEN"   (replaces "T_LOGIN")
"PLUMBER_LOGIN"   → "PLUMBER"   (replaces "_LOGIN")
```

---

## approveRejectApplication — GET (View Load)

Loads the application detail page for officer action. Key model attributes populated:

| Attribute | Source |
|---|---|
| `applicationDetail` | `ApplicationDetail` entity by `appId` |
| `attachmentLstdetails` | `fetchAttachedDocumentsByAppIdAndVersion` named query |
| `docValidFlagMap` | `Map<String, String>` — attachId (String) → validFlag (Y/N/"") |
| `mapFieldNameToValue` | Custom fields via `FieldConfigurationMaster` + BeanUtils reflection on `MasterStructure` |
| `stageId` | `EscalationDetails.escalationMaster.esmStage` |
| `esmId` | `EscalationDetails.escalationMaster.esmId` |
| `reasonMaster` | `ReasonMaster` list by pamId |
| `fetchRateDetails` | `ServiceRateSlabMaster` by pamId |
| `conditionLst` | `SystemConfiguration` by type + pamId |
| `lstChallanDetails` | `ChallanDetails` if challan exists |
| `signImageBase64` | Officer signature image as base64 data URI |
| `isAppealView` | Boolean — true if opened from appeal inbox |

Service-specific extras:
- PamId 30 (Sky Sign): `lstSkysignDetails`, `lstBoardDetails`, `lstNocCertificateDetails`
- PamId 69/70 (Tree): `detailsBranch`, `treereplantation`, `fetchTreeDetailsAppId`
- PamId 116 (Animal Rescue): `animalRescueLst`
- PamId 127: additional document list

---

## approveRejectApplication — POST (Save Action)

Triggered when `pageMode = "SaveAction"`. Saves based on service/stage combination:

### Document Validation (all services)
```java
// For each attachment in departmentInboxModel.lstAttachments:
applicationAttachments.setValidFlag(flag ? 'Y' : 'N');
applicationAttachments.setAttachRemark(remark);
commonService.saveOrUpdate(applicationAttachments);
```

### Service-Specific Save Logic

| Condition | What Gets Saved |
|---|---|
| PamParent=241, PamId=246, stage=1 | `CertificateNOCConditions` from `lstConditions` |
| PamParent=1 (Fire), stage=1 | `FireNocCondition` selections → `CertificateNOCConditions` |
| PamId=238 (Fire Pump), stage=1 | `PumpDetail`, `PumpInformation`, `AreaInspectionReport` |
| PamId=237 (Fire NOC Building), stage=2 | Merge existing `CertificateNOCConditions` |
| PamId=237, stage=1 or 3 | `BuildingInformation`, `ConstructionDetail`, manual challan amount, challan remarks (`MasterStructure.key43`) |
| PamId=240 or 238, stage=1 or 3 | Manual challan amount + remarks |
| PamParent=82, PamId=267, stage=1 | Create new `Challan` + `ChallanDetails` (budget head 99), save `AreaStatement` list, set `CHALLAN_GENERATED` |
| PamParent=82, PamId=268 or 269, stage=2 | Create `Challan` with 4 cost components (budget heads 99–102) |
| UsmId=59, PamId=18 (Dog License) | Set `app_status='CREATED'`, `appDogLicesneApprovalWardOffice='D'` |

### Manual Challan Override Pattern
```java
if ("N".equals(departmentInboxModel.getApproveChallan())) {
    Challan ch = fetchChallanByAppId(appId);
    ch.setClnAmount(departmentInboxModel.getManualAmount());
    // update all ChallanDetails amounts too
}
```

### Challan Creation Pattern (PamId 267)
```java
Challan cln = new Challan();
cln.setClnNo("CLN" + new Date().getTime());
cln.setClnAmount(amount);
cln.setClnAmountFlag('S');  // 'S' = manual/system set
cln.setActiveFlag('Y');
cln.setClnAppId(applicationDetail);
commonService.saveOrUpdate(cln);
// then create ChallanDetails per budget head
```

### Hoarding Cost Components (PamId 268/269, stage 2)
| Field | Budget Head ID |
|---|---|
| Advertisement Cost | 99 |
| Development Cost | 100 |
| Additional Development Cost | 101 |
| Regularisation Cost | 102 |

---

## EscalationDetails — Key Fields

```java
EscalationDetails esdD = commonService.fetchObjectByNamedQuery(
    "fetchActiveEscalationByUsersId", "usmId=" + users.getUsmId() + "=long");

esdD.getEscalationMaster().getEsmStage()   // Long — current stage number
esdD.getEscalationMaster().getServiceId()  // service reference
esdD.getEscalationMaster().getEsmId()      // escalation master ID
```

PamIds for an officer are fetched via:
```java
List<Long> pamIDs = commonService.fetchByNamedQuery(
    "fetchPamIdsByUserId", "usmId=" + users.getUsmId() + "=long");
```

---

## Appeal Inbox

- Only user ID 11 sees the appeal inbox (`vw_appeal_department_inbox`)
- Count queries in `loadDepartmentInboxForAppeal` are currently not executed (placeholder code)
- View: `appealDepartmentInbox`
- `isAppealView` flag passed to `approveRejectApplication` view to adjust UI
