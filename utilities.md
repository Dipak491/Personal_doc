# OnlineRTS — Utilities & Helpers Reference

All utilities live in `com.pmc.utils`.

---

## Util.java — Main Utility Class

The primary utility class. Key static methods:

### Email
```java
// Send plain email
Boolean sent = Util.sendMail(String to, String subject, String body);
```
SMTP config: `rml-mta.unifiedrml.com:587`, TLS 1.2, auth required.

### SMS
```java
// Send SMS via configured gateway
Util.sendSms(String mobileNumber, String message);
```
SMS templates are in `application-config.properties`. Always use `MessageFormat.format()` with the template:
```java
String msg = MessageFormat.format(
    PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "ONLINE_RTS_APPLICATION_SUBMITTED"),
    applicationNo
);
Util.sendSms(mobile, msg);
```

### File Download
```java
// Set response headers for file download
Util.downloadFileProperties(HttpServletRequest req, HttpServletResponse resp,
    String fileName, File downloadFile);
```

### File Path
```java
// Get the downloads directory path
String path = Util.getFilePath(HttpServletRequest req);
```

---

## PropertyFetcher.java

Loads values from properties files:
```java
String value = PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "key.name");
```

`FileName` constants (from `WebConstants`):
- `FileName.APPLICATION_CONFIG` → `application-config.properties`
- `FileName.BUFFER_SIZE` → buffer size constant for file I/O

---

## WebConstants.java

Central constants class. Key inner classes:

```java
WebConstants.SessionData.USERS      // session key for logged-in Users entity
WebConstants.FileName.APPLICATION_CONFIG
WebConstants.FileName.BUFFER_SIZE
WebConstants.StatusConstants.*      // application status string constants
```

---

## StaticData.java

Loads common data needed on every page:
```java
ModelAndView view = StaticData.loadDataForHomepage(
    CommonService commonService,
    Users users,
    ModelMap model,
    Locale locale,
    ModelAndView view
);
```
Call this on every page-load GET handler after getting the session user.

---

## CryptoUtil.java / CryptUtility.java / EncryptionUtil.java

Encryption utilities. The project uses TripleDES encryption for sensitive data (e.g. PMC Care integration):
```java
// Keys from application-config.properties:
// encryptionKey=!#P5DC5ANMONKSZEFSNNWVYV
// encryptionIV=@#RLH27Q
// checksumKey=!#N61IJSDDRC
```
Use `CryptoUtil` for general encryption/decryption. `ValidateCheckSum` for checksum verification.

---

## PDF Generation

### CertificatePdfGenerator / CertificatePdfGenerator1
Generates certificate PDFs using iText 2.0.6 and OpenHtmlToPdf 1.0.10.

### PDFWriter
Low-level PDF writing utility.

### DummyCertificatePdfGenerator
Test/dummy certificate generator — not for production use.

### External PDF Utility
For HTML-to-PDF via the external utility service:
```java
String generatePdfUtilityUrl = PropertyFetcher.getPropertyValue(
    FileName.APPLICATION_CONFIG, "generate.pdf.utility");
// POST HTML content to this URL to get PDF back
// URL: http://localhost:9094/api/generate-html (local)
//      http://115.124.113.50:9094/api/generate-html (UAT)
```

PDF base URL for certificates:
```java
String pdfUrl = PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "pdf.url");
// https://certificate.pmrda.gov.in/pmrda/rts/certificate_
```

---

## QR Code

### QRCodeGenerator (controller) + QRCodeUtil + ZXingHelper
```java
// Generate QR code image bytes
byte[] qrBytes = QRCodeUtil.generateQRCode(String content, int width, int height);

// ZXing helper for reading/writing barcodes
ZXingHelper.generate(...);
```
Dependencies: ZXing 3.4.1 (`com.google.zxing:core`) + QRGen 2.0 (`net.glxn.qrgen:javase`).

---

## DMSWebService.java

SOAP client for Primeleaf Krystal DMS integration:
```java
DMSWebService dms = new DMSWebService();
SignOnResponse signOn = dms.signOn(username, password);
WorkflowStatusResponse status = dms.getWorkflowStatus(...);
KDocumentIndex index = dms.getDocumentIndex(...);
```
Uses Apache Axis 1.4. DMS is used for document workflow and status updates.

### UpdateStatusFromDMS.java
Polls DMS and updates application status in the local database.

---

## External HTTP Clients

### HtmlApiClient.java
Simple HTTP client for calling internal HTML/API endpoints.

### RestTemplateHttps.java
Spring `RestTemplate` configured for HTTPS with custom SSL settings.

### OkHttp (via OkHttpClient)
Used directly in controllers for some external API calls:
```java
OkHttpClient client = new OkHttpClient();
Request request = new Request.Builder()
    .url(url)
    .post(RequestBody.create(body, mediaType))
    .build();
Response response = client.newCall(request).execute();
```

---

## WhatsApp Integration

### InfobipWhatsAppSender / InfobipTextSenderty / InfoWhatsAppTextSender
Send WhatsApp messages via Infobip API.

### WhatsAppService (in `com.pmc.whatsapp`)
Spring service bean for WhatsApp messaging. Inject via `@Autowired`.

WhatsApp URL from config:
```java
String whatsAppUrl = PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "whatsapp.certificate.url");
// http://115.124.110.243:9094/api/sendCertificate (UAT)
```

---

## Property Tax Integration

### PropertyTaxWebServices.java / PmcPropertyTaxWebServiceCaller.java
SOAP/REST clients for PMC property tax system integration.

---

## PenaltyCalculator.java
Calculates penalties for overdue applications or payments.

---

## PosUtil.java
Utilities for POS (Point of Sale) terminal integration at CFC counters.

---

## OtherDatabaseConnections.java
Utility for connecting to secondary databases (e.g. property tax DB). Use only when `CommonService` cannot be used.

---

## AllServicesData.java
Static data holder for all available services. Used to populate service lists without DB queries.

---

## EmailScheduler.java
Scheduled email sending. Uses `javax.mail` with the configured SMTP server.

---

## KeyGenerator.java
Generates encryption keys. Used during setup/configuration, not in request handling.

---

## SMS Template Keys (application-config.properties)

| Key | Usage |
|---|---|
| `CITIZEN_REGISTRATION` | New citizen registration |
| `CITIZEN_REGISTRATION_OTP` | Registration OTP |
| `CITIZEN_ACC_RECOVER_OTP` | Account recovery OTP |
| `ONLINE_RTS_FORGOT_PASSWORD_OTP` | Forgot password OTP |
| `ONLINE_RTS_APPLICATION_SUBMITTED` | Application submitted confirmation |
| `ONLINE_RTS_IN_PROCESS` | Application under verification |
| `ONLINE_RTS_LEVEL_APPROVED` | Approved at a level, forwarded |
| `ONLINE_RTS_FINAL_APPROVED` | Final approval |
| `ONLINE_RTS_PAYMENT_REQUIRED` | Payment required |
| `ONLINE_RTS_CHALLAN_GENERATED` | Challan generated |
| `ONLINE_RTS_REJECTED` | Application rejected |
| `ONLINE_RTS_SERVICE_COMPLETED` | Service completed |
| `CITIZEN_APPLICATION_REJECTED_NO_REOPEN` | Rejected, cannot reopen |

Each template has a corresponding `_TEMPLATE_ID` key for DLT compliance.
