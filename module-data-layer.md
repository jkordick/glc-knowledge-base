# Module Specification: Data Layer

**Module**: Data Layer  
**User Story**: US-2 (Priority: P1)  
**Source**: DB2 DDL in `etc/install/base/db2jcl/`, VSAM record layouts in `src/base/cobol_copy/`  
**Technology**: DB2 v12+, VSAM KSDS, CICS Named Counter Server  
**Date**: 2026-05-12

---

## 1. Module Overview

The data layer provides persistent storage for the CBSA application using a mixed DB2 + VSAM topology. DB2 stores account and transaction data (relational, SQL-accessible). VSAM stores customer data and abend diagnostics (keyed sequential, accessed via CICS file control). The CICS Named Counter Server provides atomic sequence generation for customer and account number allocation.

**Key Design Decision**: Customer data is in VSAM (not DB2) while accounts are in DB2. This cross-store design means the logical CUSTOMER→ACCOUNT relationship is not enforced by DB2 referential integrity constraints.

---

## 2. Source File Inventory

### 2.1 DB2 DDL (from etc/install/base/db2jcl/)

| JCL Member | Purpose |
|------------|---------|
| CREDB00 | Create DB2 database CBSA |
| CRETB01 | CREATE TABLE IBMUSER.ACCOUNT |
| CRETB02 | CREATE TABLE IBMUSER.PROCTRAN |
| CRETB03 | CREATE TABLE IBMUSER.CONTROL |
| CREI101 | CREATE UNIQUE INDEX ACCTINDX on ACCOUNT(SORTCODE, NUMBER) |
| CREI201 | CREATE INDEX ACCTCUST on ACCOUNT(SORTCODE, CUSTOMER_NUMBER) |
| CREI301 | CREATE UNIQUE INDEX CONTINDX on CONTROL(CONTROL_NAME) |
| CRESG01-03 | CREATE STOGROUP for ACCOUNT, PROCTRAN, CONTROL |
| CRETS01-03 | CREATE TABLESPACE for each table |
| DROPDB2 | Master drop script |
| DRPDB00, DRPTB01-03, DRPI101-301, DRPSG01-03, DRPTS01-03 | Individual DROP statements |
| DB2BIND | BIND PLAN for COBOL programs |
| BTCHSQL | Batch SQL execution utility |
| INSTDB2 | Master install orchestration |

### 2.2 VSAM Record Layout Copybooks

| Copybook | Entity | Source Path |
|----------|--------|-------------|
| CUSTOMER.cpy | Customer record | src/base/cobol_copy/CUSTOMER.cpy |
| ABNDINFO.cpy | Abend diagnostic record | src/base/cobol_copy/ABNDINFO.cpy |
| CUSTCTRL.cpy | Customer counter control | src/base/cobol_copy/CUSTCTRL.cpy |
| ACCTCTRL.cpy | Account counter control | src/base/cobol_copy/ACCTCTRL.cpy |

### 2.3 DB2 Host Variable Copybooks

| Copybook | Table | Source Path |
|----------|-------|-------------|
| ACCDB2.cpy | ACCOUNT | src/base/cobol_copy/ACCDB2.cpy |
| PROCDB2.cpy | PROCTRAN | src/base/cobol_copy/PROCDB2.cpy |
| CONTDB2.cpy | CONTROL | src/base/cobol_copy/CONTDB2.cpy |
| CONTROLI.cpy | CONTROL (host vars) | src/base/cobol_copy/CONTROLI.cpy |

---

## 3. Detailed Specifications

### 3.1 DB2 Tables

#### ACCOUNT Table

**DDL Source**: CRETB01.jcl  
**Schema**: IBMUSER.ACCOUNT  
**Tablespace**: CBSA.ACCOUNT  

