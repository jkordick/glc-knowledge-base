# Module Specification: Customer Services Interface

**Feature**: 001-reverse-engineer-modules  
**User Story**: US-4 — Reverse Engineer Customer Services Interface (P2)  
**Date**: 2026-05-12  
**Source**: `src/Z-OS-Connect-Customer-Services-Interface/src/` (36 Java files), `src/Z-OS-Connect-Customer-Services-Interface/pom.xml`

---

## 1. Module Overview

**Purpose**: Provides a Spring Boot web application with server-side rendered Thymeleaf forms for performing customer and account CRUD operations. Acts as an alternative UI to the Carbon React frontend, connecting to the same COBOL backend programs via z/OS Connect EE.

**Technology Stack**:
- **Framework**: Spring Boot 3.5.11, Spring MVC, Thymeleaf
- **HTTP client**: Spring WebFlux `WebClient` (used synchronously via `.block()`)
- **Validation**: Jakarta Bean Validation + Hibernate Validator `@Range`
- **Build**: Maven WAR → CICS bundle via `cics-bundle-maven-plugin` → JVM Server `CBSAWLP`
- **Java**: 17
- **Context path**: `/customerservices-1.0`
- **Server port**: 19080

**Architecture**:
```
Browser ──HTML forms──▶ WebController (Spring MVC @Controller)
                            │
                        WebClient (.block())
                            │
                            ▼
                    z/OS Connect EE (REST gateway)
                            │
                        COMMAREA
                            │
                            ▼
                    CICS COBOL Programs
                     ┌──────┼──────┐
                     ▼      ▼      ▼
                   DB2    VSAM   CICS
                (ACCOUNT, (CUSTOMER)
                 PROCTRAN,
                 CONTROL)
```

---

## 2. Source File Inventory

### 2.1 Application Bootstrap (2 files)

| Class | Purpose |
|-------|---------|
| `CustomerServices.java` | `@SpringBootApplication` entry point. JCommander CLI arg parsing, logs z/OS Connect address. |
| `ServletInitializer.java` | `SpringBootServletInitializer` for WAR deployment to Liberty. |

### 2.2 Configuration / Utility (3 files)

| Class | Purpose |
|-------|---------|
| `ConnectionInfo.java` | Static holder for z/OS Connect host/port. Reads `CBSA_ZOSCONN_HOST` and `CBSA_ZOSCONN_PORT` system properties. |
| `JsonPropertyNamingStrategy.java` | Custom Jackson `PropertyNamingStrategy`: strips 3-char prefix from getter/setter names for JSON keys. |
| `OutputFormatUtils.java` | Static utility: date formatting (`DDMMYYYY` → `DD/MM/YYYY`), leading-zero padding. |

### 2.3 Controller (1 file + 4 embedded exception classes)

| Class | Purpose |
|-------|---------|
| `WebController.java` | `@Controller` handling all 19 web routes. Single controller, server-side rendered. |
| `InsufficientFundsException` | Embedded in WebController. **Dead code** — never thrown. |
| `InvalidAccountTypeException` | Embedded in WebController. **Dead code** — never thrown. |
| `TooManyAccountsException` | Embedded in WebController. Thrown when >10 accounts per customer. |
| `ItemNotFoundException` | Embedded in WebController. Thrown on enquiry/delete not-found. |

| Class (separate file) | Purpose |
|-------|---------|
| `InvalidCustomerException.java` | Custom exception for invalid customer data. |

### 2.4 JSON/DTO Classes — Account Enquiry (3 files)

| Class | Purpose |
|-------|---------|
| `AccountEnquiryForm.java` | Form-backing bean. Validation: `@NotNull @Size(max=8) acctNumber`. |
| `AccountEnquiryJson.java` | Top-level response wrapper. Contains `InqaccJson`. |
| `InqaccJson.java` | COMMAREA mapping: account number, type, balances, interest rate, dates, success flag. |

### 2.5 JSON/DTO Classes — Customer Enquiry (4 files)

