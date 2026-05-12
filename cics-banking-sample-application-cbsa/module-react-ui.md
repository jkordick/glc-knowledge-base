# Module Specification: Carbon React UI

**Feature**: 001-reverse-engineer-modules  
**User Story**: US-3 — Reverse Engineer Carbon React UI (P2)  
**Date**: 2026-05-12  
**Source**: `src/bank-application-frontend/src/` (31 JS files), `src/webui/src/` (35 Java files)

---

## 1. Module Overview

**Purpose**: Provides the primary modern web UI for the CBSA banking application using React with IBM Carbon Design System. The React frontend communicates with a Liberty JVM Server-hosted JAX-RS REST API layer (the "WebUI" Java layer), which in turn accesses DB2 and VSAM data stores via JDBC and jCICS APIs.

**Technology Stack**:
- **Frontend**: React 18.2, react-router-dom 5.1 (HashRouter), Carbon React 1.61, axios for HTTP, SCSS
- **Backend**: Liberty JVM Server, JAX-RS (Jersey), JDBC, jCICS, IBM Record Generator for Java V3.0.2
- **Build**: Create React App (react-scripts 5.0.1), deployed as WAR to Liberty at context root `/webui-1.0/`

**Architecture**:
```
React (Carbon) ──axios──▶ Liberty JAX-RS (/webui-1.0/banking/*)
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
              web.db2.*   web.vsam.*   CICS LINK
              (ACCOUNT,   (CUSTOMER)   (GETSCODE,
               PROCTRAN)               GETCOMPY,
                                       NEWACCNO,
                                       NEWCUSNO)
```

**Base URL**: `/webui-1.0/banking` (set by `@ApplicationPath("banking")` in `BankingApplication.java`)

---

## 2. Source File Inventory

### 2.1 React Frontend (31 JS files)

| File | Type | Purpose |
|------|------|---------|
| `App.js` | Router | Root component, HashRouter with Switch/Route, theme wrapper |
| `index.js` | Entry | ReactDOM render, HashRouter wrapper |
| `App.test.js` | Test | Smoke test (renders App) |
| `serviceWorker.js` | Utility | CRA service worker (unregistered) |
| `setupTests.js` | Test config | Test setup (enzyme) |
| `components/Homepage-Header/Homepage-Header.js` | Component | Top nav bar with mock login form |
| `components/Homepage-Header/index.js` | Barrel | Re-export |
| `components/Admin-Header/Admin-Header.js` | Component | Admin nav bar + side nav with CRUD links |
| `components/Admin-Header/index.js` | Barrel | Re-export |
| `components/NewAccount/AccountForm.js` | Component | **Dead code** — empty form shell |
| `components/NewAccount/NewAccount.js` | Component | **Dead code** — empty component |
| `content/HomePage/HomePage.js` | Page | Landing page with About/User Guide tabs |
| `content/HomePage/index.js` | Barrel | Re-export |
| `content/AdminPage/AdminPage.js` | Page | Admin dashboard with navigation links |
| `content/AdminPage/index.js` | Barrel | Re-export |
| `content/CustomerCreationPage/CustomerCreationPage.js` | Page | Create customer form |
| `content/CustomerCreationPage/index.js` | Barrel | Re-export |
| `content/AccountCreationPage/AccountCreationPage.js` | Page | Create account form |
| `content/AccountCreationPage/index.js` | Barrel | Re-export |
| `content/CustomerDetailsPage/CustomerDetailsPage.js` | Page | Customer search + display |
| `content/CustomerDetailsPage/CustomerDetailsTable.js` | Component | Customer data table with inline update |
| `content/CustomerDetailsPage/index.js` | Barrel | Re-export |
| `content/AccountDetailsPage/AccountDetailsPage.js` | Page | Account search + display |
| `content/AccountDetailsPage/AccountDetailsTable.js` | Component | Account data table with inline update |
| `content/AccountDetailsPage/index.js` | Barrel | Re-export |
| `content/CustomerDeletePage/CustomerDeletePage.js` | Page | Customer search for deletion |
| `content/CustomerDeletePage/CustomerDeleteTables.js` | Component | Customer + accounts table with delete |
| `content/CustomerDeletePage/index.js` | Barrel | Re-export |
| `content/AccountDeletePage/AccountDeletePage.js` | Page | Account search for deletion |
| `content/AccountDeletePage/AccountDeleteTables.js` | Component | Account table with delete |
| `content/AccountDeletePage/index.js` | Barrel | Re-export |

