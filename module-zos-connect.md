# Module Specification: z/OS Connect Artefacts

**Feature**: 001-reverse-engineer-modules  
**User Story**: US-6 — Reverse Engineer z/OS Connect Artefacts (P3)  
**Date**: 2026-05-12  
**Source**: `src/zosconnect_artefacts/` (10 API definitions, 10 service definitions)

---

## 1. Module Overview

**Purpose**: z/OS Connect EE acts as the REST-to-CICS gateway, exposing COBOL programs as RESTful JSON APIs. Each API definition (AAR) maps an HTTP endpoint to a service definition (SAR), which in turn invokes a CICS program via COMMAREA. The gateway handles JSON ↔ EBCDIC COMMAREA translation using `.si` (service interface) definitions generated from COBOL copybooks.

**Architecture**:
```
Spring Boot UIs                          React/WebUI (Liberty)
(Customer Services,  ──HTTP──▶ z/OS Connect EE ──COMMAREA──▶ COBOL Programs ──▶ DB2/VSAM
 Payment Interface)           (10 APIs, 10 Services)
```

**Key Properties (all services)**:
- **Service type**: `cicsCommarea`
- **Provider**: `cics`
- **Connection reference**: `cicsConn`
- **CCSID**: 37 (IBM037 / EBCDIC US-Canada)
- **Runtime type**: `CICS_COMMAREA`

---

## 2. Source File Inventory

### 2.1 API Archives (AAR) — 10 APIs

Each API directory contains:
- `package.xml` — API definition (basePath, serviceRef)
- `api-docs/swagger.json` — Swagger 2.0 specification
- `api/<path>/<METHOD>/mapping.xml` — HTTP ↔ service mapping
- `api/<path>/<METHOD>/request.map` — Request transformation (MSL)
- `api/<path>/<METHOD>/response.map` — Response transformation (MSL)
- `services/<ServiceName>/` — Inline copy of service definition

### 2.2 Service Archives (SAR) — 10 Services

Each service directory contains:
- `service.properties` — CICS connection configuration
- `service-interfaces/<X>.si` — COMMAREA field layout
- `bin/<X>.xml` — Service definition XML
- `bin/schemas/<X>Request.json` / `<X>Response.json` — JSON Schema

---

## 3. API Definitions

| # | API | HTTP Method | REST Path | Service Ref | COBOL Copybook |
|---|-----|-------------|-----------|-------------|----------------|
| 1 | `creacc` | POST | `/creacc/insert` | CSacccre | CREACC.cpy |
| 2 | `crecust` | POST | `/crecust/insert` | CScustcre | CRECUST.cpy |
| 3 | `delacc` | DELETE | `/delacc/remove/{accno}` | CSaccdel | DELACCZ.cpy |
| 4 | `delcus` | DELETE | `/delcus/remove/{custno}` | CScustdel | DELCUS.cpy |
| 5 | `inqacccz` | GET | `/inqacccz/list/{custno}` | CScustacc | INQACCCZ.cpy |
| 6 | `inqaccz` | GET | `/inqaccz/enquiry/{accno}` | CSaccenq | INQACCZ.cpy |
| 7 | `inqcustz` | GET | `/inqcustz/enquiry/{custno}` | CScustenq | INQCUSTZ.cpy |
| 8 | `makepayment` | PUT | `/makepayment/dbcr` | Pay | PAYDBCR.cpy |
| 9 | `updacc` | PUT | `/updacc/update` | CSaccupd | UPDACC.cpy |
| 10 | `updcust` | PUT | `/updcust/update` | CScustupd | UPDCUST.cpy |

### 3.1 Request/Response Schemas (Key Fields)

#### `creacc` — Create Account
- **Request/Response** (`CreAcc`): `CommEyecatcher`(4), `CommCustno`(int), `CommKey`{`CommSortcode`(int), `CommNumber`(int)}, `CommAccType`(8), `CommIntRt`(decimal 0–9999.99), `CommOpened`(int), `CommOverdrLim`(int), `CommLastStmtDt`(int), `CommNextStmtDt`(int), `CommAvailBal`(decimal), `CommActBal`(decimal), `CommSuccess`(1), `CommFailCode`(1)

#### `crecust` — Create Customer
- **Request/Response** (`CreCust`): `CommEyecatcher`(4), `CommKey`{`CommSortcode`, `CommNumber`}, `CommName`(60), `CommAddress`(160), `CommDateOfBirth`(int), `CommCreditScore`(int 0–999), `CommCsReviewDate`(int), `CommSuccess`(1), `CommFailCode`(1)

#### `delacc` — Delete Account
- **Request/Response** (`DelAcc`): `DelAccEye`, `DelAccCustno`(10), `DelAccScode`(6), `DelAccAccno`(int), `DelAccAccType`, `DelAccIntRate`, balance/date fields, `DelAccSuccess`, `DelAccFailCd`, `DelAccDelSuccess`, `DelAccDelFailCd`, `DelAccDelApplid`(8)

