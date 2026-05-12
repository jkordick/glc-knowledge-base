# Module Specification: Payment Interface

**Feature**: 001-reverse-engineer-modules  
**User Story**: US-5 — Reverse Engineer Payment Interface (P2)  
**Date**: 2026-05-12  
**Source**: `src/Z-OS-Connect-Payment-Interface/src/` (10 Java files), `src/Z-OS-Connect-Payment-Interface/pom.xml`

---

## 1. Module Overview

**Purpose**: A single-function Spring Boot web application providing a Thymeleaf-rendered form for making debit/credit payments against bank accounts. Forwards payment requests to the COBOL backend via z/OS Connect EE.

**Technology Stack**:
- **Framework**: Spring Boot 3.5.11, Spring MVC, Thymeleaf
- **HTTP client**: Spring WebFlux `WebClient` (used synchronously via `.block()`)
- **Validation**: Jakarta Bean Validation
- **Build**: Maven WAR → CICS bundle via `cics-bundle-maven-plugin` → JVM Server `CBSAWLP`
- **Java**: 17
- **Context path**: `/paymentinterface-1.1`
- **Server port**: 19080

**Architecture**:
```
Browser ──HTML form──▶ WebController (Spring MVC @Controller)
                            │
  REST client ──POST──▶ ParamsController (@RestController)
                            │
                        WebClient (.block())
                            │
                            ▼
                    z/OS Connect EE
                     PUT /makepayment/dbcr
                            │
                        COMMAREA
                            │
                            ▼
                    COBOL Program DBCRFUN
                     ┌──────┼──────┐
                     ▼      ▼      ▼
                   DB2    DB2    (validation)
                (ACCOUNT) (PROCTRAN)
```

---

## 2. Source File Inventory

### 2.1 Application Bootstrap (2 files)

| Class | Purpose |
|-------|---------|
| `PaymentInterface.java` | `@SpringBootApplication` entry point. JCommander CLI arg parsing, logs z/OS Connect address. |
| `ServletInitializer.java` | `SpringBootServletInitializer` for WAR deployment to Liberty. |

### 2.2 Configuration / Utility (3 files)

| Class | Purpose |
|-------|---------|
| `ConnectionInfo.java` | Static holder for z/OS Connect host/port. Reads `CBSA_ZOSCONN_HOST` and `CBSA_ZOSCONN_PORT` system properties. **Identical copy** from Customer Services module. |
| `JsonPropertyNamingStrategy.java` | Custom Jackson `PropertyNamingStrategy`: strips 3-char prefix from method names. **Identical copy** from Customer Services module. |
| `OutputFormatUtils.java` | Static utility for date/number formatting. **Dead code** — never called in this module. Copied from Customer Services where it is used. |

### 2.3 Controllers (2 files)

| Class | Purpose |
|-------|---------|
| `WebController.java` | `@Controller` — Thymeleaf form rendering and payment processing via z/OS Connect. Contains 4 embedded exception classes. |
| `ParamsController.java` | `@RestController` — Programmatic REST API for payments (returns JSON). |

### 2.4 JSON/DTO Classes (4 files)

| Class | Purpose |
|-------|---------|
| `PaymentInterfaceJson.java` | Root JSON wrapper. Contains `DbcrJson` as `@JsonProperty("PAYDBCR")`. |
| `DbcrJson.java` | COMMAREA-mapped DTO: account number, amount, sort code, balances, success/fail codes, origin. |
| `OriginJson.java` | Caller identity DTO: applid, userid, facility name/type, network ID. Organisation name split across `commApplid` (chars 1–8) and `commUserid` (chars 9–16). |
| `TransferForm.java` | Form-backing bean: `acctNumber`, `debit` (boolean), `amount` (Float), `organisation`. |

### 2.5 Templates & Static Resources (2 files)

| File | Purpose |
|------|---------|
| `paymentInterfaceForm.html` | Single Thymeleaf template. Uses Carbon Design System (CDN). Form with account number, debit/credit radio, amount, organisation. |
| `styles/styles.css` | Custom CSS with Carbon color conventions. |

---

## 3. Detailed Specifications

### 3.1 Controller Endpoints

