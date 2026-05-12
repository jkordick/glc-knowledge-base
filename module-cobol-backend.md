# Module Specification: COBOL/BMS Backend

**Module**: COBOL/BMS Backend  
**User Story**: US-1 (Priority: P1)  
**Source Directories**: `src/base/cobol_src/` (29 programs), `src/base/bms_src/` (9 maps), `src/base/cobol_copy/` (37 copybooks)  
**Technology**: COBOL (CICS TS 6.1+), BMS (Basic Mapping Support), DB2 v12+, VSAM  
**Date**: 2026-05-12

---

## 1. Module Overview

The COBOL/BMS backend is the foundational layer of the CICS Bank Sample Application (CBSA). It implements all banking business logic as stateless CICS programs that communicate via COMMAREAs, with BMS-driven 3270 terminal screens providing the operator interface.

**Architecture**: Three-tier within CICS:
1. **BMS UI Drivers** (10 programs) — handle 3270 screen I/O and route to backend programs
2. **Backend Business Logic** (13 programs) — stateless services invoked via EXEC CICS LINK
3. **Utility/Support** (2 programs) + **Credit Agencies** (5 programs) — infrastructure and simulated external services

**Dependencies**: DB2 (ACCOUNT, PROCTRAN, CONTROL tables), VSAM (CUSTOMER, ABNDFILE files), CICS Named Counter Server

---

## 2. Source File Inventory

### 2.1 COBOL Programs (29)

| # | Program | Category | Purpose |
|---|---------|----------|---------|
| 1 | ABNDPROC | Utility | Writes abend diagnostic records to ABNDFILE VSAM |
| 2 | BANKDATA | Utility | Batch initializer for CUSTOMER VSAM and ACCOUNT DB2 |
| 3 | BNK1CAC | BMS UI Driver | Create Account screen handler |
| 4 | BNK1CCA | BMS UI Driver | List Accounts by Customer screen handler |
| 5 | BNK1CCS | BMS UI Driver | Create Customer screen handler |
| 6 | BNK1CRA | BMS UI Driver | Credit/Debit Funds screen handler |
| 7 | BNK1DAC | BMS UI Driver | Display/Delete Account screen handler |
| 8 | BNK1DCS | BMS UI Driver | Display/Delete/Update Customer screen handler |
| 9 | BNK1TFN | BMS UI Driver | Transfer Funds screen handler |
| 10 | BNK1UAC | BMS UI Driver | Update Account screen handler |
| 11 | BNKMENU | BMS UI Driver | Main Menu — routes to sub-transactions |
| 12 | CRDTAGY1 | Credit Agency | Dummy credit scoring (container CIPA) |
| 13 | CRDTAGY2 | Credit Agency | Dummy credit scoring (container CIPB) |
| 14 | CRDTAGY3 | Credit Agency | Dummy credit scoring (container CIPC) |
| 15 | CRDTAGY4 | Credit Agency | Dummy credit scoring (container CIPD) |
| 16 | CRDTAGY5 | Credit Agency | Dummy credit scoring (container CIPE) |
| 17 | CREACC | Backend Logic | Creates new account (DB2 INSERT, named counter) |
| 18 | CRECUST | Backend Logic | Creates new customer (VSAM WRITE, async credit check) |
| 19 | DBCRFUN | Backend Logic | Applies debit/credit to account (DB2 UPDATE) |
| 20 | DELACC | Backend Logic | Deletes single account (DB2 DELETE) |
| 21 | DELCUS | Backend Logic | Deletes customer and all accounts |
| 22 | GETCOMPY | Backend Logic | Returns company name string |
| 23 | GETSCODE | Backend Logic | Returns bank sort code |
| 24 | INQACC | Backend Logic | Retrieves single account from DB2 |
| 25 | INQACCCU | Backend Logic | Retrieves all accounts for a customer |
| 26 | INQCUST | Backend Logic | Retrieves customer from VSAM |
| 27 | UPDACC | Backend Logic | Updates account type/rate/overdraft in DB2 |
| 28 | UPDCUST | Backend Logic | Updates customer name/address in VSAM |
| 29 | XFRFUN | Backend Logic | Transfers funds between two accounts |

### 2.2 BMS Maps (9)

| # | Mapset/Map | Screen Purpose | Driving Program |
|---|-----------|----------------|-----------------|
| 1 | BNK1MAI / BNK1ME | Main Menu (7 options + A) | BNKMENU |
| 2 | BNK1ACC / BNK1ACC | List Accounts by Customer | BNK1CCA |
| 3 | BNK1CAM / BNK1CA | Create Account | BNK1CAC |
| 4 | BNK1CCM / BNK1CC | Create Customer | BNK1CCS |
| 5 | BNK1CDM / BNK1CD | Credit/Debit Funds | BNK1CRA |
| 6 | BNK1DAM / BNK1DA | Display/Delete Account | BNK1DAC |
| 7 | BNK1DCM / BNK1DC | Display/Delete/Update Customer | BNK1DCS |
| 8 | BNK1TFM / BNK1TF | Transfer Funds | BNK1TFN |
| 9 | BNK1UAM / BNK1UA | Update Account | BNK1UAC |