### 2.2 WebUI Java Layer (35 Java files)

#### JAX-RS REST Resources (5 files)

| Class | Path | Purpose |
|-------|------|---------|
| `BankingApplication.java` | `@ApplicationPath("banking")` | JAX-RS application entry point |
| `AccountsResource.java` | `@Path("/account")` | Account CRUD + debit/credit/transfer |
| `CustomerResource.java` | `@Path("/customer")` | Customer CRUD + search |
| `ProcessedTransactionResource.java` | `@Path("/processedTransaction")` | Transaction audit log read/write |
| `SortCodeResource.java` | `@Path("/sortCode")` | Bank sort code retrieval |
| `CompanyNameResource.java` | `@Path("/companyName")` | Bank company name retrieval |

#### JSON DTO Classes (11 files)

| Class | Purpose |
|-------|---------|
| `AccountJSON.java` | Account request/response DTO with type validation |
| `CustomerJSON.java` | Customer request/response DTO with title validation |
| `CreditScore.java` | Credit score orchestrator (delegates to CreditScoreCICS540) |
| `CreditScoreCICS540.java` | CICS Async API credit scoring (calls OCR1–OCR5) |
| `DebitCreditAccountJSON.java` | Debit/credit amount DTO |
| `TransferLocalJSON.java` | Transfer DTO (extends DebitCreditAccountJSON + targetAccount) |
| `ProcessedTransactionJSON.java` | Base transaction DTO |
| `ProcessedTransactionAccountJSON.java` | Account creation/deletion audit DTO |
| `ProcessedTransactionCreateCustomerJSON.java` | Customer creation audit DTO |
| `ProcessedTransactionDeleteCustomerJSON.java` | Customer deletion audit DTO |
| `ProcessedTransactionDebitCreditJSON.java` | Debit/credit audit DTO |
| `ProcessedTransactionTransferLocalJSON.java` | Transfer audit DTO |
| `HBankDataAccess.java` | Base class for DB2 JDBC connection management |

#### Data Interface Classes (7 files — IBM Record Generator output)

| Class | COBOL Copybook | Record Length | Purpose |
|-------|---------------|---------------|---------|
| `CUSTOMER.java` | CUSTOMER.cpy | 259 bytes | VSAM customer record mapping |
| `CRECUST.java` | CRECUST.cpy | 261 bytes | Customer creation commarea |
| `CustomerControl.java` | CUSTCTRL.cpy | 259 bytes | Customer control counters |
| `GetCompany.java` | GETCOMPY.cpy | 40 bytes | Company name commarea |
| `GetSortCode.java` | GETSCODE.cpy | 6 bytes | Sort code commarea |
| `NewAccountNumber.java` | — | 11 bytes | Account number generation commarea |
| `NewCustomerNumber.java` | — | 13 bytes | Customer number generation commarea |
| `PROCTRAN.java` | PROCTRAN.cpy | 99 bytes | Transaction record mapping |

#### Data Access Layer (5 files)

| Class | Data Store | Purpose |
|-------|-----------|---------|
| `web.db2.Account.java` | DB2 ACCOUNT | JDBC CRUD for accounts |
| `web.db2.ProcessedTransaction.java` | DB2 PROCTRAN | JDBC read/write for audit log |
| `web.vsam.Customer.java` | VSAM CUSTOMER | jCICS KSDS CRUD for customers |
| `webui.data_access.Account.java` | (delegates) | Legacy servlet wrapper for accounts |
| `webui.data_access.AccountList.java` | (delegates) | Legacy servlet wrapper for account lists |
| `webui.data_access.Customer.java` | (delegates) | Legacy servlet wrapper for customers |
| `webui.data_access.CustomerList.java` | (delegates) | Legacy servlet wrapper for customer lists |
| `webui.data_access.GetUserSortCode.java` | (delegates) | Legacy duplicate of GetSortCode |

---

## 3. Detailed Specifications

### 3.1 Routing (App.js)

| Route Path | Component | Description |
|------------|-----------|-------------|
| `/` (default) | HomePage | Landing page (About + User Guide) |
| `/*` (fallback) | HomePage | Catch-all redirect to home |
| `/profile/Admin` | AdminPage | Admin dashboard |
| `/Admin/customer_creation` | CustomerCreationPage | Create customer form |
| `/Admin/account_creation` | AccountCreationPage | Create account form |
| `/Admin/customer_details` | CustomerDetailsPage | Search/view customers |
| `/Admin/account_details` | AccountDetailsPage | Search/view accounts |
| `/Admin/customer_deletion` | CustomerDeletePage | Search/delete customers |
| `/Admin/account_deletion` | AccountDeletePage | Search/delete accounts |