#### WebController (Thymeleaf UI)

| # | HTTP | Path | Form Object | z/OS Connect Call | Response |
|---|------|------|-------------|-------------------|----------|
| 1 | GET | `/`, `` | `TransferForm` (empty) | — | `paymentInterfaceForm` template |
| 2 | POST | `/paydbcr` | `@Valid TransferForm` | `PUT /makepayment/dbcr` | `paymentInterfaceForm` with results |

**POST `/paydbcr` flow**:
1. Validate `TransferForm` via Jakarta Bean Validation
2. Construct `PaymentInterfaceJson` from form data (negative amount = debit, positive = credit)
3. Serialize to JSON via `ObjectMapper`
4. Send HTTP PUT to `{z/OS Connect}/makepayment/dbcr` via `WebClient.block()`
5. Deserialize response to `PaymentInterfaceJson`
6. Check `CommFailCode`: `1` = not found, `3` = insufficient funds, `4` = invalid type
7. Set `largeText`/`smallText` model attributes for display
8. Return same template with `results=true`

#### ParamsController (REST API)

| # | HTTP | Path | Parameters | Response |
|---|------|------|-----------|----------|
| 3 | POST | `/submit` | `acctnum` (String), `amount` (float), `organisation` (String) — all `@RequestParam` | `PaymentInterfaceJson` (JSON) |

**POST `/submit` flow**:
1. Construct `TransferForm` from params (debit defaults to `true` — always debits)
2. Same z/OS Connect call as WebController
3. Returns raw JSON (or `null` on any exception)

### 3.2 Validation Check: `checkIfResponseValidDbcr()`

| Fail Code | Exception | Message |
|-----------|-----------|---------|
| `1` | `AccountNotFoundException` | Account not found |
| `3` | `InsufficientFundsException` | Insufficient funds |
| `4` | `InvalidAccountTypeException` | Invalid account type |

### 3.3 Embedded Exception Classes (WebController.java)

| Class | Used? | Purpose |
|-------|-------|---------|
| `InsufficientFundsException` | Yes | Fail code 3 |
| `InvalidAccountTypeException` | Yes | Fail code 4 |
| `AccountNotFoundException` | Yes | Fail code 1 |
| `TooManyAccountsException` | **No** — dead code | Copied from Customer Services |
| `ItemNotFoundException` | **No** — dead code | Copied from Customer Services |

---

## 4. z/OS Connect Integration

### 4.1 Connection Configuration

- **Host**: `System.getProperty("CBSA_ZOSCONN_HOST")` (no default fallback)
- **Port**: `System.getProperty("CBSA_ZOSCONN_PORT")` (no default fallback)
- **Scheme**: `http` (hardcoded)

### 4.2 z/OS Connect Service → COBOL Program Mapping

| z/OS Connect API | Method | z/OS Connect Service | CICS Program | COMMAREA | Purpose |
|------------------|--------|---------------------|-------------|----------|---------|
| `/makepayment/dbcr` | PUT | `Pay` | `DBCRFUN` | `PAYDBCR` | Debit/credit account |

**COBOL Program `DBCRFUN` performs**:
1. SELECT account by number from DB2 ACCOUNT table
2. Validate account type (reject ISA debits if insufficient)
3. UPDATE account balance in DB2 ACCOUNT
4. INSERT audit record into DB2 PROCTRAN
5. Return success/fail code in COMMAREA

---

## 5. Data Access Patterns

This module does **not** access DB2 or VSAM directly. All data access is proxied through z/OS Connect → COBOL.

| Operation | Controller Path | z/OS Connect → COBOL | Data Store |
|-----------|-----------------|---------------------|-----------|
| Debit/credit | POST `/paydbcr` or `/submit` | `/makepayment/dbcr` → DBCRFUN | DB2 ACCOUNT (UPDATE) + PROCTRAN (INSERT) |

---

## 6. Inter-Module Integration Points

