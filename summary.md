# Reverse Engineering Summary: CICS Bank Sample Application (CBSA)

**Feature**: 001-reverse-engineer-modules  
**Date**: 2026-05-12

---

## 1. Module Coverage Summary

| Module | User Story | Document | Source Files | Coverage |
|--------|-----------|----------|-------------|----------|
| COBOL/BMS Backend | US-1 (P1) | [module-cobol-backend.md](module-cobol-backend.md) | 29 programs + 9 BMS maps + 37 copybooks = **75 files** | 100% ✓ |
| Data Layer | US-2 (P1) | [module-data-layer.md](module-data-layer.md) | 3 DB2 tables + 2 VSAM files + DDL JCL = **5 entities** | 100% ✓ |
| Carbon React UI + WebUI | US-3 (P2) | [module-react-ui.md](module-react-ui.md) | 31 React JS + 35 WebUI Java = **66 files** | 100% ✓ |
| Customer Services Interface | US-4 (P2) | [module-customer-services.md](module-customer-services.md) | 36 Java + 10 Thymeleaf templates = **47 files** | 100% ✓ |
| Payment Interface | US-5 (P2) | [module-payment-interface.md](module-payment-interface.md) | 10 Java + 1 Thymeleaf template = **12 files** | 100% ✓ |
| z/OS Connect Artefacts | US-6 (P3) | [module-zos-connect.md](module-zos-connect.md) | 10 APIs (AAR) + 10 Services (SAR) = **20 artefacts** | 100% ✓ |
| Cross-Module Integration Map | US-7 (P3) | [module-integration-map.md](module-integration-map.md) | All modules | 45 integration points, 4 traces ✓ |

**Total source files documented**: ~178 source files across all modules  
**Total REST endpoints documented**: 28 (WebUI) + 19 (Customer Services) + 3 (Payment Interface) + 10 (z/OS Connect) = **60 endpoints**

---

## 2. Central Findings Table

### High Severity

| ID | Module | Type | Location | Description |
|----|--------|------|----------|-------------|
| F-RUI-001 | React UI | Bug | `AccountDeleteTables.js` | `getAccountByNum()` called in component body without `useEffect` — triggers on every render, potential infinite API loop |
| F-RUI-002 | React UI | Bug | `AccountDetailsTable.js` | `currentAccountCustomerNumber` never populated — PUT always sends empty customer number |
| F-RUI-003 | React UI | Bug | `ProcessedTransactionCreateCustomerJSON.java` | `getReviewDate()` returns `customerDOB` instead of `customerReviewDate` — wrong audit data |
| F-RUI-004 | React UI | Bug | `CustomerResource.java` | Town/surname search reuses same JSONObject in loop — only last customer returned |
| F-CSI-001 | Customer Services | Bug | `ConnectionInfo.java` | `getAddress()`/`getPort()` ignore JCommander defaults, NPE if system properties missing |
| F-PAY-001 | Payment Interface | Bug | `ConnectionInfo.java` | Same as F-CSI-001 (identical copied class) |
| F-PAY-002 | Payment Interface | Bug | `ParamsController.java` | Returns `null` (HTTP 200) on any exception — no error info to caller |
| F-INT-001 | Integration | Dual access risk | VSAM CUSTOMER | React/WebUI accesses VSAM directly while Customer Services routes via z/OS Connect — no locking coordination |
| F-INT-002 | Integration | No cross-store RI | VSAM↔DB2 | Customer numbers link VSAM to DB2 with no foreign key — orphaned accounts possible |

### Medium Severity