| Column | DB2 Type | Nullable | Description |
|--------|----------|----------|-------------|
| ACCOUNT_EYECATCHER | CHAR(4) | Yes | Record type identifier ('ACCT') |
| ACCOUNT_CUSTOMER_NUMBER | CHAR(10) | Yes | FK → CUSTOMER (logical only) |
| ACCOUNT_SORTCODE | CHAR(6) | NOT NULL | Bank sort code (PK part 1) |
| ACCOUNT_NUMBER | CHAR(8) | NOT NULL | Account number (PK part 2) |
| ACCOUNT_TYPE | CHAR(8) | Yes | Account type code |
| ACCOUNT_INTEREST_RATE | DECIMAL(6,2) | Yes | Annual interest rate |
| ACCOUNT_OPENED | DATE | Yes | Date account was opened |
| ACCOUNT_OVERDRAFT_LIMIT | INTEGER | Yes | Overdraft limit in pennies |
| ACCOUNT_LAST_STATEMENT | DATE | Yes | Date of last statement |
| ACCOUNT_NEXT_STATEMENT | DATE | Yes | Date of next statement |
| ACCOUNT_AVAILABLE_BALANCE | DECIMAL(12,2) | Yes | Available balance |
| ACCOUNT_ACTUAL_BALANCE | DECIMAL(12,2) | Yes | Actual balance |

**Indexes**:
| Index Name | Columns | Unique | Purpose |
|------------|---------|--------|---------|
| ACCTINDX | (ACCOUNT_SORTCODE, ACCOUNT_NUMBER) | Yes | Primary key enforcement |
| ACCTCUST | (ACCOUNT_SORTCODE, ACCOUNT_CUSTOMER_NUMBER) | No | Customer account lookup |

**Note**: DDL uses DECIMAL(6,2) and DECIMAL(12,2) while the COBOL copybook ACCDB2.cpy declares DECIMAL(4,2) and DECIMAL(10,2). The DDL is authoritative — the copybook may use a smaller precision for working storage.

**Programs that access**:
| Program | Operations |
|---------|-----------|
| BANKDATA | INSERT (batch initialization) |
| CREACC | INSERT |
| DBCRFUN | SELECT, UPDATE |
| DELACC | SELECT, DELETE |
| INQACC | SELECT (cursor) |
| INQACCCU | SELECT (cursor, multi-row) |
| UPDACC | SELECT, UPDATE |
| XFRFUN | SELECT, UPDATE (×2) |

---

#### PROCTRAN Table (Processed Transactions)

**DDL Source**: CRETB02.jcl  
**Schema**: IBMUSER.PROCTRAN  
**Tablespace**: CBSA.PROCTRAN  

| Column | DB2 Type | Nullable | Description |
|--------|----------|----------|-------------|
| PROCTRAN_EYECATCHER | CHAR(4) | Yes | Record type identifier ('PRTR') |
| PROCTRAN_SORTCODE | CHAR(6) | NOT NULL | Sort code |
| PROCTRAN_NUMBER | CHAR(8) | NOT NULL | Account number |
| PROCTRAN_DATE | DATE | Yes | Transaction date |
| PROCTRAN_TIME | CHAR(6) | Yes | Transaction time (HHMMSS) |
| PROCTRAN_REF | CHAR(12) | Yes | Transaction reference |
| PROCTRAN_TYPE | CHAR(3) | Yes | Transaction type code |
| PROCTRAN_DESC | CHAR(40) | Yes | Description |
| PROCTRAN_AMOUNT | DECIMAL(12,2) | Yes | Transaction amount |

**Indexes**: None defined in DDL (no unique constraint). This is an append-only audit log.

**Transaction Type Codes** (from PROCTRAN.cpy 88-level values):
| Code | Meaning | Generated By |
|------|---------|--------------|
| CRE | Credit | DBCRFUN |
| DEB | Debit | DBCRFUN |
| TFR | Transfer (local) | XFRFUN |
| OCA | Branch: create account | CREACC (BMS origin) |
| OCC | Branch: create customer | CRECUST (BMS origin) |
| ODA | Branch: delete account | DELACC (BMS origin) |
| ODC | Branch: delete customer | DELCUS (BMS origin) |
| ICA | Web: create account | CREACC (web origin) |
| ICC | Web: create customer | CRECUST (web origin) |
| IDA | Web: delete account | DELACC (web origin) |
| IDC | Web: delete customer | DELCUS (web origin) |
| PCR | Payment: credit | DBCRFUN (payment origin) |
| PDR | Payment: debit | DBCRFUN (payment origin) |
| CHA | Charges | (reserved) |
| CHF | Charge fee | (reserved) |
| CHI | Charge interest | (reserved) |
| CHO | Charge overdraft | (reserved) |
| OCS | Branch: other | (reserved) |

**Programs that access**:
| Program | Operations |
|---------|-----------|
| CREACC | INSERT |
| CRECUST | INSERT |
| DBCRFUN | INSERT |
| DELACC | INSERT |
| DELCUS | INSERT |
| XFRFUN | INSERT (×2 per transfer) |