| Class | Purpose |
|-------|---------|
| `CustomerEnquiryForm.java` | Form-backing bean. Validation: `@NotNull @Size(min=1, max=10) custNumber`. |
| `CustomerEnquiryJson.java` | Top-level response wrapper. Contains `InqCustZJson`. |
| `InqCustZJson.java` | COMMAREA mapping: name, address, credit score, DOB, review date, success/fail codes. |
| `InqCustDob.java` | Nested date-of-birth (year/month/day fields). |
| `InqCustReviewDate.java` | Nested credit-score review date (year/month/day fields). |

### 2.6 JSON/DTO Classes — Create Account (4 files)

| Class | Purpose |
|-------|---------|
| `AccountType.java` | Enum: `CURRENT`, `LOAN`, `ISA`, `MORTGAGE`, `SAVING`. |
| `CreateAccountForm.java` | Form-backing bean. Validation: `@NotNull custNumber`, `@NotNull accountType`, overdraft, interest rate. |
| `CreateAccountJson.java` | Top-level request wrapper. Contains `CreaccJson`. |
| `CreaccJson.java` | COMMAREA mapping: account type, customer no, balances, dates, success/fail codes. |
| `CreaccKeyJson.java` | Nested sort-code + account-number key. Also reused by create-customer. |

### 2.7 JSON/DTO Classes — Create Customer (3 files)

| Class | Purpose |
|-------|---------|
| `CreateCustomerForm.java` | Form-backing bean. Validation: `@NotNull custName`, `custAddress`, `custDob(8)`. Date rearrangement `YYYY-MM-DD` → `DDMMYYYY`. |
| `CreateCustomerJson.java` | Top-level request wrapper. Contains `CrecustJson`. |
| `CrecustJson.java` | COMMAREA mapping: name, address, DOB, credit score, review date, success/fail codes. |

### 2.8 JSON/DTO Classes — Update Account (3 files)

| Class | Purpose |
|-------|---------|
| `UpdateAccountForm.java` | Form-backing bean. Uses `@Range(min=1, max=99999999)` for account number. Date conversion `YYYY-MM-DD` → `DDMMYYYY`. |
| `UpdateAccountJson.java` | Top-level request wrapper. Contains `UpdaccJson`. |
| `UpdaccJson.java` | COMMAREA mapping: all account fields, success flag. |

### 2.9 JSON/DTO Classes — Update Customer (3 files)

| Class | Purpose |
|-------|---------|
| `UpdateCustomerForm.java` | Form-backing bean. Validation: `@NotNull @Size(max=10) custNumber`. |
| `UpdateCustomerJson.java` | Top-level request wrapper. Contains `UpdcustJson`. |
| `UpdcustJson.java` | COMMAREA mapping: name, address, DOB, credit score, review date, success/fail codes. |

### 2.10 JSON/DTO Classes — Delete Account (2 files)

| Class | Purpose |
|-------|---------|
| `DeleteAccountJson.java` | Top-level response wrapper. Contains `DelaccJson`. |
| `DelaccJson.java` | COMMAREA mapping: all account fields, success/fail codes. Handles empty-string → null for `AccountType` enum. |

### 2.11 JSON/DTO Classes — Delete Customer (2 files)

| Class | Purpose |
|-------|---------|
| `DeleteCustomerJson.java` | Top-level response wrapper. Contains `DelcusJson`. |
| `DelcusJson.java` | COMMAREA mapping: name, address, DOB, credit score, success/fail codes. |

### 2.12 JSON/DTO Classes — List Accounts (3 files)

| Class | Purpose |
|-------|---------|
| `ListAccJson.java` | Top-level response wrapper. Contains `InqAccczJson`. |
| `InqAccczJson.java` | COMMAREA mapping: customer number, found flag, list of `AccountDetails`. |
| `AccountDetails.java` | Individual account in list: balances, type, dates, overdraft, eye-catcher. |

### 2.13 Thymeleaf Templates (10 templates + 1 CSS)

