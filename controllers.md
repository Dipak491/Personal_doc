# OnlineRTS — Controllers Reference

All controllers live in `com.pmc.controller`. Each is `@Controller @Transactional`.

---

## Authentication & User Management

| Controller | Key Endpoints | Notes |
|---|---|---|
| `LoginController` | `GET /login`, `POST /login`, `GET /logout`, `POST /forgotPassword`, `POST /verifyOtp` | IP + user-level lockout via `LoginAttemptService` + `UserLoginAttemptService` |
| `UsersController` | User CRUD, password change, role assignment | Admin-only |
| `CitizenRegistrationController` | `GET/POST /citizenRegistration`, OTP verify | Citizen self-registration |
| `RoleMenuController` | Role-menu mapping management | Admin |

---

## Citizen-Facing

| Controller | Key Endpoints | Notes |
|---|---|---|
| `CitizenInboxController` | `GET /citizenInbox` | Shows citizen's submitted applications with status + TAT |
| `ApplicationController` | `GET /applyService`, `POST /submitApplication` | Main application submission flow |
| `AppealController` | `GET/POST /appeal` | Citizen appeal against rejected applications |
| `DocumentController` | Upload/download citizen documents | Delegates to `UploadDocumentsController` |
| `UploadDocumentsController` | `POST /uploadDocument` | Multipart file upload, saves to filesystem |
| `PaymentController` | `GET /payment`, `POST /paymentCallback` | Online payment gateway integration |
| `ChallanController` | `GET /challan`, challan download | Challan generation and download |
| `CFCReceiptController` | CFC counter receipts | POS/CFC payment receipts |

---

## Department Officer-Facing

| Controller | Key Endpoints | Notes |
|---|---|---|
| `DepartmentInboxController` | `GET /departmentInbox`, `GET /appealDepartmentInbox`, `GET/POST /approveRejectApplication` | Core officer workflow — see department-inbox.md |
| `DepartmentController` | Department master CRUD | Admin |
| `DepartmentInboxController` | `POST /getFieldTypeForTree`, `POST /removeTreeDetailsForDepartment` | AJAX helpers for tree service |
| `EscalationController` | Escalation rules management | Defines officer routing stages |
| `CertificateController` | Certificate generation, download, e-sign | Generates PDF certificates after approval |
| `ScrollController` | Scroll lock/unlock for daily reconciliation | Finance department |

---

## Service-Specific Controllers

### Fire Department (PamParentId = 1)
| Controller | Services Handled |
|---|---|
| `FireDashboardController` | Fire NOC dashboard, reports |
| `BuildingDevelopemntController` | Building development permits (PamId 237, 238, 240) |

### Water Department (DeptId = 5, PamParentId = 44)
| Controller | Services Handled |
|---|---|
| `WaterDashboardController` | Water connection dashboard |
| `WaterTaxPaymentController` | Water tax payment, receipts |
| `PmpmlController` | PMPML water/sewerage integration |

### Health Department
| Controller | Services Handled |
|---|---|
| `HealthDashboardController` | Health license dashboard |
| `HealthRenewalServicesController` | Health license renewal |

### Tree Department
| Controller | Services Handled |
|---|---|
| `TreeCuttingController` | Tree cutting permits (PamId 69) |
| `TreeDashboardController` | Tree management reports |

### Sky Sign / Hoarding (PamParentId = 82)
| Controller | Services Handled |
|---|---|
| `SkySignLicencesController` | New hoarding (268), regularization (269), renewal (270) |
| `SkySignDashboardController` | Hoarding dashboard and reports |
| `HoardingQRController` | QR code generation for hoardings |

### Garden / Parks
| Controller | Services Handled |
|---|---|
| `GardanServiceController` | Garden ticket booking, slot management |
| `GardenDashboardController` | Garden dashboard |
| `GardenAPIForPMCCARE` | REST API for PMC Care mobile app |

### Other Services
| Controller | Service |
|---|---|
| `BirthAndDeathController` | Birth/death certificates |
| `AnimalRescueController` | Animal rescue & rehabilitation (PamId 116) |
| `ButcherServiceCOntroller` | Butcher/slaughterhouse licensing |
| `LadderController` | Ladder/scaffolding permits |
| `SaftyTankController` | Safety tank services |
| `EstateController` | Estate management |
| `SlumReconciliationController` | Slum reconciliation |
| `PropertyTaxController` | Property tax services |
| `BudgetHeadController` | Budget head master management |

---

## Dashboard & Reporting

| Controller | Purpose |
|---|---|
| `DashboardController` | Main admin/officer dashboard |
| `AllServiceMisController` | MIS reports across all services |
| `WardwiseDashboardController` | Ward-wise application reports |
| `EncroachmentDashboardController` | Encroachment tracking |

---

## API / Integration Controllers

| Controller | Purpose |
|---|---|
| `APIController` | REST endpoints for external consumers |
| `AapleSarkarSecurity` | Aapale Sarkar citizen portal JWT/security |
| `MahaITApiController` | MahaIT government portal integration |
| `QRCodeGenerator` | QR code generation endpoint |
| `POSController` | POS terminal integration for CFC |

---

## Utility Controllers / Misc

| Controller | Purpose |
|---|---|
| `CustomFormController` | Dynamic custom form rendering |
| `CustomFormGeneratorController` | Admin tool to build custom forms |
| `DocumentMasterController` | Document master data management |
| `ZoneController` | Zone/ward/prabhag master management |
| `ServiceController` | Service master management |
| `UserLoginAttemptService` | (Misplaced in controller package) — tracks login attempts |
| `A` | Scratch/utility class — not a real controller |

---

## Service PamId Reference (Key Services)

| PamId | Service Name | PamParentId |
|---|---|---|
| 18 | Dog License | — |
| 30 | Sky Sign Board | — |
| 69 | Tree Branch Cutting | — |
| 70 | Tree Replantation | — |
| 113 | Building Permission (e-sign) | — |
| 116 | Animal Rescue | — |
| 127 | Special document service | — |
| 177 | Area measurement service | — |
| 237 | Fire NOC (Building) | 1 |
| 238 | Fire NOC (Pump) | 1 |
| 240 | Fire NOC (Other) | 1 |
| 246 | NOC Conditions service | 241 |
| 267 | Sky Sign New Permission | 82 |
| 268 | New Hoarding Permission | 82 |
| 269 | Hoarding Regularization | 82 |
| 270 | Hoarding Renewal | 82 |

---

## Department ID Reference

| DeptId | Department |
|---|---|
| 1 | Fire |
| 2 | (After-payment dept) |
| 3 | (Challan dept) |
| 4 | General |
| 5 | Water |
| 6 | General |
| 7 | General |
| 11 | Building / E-sign |

---

## Hardcoded User IDs (Known Special Cases)

| UsmId | Role |
|---|---|
| 10, 11, 22 | Senior officers with e-sign access |
| 59 | DSI-level officer (dog license ward approval) |
| 215 | Ward officer (dog license) |