#### `delcus` — Delete Customer
- **Request/Response** (`DelCus`): `CommEye`, `CommScode`, `CommCustno`(int), `CommName`, `CommAddr`, `CommDob`, `CommCreditScore`, `CommCsReviewDate`, `CommDelSuccess`, `CommDelFailCd`

#### `inqacccz` — List Accounts by Customer
- **Response** (`InqAccZ`): `CustomerNumber`, `CommSuccess`, `CommFailCode`, `CustomerFound`, `AccountDetails[]`(1–20 items, each with full account fields)
- Most complex COMMAREA at 1981 bytes (variable-length array, `DEPENDS ON`)

#### `inqaccz` — Enquire Account
- **Response** (`InqAcc`): `InqAccEye`, `InqAccCustno`, `InqAccAccno`, `InqAccAccType`, `InqAccIntRate`, `InqAccOpened`, balance/date fields, `InqAccSuccess`

#### `inqcustz` — Enquire Customer
- **Response** (`INQCUSTZ`): `InqCustEye`, `InqCustCustno`, `InqCustName`, `InqCustAddress`, `InqCustDob`{Dd/Mm/Yyyy}, `InqCustCreditScore`, `InqCustCsReviewDate`{Dd/Mm/Yyyy}, `InqCustInqSuccess`, `InqCustInqFailCode`

#### `makepayment` — Debit/Credit
- **Request/Response** (`PAYDBCR`): `CommAccno`(8), `CommAmt`(decimal), `mSortC`(int), `CommAvBal`, `CommActBal`, `CommOrigin`{`CommApplid`, `CommUserid`, `CommFacilityName`, `CommNetwrkId`, `CommFaciltype`, `Fill0`}, `CommSuccess`, `CommFailCode`

#### `updacc` — Update Account
- **Request/Response** (`UpdAcc`): `CommEye`, `CommCustno`, `CommSortcode`, `CommAccno`, `CommAccType`, `CommIntRate`, `CommOpened`, balance/date fields, `CommOverdraft`, `CommSuccess`

#### `updcust` — Update Customer
- **Request/Response** (`UpdCust`): `CommEye`, `CommSortcode`, `CommCustno`, `CommName`, `CommAddress`, `CommDateOfBirth`, `CommCreditScore`, `CommCreditScoreReviewDate`, `CommUpdateSuccess`, `CommUpdateFailCode`

---

## 4. Service Definitions

| # | Service (SAR) | COBOL Program | SI Name | COMMAREA Size | Description |
|---|--------------|---------------|---------|---------------|-------------|
| 1 | CSacccre | CREACC | CREACC.si | ~100 bytes | Account creation |
| 2 | CSaccdel | DELACC | DELACCZ.si | ~110 bytes | Account deletion |
| 3 | CSaccenq | INQACC | INQACCZ.si | ~92 bytes | Account enquiry |
| 4 | CSaccupd | UPDACC | UPDACC.si | ~88 bytes | Account update |
| 5 | CScustacc | INQACCCU | INQACCCZ.si | 1981 bytes | Customer's accounts list |
| 6 | CScustcre | CRECUST | CRECUST.si | ~254 bytes | Customer creation |
| 7 | CScustdel | DELCUS | DELCUS.si | ~254 bytes | Customer deletion |
| 8 | CScustenq | INQCUST | INQCUSTZ.si | ~254 bytes | Customer enquiry |
| 9 | CScustupd | UPDCUST | UPDCUST.si | ~254 bytes | Customer update |
| 10 | Pay | DBCRFUN | PAYDBCR.si | 92 bytes | Debit/credit payment |

---

## 5. Complete End-to-End Chain

| REST API Path | Method | z/OS Connect Service | COBOL Program | Data Store |
|--------------|--------|---------------------|---------------|-----------|
| `/creacc/insert` | POST | CSacccre | CREACC | DB2 ACCOUNT + PROCTRAN |
| `/crecust/insert` | POST | CScustcre | CRECUST | VSAM CUSTOMER + PROCTRAN |
| `/delacc/remove/{accno}` | DELETE | CSaccdel | DELACC | DB2 ACCOUNT + PROCTRAN |
| `/delcus/remove/{custno}` | DELETE | CScustdel | DELCUS | VSAM CUSTOMER + DB2 ACCOUNT + PROCTRAN |
| `/inqacccz/list/{custno}` | GET | CScustacc | INQACCCU | DB2 ACCOUNT |
| `/inqaccz/enquiry/{accno}` | GET | CSaccenq | INQACC | DB2 ACCOUNT |
| `/inqcustz/enquiry/{custno}` | GET | CScustenq | INQCUST | VSAM CUSTOMER |
| `/makepayment/dbcr` | PUT | Pay | DBCRFUN | DB2 ACCOUNT + PROCTRAN |
| `/updacc/update` | PUT | CSaccupd | UPDACC | DB2 ACCOUNT |
| `/updcust/update` | PUT | CScustupd | UPDCUST | VSAM CUSTOMER |

---

## 6. Consumer Mapping

