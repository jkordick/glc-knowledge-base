# Module Specification: Cross-Module Integration Map

**Feature**: 001-reverse-engineer-modules  
**User Story**: US-7 — Produce Cross-Module Integration Map (P3)  
**Date**: 2026-05-12  
**Source**: All module documents (module-cobol-backend.md, module-data-layer.md, module-react-ui.md, module-customer-services.md, module-payment-interface.md, module-zos-connect.md)

---

## 1. System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         USER INTERFACES                                 │
│                                                                         │
│  ┌──────────────────┐  ┌─────────────────────┐  ┌───────────────────┐  │
│  │  Carbon React UI │  │ Customer Services   │  │ Payment Interface │  │
│  │  (react-scripts) │  │ (Spring Boot 3.5)   │  │ (Spring Boot 3.5) │  │
│  │  Port: N/A (SPA) │  │ Port: 19080         │  │ Port: 19080       │  │
│  └────────┬─────────┘  └────────┬────────────┘  └────────┬──────────┘  │
│           │ axios                │ WebClient               │ WebClient  │
│           │ HTTP                 │ HTTP                     │ HTTP       │
└───────────┼──────────────────────┼─────────────────────────┼────────────┘
            │                      │                         │
            ▼                      ▼                         ▼
┌──────────────────────┐  ┌──────────────────────────────────────────────┐
│  WebUI Java (Liberty)│  │         z/OS Connect EE                      │
│  JAX-RS REST API     │  │  10 APIs → 10 Services → CICS COMMAREA      │
│  /webui-1.0/banking/ │  │  JSON ↔ EBCDIC translation                  │
│  28 endpoints        │  │  Connection: cicsConn, CCSID 37              │
└────────┬─────────────┘  └────────────────┬─────────────────────────────┘
         │                                  │
         │ CICS LINK / JDBC / jCICS         │ COMMAREA
         │                                  │
         ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    COBOL/CICS BACKEND (29 programs)                     │
│                                                                         │
│  Account Programs: BNK1CCA, BNK1DCA, CREACC, DELACC, INQACC,          │
│                    INQACCCU, UPDACC, DBCRFUN                            │
│  Customer Programs: BNK1CCS, BNK1DCS, CRECUST, DELCUS, INQCUST,       │
│                     UPDCUST                                             │
│  Navigation: BNK1MAI (main menu)                                        │
│  Utility: GETSCODE, GETCOMPY, NEWACCNO, NEWCUSNO, ABNDPROC             │
│  BMS Screens: 9 maps (BNK1MAI, BNK1ACC, etc.)                         │
└────────┬────────────────────────────┬───────────────────────────────────┘
         │ SQL                        │ CICS File API
         ▼                           ▼