### 2.3 Copybooks (37)

| # | Copybook | Category | Purpose |
|---|----------|----------|---------|
| 1 | ABNDINFO | VSAM Record | Abend diagnostic record layout |
| 2 | ACCDB2 | DB2 Host Variable | ACCOUNT table DECLARE |
| 3 | ACCOUNT | VSAM Record | Account record layout with date REDEFINES |
| 4 | ACCTCTRL | VSAM Record | Account control/sequence counter record |
| 5 | BANKMAP | BMS Map Control | Main menu symbolic map (BNK1MAI) |
| 6 | BNK1DDM | BMS Map Control | BNK1DD display screen symbolic map |
| 7 | CONTDB2 | DB2 Host Variable | CONTROL table DECLARE |
| 8 | CONTROLI | DB2 Host Variable | Control counter packed-decimal host vars |
| 9 | CREACC | COMMAREA | Create Account operation layout |
| 10 | CRECUST | COMMAREA | Create Customer operation layout |
| 11 | CUSTCTRL | VSAM Record | Customer control/sequence counter record |
| 12 | CUSTMAP | BMS Map Control | Customer detail screen symbolic map |
| 13 | CUSTOMER | VSAM Record | Customer record layout |
| 14 | DELACC | COMMAREA | Delete Account operation layout |
| 15 | DELACCZ | COMMAREA | Delete Account — z/OS Connect variant |
| 16 | DELCUS | COMMAREA | Delete Customer operation layout |
| 17 | GETCOMPY | COMMAREA | Get Company Name operation layout |
| 18 | GETSCODE | COMMAREA | Get Sort Code operation layout |
| 19 | INQACC | COMMAREA | Inquire Account operation layout |
| 20 | INQACCCU | COMMAREA | Inquire Accounts by Customer layout |
| 21 | INQACCCZ | COMMAREA | Inquire Accounts by Customer — z/OS Connect variant |
| 22 | INQACCZ | COMMAREA | Inquire Account — z/OS Connect variant |
| 23 | INQCUST | COMMAREA | Inquire Customer operation layout |
| 24 | INQCUSTZ | COMMAREA | Inquire Customer — z/OS Connect variant |
| 25 | NEWACCNO | Named Counter | New account number allocation layout |
| 26 | NEWCUSNO | Named Counter | New customer number allocation layout |
| 27 | PAYDBCR | COMMAREA | Debit/Credit payment operation layout |
| 28 | PROCDB2 | DB2 Host Variable | PROCTRAN table DECLARE |
| 29 | PROCISRT | COMMAREA | Processed Transaction Insert (polymorphic) |
| 30 | PROCTRAN | VSAM Record | Processed transaction record (18 types) |
| 31 | RESPSTR | Utility | EIBRESP-to-string conversion paragraph |
| 32 | SORTCODE | Utility | Bank sort code constant (987654) |
| 33 | STCUSTNO | Utility | Customer number alternate index key |
| 34 | UPDACC | COMMAREA | Update Account operation layout |
| 35 | UPDCUST | COMMAREA | Update Customer operation layout |
| 36 | WAZI | Utility | Empty placeholder for IBM Wazi tooling |
| 37 | XFRFUN | COMMAREA | Transfer Funds operation layout |

---

## 3. Detailed Specifications

### 3.1 BMS UI Driver Programs

#### BNKMENU — Main Menu

| Attribute | Detail |
|-----------|--------|
| **Program ID** | BNKMENU |
| **Purpose** | Displays main menu and routes to sub-transactions based on user selection |
| **Transaction ID** | OMEN |
| **BMS Map** | BNK1MAI / BNK1ME |
| **Inputs** | ACTION field (1-7 or A) |
| **CICS Commands** | SEND MAP, RECEIVE MAP, RETURN TRANSID (IMMEDIATE) |
| **Programs Called** | ABNDPROC (error path) |
| **Key Logic** | EVALUATE on ACTION: 1→ODCS, 2→ODAC, 3→OCCS, 4→OCAC, 5→OUAC, 6→OCRA, 7→OTFN, A→OCCA |
| **Error Handling** | RESP checks, LINK ABNDPROC, ABEND-THIS-TASK |

**Menu Options**:
| Option | Transaction | Program | Function |
|--------|-------------|---------|----------|
| 1 | ODCS | BNK1DCS | Display/Delete/Update Customer |
| 2 | ODAC | BNK1DAC | Display/Delete Account |
| 3 | OCCS | BNK1CCS | Create Customer |
| 4 | OCAC | BNK1CAC | Create Account |
| 5 | OUAC | BNK1UAC | Update Account |
| 6 | OCRA | BNK1CRA | Credit/Debit |
| 7 | OTFN | BNK1TFN | Transfer Funds |
| A | OCCA | BNK1CCA | List Accounts by Customer |

---

#### BNK1CCS — Create Customer

