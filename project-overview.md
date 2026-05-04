# OnlineRTS — Project Overview

## What This Is

**OnlineRTS** (Online Right to Service) is a government web portal for **PMRDA (Pune Metropolitan Region Development Authority)**. It allows citizens to apply for municipal services online, and department officers to review, approve, reject, or query those applications.

The deployed WAR is named **ServicePortal** and runs on **Apache Tomcat 7.0.72**.

---

## Multi-Module Maven Structure

The project is split into 4 Maven modules under the parent `OnlineRTS_Spring`:

| Module | Role |
|---|---|
| `OnlineRTS_Controller` | Spring MVC web layer — controllers, models, config, utils |
| `OnlineRTS_Service` | Business/service layer (separate module, consumed as dependency) |
| `OnlineRTS_DAO` | Data access layer — JPA repositories (separate module) |
| `OnlineRTS_Entities` | JPA entity classes (separate module) |

This steering file covers **OnlineRTS_Controller** only.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Language | Java 8 |
| Web Framework | Spring MVC 4.2.6.RELEASE |
| Security | Spring Security 4.0.4.RELEASE |
| ORM | Hibernate 4.3.10.Final + Spring Data JPA 1.7.0.RELEASE |
| View Layer | JSP + JSTL + Apache Tiles 3.0.5 |
| Build | Maven (WAR packaging) |
| Server | Apache Tomcat 7.0.72 |
| JSON | Jackson 2.17.2 + GSON 2.8.6 |
| PDF | iText 2.0.6 + OpenHtmlToPdf 1.0.10 |
| Excel | Apache POI 5.2.5 |
| QR Code | ZXing 3.4.1 + QRGen 2.0 |
| HTTP Client | OkHttp 4.12.0 + Apache HttpClient 4.5.14 |
| JWT | Nimbus JOSE JWT 9.31 |
| Email | JavaMail 1.4.4 |
| SOAP | Apache Axis 1.4 |

---

## Languages Supported

English (`en`) and Marathi (`mr`). Message bundles:
- `messages_en.properties` / `messages_mr.properties`
- `validation_en.properties` / `validation_mr.properties`
- `marathi_text.properties`

---

## Deployment Paths

| Environment | Attachments | Challan | Application Forms |
|---|---|---|---|
| Windows (dev) | `C:\RTS\Attachments` | `C:\RTS\Challan` | `C:\RTS\Application` |
| Linux (server) | `/home/document/RTS/Attachments` | `/home/document/RTS/Challan` | `/home/document/RTS/Application` |
| Officer Signatures | `/home/document/RTS/Attachments/OfficerSigns` | — | — |

PDF base dir on server: `/home/pmrda/rts/`

---

## External Integrations

- **WhatsApp** — Infobip API (`whatsapp.certificate.url`)
- **PDF Generation** — Internal utility service (`generate.pdf.utility`)
- **DMS** — Primeleaf Krystal document management (SOAP via Apache Axis)
- **Property Tax** — PMC property tax web services
- **MahaIT** — Maharashtra government IT portal API
- **Aapale Sarkar** — Citizen portal security integration
- **PMPML** — Water/sewerage integration
- **Payment Gateway** — Online payment processing
- **POS** — Point of Sale for CFC counters