**Navigation flow**: HomepageHeader (mock login) → AdminPage → CRUD pages via AdminHeader side nav

### 3.2 React Component Specifications

#### HomePage.js
- **Purpose**: Static landing page with Carbon Tabs (About, User Guide)
- **Props/State**: None
- **API calls**: None
- **Children**: Carbon `Tabs`, `Grid`, `Button`

#### AdminPage.js
- **Purpose**: Admin dashboard with navigation links to CRUD operations
- **Props/State**: `isModalOpened`, `isSuccessModalOpened` (both unused)
- **API calls**: None
- **Children**: Carbon `HeaderName` with routing links

#### CustomerCreationPage.js
- **Purpose**: Form to create a new customer
- **State**: `customerFullName`, `line1`, `dateOfBirth`, `city`, `title`, `successText`, modal states
- **API calls**: `POST /webui-1.0/banking/customer` — body: `{customerAddress, dateOfBirth, sortCode: "987654", customerName}`
- **Children**: Carbon `Dropdown`, `TextInput`, `DatePicker`, `Modal`

#### AccountCreationPage.js
- **Purpose**: Form to create a new account for an existing customer
- **State**: `enteredCustomerID`, `enteredAccountType`, `enteredOverdraftLimit`, `enteredInterestRate`, modal states
- **API calls**: `POST /webui-1.0/banking/account` — body: `{interestRate, dateOpened, overdraft, accountType, customerNumber, sortCode: "987654"}`
- **Children**: Carbon `Form`, `Stack`, `TextInput`, `Dropdown`, `Modal`

#### CustomerDetailsPage.js
- **Purpose**: Search customers by number or name, display results with expandable account rows
- **State**: `customerDetailsRows`, `accountDetailsRows`, `numSearch`, `nameSearch`, modal states
- **API calls**: `GET /customer/{id}`, `GET /customer/name?name={name}&limit=10`, `GET /account/retrieveByCustomerNumber/{id}`
- **Children**: `CustomerDetailsTable`

#### CustomerDetailsTable.js
- **Purpose**: Expandable data table for customer data with inline update modal
- **Props**: `customerDetailsRows`, `accountDetailsRows`
- **State**: Current customer field values, `enteredNameChange`, `enteredAddressChange`, modal states
- **API calls**: `PUT /customer/{id}` — body: `{customerAddress, creditScore, dateOfBirth, sortCode, customerName}`

#### AccountDetailsPage.js
- **Purpose**: Search for a single account by number
- **State**: `userInput`, `accountMainRow`, modal states
- **API calls**: `GET /account/{accountNumber}`
- **Children**: `AccountDetailsTable`

#### AccountDetailsTable.js
- **Purpose**: Account data table with inline update modal
- **Props**: `accountMainRow`
- **State**: All current + edited account field values, modal states
- **API calls**: `PUT /account/{accountNumber}` — body: `{interestRate, lastStatementDate, nextStatementDate, dateOpened, actualBalance, overdraft, accountType, id, customerNumber, sortCode, availableBalance}`

#### CustomerDeletePage.js
- **Purpose**: Search customer by number, show customer + accounts, delegate to delete tables
- **State**: `searchCustomerValue`, `customerDetailsRows`, `accountDetailsRows`, modal states
- **API calls**: `GET /customer/{id}`, `GET /account/retrieveByCustomerNumber/{id}`
- **Children**: `CustomerDeleteTables`

#### CustomerDeleteTables.js
- **Purpose**: Customer + account tables with delete buttons and confirmation modals
- **Props**: `customerRow`, `accountRow`
- **API calls**: `GET /account/retrieveByCustomerNumber/{id}` (check account count), `DELETE /customer/{id}`, `DELETE /account/{id}`
- **Business rule**: Customer deletion requires zero accounts (enforced on frontend)

#### AccountDeletePage.js
- **Purpose**: Search for account by number, delegate to delete tables
- **State**: `searchAccountValue`
- **Children**: `AccountDeleteTables`

#### AccountDeleteTables.js
- **Purpose**: Show target account + sibling accounts, delete with confirmation
- **Props**: `accountQuery`
- **API calls**: `GET /account/{id}`, `GET /account/retrieveByCustomerNumber/{id}`, `DELETE /account/{id}`