| From | To | Mechanism | Data |
|------|-----|-----------|------|
| Browser | PaymentInterface (Spring Boot) | HTTP/HTML forms | Form POST data |
| REST client | PaymentInterface (Spring Boot) | HTTP/JSON | `@RequestParam` |
| PaymentInterface | z/OS Connect EE | HTTP PUT via `WebClient.block()` | JSON (COMMAREA-mapped) |
| z/OS Connect EE | CICS COBOL Program DBCRFUN | COMMAREA | PAYDBCR copybook |
| DBCRFUN | DB2 | SQL | ACCOUNT (UPDATE), PROCTRAN (INSERT) |

**Shared infrastructure**: Uses same z/OS Connect gateway as Customer Services Interface. Both modules share system properties `CBSA_ZOSCONN_HOST`/`CBSA_ZOSCONN_PORT`.

---

## 7. Code Duplication with Customer Services Interface

The following 4 classes are identical copies between the two modules:

| Class | Used in Payment? | Used in Customer Services? |
|-------|-----------------|---------------------------|
| `ConnectionInfo.java` | Yes | Yes |
| `JsonPropertyNamingStrategy.java` | Effectively no (overridden by `@JsonProperty`) | Effectively no |
| `OutputFormatUtils.java` | **No** — dead code | Yes |
| `ServletInitializer.java` | Yes | Yes |

These should be extracted to a shared library.

---

## 8. Findings

| ID | Severity | Type | Location | Description |
|----|----------|------|----------|-------------|
| F-PAY-001 | **High** | Bug | `ConnectionInfo.java` | `getAddress()`/`getPort()` always read from `System.getProperty(...)`, ignoring JCommander defaults. Missing system properties cause NPE/NFE. JCommander annotations are dead code. |
| F-PAY-002 | **High** | Bug | `ParamsController.java` | Returns `null` (HTTP 200 with null body) on any exception. No error information returned to caller. |
| F-PAY-003 | **Medium** | Bug | `WebController.java` | `checkIfResponseValidDbcr()` parses `CommFailCode` which defaults to `" "` (space). `Integer.parseInt(" ")` throws `NumberFormatException`, caught by generic handler with confusing message. |
| F-PAY-004 | **Medium** | Precision | `DbcrJson.java`, `TransferForm.java` | `float` used for monetary amounts (`commAmt`, `amount`). Should use `BigDecimal` to avoid floating-point rounding errors. |
| F-PAY-005 | **Medium** | Dead code | `OutputFormatUtils.java` | Entirely unused in this module. Copied from Customer Services. |
| F-PAY-006 | **Medium** | Dead code | `WebController.java` | `TooManyAccountsException` and `ItemNotFoundException` declared but never thrown. Copied from Customer Services. |
| F-PAY-007 | **Medium** | Dead code | `JsonPropertyNamingStrategy.java` | Applied via `@JsonNaming` but all DTO fields have explicit `@JsonProperty`, making the strategy redundant. |
| F-PAY-008 | **Low** | Inconsistency | `DbcrJson.java` | `@JsonProperty("mSortC")` doesn't follow `Comm*` pattern of all other fields. Setter `setCommC()` doesn't match field `commSortC`. |
| F-PAY-009 | **Low** | Inconsistency | `ParamsController.java` / README | README says endpoint uses GET but actual annotation is `@PostMapping`. |
| F-PAY-010 | **Low** | Design issue | `ParamsController.java` | Always performs debit (no credit option). 3-arg `TransferForm` constructor doesn't set `debit` flag, defaulting to `true`. |
| F-PAY-011 | **Low** | Misleading | `WebController.java` | Method `checkPersonInfo` processes payments, not person info. Copy-paste artifact. |
| F-PAY-012 | **Low** | No timeout | `WebController.java` | `WebClient` calls have no timeout. Hung z/OS Connect blocks servlet thread indefinitely. |
| F-PAY-013 | **Low** | No authentication | All endpoints | No authentication or authorization. Relies on network-level security. |
| F-PAY-014 | **Low** | Validation gap | `TransferForm.java` | `@NotNull` on primitive `boolean debit` is meaningless — primitive can never be null. |

---

**Coverage Validation**: This document covers 100% of source files:
- 10/10 Java source files documented ✓
- 1/1 Thymeleaf template documented ✓
- 1/1 z/OS Connect service mapping documented ✓
- All endpoints (3 total) documented ✓
- All COBOL program mappings documented ✓