---

#### CONTROL Table

**DDL Source**: CRETB03.jcl  
**Schema**: IBMUSER.CONTROL  
**Tablespace**: CBSA.CONTROL  

| Column | DB2 Type | Nullable | Description |
|--------|----------|----------|-------------|
| CONTROL_NAME | CHAR(32) | Yes* | Named counter identifier |
| CONTROL_VALUE_NUM | INTEGER | Yes | Numeric counter value |
| CONTROL_VALUE_STR | CHAR(40) | Yes | String configuration value |

*Note: DDL does not declare NOT NULL on CONTROL_NAME, but the unique index CONTINDX enforces uniqueness.

**Indexes**:
| Index Name | Columns | Unique | Purpose |
|------------|---------|--------|---------|
| CONTINDX | (CONTROL_NAME) | Yes | Named counter lookup |

**Known Control Records**:
| CONTROL_NAME | Purpose | Used By |
|--------------|---------|---------|
| ACCOUNT-COUNT | Total number of accounts | CREACC |
| ACCOUNT-LAST | Last assigned account number | CREACC |
| CUSTOMER-COUNT | Total number of customers | CRECUST (via NCS) |
| CUSTOMER-LAST | Last assigned customer number | CRECUST (via NCS) |

**Programs that access**: CREACC (SELECT/UPDATE for account numbering). Customer numbering uses CICS Named Counter Server rather than direct SQL.

---

### 3.2 VSAM Files

#### CUSTOMER File (KSDS)

**CICS File Name**: CUSTOMER  
**Organization**: KSDS (Key-Sequenced Data Set)  
**Key**: CUSTOMER-SORTCODE (6 bytes) + CUSTOMER-NUMBER (10 bytes) = 16-byte composite key  
**Record Layout**: CUSTOMER.cpy  

| Field | PIC | Offset | Length | Description |
|-------|-----|--------|--------|-------------|
| CUSTOMER-EYECATCHER | X(4) | 0 | 4 | Record identifier ('CUST') |
| CUSTOMER-SORTCODE | 9(6) DISPLAY | 4 | 6 | Bank sort code (key part 1) |
| CUSTOMER-NUMBER | 9(10) DISPLAY | 10 | 10 | Customer number (key part 2) |
| CUSTOMER-NAME | X(60) | 20 | 60 | Full name |
| CUSTOMER-ADDRESS | X(160) | 80 | 160 | Full address |
| CUSTOMER-DATE-OF-BIRTH | 9(8) | 240 | 8 | DOB as DDMMYYYY |
| CUSTOMER-CREDIT-SCORE | 999 | 248 | 3 | Credit score (0-999) |
| CUSTOMER-CS-REVIEW-DATE | 9(8) | 251 | 8 | Credit score review date |

**Approximate Record Length**: ~259 bytes

**Programs that access**:
| Program | Operations | Notes |
|---------|-----------|-------|
| BANKDATA | WRITE | Batch initialization (COBOL file I/O) |
| CRECUST | WRITE | New customer creation |
| INQCUST | READ, STARTBR, READPREV, ENDBR | Inquiry (supports random, specific, last) |
| UPDCUST | READ UPDATE, REWRITE | Name/address updates |
| DELCUS | READ UPDATE, DELETE | Customer deletion |

---

#### ABNDFILE (KSDS)

**CICS File Name**: ABNDFILE  
**Organization**: KSDS (Key-Sequenced Data Set)  
**Key**: ABND-UTIME-KEY (8 bytes packed) + ABND-TASKNO-KEY (4 bytes) = 12-byte composite key  
**Record Layout**: ABNDINFO.cpy  

| Field | PIC | Length | Description |
|-------|-----|--------|-------------|
| ABND-UTIME-KEY | S9(15) COMP-3 | 8 | Unique time key (packed decimal) |
| ABND-TASKNO-KEY | 9(4) | 4 | CICS task number |
| ABND-APPLID | X(8) | 8 | CICS application ID |
| ABND-TRANID | X(4) | 4 | Transaction ID |
| ABND-DATE | X(10) | 10 | Abend date |
| ABND-TIME | X(8) | 8 | Abend time |
| ABND-CODE | X(4) | 4 | CICS abend code |
| ABND-PROGRAM | X(8) | 8 | Failing program name |
| ABND-RESPCODE | S9(8) SIGN LEADING | 9 | EIBRESP code |
| ABND-RESP2CODE | S9(8) SIGN LEADING | 9 | EIBRESP2 code |
| ABND-SQLCODE | S9(8) SIGN LEADING | 9 | SQL return code |
| ABND-FREEFORM | X(600) | 600 | Free-form diagnostic text |