| Attribute | Detail |
|-----------|--------|
| **Program ID** | BNK1CCS |
| **Purpose** | Collects customer demographics, validates input, LINKs to CRECUST |
| **Transaction ID** | OCCS |
| **BMS Map** | BNK1CCM / BNK1CC |
| **Inputs** | CUSTTIT, CHRISTN, CUSTINS, CUSTSN, CUSTAD1-3, DOBDD/MM/YY |
| **Outputs** | SORTC, CUSTNO2, CREDSC, SCRDTDD/MM/YY |
| **CICS Commands** | SEND MAP, RECEIVE MAP (ASIS), RETURN TRANSID, LINK, HANDLE ABEND, INQUIRE/SET TERMINAL |
| **Programs Called** | CRECUST, ABNDPROC |
| **Key Logic** | Disable UCTRAN for mixed-case; validate title/name/address/DOB; LINK CRECUST; restore UCTRAN |
| **Error Handling** | HANDLE ABEND, RESP checks, LINK ABNDPROC |

---

#### BNK1CAC — Create Account

| Attribute | Detail |
|-----------|--------|
| **Program ID** | BNK1CAC |
| **Purpose** | Collects account details, validates, LINKs to CREACC |
| **Transaction ID** | OCAC |
| **BMS Map** | BNK1CAM / BNK1CA |
| **Inputs** | CUSTNO, ACCTYP, INTRT, OVERDR |
| **Outputs** | ACCNO, SRTCD, dates, balances |
| **CICS Commands** | SEND MAP, RECEIVE MAP, RETURN TRANSID, LINK, BIF DEEDIT |
| **Programs Called** | CREACC, ABNDPROC |
| **Key Logic** | EVALUATE EIBAID: first→send empty; ENTER→validate & LINK CREACC; PF3→menu |
| **Error Handling** | RESP checks, LINK ABNDPROC |

---

#### BNK1CCA — List Accounts by Customer

| Attribute | Detail |
|-----------|--------|
| **Program ID** | BNK1CCA |
| **Purpose** | Lists up to 10 accounts for a given customer number |
| **Transaction ID** | OCCA |
| **BMS Map** | BNK1ACC / BNK1ACC |
| **Inputs** | CUSTNO (10 digits) |
| **Outputs** | Array of up to 10 accounts |
| **CICS Commands** | SEND MAP, RECEIVE MAP, RETURN TRANSID, LINK |
| **Programs Called** | INQACCCU, ABNDPROC |
| **Key Logic** | Validate numeric customer number; LINK INQACCCU; populate screen array |
| **Error Handling** | RESP checks, LINK ABNDPROC |

---

#### BNK1CRA — Credit/Debit Funds

| Attribute | Detail |
|-----------|--------|
| **Program ID** | BNK1CRA |
| **Purpose** | Applies credit or debit to an account via DBCRFUN |
| **Transaction ID** | OCRA |
| **BMS Map** | BNK1CDM / BNK1CD |
| **Inputs** | ACCNO, SIGN (+/-), AMT |
| **Outputs** | SORTC, AVBAL, ACTBAL |
| **CICS Commands** | SEND MAP, RECEIVE MAP, RETURN TRANSID, LINK |
| **Programs Called** | DBCRFUN, ABNDPROC |
| **Key Logic** | Validate numeric account & amount; parse decimal; determine sign; LINK DBCRFUN |
| **Error Handling** | RESP checks, LINK ABNDPROC |

---

#### BNK1DAC — Display/Delete Account

| Attribute | Detail |
|-----------|--------|
| **Program ID** | BNK1DAC |
| **Purpose** | Shows account details; deletes on PF5 |
| **Transaction ID** | ODAC |
| **BMS Map** | BNK1DAM / BNK1DA |
| **Inputs** | ACCNO |
| **Outputs** | Full account details |
| **CICS Commands** | SEND MAP, RECEIVE MAP, RETURN TRANSID, LINK |
| **Programs Called** | INQACC, DELACC, ABNDPROC |
| **Key Logic** | ENTER→LINK INQACC; PF5→validate & LINK DELACC |
| **Error Handling** | RESP checks, LINK ABNDPROC |

---

#### BNK1DCS — Display/Delete/Update Customer

| Attribute | Detail |
|-----------|--------|
| **Program ID** | BNK1DCS |
| **Purpose** | Shows customer details; deletes on PF5; updates on PF10 |
| **Transaction ID** | ODCS |
| **BMS Map** | BNK1DCM / BNK1DC |
| **Inputs** | CUSTNO; update: name, address, DOB, credit score |
| **Outputs** | Full customer details |
| **CICS Commands** | SEND MAP, RECEIVE MAP, RETURN TRANSID, LINK, HANDLE ABEND, INQUIRE/SET TERMINAL |
| **Programs Called** | INQCUST, DELCUS, UPDCUST, ABNDPROC |
| **Key Logic** | ENTER→LINK INQCUST; PF5→LINK DELCUS; PF10→LINK UPDCUST; manage UCTRAN |
| **Error Handling** | HANDLE ABEND, RESP checks, LINK ABNDPROC |

---

#### BNK1TFN — Transfer Funds