| Template | Purpose |
|----------|---------|
| `customerServices.html` | Landing page / service menu |
| `accountEnquiryForm.html` | Enquire account by number |
| `customerEnquiryForm.html` | Enquire customer by number |
| `listAccountsForm.html` | List all accounts for a customer |
| `createAccountForm.html` | Create a new account |
| `createCustomerForm.html` | Create a new customer |
| `updateAccountForm.html` | Update account details |
| `updateCustomerForm.html` | Update customer details |
| `deleteAccountForm.html` | Delete an account |
| `deleteCustomerForm.html` | Delete a customer |
| `styles/styles.css` | Single CSS stylesheet |

---

## 3. Detailed Specifications

### 3.1 Controller Endpoints (WebController.java)

| # | HTTP | Path | Form Object | z/OS Connect URL | z/OS Method | Response DTO | Template |
|---|------|------|-------------|-----------------|-------------|-------------|----------|
| 1 | GET | `/`, `/services` | — | — | — | — | `customerServices` |
| 2 | GET | `/enqacct` | `AccountEnquiryForm` | — | — | — | `accountEnquiryForm` |
| 3 | POST | `/enqacct` | `@Valid AccountEnquiryForm` | `/inqaccz/enquiry/{acctNo}` | GET | `AccountEnquiryJson` | `accountEnquiryForm` |
| 4 | GET | `/enqcust` | `CustomerEnquiryForm` | — | — | — | `customerEnquiryForm` |
| 5 | POST | `/enqcust` | `@Valid CustomerEnquiryForm` | `/inqcustz/enquiry/{custNo}` | GET | `CustomerEnquiryJson` | `customerEnquiryForm` |
| 6 | GET | `/listacc` | `CustomerEnquiryForm` | — | — | — | `listAccountsForm` |
| 7 | POST | `/listacc` | `@Valid CustomerEnquiryForm` | `/inqacccz/list/{custNo}` | GET | `ListAccJson` | `listAccountsForm` |
| 8 | GET | `/createacc` | `CreateAccountForm` | — | — | — | `createAccountForm` |
| 9 | POST | `/createacc` | `@Valid CreateAccountForm` | `/creacc/insert` | POST | `CreateAccountJson` | `createAccountForm` |
| 10 | GET | `/createcust` | `CreateCustomerForm` | — | — | — | `createCustomerForm` |
| 11 | POST | `/createcust` | `@Valid CreateCustomerForm` | `/crecust/insert` | POST | `CreateCustomerJson` | `createCustomerForm` |
| 12 | GET | `/updateacc` | `UpdateAccountForm` | — | — | — | `updateAccountForm` |
| 13 | POST | `/updateacc` | `@Valid UpdateAccountForm` | `/updacc/update` | PUT | `UpdateAccountJson` | `updateAccountForm` |
| 14 | GET | `/updatecust` | `UpdateCustomerForm` | — | — | — | `updateCustomerForm` |
| 15 | POST | `/updatecust` | `@Valid UpdateCustomerForm` | `/updcust/update` | PUT | `UpdateCustomerJson` | `updateCustomerForm` |
| 16 | GET | `/delacct` | `AccountEnquiryForm` | — | — | — | `deleteAccountForm` |
| 17 | POST | `/delacct` | `@Valid AccountEnquiryForm` | `/delacc/remove/{acctNo}` | DELETE | `DeleteAccountJson` | `deleteAccountForm` |
| 18 | GET | `/delcust` | `CustomerEnquiryForm` | — | — | — | `deleteCustomerForm` |
| 19 | POST | `/delcust` | `@Valid CustomerEnquiryForm` | `/delcus/remove/{custNo}` | DELETE | `DeleteCustomerJson` | `deleteCustomerForm` |

### 3.2 Validation Check Methods (WebController.java)