| ID | Module | Type | Location | Description |
|----|--------|------|----------|-------------|
| F-RUI-005 | React UI | Dead code | `AccountForm.js`, `NewAccount.js` | Two empty/non-functional components never referenced |
| F-RUI-006 | React UI | Dead code | `AdminPage.js` | Unused imports, modal state, and handler functions |
| F-RUI-008 | React UI | Inconsistency | Multiple components | Sort code hardcoded `"987654"` in React vs dynamically fetched on backend |
| F-RUI-009 | React UI | Pattern issue | `CustomerDetailsTable.js` | Axios interceptors accumulate on every update call (memory leak) |
| F-RUI-010 | React UI | Pattern issue | Multiple components | Modals in `map()` loops share state — all open/close together |
| F-RUI-011 | React UI | Legacy code | `webui.data_access.*` | 5 wrapper classes for older JSP/Servlet UI, potentially unused |
| F-CSI-002 | Customer Services | Dead code | `WebController.java` | `InsufficientFundsException` and `InvalidAccountTypeException` never thrown |
| F-CSI-003 | Customer Services | Dead code | `CreateCustomerForm.java` | `isValidTitle()` defined but never called |
| F-CSI-004 | Customer Services | Bug | `WebController.java` | `InvalidCustomerException` thrown but caught as generic `Exception` |
| F-CSI-005 | Customer Services | Deprecated API | `JsonPropertyNamingStrategy.java` | Extends deprecated `PropertyNamingStrategy` |
| F-PAY-003 | Payment Interface | Bug | `WebController.java` | `Integer.parseInt(" ")` on default `CommFailCode` causes NFE |
| F-PAY-004 | Payment Interface | Precision | `DbcrJson.java` | `float` used for monetary amounts — should be `BigDecimal` |
| F-PAY-005 | Payment Interface | Dead code | `OutputFormatUtils.java` | Entirely unused in this module |
| F-PAY-006 | Payment Interface | Dead code | `WebController.java` | `TooManyAccountsException` and `ItemNotFoundException` never thrown |
| F-ZOS-001 | z/OS Connect | Non-standard REST | `inqaccz`, `inqacccz`, `inqcustz` | GET requests with request body |
| F-ZOS-002 | z/OS Connect | Non-standard REST | `delacc`, `delcus` | DELETE requests with request body |
| F-ZOS-003 | z/OS Connect | Missing mappings | `updacc` | No request/response mapping files |
| F-DL-001 | Data Layer | Design observation | PROCTRAN | No primary key or unique index on append-only audit table |
| F-DL-002 | Data Layer | Design observation | CUSTOMER→ACCOUNT | Cross-store relationship with no referential integrity |
| F-COB-004 | COBOL | Code duplication | CRDTAGY1-5 | Five nearly identical programs differing only in container name |
| F-COB-006 | COBOL | Design observation | INQCUST.cbl | Random customer lookup with retry loop (up to 100 attempts) |
| F-INT-003 | Integration | Inconsistency | Sort code | Different fetch mechanisms across three access paths |

### Low Severity

| ID | Module | Type | Location | Description |
|----|--------|------|----------|-------------|
| F-RUI-007 | React UI | Dead code | `App.js` | Dead `DebugRouter` class |
| F-RUI-012 | React UI | JSX anti-pattern | Multiple files | `class=` instead of `className=` |
| F-RUI-013 | React UI | Naming | `HBankDataAccess.java` | DB2 connection cache named `cornedBeef` |
| F-RUI-014 | React UI | Unused endpoints | REST API | 18 of 28 endpoints not called by React |
| F-RUI-015 | React UI | Deprecated API | `App.js` | react-router-dom v5 `Switch`/`Route` |
| F-CSI-006–013 | Customer Services | Various | Various | Type inconsistencies, duplicate fields, no timeout, no auth |
| F-PAY-007–014 | Payment Interface | Various | Various | Dead naming strategy, README inconsistency, no timeout, no auth |
| F-ZOS-004–010 | z/OS Connect | Various | Various | Missing auth header, duplicate artefacts, typo, naming, date format |
| F-DL-003 | Data Layer | Inconsistency | DDL vs DCLGEN | DECIMAL precision mismatches (DDL authoritative) |
| F-DL-004 | Data Layer | Design observation | CONTROL table | CONTROL_NAME not declared NOT NULL despite unique index |
| F-DL-005 | Data Layer | Concern | ABNDFILE | Write-only, could grow unbounded |
| F-COB-001–010 | COBOL | Various | Various | Empty copybooks, orphan symbolic maps, unused copybooks |

**Total findings**: 9 High, ~15 Medium, ~25 Low = **~49 findings**

---

## 3. Cross-Module Code Duplication

| Duplicated Class | Customer Services | Payment Interface | Action Recommended |
|-----------------|-------------------|-------------------|--------------------|
| `ConnectionInfo.java` | ✓ | ✓ (identical) | Extract to shared library |
| `JsonPropertyNamingStrategy.java` | ✓ | ✓ (identical) | Extract to shared library |
| `OutputFormatUtils.java` | ✓ (used) | ✓ (dead code) | Extract to shared library, remove from Payment |
| `ServletInitializer.java` | ✓ | ✓ (identical) | Extract to shared library |

---

## 4. Success Criteria Validation

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| SC-001 | Each module document has ≥ 90% source file coverage | ✓ PASS | All 6 modules at 100% coverage |
| SC-002 | Findings flagged with severity and location | ✓ PASS | ~49 findings across all modules, all with severity + location |
| SC-003 | Integration map produced with ≥ 3 end-to-end traces | ✓ PASS | 4 traces: Create Customer (React), Delete Customer (CS), Make Payment (PI), Update Account (React) |
| SC-004 | Every inter-module integration point documented | ✓ PASS | 45 integration points enumerated in module-integration-map.md §3 |
| SC-005 | All documents follow consistent structure | ✓ PASS | All use: Overview, File Inventory, Detailed Specs, Data Access, Integration Points, Findings |