| Attribute | Detail |
|-----------|--------|
| **Program ID** | BNK1TFN |
| **Purpose** | Transfers funds between two accounts in the same bank |
| **Transaction ID** | OTFN |
| **BMS Map** | BNK1TFM / BNK1TF |
| **Inputs** | FACCNO, TACCNO, AMT |
| **Outputs** | From/to sort codes, actual/available balances |
| **CICS Commands** | SEND MAP, RECEIVE MAP, RETURN TRANSID, LINK |
| **Programs Called** | XFRFUN, ABNDPROC |
| **Key Logic** | Validate from/to account numbers and amount; LINK XFRFUN |
| **Error Handling** | RESP checks, LINK ABNDPROC |

---

#### BNK1UAC — Update Account

| Attribute | Detail |
|-----------|--------|
| **Program ID** | BNK1UAC |
| **Purpose** | Displays account; allows modification of type/rate/overdraft on PF5 |
| **Transaction ID** | OUAC |
| **BMS Map** | BNK1UAM / BNK1UA |
| **Inputs** | ACCNO; update: ACTYPE, INTRT, OVERDR |
| **Outputs** | Full account details |
| **CICS Commands** | SEND MAP, RECEIVE MAP, RETURN TRANSID, LINK |
| **Programs Called** | INQACC, UPDACC, ABNDPROC |
| **Key Logic** | ENTER→LINK INQACC; PF5→validate & LINK UPDACC |
| **Error Handling** | RESP checks, LINK ABNDPROC |

---

### 3.2 Backend Business Logic Programs

#### CREACC — Create Account

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Creates new account: validates customer, generates number via named counter, INSERTs DB2 |
| **COMMAREA** | CREACC.cpy |
| **DB2 Tables** | ACCOUNT (INSERT), PROCTRAN (INSERT), CONTROL (SELECT/UPDATE) |
| **VSAM Files** | None |
| **Programs Called** | INQCUST, INQACCCU, ABNDPROC |
| **Key Logic** | Validate customer exists; check <10 accounts; ENQ counter; get next number; INSERT ACCOUNT; INSERT PROCTRAN; DEQ |
| **Error Handling** | RESP/SQLCODE checks, ENQ/DEQ management, LINK ABNDPROC |

---

#### CRECUST — Create Customer

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Creates new customer: validates, async credit checks (5 agencies), writes VSAM, logs PROCTRAN |
| **COMMAREA** | CRECUST.cpy |
| **DB2 Tables** | PROCTRAN (INSERT) |
| **VSAM Files** | CUSTOMER (WRITE) |
| **Programs Called** | CRDTAGY1-5 (async RUN TRANSID), ABNDPROC |
| **Key Logic** | Validate title; ENQ counter; generate number; WRITE CUSTOMER; fire 5 async credit checks; FETCH CHILD (3-sec timeout); average scores; INSERT PROCTRAN; DEQ |
| **Error Handling** | RESP checks, CEEDAYS date validation, ENQ/DEQ rollback |

---

#### DBCRFUN — Debit/Credit

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Applies credit or debit to account, updates balances, writes audit log |
| **COMMAREA** | PAYDBCR.cpy |
| **DB2 Tables** | ACCOUNT (SELECT, UPDATE), PROCTRAN (INSERT) |
| **VSAM Files** | None |
| **Key Logic** | SELECT account; validate funds for debits; reject MORTGAGE/LOAN payments; UPDATE balances; INSERT PROCTRAN |
| **Error Handling** | HANDLE ABEND, SQLCODE checks, storm drain pattern |

---

#### DELACC — Delete Account

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Deletes single account from DB2 and logs deletion |
| **COMMAREA** | DELACC.cpy |
| **DB2 Tables** | ACCOUNT (SELECT, DELETE), PROCTRAN (INSERT) |
| **VSAM Files** | None |
| **Key Logic** | SELECT account; DELETE row; INSERT PROCTRAN |
| **Error Handling** | SQLCODE checks, ABEND('HRAC') |

---

#### DELCUS — Delete Customer

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Deletes customer and ALL associated accounts (cascade) |
| **COMMAREA** | DELCUS.cpy |
| **DB2 Tables** | PROCTRAN (INSERT) |
| **VSAM Files** | CUSTOMER (READ UPDATE, DELETE) |
| **Programs Called** | INQCUST, INQACCCU, DELACC (loop), ABNDPROC |
| **Key Logic** | Verify customer; get all accounts; loop LINK DELACC for each; READ/DELETE CUSTOMER; INSERT PROCTRAN |
| **Error Handling** | RESP checks with SYSIDERR retry, LINK ABNDPROC |

---

#### INQACC — Inquire Account

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Retrieves single account by sort code + account number (or last account if 99999999) |
| **COMMAREA** | INQACC.cpy |
| **DB2 Tables** | ACCOUNT (SELECT via CURSOR) |
| **Key Logic** | OPEN cursor; FETCH row; CLOSE cursor; map to COMMAREA |
| **Error Handling** | HANDLE ABEND, SQLCODE checks, storm drain, ABEND('HRAC') |

---