#### HomepageHeader.js
- **Purpose**: Top navigation bar with mock login form (non-functional authentication)
- **API calls**: None
- **Children**: Carbon `Header`, `HeaderNavigation`, mock login form

#### AdminHeader.js
- **Purpose**: Admin navigation bar + side navigation with CRUD page links
- **API calls**: None
- **Children**: Carbon `Header`, `SideNav` with page links

### 3.3 Complete REST API Surface (WebUI Java)

#### Account Endpoints (`/webui-1.0/banking/account`)

| # | Method | Path | Description | Called by React? |
|---|--------|------|-------------|-----------------|
| 1 | POST | `/account` | Create account | Yes |
| 2 | GET | `/account` | List all (paginated) | No |
| 3 | GET | `/account/{accountNumber}` | Get single account | Yes |
| 4 | GET | `/account/retrieveByCustomerNumber/{customerNumber}` | Get by customer | Yes |
| 5 | PUT | `/account/{id}` | Update account | Yes |
| 6 | PUT | `/account/debit/{id}` | Debit account | No |
| 7 | PUT | `/account/credit/{id}` | Credit account | No |
| 8 | PUT | `/account/transfer/{id}` | Transfer funds | No |
| 9 | DELETE | `/account/{accountNumber}` | Delete account | Yes |
| 10 | GET | `/account/balance` | Filter by balance | No |

#### Customer Endpoints (`/webui-1.0/banking/customer`)

| # | Method | Path | Description | Called by React? |
|---|--------|------|-------------|-----------------|
| 11 | POST | `/customer` | Create customer | Yes |
| 12 | GET | `/customer` | List all (paginated) | No |
| 13 | GET | `/customer/{id}` | Get single customer | Yes |
| 14 | PUT | `/customer/{id}` | Update customer | Yes |
| 15 | DELETE | `/customer/{id}` | Delete customer (cascades) | Yes |
| 16 | GET | `/customer/name` | Search by name | Yes |
| 17 | GET | `/customer/all/town/{town}` | Search by town | No |
| 18 | GET | `/customer/all/surname/{surname}` | Search by surname | No |
| 19 | GET | `/customer/all/age/{age}` | Search by age | No |

#### Transaction Endpoints (`/webui-1.0/banking/processedTransaction`)

| # | Method | Path | Description | Called by React? |
|---|--------|------|-------------|-----------------|
| 20 | GET | `/processedTransaction` | List transactions | No |
| 21-26 | POST | `/processedTransaction/*` | Audit trail writes | No (internal) |

#### Utility Endpoints

| # | Method | Path | Description | Called by React? |
|---|--------|------|-------------|-----------------|
| 27 | GET | `/sortCode` | Get sort code | No |
| 28 | GET | `/companyName` | Get company name | No |

**Summary**: 28 total endpoints; 10 used by React frontend, 18 unused by React (available to other consumers like Spring Boot UIs or z/OS Connect).

---

## 4. Data Access Patterns

| Java Class | Data Store | Access Method | Operations |
|-----------|-----------|---------------|------------|
| `web.db2.Account` | DB2 ACCOUNT table | JDBC PreparedStatement | CRUD + debit/credit/transfer |
| `web.db2.ProcessedTransaction` | DB2 PROCTRAN table | JDBC PreparedStatement | SELECT (paginated), INSERT (audit) |
| `web.vsam.Customer` | VSAM CUSTOMER file | jCICS KSDS API (KeyedFileBrowse) | CRUD + search by name/town/surname/age |
| `HBankDataAccess` | DB2 connection pool | JNDI `jdbc/defaultCICSDataSource` | Connection lifecycle, static cache by task |

**Connection management**: `HBankDataAccess` maintains a static `HashMap` keyed by CICS task number to reuse JDBC connections within a single CICS task. Uses `TRANSACTION_READ_UNCOMMITTED` isolation level.

**CICS program calls** (via data interfaces):
| Program | Interface Class | Purpose |
|---------|----------------|---------|
| GETSCODE | `GetSortCode` | Retrieve bank sort code |
| GETCOMPY | `GetCompany` | Retrieve company name |
| NEWACCNO | `NewAccountNumber` | Generate sequential account number |
| NEWCUSNO | `NewCustomerNumber` | Generate sequential customer number |
| OCR1–OCR5 | `CRECUST` | Async credit scoring via CICS channels/containers |

---

## 5. Inter-Module Integration Points