| z/OS Connect API | Customer Services Interface | Payment Interface | React/WebUI (direct CICS LINK) |
|-----------------|----------------------------|-------------------|-------------------------------|
| `/creacc/insert` | ✓ (`POST /createacc`) | — | ✓ (via `AccountsResource`) |
| `/crecust/insert` | ✓ (`POST /createcust`) | — | ✓ (via `CustomerResource`) |
| `/delacc/remove/{accno}` | ✓ (`POST /delacct`) | — | ✓ (via `AccountsResource`) |
| `/delcus/remove/{custno}` | ✓ (`POST /delcust`) | — | ✓ (via `CustomerResource`) |
| `/inqacccz/list/{custno}` | ✓ (`POST /listacc`) | — | ✓ (via `AccountsResource`) |
| `/inqaccz/enquiry/{accno}` | ✓ (`POST /enqacct`) | — | ✓ (via `AccountsResource`) |
| `/inqcustz/enquiry/{custno}` | ✓ (`POST /enqcust`) | — | ✓ (via `CustomerResource`) |
| `/makepayment/dbcr` | — | ✓ (`POST /paydbcr`) | — |
| `/updacc/update` | ✓ (`POST /updateacc`) | — | ✓ (via `AccountsResource`) |
| `/updcust/update` | ✓ (`POST /updatecust`) | — | ✓ (via `CustomerResource`) |

**Note**: The React/WebUI path calls the same COBOL programs but via **direct CICS LINK** from the Liberty JVM server, bypassing z/OS Connect entirely.

---

## 7. Inter-Module Integration Points

| From | To | Mechanism | Data |
|------|-----|-----------|------|
| Customer Services (Spring Boot) | z/OS Connect | HTTP (`WebClient`) | JSON |
| Payment Interface (Spring Boot) | z/OS Connect | HTTP (`WebClient`) | JSON |
| z/OS Connect | CICS Programs | COMMAREA | EBCDIC bytes |
| CICS Programs | DB2 | Embedded SQL | ACCOUNT, PROCTRAN, CONTROL tables |
| CICS Programs | VSAM | CICS File API | CUSTOMER KSDS |

---

## 8. Findings

| ID | Severity | Type | Location | Description |
|----|----------|------|----------|-------------|
| F-ZOS-001 | **Medium** | Non-standard REST | `inqaccz`, `inqacccz`, `inqcustz` | GET requests define a request body in their Swagger specs. Non-standard per RFC 7231; some HTTP clients/proxies strip GET bodies. |
| F-ZOS-002 | **Medium** | Non-standard REST | `delacc`, `delcus` | DELETE requests require a request body. Also non-standard REST practice. |
| F-ZOS-003 | **Medium** | Missing mappings | `updacc` | API's `mapping.xml` has no `<requestMessage>` element and empty `<responseMessages>`. No `request.map`/`response.map` files. JSON passes through without field-level transformation. |
| F-ZOS-004 | **Medium** | Missing header | `updacc` swagger.json | Only API that omits the optional `Authorization` header parameter; all other 9 APIs include it. |
| F-ZOS-005 | **Low** | Duplicate artefacts | All APIs | Each AAR's `services/` subdirectory contains a full copy of the SAR definition, duplicating what exists in the top-level `services/` directory. Changes must be synchronized in both locations. |
| F-ZOS-006 | **Low** | Typo | `CScustacc` service.properties | Description reads "Customer Servoce customer's accounts" (misspelling of "Service"). |
| F-ZOS-007 | **Low** | Naming inconsistency | SI names vs executables | SI files use "Z" suffix (DELACCZ.si, INQACCZ.si, INQACCCZ.si, INQCUSTZ.si) distinguishing z/OS Connect versions, but the COBOL program names lack the suffix (DELACC, INQACC, INQACCCU, INQCUST). |
| F-ZOS-008 | **Low** | Field naming | Cross-service | JSON field naming conventions vary: account services use `Comm*`, delete account uses `DelAcc*`, inquiry customer uses `InqCust*`, payment uses `Comm*`. Derived from COBOL copybooks but inconsistent. |
| F-ZOS-009 | **Low** | Date handling | All services | Date fields use COBOL REDEFINES: `CommOpened` (8-digit int DDMMYYYY) redefines to `CommOpenedGroup`{Day, Month, Year}. Group variants excluded from JSON (`included="N"`), so REST consumers must parse dates from integers. |
| F-ZOS-010 | **Low** | No error schemas | All swagger.json | All Swagger specs only define `200 OK` response. No 400, 404, or 500 error schemas despite COMMAREA including `CommSuccess`/`CommFailCode` fields. |

---

**Coverage Validation**: This document covers 100% of z/OS Connect artefacts:
- 10/10 API definitions (AAR) documented ✓
- 10/10 service definitions (SAR) documented ✓
- 10/10 COBOL program mappings documented ✓
- All consumer mappings (Customer Services, Payment Interface, React/WebUI) documented ✓
- No unused/orphaned artefacts found ✓