#### INQACCCU — Inquire Accounts by Customer

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Retrieves all accounts for a given customer (up to 20) |
| **COMMAREA** | INQACCCU.cpy |
| **DB2 Tables** | ACCOUNT (SELECT via CURSOR, multiple FETCH) |
| **Programs Called** | INQCUST, ABNDPROC |
| **Key Logic** | Verify customer; OPEN cursor; FETCH loop (max 20); CLOSE; populate array |
| **Error Handling** | HANDLE ABEND, SQLCODE checks, SYNCPOINT ROLLBACK |

---

#### INQCUST — Inquire Customer

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Retrieves customer from VSAM (supports random, specific, or last-customer lookup) |
| **COMMAREA** | INQCUST.cpy |
| **VSAM Files** | CUSTOMER (READ, STARTBR, READPREV, ENDBR) |
| **Key Logic** | 0→random (NCS last, random number); 9999999999→last; else→specific READ; retry on SYSIDERR (up to 100); retry random on NOTFND |
| **Error Handling** | HANDLE ABEND, RESP checks with retry, ABEND('CVR1') |

---

#### UPDACC — Update Account

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Updates account type, interest rate, and overdraft limit |
| **COMMAREA** | UPDACC.cpy |
| **DB2 Tables** | ACCOUNT (SELECT, UPDATE) |
| **Key Logic** | SELECT account; reject if type blank; UPDATE fields |
| **Error Handling** | SQLCODE checks, success/fail flag |

---

#### UPDCUST — Update Customer

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Updates customer name and/or address in VSAM |
| **COMMAREA** | UPDCUST.cpy |
| **VSAM Files** | CUSTOMER (READ UPDATE, REWRITE) |
| **Key Logic** | Validate title; READ for UPDATE; reject if both name/addr blank; REWRITE |
| **Error Handling** | RESP checks (NOTFND→fail '1', empty data→fail '4') |

---

#### XFRFUN — Transfer Funds

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Transfers funds between two accounts with proper locking and audit trail |
| **COMMAREA** | XFRFUN.cpy |
| **DB2 Tables** | ACCOUNT (SELECT, UPDATE ×2), PROCTRAN (INSERT ×2) |
| **Key Logic** | Reject same-account; lock in sort order (deadlock avoidance); UPDATE FROM (debit); UPDATE TO (credit); INSERT 2 PROCTRANs; SYNCPOINT ROLLBACK on failure |
| **Error Handling** | HANDLE ABEND, deadlock retry, SYNCPOINT ROLLBACK, ABEND('SAME') |

---

#### GETCOMPY — Get Company Name

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Returns literal 'CICS Bank Sample Application' |
| **COMMAREA** | GETCOMPY.cpy |
| **Key Logic** | MOVE literal to COMMAREA |

---

#### GETSCODE — Get Sort Code

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Returns bank sort code (987654) |
| **COMMAREA** | GETSCODE.cpy |
| **Key Logic** | MOVE SORTCODE to COMMAREA |

---

### 3.3 Utility/Support Programs

#### ABNDPROC — Abend Handler

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Centralized error handler — writes abend diagnostic record to VSAM |
| **VSAM Files** | ABNDFILE (WRITE) |
| **Copybooks** | ABNDINFO |
| **Key Logic** | WRITE abend record; DISPLAY error on failure |
| **Error Handling** | RESP/RESP2 check on WRITE |

---

#### BANKDATA — Batch Data Initializer

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Populates CUSTOMER VSAM and ACCOUNT DB2 with generated test data |
| **DB2 Tables** | ACCOUNT (INSERT) |
| **VSAM Files** | CUSTOMER (WRITE via COBOL file I/O) |
| **Key Logic** | Loop from-key to to-key; generate random names/addresses/titles; WRITE CUSTOMER; INSERT ACCOUNT |
| **Note** | Batch program — no CICS commands; uses native COBOL file I/O |

---

### 3.4 Credit Agency Programs (CRDTAGY1–CRDTAGY5)

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Simulate external credit scoring with random delay (1-3 sec) and random score (1-999) |
| **Invocation** | Async via EXEC CICS RUN TRANSID from CRECUST |
| **Channel** | CIPCREDCHANN |
| **Containers** | CIPA (AGY1), CIPB (AGY2), CIPC (AGY3), CIPD (AGY4), CIPE (AGY5) |
| **Key Logic** | DELAY random seconds; GET container; compute random score; PUT container |
| **Error Handling** | RESP checks, LINK ABNDPROC then ABEND('PLOP') |

---

### 3.5 BMS Map Details

#### BNK1MAI / BNK1ME — Main Menu

| Field | Length | Type | Description |
|-------|--------|------|-------------|
| ACTION | 1 | UNPROT, IC | Menu option selection |
| COMPANY | 43 | PROT | Company name display |
| MESSAGE | 79 | PROT | Status/error message |

**Navigation**: F3=Exit, F12=Cancel

---

#### BNK1ACC / BNK1ACC — List Accounts

| Field | Length | Type | Description |
|-------|--------|------|-------------|
| CUSTNO | 10 | NUM, IC | Customer number input |
| ACCOUNT (×10) | 79 | PROT | Account detail array |
| MESSAGE | 79 | PROT | Status message |

---

#### BNK1CAM / BNK1CA — Create Account