| From | To | Mechanism | Data |
|------|-----|-----------|------|
| React Frontend | WebUI Java (Liberty) | HTTP/axios via `REACT_APP_*_URL` env vars | JSON (account/customer payloads) |
| WebUI Java | DB2 | JDBC via `HBankDataAccess` | ACCOUNT, PROCTRAN tables |
| WebUI Java | VSAM | jCICS KSDS API | CUSTOMER file |
| WebUI Java | COBOL Programs | CICS LINK via data interfaces | GETSCODE, GETCOMPY, NEWACCNO, NEWCUSNO |
| WebUI Java | CICS Async (credit scoring) | `RUN TRANSID` via channels/containers | OCR1–OCR5 → CRDTAGY1–5 |
| Spring Boot UIs | WebUI Java (Liberty) | REST via z/OS Connect (not direct) | N/A (separate path) |

---

## 6. Findings

| ID | Severity | Type | Location | Description |
|----|----------|------|----------|-------------|
| F-RUI-001 | **High** | Bug | `AccountDeleteTables.js` | `getAccountByNum()` called in component body without `useEffect` — triggers on every render, causing potential infinite API call loop. Mitigated by `mainAccountRow.length === 0` guard but fragile. |
| F-RUI-002 | **High** | Bug | `AccountDetailsTable.js` | `currentAccountCustomerNumber` is never populated (always `""`) — PUT request always sends empty customer number, risking data corruption on account update. |
| F-RUI-003 | **High** | Bug | `ProcessedTransactionCreateCustomerJSON.java` | `getReviewDate()` returns `customerDOB` instead of `customerReviewDate` — audit records for customer creation log the wrong review date. |
| F-RUI-004 | **High** | Bug | `CustomerResource.java` | `getCustomersTownInternal()` and `getCustomersSurnameInternal()` reuse the same `response` JSONObject in a loop — only the last customer's data is returned. |
| F-RUI-005 | **Medium** | Dead code | `AccountForm.js`, `NewAccount.js` | Two React components that are completely empty/non-functional. Never referenced in routing or other components. |
| F-RUI-006 | **Medium** | Dead code | `AdminPage.js` | Unused imports (`TextInput`, `NumberInput`, `Modal`, `Form`), unused modal state (`isModalOpened`, `isSuccessModalOpened`), and unused handler functions (`displayModal()`, `displaySuccessModal()`, `deleteModal()`). |
| F-RUI-007 | **Medium** | Dead code | `App.js` | Dead `DebugRouter` class at bottom of file — never used. |
| F-RUI-008 | **Medium** | Inconsistency | `CustomerCreationPage.js`, `AccountCreationPage.js` | Sort code is hardcoded as `"987654"` in React but dynamically fetched via COBOL program GETSCODE on the Java backend. If sort code changes, React won't reflect it. |
| F-RUI-009 | **Medium** | Pattern issue | `CustomerDetailsTable.js` | Axios response interceptor added on every update call — these accumulate as global interceptors, causing a memory leak pattern. |
| F-RUI-010 | **Medium** | Pattern issue | Multiple components | Modals rendered inside `map()` loops share the same state — all modals open/close together instead of individually. Affects `CustomerDetailsTable`, `CustomerDeleteTables`. |
| F-RUI-011 | **Medium** | Legacy code | `webui.data_access.*` (5 files) | Five wrapper classes (`Account`, `AccountList`, `Customer`, `CustomerList`, `GetUserSortCode`) exist for the older JSP/Servlet WebUI layer. They delegate to the same JAX-RS resources used by React. Potentially unused if the servlet UI is decommissioned. |
| F-RUI-012 | **Low** | JSX anti-pattern | Multiple React files | Uses HTML `class=` instead of React `className=` in several components (HomepageHeader, AdminHeader, HomePage). |
| F-RUI-013 | **Low** | Naming | `HBankDataAccess.java` | DB2 connection cache HashMap is named `cornedBeef` — unconventional naming reduces readability. |
| F-RUI-014 | **Low** | Unused endpoints | REST API | 18 of 28 REST endpoints are not called by the React frontend. They may be consumed by other modules (Spring Boot UIs) or are unused. |
| F-RUI-015 | **Low** | Deprecated API | `App.js` | Uses react-router-dom v5 `Switch`/`Route` instead of v6 `Routes`/`Route` with `element` prop. |

---

**Coverage Validation**: This document covers 100% of source files:
- 31/31 React JS files documented ✓ (including 2 dead code files)
- 35/35 WebUI Java files documented ✓ (including 5 legacy wrapper files)
- 28/28 REST API endpoints documented ✓
- All React-to-Java endpoint mappings documented ✓
- All data access patterns documented ✓