| Method | COBOL Fail Codes | Exceptions Thrown |
|--------|-----------------|-------------------|
| `checkIfResponseValidListAcc(AccountEnquiryJson)` | success == "N", custno == 0 | `ItemNotFoundException` |
| `checkIfResponseValidEnqCust(CustomerEnquiryJson)` | inqSuccess == "N" | `ItemNotFoundException` |
| `checkIfResponseValidListAcc(ListAccJson)` | customerFound == "N" | `ItemNotFoundException` |
| `checkIfResponseValidCreateAcc(CreateAccountJson)` | "1" = not found, "8" = too many, "A" = invalid type | `ItemNotFoundException`, `TooManyAccountsException`, `InvalidAccountTypeException` |
| `checkIfResponseValidCreateCust(CreateCustomerJson)` | "T" = invalid title | `InvalidCustomerException` |
| `checkIfResponseValidUpdateAcc(UpdateAccountJson)` | success == "N" | `ItemNotFoundException` |
| `checkIfResponseValidUpdateCust(UpdateCustomerJson)` | "4" = no name/address, "T" = invalid title | `ItemNotFoundException`, `InvalidCustomerException` |
| `checkIfResponseValidDeleteAcc(DeleteAccountJson)` | failCode == 1 | `ItemNotFoundException` |
| `checkIfResponseValidDeleteCust(DeleteCustomerJson)` | failCode == 1 | `ItemNotFoundException` |

---

## 4. z/OS Connect Integration

### 4.1 Connection Configuration

- **Host**: `System.getProperty("CBSA_ZOSCONN_HOST")` (no default fallback)
- **Port**: `System.getProperty("CBSA_ZOSCONN_PORT")` (no default fallback)
- **Scheme**: `http` (hardcoded)
- **Base URL**: `http://<host>:<port>`

### 4.2 z/OS Connect Service → COBOL Program Mapping

| z/OS Connect Path | HTTP Method | COBOL Copybook | COBOL Program |
|-------------------|-------------|----------------|---------------|
| `/inqaccz/enquiry/{acctNo}` | GET | INQACCZ.cpy | INQACC |
| `/inqcustz/enquiry/{custNo}` | GET | INQCUSTZ.cpy | INQCUST |
| `/inqacccz/list/{custNo}` | GET | INQACCCZ.cpy | INQACCCU |
| `/creacc/insert` | POST | CREACC.cpy | CREACC |
| `/crecust/insert` | POST | CRECUST.cpy | CRECUST |
| `/updacc/update` | PUT | UPDACC.cpy | UPDACC |
| `/updcust/update` | PUT | UPDCUST.cpy | UPDCUST |
| `/delacc/remove/{acctNo}` | DELETE | DELACC.cpy | DELACC |
| `/delcus/remove/{custNo}` | DELETE | DELCUS.cpy | DELCUS |

**JSON property naming**: JSON field names (e.g., `InqAccAccno`, `CommCustno`) directly mirror COBOL COMMAREA copybook field names via the custom `JsonPropertyNamingStrategy`.

---

## 5. Data Access Patterns

This module does **not** access DB2 or VSAM directly. All data access is proxied through z/OS Connect → COBOL programs.

| Operation | WebController Path | z/OS Connect → COBOL | Ultimate Data Store |
|-----------|-------------------|---------------------|-------------------|
| Enquire account | POST `/enqacct` | `/inqaccz` → INQACC | DB2 ACCOUNT |
| Enquire customer | POST `/enqcust` | `/inqcustz` → INQCUST | VSAM CUSTOMER |
| List accounts | POST `/listacc` | `/inqacccz` → INQACCCU | DB2 ACCOUNT |
| Create account | POST `/createacc` | `/creacc` → CREACC | DB2 ACCOUNT + PROCTRAN |
| Create customer | POST `/createcust` | `/crecust` → CRECUST | VSAM CUSTOMER + PROCTRAN |
| Update account | POST `/updateacc` | `/updacc` → UPDACC | DB2 ACCOUNT |
| Update customer | POST `/updatecust` | `/updcust` → UPDCUST | VSAM CUSTOMER |
| Delete account | POST `/delacct` | `/delacc` → DELACC | DB2 ACCOUNT + PROCTRAN |
| Delete customer | POST `/delcust` | `/delcus` → DELCUS | VSAM CUSTOMER + PROCTRAN |

---

## 6. Inter-Module Integration Points