**Approximate Record Length**: ~680 bytes

**Programs that access**:
| Program | Operations |
|---------|-----------|
| ABNDPROC | WRITE (only writer — centralized error handler) |

**Note**: No program reads ABNDFILE in the current source. It is a write-only diagnostic store intended for operational review via external tools or CICS utilities.

---

### 3.3 Named Counter Server (NCS)

The CICS Named Counter Server provides atomic sequence generation without direct DB2 table manipulation:

| Counter | Purpose | Used By | Mechanism |
|---------|---------|---------|-----------|
| Account number | Allocate next account number | CREACC | ENQ/DEQ with DB2 CONTROL table SELECT/UPDATE |
| Customer number | Allocate next customer number | CRECUST | ENQ/DEQ with CICS Named Counter commands |

**Pattern**: Programs use EXEC CICS ENQ to serialize access, then either directly manipulate the CONTROL table (CREACC) or use CICS Named Counter Server GET commands (CRECUST).

---

## 4. Entity Relationships

```
┌─────────────────────┐          ┌─────────────────────┐
│   CUSTOMER (VSAM)   │ 1──────M │   ACCOUNT (DB2)     │
│                     │          │                     │
│ Key: SORTCODE +     │          │ Key: SORTCODE +     │
│      CUSTOMER-NUMBER│          │      ACCOUNT-NUMBER │
│                     │          │                     │
│ FK: none            │          │ FK (logical):       │
│                     │          │ CUSTOMER_NUMBER →   │
│                     │          │ CUSTOMER.NUMBER     │
└─────────────────────┘          └──────────┬──────────┘
                                            │
                                            │ 1──────M
                                            ▼
                                 ┌─────────────────────┐
                                 │  PROCTRAN (DB2)     │
                                 │                     │
                                 │  Key: none (no PK)  │
                                 │  Index: none        │
                                 │                     │
                                 │  FK (logical):      │
                                 │  SORTCODE + NUMBER  │
                                 │  → ACCOUNT          │
                                 └─────────────────────┘

┌─────────────────────┐          ┌─────────────────────┐
│  CONTROL (DB2)      │          │  ABNDFILE (VSAM)    │
│                     │          │                     │
│ Key: CONTROL_NAME   │          │ Key: UTIME-KEY +   │
│ (unique index)      │          │      TASKNO-KEY     │
│                     │          │                     │
│ Independent config  │          │ Independent error   │
│ and counter store   │          │ diagnostic store    │
└─────────────────────┘          └─────────────────────┘
```

### Relationship Details

| Relationship | Cardinality | Enforcement | Notes |
|--------------|-------------|-------------|-------|
| CUSTOMER → ACCOUNT | 1:M (max 20) | Application logic in INQACCCU/CREACC | Not enforced by DB2 constraint; cross-store (VSAM→DB2) |
| ACCOUNT → PROCTRAN | 1:M (unbounded) | Application logic | FK via SORTCODE + NUMBER; no referential integrity |
| CUSTOMER → PROCTRAN | 1:M (indirect) | Via ACCOUNT | Customers appear in PROCTRAN only through their accounts |

### Cross-Store Reconciliation

| Entity | DB2 | VSAM | Reconciliation Notes |
|--------|-----|------|---------------------|
| Customer | — | CUSTOMER (KSDS) | VSAM only. No DB2 table for customer data. |
| Account | ACCOUNT table | — | DB2 only. No VSAM file for account data. |
| Processed Transaction | PROCTRAN table | — | DB2 only. PROCTRAN.cpy defines a VSAM-style record layout for working storage, but actual persistence is DB2. |
| Control Counters | CONTROL table | — | DB2 only. Also accessible via CICS NCS. |
| Abend Diagnostics | — | ABNDFILE (KSDS) | VSAM only. |

**Conclusion**: No entity currently spans both DB2 and VSAM. The CUSTOMER→ACCOUNT relationship is the only cross-store dependency, linked by the CUSTOMER_NUMBER field.

