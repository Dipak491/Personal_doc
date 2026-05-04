# OnlineRTS — Coding Conventions & Standards

## General Rules

- Java 8 — use streams, lambdas, `Optional` where appropriate but match existing style in the file you're editing
- All new controller classes: `@Controller @Transactional` only — do **not** add `@Repository` (it's a mistake in existing code)
- Always wrap controller method bodies in `try/catch(Exception e)` and call `e.printStackTrace()` — this is the project standard
- Never swallow exceptions silently with just `e.getMessage()` on page-load methods; use `e.printStackTrace()` so errors appear in logs
- Match the existing code style in whichever file you are editing — don't introduce new patterns unless asked

---

## Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Controllers | `XyzController.java` | `DepartmentInboxController` |
| Models/DTOs | `XyzModel.java` | `DepartmentInboxModel` |
| Views (Tiles) | camelCase string | `"departmentInbox"` |
| Native query vars | `lstXxx` for lists | `lstInbox`, `lstFilter` |
| Model attributes | camelCase | `"lstInbox"`, `"departmentInboxModel"` |
| Session attributes | `SessionData.USERS` constant | never raw strings for user |
| Config keys | UPPER_SNAKE_CASE | `UPLOADFILE_ATTACHMENTS` |

---

## Controller Method Signatures

Standard page controller:
```java
@RequestMapping(value = "/path", method = RequestMethod.GET)
public ModelAndView methodName(HttpServletRequest request, HttpSession session,
        ModelMap model, Locale locale) {
    ModelAndView view = new ModelAndView("viewName");
    try {
        Users users = (Users) session.getAttribute(SessionData.USERS);
        view = StaticData.loadDataForHomepage(commonService, users, model, locale, view);
        // ... logic
    } catch (Exception e) {
        e.printStackTrace();
    }
    return view;
}
```

Standard AJAX endpoint:
```java
@RequestMapping(value = "/ajaxPath", method = RequestMethod.POST)
public @ResponseBody Map<String, Object> methodName(
        @RequestParam Long someId, HttpSession session) {
    Map<String, Object> map = new HashMap<String, Object>();
    try {
        // ... logic
        map.put("key", value);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return map;
}
```

Form POST handler:
```java
@RequestMapping(value = "/path", method = RequestMethod.POST)
public ModelAndView methodName(@ModelAttribute XyzModel model,
        HttpSession session, ModelMap modelMap, Locale locale) {
    ModelAndView view = new ModelAndView("viewName");
    try {
        Users users = (Users) session.getAttribute(SessionData.USERS);
        // ... logic
    } catch (Exception e) {
        e.printStackTrace();
    }
    return view;
}
```

---

## Data Access Conventions

Always cast results from `commonService` — it returns `Object`:
```java
// Single entity
ApplicationDetail app = (ApplicationDetail) commonService.findById(ApplicationDetail.class, appId);

// Named query list
List<ReasonMaster> list = commonService.fetchByNamedQuery("fetchReasonByPamId",
        "mrmPamId=" + pamId + "=long");

// Native query — rows are Object[]
List<Object[]> rows = commonService.fetchByNativeQuery("SELECT * FROM vw_... WHERE ...");

// Access row columns by index
Long id = ((BigInteger) row[0]).longValue();   // numeric columns come as BigInteger from native queries
String status = row[5] != null ? row[5].toString() : "";
Date dt = (Date) row[26];
```

Always null-check before accessing row columns:
```java
if (row[26] != null) {
    Date expectedDate = (Date) row[26];
    // use it
}
```

---

## File Path Handling

Always use OS detection for file paths:
```java
String osName = System.getProperty("os.name");
String filePath;
if (osName.toLowerCase().contains("windows")) {
    filePath = PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "WINDOWS_UPLOADFILE_ATTACHMENTS");
} else {
    filePath = PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "UPLOADFILE_ATTACHMENTS");
}
File file = new File(filePath + File.separator + fileName);
```

Never hardcode file paths directly in code.

---

## SQL / Native Queries

- All native queries use **string concatenation** with IDs — this is the existing pattern, match it
- Always use `Long` / `users.getUsmId()` directly in query strings
- Use `DISTINCT` when joining `tbl_application_transaction` to avoid duplicate rows
- Standard inbox query pattern:
```java
// Approved/Rejected — use transaction join
"SELECT DISTINCT v.* FROM vw_department_inbox_approved v " +
"JOIN tbl_application_transaction tn ON tn.tn_esd_id = v.esd_id AND tn.tn_app_id = v.app_id " +
"WHERE v.esd_usm_id = " + users.getUsmId() + " AND tn.tn_action_taken = 'APPROVED' order by v.app_id DESC"

// Pending/Query — use main inbox view
"SELECT * from vw_department_inbox where esd_usm_id = " + users.getUsmId() +
" AND app_status = 'PENDING' order by app_id DESC"

// ALL filter — UNION both
"SELECT DISTINCT v.* FROM vw_department_inbox_approved v " +
"JOIN tbl_application_transaction tn ON tn.tn_esd_id = v.esd_id AND tn.tn_app_id = v.app_id " +
"WHERE v.esd_usm_id = " + users.getUsmId() + " AND tn.tn_action_taken IN ('APPROVED','REJECTED') " +
"UNION SELECT * FROM vw_department_inbox WHERE app_status IN ('PENDING','QUERY') AND esd_usm_id = " +
users.getUsmId() + " order by app_id DESC"
```

---

## Entity Save Patterns

New entity:
```java
Entity e = new Entity();
e.setField(value);
e.setActiveFlag('Y');
e.setCreatedBy(users.getUsmId());
e.setCreatedOn(new Date());
commonService.saveOrUpdate(e);
```

Update existing entity:
```java
Entity e = (Entity) commonService.findById(Entity.class, id);
e.setField(newValue);
commonService.merge(e);
```

---

## Challan Creation Pattern

```java
Challan cln = new Challan();
cln.setClnDate(new Date());
cln.setClnNo("CLN" + new Date().getTime());   // unique number
cln.setCreatedBy(users.getUsmId());
cln.setCreatedOn(new Date());
cln.setClnAmount(amount);
cln.setClnAmountFlag('S');   // 'S' = system/manual set
cln.setActiveFlag('Y');
cln.setClnAppId(applicationDetail);
if (commonService.saveOrUpdate(cln)) {
    // create ChallanDetails per budget head
    ChallanDetails cd = new ChallanDetails();
    cd.setBudgetHeadId((BudgetHead) commonService.findById(BudgetHead.class, budgetHeadId));
    cd.setClndtBudgetHeadName(budgetHead.getHeadName());
    cd.setClndtBudgetHeadAmount(amount);
    cd.setClndtPaid('N');
    cd.setClndtClnId(cln);
    commonService.saveOrUpdate(cd);
}
```

---

## Model Attribute Keys (Consistent Names)

Always use these exact keys when adding to `ModelMap` / `model`:

| Key | Type | Content |
|---|---|---|
| `"lstInbox"` | `List<Object[]>` | Inbox rows |
| `"lstFilter"` | `List<SystemConfiguration>` | Filter dropdown |
| `"lstActions"` | `List<SystemConfiguration>` | Action buttons |
| `"departmentInboxModel"` | `DepartmentInboxModel` | Form backing bean |
| `"applicationDetail"` | `ApplicationDetail` | Current application |
| `"attachmentLstdetails"` | `List<ApplicationAttachments>` | Attached documents |
| `"docValidFlagMap"` | `Map<String, String>` | attachId → Y/N/"" |
| `"reasonMaster"` | `List<ReasonMaster>` | Rejection reasons |
| `"treeDetails1"` | `List<TreeNameDetails>` | Tree species list |
| `"lstDetails"` | `List<YesNoDetailsMaster>` | Yes/No options |
| `"wardList"` | `List<WardMaster>` | Ward list for forwarding |
| `"total"`, `"approved"`, `"rejected"`, `"pending"`, `"queryCount"` | `Long` | Dashboard card counts |
| `"tatIndex"` | `Integer` | Column index of TAT in row array |
| `"stageId"` | `Long` | Escalation stage number |
| `"pamId"` | `Long` | Service parent app master ID |
| `"isAppealView"` | `Boolean` | Whether opened from appeal inbox |
| `"groupedList"` | `List<Map.Entry<String, List<ServiceInfo>>>` | Services grouped by dept |
| `"locale"` | `Locale` | Current locale object |
| `"currentLocale"` | `String` | Display language name |

---

## Null Safety Patterns

For `Optional`-style null checks on nested objects:
```java
String name = Optional.ofNullable(users)
    .map(u -> u.getUsmEpmId())
    .map(epm -> Objects.toString(epm.getEpmFirstName(), "") + " " + Objects.toString(epm.getEpmLastName(), ""))
    .orElse("");
```

For collections before iterating:
```java
if (list != null && !list.isEmpty()) {
    for (Item item : list) {
        if (item == null) continue;
        // process
    }
}
```

For TAT index:
```java
Integer tatIndex = -1;
if (deptInboxWithTat != null && !deptInboxWithTat.isEmpty() && deptInboxWithTat.get(0) != null) {
    tatIndex = deptInboxWithTat.get(0).length - 1;
}
model.addAttribute("tatIndex", tatIndex);
```

---

## Internationalisation in Controllers

When loading grouped services for the nav menu (required on all main page loads):
```java
Locale currentLocale = LocaleContextHolder.getLocale();
Map<String, List<ServiceInfo>> groupedServices;
if (currentLocale.getLanguage().equalsIgnoreCase("en")) {
    groupedServices = commonService.getGroupedServicesByDepartment();
} else {
    groupedServices = commonService.getGroupedServicesByDepartmentMr();
}
List<Map.Entry<String, List<ServiceInfo>>> groupedList = new ArrayList<>(groupedServices.entrySet());
model.addAttribute("groupedList", groupedList);
model.addAttribute("locale", currentLocale);
model.addAttribute("currentLocale", locale.getDisplayLanguage());
```

---

## Known Code Smells to Be Aware Of (Do Not Replicate)

- `@Repository` on `@Controller` classes — already present, don't add to new code
- Hardcoded user IDs (10, 11, 22, 59, 215) — existing pattern, document when adding new ones
- Native query string concatenation without parameterization — existing pattern, be aware of SQL injection risk
- Large commented-out code blocks — existing pattern, don't add new ones; use version control instead
- `Main.java` / `Main1.java` in utils — scratch files, not production code
- `.bak` files in source — old backups, ignore them
