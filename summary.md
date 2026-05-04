Here's a summary of `DepartmentInboxController.java`:

---

## Overview

A Spring MVC `@Controller` for a government online RTS (Right to Service) portal, handling the **department officer's inbox** — where officers review, approve, reject, or query citizen applications.

---

## Key Dependencies

- `CommonService` — generic data access layer (CRUD, named queries, native queries)
- `CertificateService` — certificate generation
- `HoardingQRService` — QR code management for hoarding/signboard services
- `EntityManager` — JPA persistence context
- Views rendered as JSPs via `ModelAndView`

---

## Endpoints & Core Logic

### `GET /departmentInbox`
Loads the main department inbox page. Calls `loadDepartmentInbox()` to populate the model, also loads:
- Filter configs (`FILTER` type from `SystemConfiguration`)
- Grouped services by department (English/Marathi based on locale)
- Hoarding services map (IDs 268, 269)
- QR code list

### `GET /appealDepartmentInbox`
Same structure as above but for **appeal cases**, calls `loadDepartmentInboxForAppeal()`.

---

## `loadDepartmentInbox()` — Core Inbox Loading Logic

This is the most complex method. It determines **what applications to show** based on the logged-in officer's department, user ID, and selected filter.

**Card Counts (Approved / Rejected / Pending / Query / Total):**
- Pending: from `vw_department_inbox` where `app_status = 'PENDING'`
- Approved/Rejected: from `vw_department_inbox_approved` joined with `tbl_application_transaction` on action taken
- Query: from `vw_department_inbox_approved` where `app_status = 'QUERY'`
- Total = sum of all four

**Inbox list query selection — branching by dept/user:**

| Condition | Query Used |
|---|---|
| Dept ≠ 1 & ≠ 5, user = 215 | `vw_department_inbox_for_ward_officer` (dog license ward officer) |
| Dept = 1 or 5 (Fire/Water), filter = APPROVED/REJECTED | `vw_department_inbox_approved` JOIN `tbl_application_transaction` |
| Dept = 1 or 5, filter = PENDING/QUERY | `vw_department_inbox` |
| Dept = 1 or 5, ALL filter | UNION of approved/rejected transactions + pending/query view |
| Dept = 2 (after payment) | `vw_department_inbox_after_payment` |
| Dept = 3 (with challan filter) | `vw_department_inbox` or `vw_department_inbox_approved` with `cln_department_generated_challan is null` |
| Dept = 11, stage 2, pamId 113 (e-sign officer) | `vw_department_inbox_approved` or UNION query + TAT calculation |
| Dept = 11, stage 1, pamIds 267/268/269/270 (Hoarding) | Same pattern with FORWARD_WITH_REJECTION filter support |
| Users 10, 11, 22 (specific senior officers) | UNION query + TAT calculation |
| All other officers | UNION query + TAT calculation, also loads `stage` and `pamId` per row |

**TAT (Turnaround Time) Calculation:**
- Appended as an extra column to each inbox row
- Uses `calculateWorkingDays(today, expectedDate, holidays)` — skips holidays loaded from `HolidayMaster`
- TAT index stored in model as `tatIndex`

**Status label cleanup:**
- `CITIZENT_LOGIN` → `CITIZEN` (strips `T_LOGIN`)
- `PLUMBER_LOGIN` → `PLUMBER` (strips `_LOGIN`)

---

## `loadDepartmentInboxForAppeal()`
Simplified version for appeal inbox. Only user ID 11 gets the full appeal list from `vw_appeal_department_inbox`. Count logic is present but queries are not executed (variables left null — appears incomplete/placeholder).

---

## `GET /approveRejectApplication`
Loads the application detail page for an officer to take action. Populates:
- Application details, attachments, document validity flags (`docValidFlagMap`)
- Custom form fields via `FieldConfigurationMaster` + `BeanUtils` reflection
- Service rate slabs, conditions, officer names, tree details, skysign board details
- Challan details if present
- Business partners, animal rescue details (service-specific)
- Department signature image (base64 encoded, path varies Windows/Linux)
- Escalation stage info

---

## `POST /approveRejectApplication`
Handles the officer's action submission. Key save operations:

| Condition | Action |
|---|---|
| `pageMode = SaveAction` | Saves document validity flags + remarks per attachment |
| PamParent = 241, PamId = 246, stage 1 | Saves `CertificateNOCConditions` |
| PamParent = 1 (Fire), stage 1 | Saves `FireNocCondition` selections as `CertificateNOCConditions` |
| PamId = 238 (Fire NOC), stage 1 | Saves pump details, pump info, area inspection reports |
| PamId = 237, stage 2 | Merges saved fire conditions |
| PamId = 237, stage 1 or 3 | Saves building sides, construction details, manual challan amount override, challan remarks |
| PamId = 240 or 238, stage 1 or 3 | Manual challan amount + remarks |
| PamParent = 82, PamId = 267, stage 1 | Creates new `Challan` + `ChallanDetails` (budget head 99), saves area statements, sets `CHALLAN_GENERATED` status |
| PamParent = 82, PamId = 268/269, stage 2 | Creates challan with 4 cost components (advertisement, development, additional dev, regularisation) mapped to budget heads 99–102 |
| User 59, PamId = 18 (dog license) | Sets `app_status = CREATED`, marks ward office approval |

---

## Helper Methods

- `approveReject()` — loads action list and tree/replantation details into model
- `getFieldTypeForTree()` — AJAX endpoint returning field types, tree names, reason master, yes/no options for a given `pamId`
- `removeTreeDetailsForDepartment()` — AJAX endpoint to delete a `TreeDetails` record

---

## Notable Issues / Code Smells

- **SQL injection risk** — all native queries use string concatenation with user-supplied IDs (no parameterized queries)
- **Large commented-out block** — the original `loadDepartmentInbox` is fully commented out above the active version
- **Hardcoded user IDs** (10, 11, 22, 59, 215) and **department IDs** (1–7, 11) scattered throughout
- **Hardcoded paths** — `/home/pmrda/rts/` for server, with a commented-out Windows path
- **`@Repository` on a `@Controller`** — incorrect annotation combination
- **`loadDepartmentInboxForAppeal`** — count queries are never executed (dead code)
- The file is truncated in the paste — there's more logic after the dog license section