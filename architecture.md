# OnlineRTS — Architecture & Patterns

## Package Structure (OnlineRTS_Controller)

```
com.pmc
├── controller/          # Spring MVC controllers (61 classes)
│   ├── servicesimpl/    # CertificateService and similar service impls
│   └── daoimpl/         # Spring Data JPA repositories
├── model/               # Request/response DTOs and form-backing beans (96 classes)
├── spring/
│   └── config/          # All Spring configuration classes
├── utils/               # Utilities, PDF generators, external service clients
├── payment/             # Payment gateway integration
├── whatsapp/            # WhatsApp messaging integration
└── domain/              # Domain helpers (e.g. LocaleLanguage)
```

---

## Request Flow

```
Browser → Spring DispatcherServlet
       → Security Filter Chain (XSS, CSP, Session Hijack, SameSite)
       → NoCacheInterceptor (for /secure/**, /admin/**)
       → @Controller method
       → CommonService (or specialized service)
       → JPA/Hibernate → Database
       → ModelAndView → Apache Tiles → JSP → Response
```

---

## Controller Pattern

All controllers follow this structure:

```java
@Controller
@Repository        // ← incorrect annotation, present in existing code — do not add to new controllers
@Transactional
public class XyzController {

    @Autowired
    CommonService commonService;

    @RequestMapping(value = "/path", method = RequestMethod.GET)
    public ModelAndView getPage(HttpSession session, ModelMap model, Locale locale) {
        Users users = (Users) session.getAttribute(SessionData.USERS);
        ModelAndView view = new ModelAndView("viewName");
        view = StaticData.loadDataForHomepage(commonService, users, model, locale, view);
        // ... populate model
        return view;
    }

    @RequestMapping(value = "/ajaxEndpoint", method = RequestMethod.POST)
    public @ResponseBody Map<String, Object> ajaxMethod(...) {
        Map<String, Object> map = new HashMap<>();
        // ... logic
        return map;
    }
}
```

Key points:
- Session user is always retrieved via `session.getAttribute(SessionData.USERS)` → cast to `Users`
- `StaticData.loadDataForHomepage()` is called on every page load to populate common nav/menu data
- AJAX endpoints return `Map<String, Object>` with `@ResponseBody`
- File paths are resolved via `PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "key")`

---

## Data Access Pattern

`CommonService` is the single generic data access facade used everywhere:

```java
// Find by primary key
Entity e = (Entity) commonService.findById(Entity.class, id);

// Named query (defined on entity with @NamedQuery)
List<X> list = commonService.fetchByNamedQuery("queryName", "param1=value1=type&param2=value2=type");

// Native SQL query
List<Object[]> rows = commonService.fetchByNativeQuery("SELECT * FROM table WHERE ...");

// Native query mapped to entity class
List<Entity> list = commonService.fetchByNativeQuery("SELECT ...", Entity.class);

// Single result from named query
X obj = (X) commonService.fetchObjectByNamedQuery("queryName", "param=value=type");

// Save / update
commonService.saveOrUpdate(entity);
commonService.merge(entity);
commonService.delete(entity);
```

Named query parameter format: `"paramName=value=type"` — multiple params separated by `&`.
Types: `long`, `string`, `date`, etc.

---

## Security Architecture

### Filter Chain (in order)
1. `XssFilter` + `XssRequestWrapper` — strips XSS from all request parameters
2. `ContentSecurityPolicyFilter` — adds CSP headers
3. `SameSiteCookieFilter` — adds SameSite=Strict to cookies
4. `SessionHijackProtectionFilter` — validates session integrity
5. `ETagHeaderFilter` — ETag caching
6. Spring Security filter chain

### Spring Security Config (`SpringSecurityConfiguration`)
- CSRF: **disabled**
- Sessions: max 1 per user, `maxSessionsPreventsLogin = true`
- Invalid session redirect: `/login?expired=true`
- Logout: `/logout` → `/login?logout`, invalidates session, deletes JSESSIONID
- Frame options: `SAMEORIGIN`
- HSTS: enabled, 1 year, includeSubDomains
- CORS: restricted to `https://services.pmc.gov.in/`

### Login Attempt Tracking
- `LoginAttemptService` — IP-based lockout (Guava cache)
- `UserLoginAttemptService` — user-specific attempt tracking

---

## Session Data Keys

Defined in `WebConstants.SessionData`:

```java
SessionData.USERS        // → Users entity (logged-in officer/citizen)
"usmId"                  // → Long, set on login
```

---

## Configuration Loading

All runtime config is loaded from `application-config.properties` via:

```java
PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "key.name")
```

File paths are OS-aware — always check both Windows and Linux keys:
```java
String osName = System.getProperty("os.name");
if (osName.toLowerCase().contains("windows")) {
    filePath = PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "WINDOWS_UPLOADFILE_ATTACHMENTS");
} else {
    filePath = PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "UPLOADFILE_ATTACHMENTS");
}
```

---

## Internationalisation

```java
Locale currentLocale = LocaleContextHolder.getLocale();
if (currentLocale.getLanguage().equalsIgnoreCase("en")) {
    // English path
} else {
    // Marathi path
}
```

Views receive `locale` and `currentLocale` as model attributes on every page load.

---

## Transaction Management

- All controllers are `@Transactional` at class level
- JPA transaction manager (`JpaTransactionManager`) backed by Hibernate
- C3P0 connection pool configured in `JPAConfig` via `jdbc.properties`
- Entity scanning: `com.pmc.entities` package (in `OnlineRTS_Entities` module)
- JPA repositories: `com.pmc.controller.daoimpl` package

---

## View Resolution

Apache Tiles 3.0.5 is used as the template engine. View names in `ModelAndView` map to Tiles definitions, not directly to JSP files.

---

## Notification Pattern

After application status changes, notifications are sent via:

```java
// SMS
Util.sendSms(mobileNumber, MessageFormat.format(
    PropertyFetcher.getPropertyValue(FileName.APPLICATION_CONFIG, "TEMPLATE_KEY"),
    param1, param2
));

// Email
Util.sendMail(toAddress, subject, body);

// WhatsApp
// Via InfobipWhatsAppSender or WhatsAppService
```

SMS templates and DLT template IDs are all stored in `application-config.properties`.