┌────────────────────────┐  ┌────────────────────────┐
│    DB2 v12+            │  │    VSAM KSDS           │
│  ┌──────────────────┐  │  │  ┌──────────────────┐  │
│  │ ACCOUNT          │  │  │  │ CUSTOMER         │  │
│  │ (12 columns)     │  │  │  │ (259 bytes/rec)  │  │
│  ├──────────────────┤  │  │  ├──────────────────┤  │
│  │ PROCTRAN         │  │  │  │ ABNDFILE         │  │
│  │ (audit log)      │  │  │  │ (abend records)  │  │
│  ├──────────────────┤  │  │  └──────────────────┘  │
│  │ CONTROL          │  │  └────────────────────────┘
│  │ (config)         │  │
│  └──────────────────┘  │
└────────────────────────┘
```

---

## 2. End-to-End User Action Traces

### 2.1 Trace 1: Create a Customer (React UI Path)

| Step | Layer | Component | Action | Data |
|------|-------|-----------|--------|------|
| 1 | React | `CustomerCreationPage.js` | User fills form: name, address, DOB, title | Form state |
| 2 | React | `CustomerCreationPage.js` | Hardcodes sort code `"987654"`, sends POST | axios POST |
| 3 | HTTP | — | `POST /webui-1.0/banking/customer` | JSON: `{customerAddress, dateOfBirth, sortCode, customerName}` |
| 4 | WebUI Java | `CustomerResource.createCustomerExternal()` | Creates `HBankDataAccess`, gets DB2 connection | JDBC setup |
| 5 | WebUI Java | `web.vsam.Customer.createCustomer()` | Calls COBOL `NEWCUSNO` via CICS LINK for next customer number | `NewCustomerNumber` commarea (13 bytes) |
| 6 | COBOL | `NEWCUSNO` | Reads VSAM CONTROL record, increments counter, returns new number | VSAM CUSTOMER KSDS |
| 7 | WebUI Java | `CreditScore.populateCreditScoreAndReviewDate()` | Calls CICS ASYNC (OCR1–OCR5) for parallel credit scoring | `CRECUST` commarea (261 bytes) |
| 8 | COBOL | `CRDTAGY1–5` | 5 credit agency simulations run in parallel, return scores | CICS channels/containers |
| 9 | WebUI Java | `web.vsam.Customer` | Writes CUSTOMER record to VSAM KSDS | jCICS KSDS WRITE (259 bytes) |
| 10 | WebUI Java | `ProcessedTransactionResource` | Writes audit record to PROCTRAN | JDBC INSERT (DB2) |
| 11 | React | `CustomerCreationPage.js` | Shows success modal with customer number | UI update |

### 2.2 Trace 2: Delete a Customer (Customer Services Path)

| Step | Layer | Component | Action | Data |
|------|-------|-----------|--------|------|
| 1 | Browser | `deleteCustomerForm.html` | User enters customer number | Thymeleaf form |
| 2 | Spring Boot | `WebController.checkPersonInfo()` | Validates `CustomerEnquiryForm` | Bean Validation |
| 3 | HTTP | — | `DELETE {z/OS Connect}/delcus/remove/{custno}` | WebClient `.block()` |
| 4 | z/OS Connect | `delcus` API → `CScustdel` service | JSON → COMMAREA translation | EBCDIC CCSID 37 |
| 5 | CICS | COMMAREA | Invokes COBOL program `DELCUS` | DELCUS.cpy (254 bytes) |
| 6 | COBOL | `DELCUS` | Retrieves customer from VSAM CUSTOMER | CICS File READ |
| 7 | COBOL | `DELCUS` | Reads all accounts for customer from DB2 ACCOUNT | SQL SELECT |
| 8 | COBOL | `DELCUS` | Deletes each account from DB2 ACCOUNT | SQL DELETE (loop) |
| 9 | COBOL | `DELCUS` | Writes deletion audit to DB2 PROCTRAN | SQL INSERT |
| 10 | COBOL | `DELCUS` | Deletes customer record from VSAM CUSTOMER | CICS File DELETE |
| 11 | z/OS Connect | Response | COMMAREA → JSON translation | Success/fail codes |
| 12 | Spring Boot | `WebController` | Checks `CommDelFailCode`, renders result | Thymeleaf template |

### 2.3 Trace 3: Make a Payment (Payment Interface Path)

| Step | Layer | Component | Action | Data |
|------|-------|-----------|--------|------|
| 1 | Browser | `paymentInterfaceForm.html` | User enters account number, amount, debit/credit, organisation | Thymeleaf form |
| 2 | Spring Boot | `WebController.checkPersonInfo()` | Validates `TransferForm`, constructs `PaymentInterfaceJson` | Bean Validation |
| 3 | Spring Boot | `WebController` | Negates amount if debit, splits org name across `CommApplid`/`CommUserid` | DTO construction |
| 4 | HTTP | — | `PUT {z/OS Connect}/makepayment/dbcr` | WebClient `.block()` + JSON body |
| 5 | z/OS Connect | `makepayment` API → `Pay` service | JSON → COMMAREA translation | PAYDBCR.si (92 bytes) |
| 6 | COBOL | `DBCRFUN` | Reads account from DB2 ACCOUNT | SQL SELECT |
| 7 | COBOL | `DBCRFUN` | Validates account type, checks balance for debit | Business rules |
| 8 | COBOL | `DBCRFUN` | Updates balance in DB2 ACCOUNT | SQL UPDATE |
| 9 | COBOL | `DBCRFUN` | Writes audit record to DB2 PROCTRAN | SQL INSERT |
| 10 | z/OS Connect | Response | COMMAREA → JSON, includes new balances | JSON response |
| 11 | Spring Boot | `WebController` | Checks `CommFailCode` (1=not found, 3=insufficient, 4=invalid type) | Exception or success |
| 12 | Browser | `paymentInterfaceForm.html` | Displays result (success message or error) | Thymeleaf render |

### 2.4 Trace 4: Update Account Details (React UI Path)

| Step | Layer | Component | Action | Data |
|------|-------|-----------|--------|------|
| 1 | React | `AccountDetailsPage.js` | User searches by account number | State update |
| 2 | HTTP | — | `GET /webui-1.0/banking/account/{accountNumber}` | axios GET |
| 3 | WebUI Java | `AccountsResource.getAccountExternal()` | Reads from DB2 ACCOUNT via JDBC | SQL SELECT |
| 4 | React | `AccountDetailsTable.js` | Displays account in DataTable, user edits fields | Carbon DataTable |
| 5 | React | `AccountDetailsTable.js` | User clicks update, reformats dates (dd-mm-yyyy → yyyy-mm-dd) | Substring operations |
| 6 | HTTP | — | `PUT /webui-1.0/banking/account/{id}` | JSON body (⚠️ customerNumber always `""`) |
| 7 | WebUI Java | `AccountsResource.updateAccountExternal()` | Validates account type, updates DB2 ACCOUNT | SQL UPDATE |
| 8 | React | `AccountDetailsTable.js` | Shows success modal, refreshes data | UI update |

---

## 3. Complete Integration Point Inventory

### 3.1 UI → Backend Integration Points

| # | Source | Target | Protocol | Endpoint | Direction |
|---|--------|--------|----------|----------|-----------|
| 1 | React UI | WebUI Java (Liberty) | HTTP/JSON | `POST /webui-1.0/banking/customer` | Create customer |
| 2 | React UI | WebUI Java (Liberty) | HTTP/JSON | `GET /webui-1.0/banking/customer/{id}` | Get customer |
| 3 | React UI | WebUI Java (Liberty) | HTTP/JSON | `PUT /webui-1.0/banking/customer/{id}` | Update customer |
| 4 | React UI | WebUI Java (Liberty) | HTTP/JSON | `DELETE /webui-1.0/banking/customer/{id}` | Delete customer |
| 5 | React UI | WebUI Java (Liberty) | HTTP/JSON | `GET /webui-1.0/banking/customer/name` | Search by name |
| 6 | React UI | WebUI Java (Liberty) | HTTP/JSON | `POST /webui-1.0/banking/account` | Create account |
| 7 | React UI | WebUI Java (Liberty) | HTTP/JSON | `GET /webui-1.0/banking/account/{id}` | Get account |
| 8 | React UI | WebUI Java (Liberty) | HTTP/JSON | `PUT /webui-1.0/banking/account/{id}` | Update account |
| 9 | React UI | WebUI Java (Liberty) | HTTP/JSON | `DELETE /webui-1.0/banking/account/{id}` | Delete account |
| 10 | React UI | WebUI Java (Liberty) | HTTP/JSON | `GET /webui-1.0/banking/account/retrieveByCustomerNumber/{id}` | Get accounts by customer |
| 11 | Customer Services | z/OS Connect | HTTP/JSON | 9 endpoints (see module-customer-services.md §4.2) | CRUD via z/OS Connect |
| 12 | Payment Interface | z/OS Connect | HTTP/JSON | `PUT /makepayment/dbcr` | Debit/credit via z/OS Connect |
| 13 | Payment Interface (REST) | z/OS Connect | HTTP/JSON | `POST /submit` (ParamsController) | Programmatic payment |

### 3.2 Backend → Data Store Integration Points

| # | Source | Target | Protocol | Purpose |
|---|--------|--------|----------|---------|
| 14 | WebUI Java (Liberty) | DB2 ACCOUNT | JDBC | Account CRUD, debit/credit |
| 15 | WebUI Java (Liberty) | DB2 PROCTRAN | JDBC | Audit trail writes |
| 16 | WebUI Java (Liberty) | VSAM CUSTOMER | jCICS KSDS | Customer CRUD |
| 17 | COBOL programs | DB2 ACCOUNT | Embedded SQL | Account operations |
| 18 | COBOL programs | DB2 PROCTRAN | Embedded SQL | Audit trail |
| 19 | COBOL programs | DB2 CONTROL | Embedded SQL | Configuration reads |
| 20 | COBOL programs | VSAM CUSTOMER | CICS File API | Customer operations |
| 21 | COBOL programs | VSAM ABNDFILE | CICS File API | Abend recording |

### 3.3 Backend → COBOL Integration Points (CICS LINK / CICS ASYNC)

| # | Source | COBOL Program | Mechanism | Purpose |
|---|--------|---------------|-----------|---------|
| 22 | WebUI Java | GETSCODE | CICS LINK (`GetSortCode` commarea) | Retrieve sort code |
| 23 | WebUI Java | GETCOMPY | CICS LINK (`GetCompany` commarea) | Retrieve company name |
| 24 | WebUI Java | NEWACCNO | CICS LINK (`NewAccountNumber` commarea) | Generate account number |
| 25 | WebUI Java | NEWCUSNO | CICS LINK (`NewCustomerNumber` commarea) | Generate customer number |
| 26 | WebUI Java | OCR1–OCR5 | CICS ASYNC (`CRECUST` commarea) | Parallel credit scoring |
| 27 | z/OS Connect | CREACC | COMMAREA | Create account |
| 28 | z/OS Connect | CRECUST | COMMAREA | Create customer |
| 29 | z/OS Connect | DELACC | COMMAREA | Delete account |
| 30 | z/OS Connect | DELCUS | COMMAREA | Delete customer |
| 31 | z/OS Connect | INQACC | COMMAREA | Enquire account |
| 32 | z/OS Connect | INQACCCU | COMMAREA | List customer's accounts |
| 33 | z/OS Connect | INQCUST | COMMAREA | Enquire customer |
| 34 | z/OS Connect | UPDACC | COMMAREA | Update account |
| 35 | z/OS Connect | UPDCUST | COMMAREA | Update customer |
| 36 | z/OS Connect | DBCRFUN | COMMAREA | Debit/credit payment |

### 3.4 z/OS Connect Gateway Integration Points

| # | Source | Target | Mechanism | Purpose |
|---|--------|--------|-----------|---------|
| 37 | z/OS Connect API (AAR) | z/OS Connect Service (SAR) | Service reference | API → service routing |
| 38 | z/OS Connect Service | CICS programs | CICS COMMAREA (`cicsConn`) | Service → program execution |
| 39 | z/OS Connect | JSON ↔ EBCDIC | `.si` interface definitions | Data translation (CCSID 37) |

### 3.5 COBOL → COBOL Integration Points (Internal)

| # | Source Program | Target Program | Mechanism | Purpose |
|---|----------------|----------------|-----------|---------|
| 40 | BNK1CCA | CREACC | CICS LINK | Create account from 3270 |
| 41 | BNK1CCS | CRECUST | CICS LINK | Create customer from 3270 |
| 42 | BNK1DCA | DELACC | CICS LINK | Delete account from 3270 |
| 43 | BNK1DCS | DELCUS | CICS LINK | Delete customer from 3270 |
| 44 | BNK1MAI | BNK1CCA/DCA/CCS/DCS | CICS XCTL | Menu navigation |
| 45 | Any program | ABNDPROC | CICS LINK | Abend handling |

**Total integration points**: 45

---

## 4. Parallel Access Paths to Same Data

The CBSA application provides three independent paths to the same backend data. Each path has different middleware but ultimately invokes the same COBOL programs or accesses the same data stores.

| Operation | Path A: React + WebUI | Path B: Customer Services + z/OS Connect | Path C: 3270 Terminal |
|-----------|----------------------|------------------------------------------|----------------------|
| Create Customer | `POST /banking/customer` → jCICS VSAM direct | `POST /createcust` → z/OS Connect → CRECUST | BNK1CCS → CRECUST |
| Enquire Customer | `GET /banking/customer/{id}` → jCICS VSAM direct | `POST /enqcust` → z/OS Connect → INQCUST | BNK1MAI → inline SQL |
| Update Customer | `PUT /banking/customer/{id}` → jCICS VSAM direct | `POST /updatecust` → z/OS Connect → UPDCUST | N/A (not in BMS) |
| Delete Customer | `DELETE /banking/customer/{id}` → jCICS VSAM direct | `POST /delcust` → z/OS Connect → DELCUS | BNK1DCS → DELCUS |
| Create Account | `POST /banking/account` → JDBC DB2 direct | `POST /createacc` → z/OS Connect → CREACC | BNK1CCA → CREACC |
| Enquire Account | `GET /banking/account/{id}` → JDBC DB2 direct | `POST /enqacct` → z/OS Connect → INQACC | BNK1MAI → inline SQL |
| Update Account | `PUT /banking/account/{id}` → JDBC DB2 direct | `POST /updateacc` → z/OS Connect → UPDACC | N/A (not in BMS) |
| Delete Account | `DELETE /banking/account/{id}` → JDBC DB2 direct | `POST /delacct` → z/OS Connect → DELACC | BNK1DCA → DELACC |
| Debit/Credit | N/A | `POST /paydbcr` → z/OS Connect → DBCRFUN | BNK1TFM → DBCRFUN |

**Key difference**: Path A (React/WebUI) accesses DB2 and VSAM directly via JDBC/jCICS APIs from the Liberty JVM server. Path B (Customer Services/Payment Interface) routes through z/OS Connect, which performs JSON ↔ EBCDIC translation and then uses CICS COMMAREA to invoke COBOL programs that access the same data stores. Path C uses 3270 terminal BMS screens.

---

## 5. Data Flow Diagram: Cross-Store Relationships

```
VSAM CUSTOMER ──────key: sortcode(6)+custno(10)─────┐
     │                                               │
     │ customer_number referenced in                  │
     ▼                                               ▼