| Field | Length | Type | Description |
|-------|--------|------|-------------|
| CUSTNO | 10 | NUM | Customer number |
| ACCTYP | 8 | UNPROT | Account type |
| INTRT | 7 | NUM | Interest rate |
| OVERDR | 8 | NUM | Overdraft limit |
| ACCNO | 8 | PROT (output) | Generated account number |
| SRTCD | 6 | PROT (output) | Sort code |

---

#### BNK1CCM / BNK1CC — Create Customer

| Field | Length | Type | Description |
|-------|--------|------|-------------|
| CUSTTIT | 10 | UNPROT, IC | Title |
| CHRISTN | 20 | UNPROT | First name |
| CUSTINS | 2 | UNPROT | Initials |
| CUSTSN | 20 | UNPROT | Surname |
| CUSTAD1-3 | 60/60/40 | UNPROT | Address lines |
| DOBDD/MM/YY | 2/2/4 | NUM | Date of birth |
| SORTC | 6 | PROT (output) | Sort code |
| CUSTNO2 | 10 | PROT (output) | Customer number |
| CREDSC | 3 | PROT (output) | Credit score |

---

#### BNK1CDM / BNK1CD — Credit/Debit

| Field | Length | Type | Description |
|-------|--------|------|-------------|
| ACCNO | 8 | NUM, IC | Account number |
| SIGN | 1 | UNPROT (init '+') | Credit/debit sign |
| AMT | 13 | UNPROT (init '0000000000.00') | Amount |
| SORTC | 6 | PROT (output) | Sort code |
| AVBAL | 14 | PROT (output) | Available balance |
| ACTBAL | 14 | PROT (output) | Actual balance |

---

#### BNK1DAM / BNK1DA — Display/Delete Account

| Field | Length | Type | Description |
|-------|--------|------|-------------|
| ACCNO | 8 | NUM, IC | Account number input |
| All others | various | PROT | Full account details display |

**Navigation**: F3=Exit, F12=Cancel, PF5=Delete

---

#### BNK1DCM / BNK1DC — Display/Delete/Update Customer

| Field | Length | Type | Description |
|-------|--------|------|-------------|
| CUSTNO | 10 | IC | Customer number input |
| All others | various | PROT | Full customer details display |

**Navigation**: F3=Exit, F12=Cancel, PF5=Delete, PF10=Update

---

#### BNK1TFM / BNK1TF — Transfer Funds

| Field | Length | Type | Description |
|-------|--------|------|-------------|
| FACCNO | 8 | NUM | From account |
| TACCNO | 8 | NUM | To account |
| AMT | 13 | UNPROT (init '0000000000.00') | Transfer amount |
| FSORTC/TSORTC | 6 | PROT (output) | Sort codes |
| FACTBAL/TACTBAL | various | PROT (output) | Actual balances |
| FAVBAL/TAVBAL | various | PROT (output) | Available balances |

---

#### BNK1UAM / BNK1UA — Update Account

| Field | Length | Type | Description |
|-------|--------|------|-------------|
| ACCNO | 8 | NUM, IC | Account number input |
| ACTYPE | 8 | UNPROT | Account type (editable) |
| INTRT | 7 | NUM | Interest rate (editable) |
| OVERDR | 8 | NUM | Overdraft (editable) |
| All others | various | PROT | Display-only fields |

**Navigation**: F3=Exit, F12=Cancel, PF5=Update

---

### 3.6 Copybook Specifications

#### COMMAREA Copybooks (15)

| Copybook | Operation | Key Fields | Consumer Programs |
|----------|-----------|------------|-------------------|
| CREACC | Create Account | custno, sortcode, accno, type, rate, overdraft, balances, success/fail | CREACC, BNK1CAC |
| CRECUST | Create Customer | sortcode, custno, name, address, DOB, credit-score, success/fail | CRECUST, BNK1CCS |
| DELACC | Delete Account | custno, sortcode, accno, full account fields, del-success/fail, APPLID, PCB ptrs | DELACC, BNK1DAC |
| DELACCZ | Delete Account (z/OS Connect) | number-of-accounts, custno, account array (1-20), PCB as X(4) | z/OS Connect services |
| DELCUS | Delete Customer | sortcode, custno, name, addr, DOB, credit-score, del-success/fail | DELCUS, BNK1DCS |
| GETCOMPY | Get Company Name | company-name X(40) | GETCOMPY |
| GETSCODE | Get Sort Code | sortcode X(6) | GETSCODE |
| INQACC | Inquire Account | full account fields + success/fail + PCB POINTER | INQACC, BNK1DAC, BNK1UAC |
| INQACCCU | Inquire Accounts by Customer | num-accounts, custno, customer-found, PCB POINTER, account array (1-20) | INQACCCU, BNK1CCA, CREACC, DELCUS |
| INQACCCZ | Inquire Accounts (z/OS Connect) | Same as INQACCCU with PCB as X(4) | z/OS Connect services |
| INQACCZ | Inquire Account (z/OS Connect) | Same as INQACC with PCB as X(4) | z/OS Connect services |
| INQCUST | Inquire Customer | sortcode, custno, name, addr, DOB, credit-score, success/fail, PCB POINTER | INQCUST, BNK1DCS, CREACC, DELCUS, INQACCCU |
| INQCUSTZ | Inquire Customer (z/OS Connect) | Same as INQCUST with PCB as X(4) | z/OS Connect services |
| PAYDBCR | Debit/Credit | accno, amount, sortcode, balances, origin (APPLID/USERID/facility), success/fail | DBCRFUN, BNK1CRA |
| UPDACC | Update Account | full account fields + success | UPDACC, BNK1UAC |
| UPDCUST | Update Customer | sortcode, custno, name, addr, DOB, credit-score, upd-success/fail | UPDCUST, BNK1DCS |
| XFRFUN | Transfer Funds | from-accno/sortcode, to-accno/sortcode, amount, all balances, success/fail | XFRFUN, BNK1TFN |