| From | To | Mechanism | Data |
|------|-----|-----------|------|
| Browser | CustomerServices (Spring Boot) | HTTP/HTML forms | Form POST data |
| CustomerServices | z/OS Connect EE | HTTP via `WebClient.block()` | JSON (COMMAREA-mapped) |
| z/OS Connect EE | CICS COBOL Programs | COMMAREA | Copybook-structured bytes |
| COBOL Programs | DB2 | SQL (embedded) | ACCOUNT, PROCTRAN, CONTROL |
| COBOL Programs | VSAM | CICS File API | CUSTOMER KSDS |

**Shared COBOL programs**: This module invokes the same 9 COBOL programs (INQACC, INQCUST, INQACCCU, CREACC, CRECUST, UPDACC, UPDCUST, DELACC, DELCUS) as the React/WebUI path, but via z/OS Connect rather than direct CICS LINK.

---

## 7. Findings

| ID | Severity | Type | Location | Description |
|----|----------|------|----------|-------------|
| F-CSI-001 | **High** | Bug | `ConnectionInfo.java` | `getAddress()` and `getPort()` always read from `System.getProperty(...)`, ignoring JCommander `@Parameter` defaults. Missing system properties cause `NullPointerException`/`NumberFormatException`. JCommander annotations are dead code. |
| F-CSI-002 | **Medium** | Dead code | `WebController.java` | `InsufficientFundsException` and `InvalidAccountTypeException` are defined but never thrown or caught. Likely copied from the Payment Interface module. |
| F-CSI-003 | **Medium** | Dead code | `CreateCustomerForm.java` | `isValidTitle()` method defined but never called. Title validation relies solely on COBOL backend fail code `"T"`. |
| F-CSI-004 | **Medium** | Bug | `WebController.java` | `InvalidCustomerException` is thrown in `checkIfResponseValidCreateCust` but the controller catches `IllegalArgumentException`, not `InvalidCustomerException`. It propagates to the generic `catch (Exception e)` with a vague "error processing request" message. |
| F-CSI-005 | **Medium** | Deprecated API | `JsonPropertyNamingStrategy.java` | Extends deprecated `PropertyNamingStrategy` (Jackson). Should use `PropertyNamingStrategies` (Jackson 2.12+). |
| F-CSI-006 | **Medium** | Design smell | `WebController.java` | All 4 exception classes are package-private classes embedded within `WebController.java`. Should be separate files for maintainability. |
| F-CSI-007 | **Medium** | Type inconsistency | Multiple DTOs | Same logical fields (sort code, customer number, dates) use `int` in some DTOs and `String` in others depending on which z/OS Connect service is called. |
| F-CSI-008 | **Low** | Duplicate field | `DelaccJson.java` | Two fail-code fields: `delaccDelFailCode` (`@JsonProperty("DelAccFailCd")`) and `delaccFailCode` (`@JsonProperty("DelAccDelFailCd")`). Controller only checks the first. |
| F-CSI-009 | **Low** | Blocking reactive | `WebController.java` | `WebClient` used with `.block()` throughout, negating reactive benefits. Deliberate simplicity choice. |
| F-CSI-010 | **Low** | No timeout | `WebController.java` | `WebClient` calls have no timeout configured. A hung z/OS Connect service blocks the servlet thread indefinitely. |
| F-CSI-011 | **Low** | Comment numbering | `WebController.java` | Section numbering skips: "6. Update an account" → "6. Update a customer" (should be 7), then jumps to 8, 9. |
| F-CSI-012 | **Low** | No authentication | All endpoints | No authentication or authorization on any endpoint. Relies on network-level security (z/OS, CICS). |
| F-CSI-013 | **Low** | Unused import | `WebController.java` | `PropertyNamingStrategies.SnakeCaseStrategy` imported but never used. |

---

**Coverage Validation**: This document covers 100% of source files:
- 36/36 Java source files documented ✓ (including bootstrap, config, controller, DTOs, exceptions)
- 10/10 Thymeleaf templates documented ✓
- 9/9 z/OS Connect service mappings documented ✓
- All form validation rules documented ✓
- All COBOL program mappings documented ✓