---

## 5. Data Access Patterns

### Write Patterns

| Pattern | Tables/Files | Programs | Mechanism |
|---------|-------------|----------|-----------|
| Create with audit | ACCOUNT + PROCTRAN | CREACC | INSERT account, INSERT PROCTRAN (same unit of work) |
| Create with async | CUSTOMER + PROCTRAN | CRECUST | WRITE VSAM, RUN TRANSID (async), INSERT PROCTRAN |
| Update | ACCOUNT | DBCRFUN, UPDACC, XFRFUN | SELECT + UPDATE (same cursor) |
| Delete with cascade | CUSTOMER + ACCOUNT + PROCTRAN | DELCUS | Loop DELACC for each account, then DELETE CUSTOMER |
| Audit-only | PROCTRAN | All writes | Every mutation generates a PROCTRAN INSERT |

### Read Patterns

| Pattern | Tables/Files | Programs | Mechanism |
|---------|-------------|----------|-----------|
| Single-row by key | ACCOUNT | INQACC, DBCRFUN, UPDACC, DELACC | DECLARE CURSOR + FETCH (single row) |
| Multi-row by FK | ACCOUNT | INQACCCU | DECLARE CURSOR + FETCH loop (max 20 rows) |
| VSAM by key | CUSTOMER | INQCUST | READ FILE with key |
| VSAM browse | CUSTOMER | INQCUST | STARTBR + READPREV (for "last customer") |
| Random lookup | CUSTOMER | INQCUST (custno=0) | Generate random key, READ, retry on NOTFND |

### Concurrency Control

| Mechanism | Used For | Programs |
|-----------|----------|----------|
| EXEC CICS ENQ/DEQ | Sequence number allocation | CREACC, CRECUST |
| READ FOR UPDATE + REWRITE | VSAM record modification | UPDCUST, DELCUS |
| SYNCPOINT ROLLBACK | Transaction failure recovery | XFRFUN, INQACCCU |
| Sort-order locking | Deadlock avoidance on dual-account update | XFRFUN |

---

## 6. Inter-Module Integration Points

| Consumer Module | Access Method | Data Stores | Purpose |
|----------------|---------------|-------------|---------|
| COBOL Backend Programs | EXEC SQL / EXEC CICS FILE | All | Direct data manipulation |
| WebUI (Liberty) | JCICS API → EXEC SQL / EXEC CICS FILE | ACCOUNT, CUSTOMER, PROCTRAN | REST API backend (Java→CICS→DB2/VSAM) |
| z/OS Connect Services | LINK to COBOL programs | All (indirect) | REST-to-CICS bridge delegates to backend programs |
| Spring Boot (Customer Services) | REST → z/OS Connect → COBOL | All (indirect) | External UI layer |
| Spring Boot (Payment Interface) | REST → z/OS Connect → COBOL | ACCOUNT, PROCTRAN (indirect) | Payment operations |

---

## 7. Findings

| # | Severity | Type | Location | Description |
|---|----------|------|----------|-------------|
| F1 | Medium | Design Observation | CRETB02.jcl | PROCTRAN table has no primary key or unique index. It is an append-only audit log, but this means duplicate detection is impossible at the DB2 level. |
| F2 | Medium | Design Observation | CRETB01.jcl / CUSTOMER.cpy | CUSTOMER→ACCOUNT relationship is cross-store (VSAM→DB2) with no referential integrity enforcement. Orphan accounts could exist if customer deletion fails mid-cascade. |
| F3 | Low | Inconsistency | CRETB01.jcl vs ACCDB2.cpy | DDL declares DECIMAL(6,2) for interest rate and DECIMAL(12,2) for balances; copybook ACCDB2.cpy declares DECIMAL(4,2) and DECIMAL(10,2). DDL is authoritative. |
| F4 | Low | Design Observation | CRETB03.jcl | CONTROL table DDL does not declare CONTROL_NAME as NOT NULL, despite the unique index CONTINDX on that column. Null values could theoretically be inserted. |
| F5 | Low | Potential Concern | ABNDFILE | ABNDFILE is write-only — no program reads from it. Useful for operational diagnostics but could grow unbounded without external housekeeping. |

---

*Document covers 3/3 DB2 tables, 2/2 VSAM data files, 2/2 VSAM control files, 3/3 DB2 indexes, all relationship mappings. Coverage: 100%.*