**Design Pattern**: The "Z" suffix variants (DELACCZ, INQACCCZ, INQACCZ, INQCUSTZ) replace native POINTER with PIC X(4) for z/OS Connect JSON serialization compatibility.

#### VSAM Record Copybooks (5)

| Copybook | Entity | Key | Record Length | Key Programs |
|----------|--------|-----|---------------|--------------|
| CUSTOMER | Customer | SORTCODE + CUSTOMER-NUMBER | ~259 bytes | BANKDATA, CRECUST, DELCUS, INQCUST, UPDCUST |
| ACCOUNT | Account | SORT-CODE + ACCOUNT-NUMBER | ~100 bytes | BANKDATA, CREACC, DBCRFUN, DELACC, DELCUS, INQACC, INQACCCU, UPDACC, XFRFUN |
| ABNDINFO | Abend Record | UTIME-KEY + TASKNO-KEY | ~680 bytes | ABNDPROC + all UI drivers |
| ACCTCTRL | Account Counter | SORT-CODE + CONTROL-NUMBER | matches ACCOUNT | BANKDATA, CREACC, DELACC |
| CUSTCTRL | Customer Counter | SORTCODE + CONTROL-NUMBER | matches CUSTOMER | BANKDATA, CRECUST |
| PROCTRAN | Transaction | SORT-CODE + NUMBER + DATE/TIME | ~100 bytes | CREACC, CRECUST, DBCRFUN, DELACC, DELCUS, XFRFUN |

#### DB2 Host Variable Copybooks (3)

| Copybook | Table | Purpose |
|----------|-------|---------|
| ACCDB2 | ACCOUNT | SQL DECLARE TABLE for precompiler |
| CONTDB2 | CONTROL | SQL DECLARE TABLE for precompiler |
| PROCDB2 | PROCTRAN | SQL DECLARE TABLE for precompiler |

#### Named Counter Copybooks (2)

| Copybook | Purpose | Functions |
|----------|---------|-----------|
| NEWACCNO | Account number allocation | Get-new (G), Rollback (R), Current (C) |
| NEWCUSNO | Customer number allocation | Get-new (G), Rollback (R), Current (C) |

#### Utility Copybooks (5)

| Copybook | Purpose |
|----------|---------|
| SORTCODE | Bank sort code constant (987654) — used by 18 programs |
| RESPSTR | EIBRESP-to-string conversion paragraph (~80 response codes) |
| STCUSTNO | Customer number alternate index key structure |
| CONTROLI | Packed-decimal host variables for control counters |
| WAZI | Empty placeholder for IBM Wazi tooling |

#### BMS Map Control Copybooks (3)

| Copybook | Mapset | Purpose |
|----------|--------|---------|
| BANKMAP | BNK1MAI | Main menu symbolic map (input/output structures) |
| CUSTMAP | CSTMAP1 | Customer detail screen symbolic map |
| BNK1DDM | BNK1DD | Display-only screen symbolic map |

---

## 4. Data Access Patterns

| Datastore | Type | Programs | Operations |
|-----------|------|----------|------------|
| **ACCOUNT** | DB2 | BANKDATA, CREACC, DBCRFUN, DELACC, INQACC, INQACCCU, UPDACC, XFRFUN | SELECT, INSERT, UPDATE, DELETE |
| **PROCTRAN** | DB2 | CREACC, CRECUST, DBCRFUN, DELACC, DELCUS, XFRFUN | INSERT only |
| **CONTROL** | DB2 | CREACC | SELECT, UPDATE (named counter) |
| **CUSTOMER** | VSAM KSDS | BANKDATA, CRECUST, DELCUS, INQCUST, UPDCUST | READ, WRITE, REWRITE, DELETE |
| **ABNDFILE** | VSAM KSDS | ABNDPROC | WRITE only |

**Patterns**:
- All DB2 writes include a corresponding PROCTRAN INSERT (audit trail)
- VSAM CUSTOMER access uses READ FOR UPDATE + REWRITE for modifications
- Named Counter Server used for ID generation (account numbers, customer numbers)
- Cursor-based retrieval for multi-row queries (INQACC, INQACCCU)
- SYNCPOINT ROLLBACK for transactional integrity (XFRFUN, INQACCCU)

---

## 5. Inter-Module Integration Points