DB2 ACCOUNT ──────ACCOUNT_CUSTOMER_NUMBER────────── DB2 PROCTRAN
     │                  (no foreign key)                │
     │                                               │ audit log for
     │ ACCOUNT_NUMBER                                │ all operations
     │                                               │
     └───── balance operations ──────────────────────┘
                                                      
DB2 CONTROL ─── configuration (sort code, company name)
VSAM ABNDFILE ─── abend records (write-only, no cleanup)
```

**Cross-store reconciliation**: No referential integrity exists between VSAM CUSTOMER and DB2 ACCOUNT. The `ACCOUNT_CUSTOMER_NUMBER` column in DB2 references the VSAM customer key, but this is enforced only by application logic (COBOL programs and WebUI Java code), not by database constraints.

---

## 6. Findings

| ID | Severity | Type | Description |
|----|----------|------|-------------|
| F-INT-001 | **High** | Dual access path risk | React/WebUI accesses VSAM CUSTOMER directly via jCICS while Customer Services accesses the same data via z/OS Connect → COBOL. No locking coordination between paths; concurrent updates could cause data inconsistency. |
| F-INT-002 | **High** | No cross-store RI | Customer numbers link VSAM CUSTOMER to DB2 ACCOUNT with no foreign key or referential integrity. Orphaned accounts possible if customer deletion fails mid-operation. |
| F-INT-003 | **Medium** | Sort code inconsistency | React hardcodes `"987654"`, WebUI dynamically calls GETSCODE, Customer Services passes through from z/OS Connect. If sort code changes, React breaks while other paths adapt. |
| F-INT-004 | **Medium** | Audit gaps | React/WebUI path writes to PROCTRAN for create/delete but not for enquiry operations. Customer Services path doesn't write to PROCTRAN at all (relies on COBOL programs). Audit coverage varies by path. |
| F-INT-005 | **Low** | Unused endpoints | 18 of 28 WebUI REST endpoints are not called by React. 1 z/OS Connect endpoint (`GET /companyName`) has no UI consumer. These may be consumed by other unanalyzed systems. |

---

**Coverage Validation**:
- All 6 modules mapped to integration points ✓
- 45 total integration points enumerated ✓
- 3 user action traces documented end-to-end ✓ (plus 1 shorter trace)
- All parallel access paths documented ✓
- Cross-store relationships documented ✓