| Source | Target | Mechanism | Data Format |
|--------|--------|-----------|-------------|
| BMS UI Drivers | Backend Logic | EXEC CICS LINK PROGRAM with COMMAREA | Copybook-defined COMMAREA |
| CRECUST | CRDTAGY1-5 | EXEC CICS RUN TRANSID (async) + FETCH CHILD | Channel CIPCREDCHANN / Containers |
| All programs | ABNDPROC | EXEC CICS LINK PROGRAM | ABNDINFO COMMAREA |
| Backend Logic | DB2 | Embedded SQL (EXEC SQL) | Host variables from ACCDB2/PROCDB2/CONTDB2 |
| Backend Logic | VSAM | EXEC CICS READ/WRITE/REWRITE/DELETE FILE | Record layouts from CUSTOMER/ACCOUNT/ABNDINFO |
| z/OS Connect Services | Backend Logic | EXEC CICS LINK PROGRAM | z/OS Connect variant copybooks (*Z suffix) |
| WebUI (Liberty) | Backend Logic | JCICS API (Java) → EXEC CICS LINK | Same COMMAREA structures |

---

## 6. Findings

| # | Severity | Type | Location | Description |
|---|----------|------|----------|-------------|
| F1 | Low | Potential Dead Code | WAZI.cpy | Empty copybook — contains only a comment. Placeholder for IBM Wazi tooling, never referenced by any program. |
| F2 | Low | Inconsistency | BNK1DDM.cpy | Symbolic map for BNK1DD screen — no corresponding BMS source in bms_src/ (may be generated externally or from a missing BMS map). |
| F3 | Low | Inconsistency | BANKMAP.cpy | Complex symbolic map appears to define a richer menu than what BNKMENU actually uses (includes customer/account/proctran controls). May be from an earlier design that was simplified. |
| F4 | Medium | Code Duplication | CRDTAGY1-5 | Five nearly identical programs differing only in container name (CIPA-CIPE). Could be consolidated into a single parameterized program. |
| F5 | Low | Inconsistency | CUSTMAP.cpy | BMS symbolic map for a customer screen that doesn't obviously map to any of the 9 BMS sources in bms_src/. May be generated from an external tool. |
| F6 | Medium | Design Observation | INQCUST.cbl | Random customer lookup (CUSTNO=0) uses retry loop up to 100 times on NOTFND, generating random numbers until one exists. Inefficient but functional for demo purposes. |
| F7 | Low | Naming Inconsistency | PROCISRT.cpy | Copybook named PROCISRT but no corresponding PROCISRT.cbl program exists in cobol_src/. The INSERT logic is inline in each business logic program. |
| F8 | Medium | Design Observation | DELCUS.cbl | Uses DELAY + retry on SYSIDERR for cascade delete. This is a polling pattern that may be fragile under load. |
| F9 | Low | Unused Variant | DELACCZ.cpy, INQACCCZ.cpy, INQACCZ.cpy, INQCUSTZ.cpy | z/OS Connect variant copybooks — not directly referenced by any program in cobol_src/. Used by z/OS Connect service wrappers (documented in module-zos-connect.md). |
| F10 | Low | Potential Dead Code | STCUSTNO.cpy | Alternate index key structure — no program in cobol_src/ references it via COPY. May be used by CICS resource definitions or external utilities. |

---

## 7. Program Call Graph

```
BNKMENU (OMEN)
├─[1]─ BNK1DCS (ODCS) ──→ INQCUST, DELCUS, UPDCUST
│                                    └─→ INQCUST, INQACCCU, DELACC (loop)
├─[2]─ BNK1DAC (ODAC) ──→ INQACC, DELACC
├─[3]─ BNK1CCS (OCCS) ──→ CRECUST ──→ CRDTAGY1-5 (async)
├─[4]─ BNK1CAC (OCAC) ──→ CREACC ──→ INQCUST, INQACCCU
├─[5]─ BNK1UAC (OUAC) ──→ INQACC, UPDACC
├─[6]─ BNK1CRA (OCRA) ──→ DBCRFUN
├─[7]─ BNK1TFN (OTFN) ──→ XFRFUN
└─[A]─ BNK1CCA (OCCA) ──→ INQACCCU ──→ INQCUST

Error paths: All BMS UI Drivers ──→ ABNDPROC ──→ ABNDFILE (VSAM)
```

---

## 8. Transaction ID Map

| TRANSID | Program | Menu Option | Function |
|---------|---------|-------------|----------|
| OMEN | BNKMENU | — | Main Menu |
| ODCS | BNK1DCS | 1 | Display/Delete/Update Customer |
| ODAC | BNK1DAC | 2 | Display/Delete Account |
| OCCS | BNK1CCS | 3 | Create Customer |
| OCAC | BNK1CAC | 4 | Create Account |
| OUAC | BNK1UAC | 5 | Update Account |
| OCRA | BNK1CRA | 6 | Credit/Debit |
| OTFN | BNK1TFN | 7 | Transfer Funds |
| OCCA | BNK1CCA | A | List Accounts by Customer |

---

*Document covers 29/29 COBOL programs, 9/9 BMS maps, 37/37 copybooks. Coverage: 100%.*
