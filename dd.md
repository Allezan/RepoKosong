# Dokumentasi Hubungan Tabel dan Stored Procedures Aplikasi AAF

---

## 1. Pendahuluan

Dokumen ini menjelaskan hubungan antar tabel, stored procedures, dan Oracle views yang digunakan dalam aplikasi AAF (Actual Accrued & Freight). Dokumentasi ini disusun berdasarkan analisis file-file handler backend.

---

## 2. Arsitektur Database

### 2.1. Overview Database

```mermaid
flowchart TB
    subgraph MSSQL["🗄️ MS SQL Server"]
        subgraph SASKI["Database: SASKI"]
            MU["master.master_user"]
            MR["master.master_role"]
            MP["master.master_permission"]
            RPM["master.role_permission_map"]
        end

        subgraph APIHUB["Database: APIHUB"]
            AAF_H["AAF Headers<br/>(Accrued/Actual)"]
            AAF_D["AAF Details<br/>(Accrued/Actual)"]
            SP_AAF["Stored Procedures<br/>SP_APIHUB_ERPORACLE_*"]
        end

        subgraph APPROVAL["Database: APPROVAL"]
            UM["M_USER_MAPPING"]
        end
    end

    subgraph Oracle["🏢 Oracle EBS (via Linked Server)"]
        subgraph Views["Oracle Views"]
            KLICI_V["XXGL_KICI_COMMERCIAL_INVOICE_V"]
            ACCRUE_V["XXKI_SASKI_AAF_ACCRUE_DTL_V"]
            ACTF_V["XXKI_SASKI_AAF_KLICI_AVAILABLE_ACTF_V"]
            ANAF_V["XXKI_SASKI_AAF_KLICI_AVAILABLE_ANAF_V"]
            SUPP_V["Supplier Views"]
        end

        subgraph Staging["Staging Tables"]
            GL_STG["XXGL_JOURNAL_*_STAGING"]
            AP_STG["XXAP_INVOICES_*_STG"]
        end
    end

    MU --> MR
    MU --> RPM
    MR --> RPM
    MP --> RPM

    SP_AAF --> AAF_H
    SP_AAF --> AAF_D
    SP_AAF --> KLICI_V
    SP_AAF --> GL_STG
    SP_AAF --> AP_STG
```

---

## 3. Tabel dan Relasi per Handler

### 3.1. User Handler (`user.js`)

#### Tabel yang Digunakan

| Database | Schema | Tabel               | Deskripsi                  |
| -------- | ------ | ------------------- | -------------------------- |
| SASKI    | master | master_user         | Data user aplikasi         |
| SASKI    | master | master_role         | Data role user             |
| SASKI    | master | master_permission   | Data permission            |
| SASKI    | master | role_permission_map | Mapping role ke permission |
| APPROVAL | dbo    | M_USER_MAPPING      | Mapping user ke EBS        |

#### Entity Relationship Diagram

```mermaid
erDiagram
    MASTER_USER {
        int id PK
        varchar username
        varchar email
        int masterRoleId FK
    }

    MASTER_ROLE {
        int id PK
        varchar name
    }

    MASTER_PERMISSION {
        int id PK
        varchar name
    }

    ROLE_PERMISSION_MAP {
        int masterRoleId FK
        int masterPermissionId FK
    }

    M_USER_MAPPING {
        varchar User_Activity
        varchar userEBSID
    }

    MASTER_USER ||--o{ MASTER_ROLE : "has"
    MASTER_ROLE ||--o{ ROLE_PERMISSION_MAP : "has"
    MASTER_PERMISSION ||--o{ ROLE_PERMISSION_MAP : "has"
    MASTER_USER ||--o| M_USER_MAPPING : "mapped to"
```

#### Flow Login User

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant BE as user.js
    participant SASKI as SASKI DB
    participant SSO as SSO Service

    FE->>BE: POST /login {username, password}
    BE->>SSO: service.login()
    SSO->>BE: {accName, accEmail, accId}

    BE->>SASKI: SELECT from master_user WHERE username
    SASKI->>BE: User data

    BE->>SASKI: SELECT from role_permission_map WHERE roleId
    SASKI->>BE: Permissions (check id=68)

    alt Has Permission
        BE->>FE: {token, userData}
    else No Permission
        BE->>FE: 403 Forbidden
    end
```

---

### 3.2. Dashboard Handler (`dashboardAAF.js`)

#### Stored Procedures yang Digunakan

| Stored Procedure                                           | Fungsi                            |
| ---------------------------------------------------------- | --------------------------------- |
| SP_APIHUB_ERPORACLE_AAF_ACCRUED_INDEX_LIST                 | List dokumen Accrued              |
| SP_APIHUB_ERPORACLE_AAF_ACCRUED_INDEX_LIST_COUNT           | Count dokumen Accrued             |
| SP_APIHUB_ERPORACLE_AAF_ACCRUED_INDEX_DETAIL               | Detail dokumen Accrued            |
| SP_APIHUB_ERPORACLE_AAF_ACTUAL_ACCRUED_INDEX_V3            | List Actual Accrued               |
| SP_APIHUB_ERPORACLE_AAF_ACTUAL_ACCRUED_INDEX_V3_COUNT      | Count Actual Accrued              |
| SP_APIHUB_ERPORACLE_AAF_ACTUAL_ACCRUED_INDEX_DETAIL_V3     | Detail Actual Accrued             |
| SP_APIHUB_ERPORACLE_AAF_ACTUAL_NON_ACCRUED_INDEX_V3        | List Actual Non Accrued           |
| SP_APIHUB_ERPORACLE_AAF_ACTUAL_NON_ACCRUED_INDEX_V3_COUNT  | Count Actual Non Accrued          |
| SP_APIHUB_ERPORACLE_AAF_ACTUAL_NON_ACCRUED_INDEX_DETAIL_V3 | Detail Actual Non Accrued         |
| SP_APIHUB_ERPORACLE_AAF_DASHBOARD_COUNT                    | Aggregated counts untuk dashboard |

#### Flow Dashboard Data

```mermaid
flowchart TB
    subgraph Dashboard["Dashboard Handler"]
        IDX["indexAAF()"]
        CNT["getDashboardCounts()"]
    end

    subgraph DocTypes["Document Types"]
        ACC["Accrued"]
        ACTF["Actual Accrued"]
        ANAF["Actual Non Accrued"]
    end

    subgraph Status["Status Filter"]
        DRF["Draft"]
        SUB["Submitted"]
        ERR["Interface Error"]
    end

    IDX --> ACC
    IDX --> ACTF
    IDX --> ANAF

    ACC --> DRF
    ACC --> SUB
    ACC --> ERR

    CNT --> |SP_DASHBOARD_COUNT| RESULT["Counts per Status"]
```

---

### 3.3. Invoice/Report Handler (`invoice.js`)

#### Stored Procedures dan Views

| SP/View                                                | Tipe        | Fungsi                                  |
| ------------------------------------------------------ | ----------- | --------------------------------------- |
| SP_APIHUB_ERPORACLE_AAF_ACCRUED_CHECK_KLICI_VALIDATION | SP          | Validasi KLICI untuk Accrued            |
| SP_APIHUB_ERPORACLE_AAF_CHECK_KLICI_ACTF               | SP          | Validasi KLICI untuk Actual Accrued     |
| SP_APIHUB_ERPORACLE_AAF_CHECK_KLICI_ANAF               | SP          | Validasi KLICI untuk Actual Non Accrued |
| XXGL_KICI_COMMERCIAL_INVOICE_V                         | Oracle View | Master data Commercial Invoice          |
| XXKI_SASKI_AAF_KLICI_AVAILABLE_ACTF_V                  | Oracle View | KLICI available untuk ACTF              |
| XXKI_SASKI_AAF_KLICI_AVAILABLE_ANAF_V                  | Oracle View | KLICI available untuk ANAF              |

#### Flow Validasi KLICI

```mermaid
flowchart TB
    START["checkKLICI(KLICI)"]

    subgraph Step1["Step 1: Check Existence"]
        CHK["utils.checkKLICICount()"]
        ORA1["OPENQUERY → XXGL_KICI_COMMERCIAL_INVOICE_V"]
    end

    subgraph Step2["Step 2: Get Details"]
        DTL["utils.getKLICIDetail()"]
        ORA2["OPENQUERY → Commercial Invoice Details"]
    end

    subgraph Step3["Step 3: Validation SP"]
        VAL["SP_ACCRUED_CHECK_KLICI_VALIDATION"]
        RES["Returns: IsValid, AvailableTypes, Message"]
    end

    START --> CHK
    CHK --> ORA1
    ORA1 --> |count > 0| DTL
    ORA1 --> |count = 0| NOTFOUND["KLICI NOT EXIST"]
    DTL --> ORA2
    ORA2 --> VAL
    VAL --> RES
```

---

### 3.4. Supplier Handler (`supplier.js`)

#### Stored Procedures yang Digunakan

| Stored Procedure                            | Fungsi                                 |
| ------------------------------------------- | -------------------------------------- |
| SP_APIHUB_ERPORACLE_AAF_INDEX_APSUPPLIER_v2 | Get list supplier by Paid Country      |
| Dynamic OPENQUERY                           | Get Paid Country from Transaction Type |

#### Flow Get Suppliers by Transaction Type

```mermaid
flowchart LR
    REQ["Request: trxType=KIN-KBL"]

    subgraph GetPaidCountry["Get Paid Country"]
        PC["utils.getPaidCountryFromTrxType()"]
        ORA["OPENQUERY → Oracle OU Mapping"]
    end

    subgraph GetSuppliers["Get Suppliers"]
        SUP["utils.getAllSupplier(paidCountry)"]
        SP["SP_INDEX_APSUPPLIER_v2"]
    end

    RES["Response: Supplier List"]

    REQ --> PC
    PC --> ORA
    ORA --> SUP
    SUP --> SP
    SP --> RES
```

---

### 3.5. Accrued Handler (`accrued.js`)

#### Stored Procedures Utama

| Kategori   | Stored Procedure                                        | Fungsi                       |
| ---------- | ------------------------------------------------------- | ---------------------------- |
| **Header** | SP_APIHUB_ERPORACLE_AAF_ACCRUED_CREATE_HEADER           | Create header dokumen        |
| **Header** | SP_APIHUB_ERPORACLE_AAF_ACCRUED_INDEX_LIST              | List dokumen                 |
| **Header** | SP_APIHUB_ERPORACLE_AAF_ACCRUED_DELETE_HEADER           | Delete header                |
| **Detail** | SP_APIHUB_ERPORACLE_AAF_ACCRUED_INDEX_DETAIL            | Get detail lines             |
| **Detail** | SP_APIHUB_ERPORACLE_AAF_ACCRUED_DELETE_DETAIL           | Delete detail                |
| **Submit** | SP_APIHUB_ERPORACLE_AAF_ACCRUED_CREATE_JOURNAL          | Create GL journal via Oracle |
| **Status** | SP_APIHUB_ERPORACLE_AAF_ACCRUED_UPDATE_INTERFACE_STATUS | Update status interface      |

#### Alur Data Accrued

```mermaid
flowchart TB
    subgraph Create["Create Document"]
        C1["Create Header SP"]
        C2["Local DB: AAF_ACCRUE_HDR"]
    end

    subgraph AddDetail["Add Details"]
        D1["Create Detail SP"]
        D2["Local DB: AAF_ACCRUE_DTL"]
        D3["OPENQUERY → XXGL_KICI_COMMERCIAL_INVOICE_V"]
    end

    subgraph Submit["Submit to Oracle"]
        S1["Get COA Segments"]
        S2["Generate Transmission ID"]
        S3["Insert GL Staging Header"]
        S4["Insert GL Staging Lines"]
        S5["Call Create Journal SP"]
    end

    subgraph Oracle["Oracle Processing"]
        O1["XXGL_JOURNAL_HEADERS_STAGING"]
        O2["XXGL_JOURNAL_DETAILS_STAGING"]
        O3["Oracle Concurrent Program"]
        O4["GL Module"]
    end

    C1 --> C2
    C2 --> D1
    D1 --> D2
    D1 --> D3
    D2 --> S1
    S1 --> S2
    S2 --> S3
    S3 --> S4
    S4 --> S5
    S5 --> O1
    S5 --> O2
    O1 --> O3
    O2 --> O3
    O3 --> O4
```

---

### 3.6. Actual Accrued Handler (`actual.js`)

#### Stored Procedures Utama

| Kategori   | Stored Procedure                                                  | Fungsi             |
| ---------- | ----------------------------------------------------------------- | ------------------ |
| **Header** | SP_APIHUB_ERPORACLE_AAF_ACTUAL_CREATE                             | Create header      |
| **Header** | SP_APIHUB_ERPORACLE_AAF_ACTUAL_ACCRUED_INDEX_V3                   | List header        |
| **Detail** | SP_APIHUB_ERPORACLE_AAF_ACTUAL_CREATE_DETAIL                      | Create detail      |
| **Detail** | SP_APIHUB_ERPORACLE_AAF_ACTUAL_ACCRUED_INDEX_DETAIL_V3            | Get detail         |
| **GL**     | SP_APIHUB_ERPORACLE_AAF_GL_CREATE_HEADER                          | Create GL staging  |
| **GL**     | SP_APIHUB_ERPORACLE_AAF_UPDATE_GL_STAGING_HEADER                  | Update GL staging  |
| **GL**     | SP_APIHUB_ERPORACLE_AAF_DELETE_GL_HEADER                          | Delete GL staging  |
| **AP**     | SP_APIHUB_ERPORACLE_AAF_AP_CREATE_HEADER                          | Create AP staging  |
| **Submit** | SP_APIHUB_ERPORACLE_AAF_POST_REVERSE_TO_EBS                       | Post to Oracle EBS |
| **Batch**  | SP_APIHUB_ERPORACLE_AAF_SAVE_BATCH_TRANSMISSION_TO_DETAIL_ACCRUED | Save batch info    |

#### Alur Submit Actual Accrued (Dual Interface)

```mermaid
flowchart TB
    subgraph Phase1["Phase 1: GL Reverse Accrual"]
        G1["Fetch Accrued GL Amounts"]
        G2["Generate Transmission ID"]
        G3["Insert GL Header Staging"]
        G4["Insert GL Lines Staging (Reverse)"]
        G5["Call Create Journal SP"]
    end

    subgraph Phase2["Phase 2: AP Invoice"]
        A1["Calculate Net Amount"]
        A2["Round to Nearest 100"]
        A3["Insert AP Header Staging"]
        A4["Insert AP Lines Staging"]
        A5["Call Post AP Invoice SP"]
    end

    subgraph Oracle["Oracle EBS"]
        GL["GL Journal Entry<br/>(Reverse Accrual)"]
        AP["AP Invoice<br/>(Vendor Payment)"]
    end

    subgraph Update["Update Status"]
        U1["Save Batch/Transmission to Detail"]
        U2["Update Header Status → Submitted"]
    end

    G1 --> G2
    G2 --> G3
    G3 --> G4
    G4 --> G5
    G5 --> GL

    GL --> A1
    A1 --> A2
    A2 --> A3
    A3 --> A4
    A4 --> A5
    A5 --> AP

    AP --> U1
    U1 --> U2
```

---

### 3.7. Actual Non Accrued Handler (`actualNonAccrued.js`)

#### Stored Procedures Utama

| Kategori   | Stored Procedure                                           | Fungsi                  |
| ---------- | ---------------------------------------------------------- | ----------------------- |
| **Header** | SP_APIHUB_ERPORACLE_AAF_ACTUAL_NON_ACCRUED_CREATE          | Create header           |
| **Header** | SP_APIHUB_ERPORACLE_AAF_ACTUAL_NON_ACCRUED_INDEX_V3        | List header             |
| **Detail** | SP_APIHUB_ERPORACLE_AAF_ACTUAL_NON_ACCRUED_CREATE_DETAIL   | Create detail           |
| **Detail** | SP_APIHUB_ERPORACLE_AAF_ACTUAL_NON_ACCRUED_INDEX_DETAIL_V3 | Get detail              |
| **AP**     | SP_APIHUB_ERPORACLE_AAF_UPDATE_AP_STAGING_HEADER           | Update AP staging       |
| **AP**     | SP_APIHUB_ERPORACLE_AAF_UPDATE_AP_STAGING_LINES            | Update AP staging lines |
| **Submit** | SP_APIHUB_ERPORACLE_AAF_POST_AP_NON_ACCRUE_TO_EBS          | Post to Oracle EBS      |
| **Batch**  | SP_APIHUB_ERPORACLE_AAF_SAVE_BATCH_NAME_TO_DETAIL          | Save batch info         |

#### Alur Submit Actual Non Accrued (AP Only)

```mermaid
flowchart TB
    subgraph Validate["Validation"]
        V1["Get Document Details"]
        V2["Get KLICI Breakdown"]
        V3["Calculate Amounts"]
    end

    subgraph CreateAP["Create AP Invoice"]
        A1["Generate Batch Name"]
        A2["Create AP Header Staging"]
        A3["Create AP Lines Staging"]
        A4["Round Adjustment if Needed"]
    end

    subgraph PostEBS["Post to EBS"]
        P1["Call Post AP Non Accrue SP"]
        P2["Oracle Concurrent Program"]
        P3["AP Module"]
    end

    subgraph Update["Update Status"]
        U1["Save Batch Name to Detail"]
        U2["Update Document Status → Submitted"]
    end

    V1 --> V2
    V2 --> V3
    V3 --> A1
    A1 --> A2
    A2 --> A3
    A3 --> A4
    A4 --> P1
    P1 --> P2
    P2 --> P3
    P3 --> U1
    U1 --> U2
```

---

## 4. Oracle Views dan Fungsinya

### 4.1. Daftar Oracle Views

| View                                       | Lokasi | Fungsi                                   |
| ------------------------------------------ | ------ | ---------------------------------------- |
| XXGL_KICI_COMMERCIAL_INVOICE_V             | APPS   | Master data Commercial Invoice (KLICI)   |
| XXKI_SASKI_AAF_ACCRUE_DTL_V                | APPS   | Detail transaksi Accrued                 |
| XXKI_SASKI_AAF_ACTUAL_ACCRUE_DETAILS_V     | APPS   | Detail transaksi Actual Accrued          |
| XXKI_SASKI_AAF_ACTUAL_NON_ACCRUE_DETAILS_V | APPS   | Detail transaksi Actual Non Accrued      |
| XXKI_SASKI_AAF_KLICI_VALIDATION_V          | APPS   | Ringkasan validasi KLICI                 |
| XXKI_SASKI_AAF_KLICI_AVAILABLE_ACTF_V      | APPS   | KLICI available untuk Actual Accrued     |
| XXKI_SASKI_AAF_KLICI_AVAILABLE_ANAF_V      | APPS   | KLICI available untuk Actual Non Accrued |

### 4.2. Hubungan Antar Oracle Views

```mermaid
flowchart TB
    subgraph Master["Master Data"]
        KLICI["XXGL_KICI_COMMERCIAL_INVOICE_V<br/>Master KLICI"]
    end

    subgraph Transaction["Transaction Views"]
        ACC["XXKI_SASKI_AAF_ACCRUE_DTL_V<br/>Accrued Details"]
        ACTF["XXKI_SASKI_AAF_ACTUAL_ACCRUE_DETAILS_V<br/>Actual Accrued Details"]
        ANAF["XXKI_SASKI_AAF_ACTUAL_NON_ACCRUE_DETAILS_V<br/>Actual Non Accrued Details"]
    end

    subgraph Validation["Validation Views"]
        VAL["XXKI_SASKI_AAF_KLICI_VALIDATION_V<br/>KLICI Status Summary"]
        AVL_ACTF["XXKI_SASKI_AAF_KLICI_AVAILABLE_ACTF_V<br/>Available for ACTF"]
        AVL_ANAF["XXKI_SASKI_AAF_KLICI_AVAILABLE_ANAF_V<br/>Available for ANAF"]
    end

    KLICI --> ACC
    KLICI --> ACTF
    KLICI --> ANAF

    ACC --> VAL
    ACTF --> VAL
    ANAF --> VAL

    ACC --> AVL_ACTF
    ACTF --> AVL_ACTF

    ACC --> AVL_ANAF
    ACTF --> AVL_ANAF
    ANAF --> AVL_ANAF
```

---

## 5. Staging Tables Oracle

### 5.1. GL Staging (Journal)

| Tabel                        | Schema | Fungsi                |
| ---------------------------- | ------ | --------------------- |
| XXGL_JOURNAL_HEADERS_STAGING | XKIN   | Header jurnal GL      |
| XXGL_JOURNAL_DETAILS_STAGING | XKIN   | Detail line jurnal GL |

### 5.2. AP Staging (Invoice)

| Tabel                  | Schema | Fungsi                 |
| ---------------------- | ------ | ---------------------- |
| XXAP_INVOICES_HDR_STG  | XKIN   | Header invoice AP      |
| XXAP_INVOICES_LINE_STG | XKIN   | Detail line invoice AP |

### 5.3. Flow Data ke Staging

```mermaid
flowchart LR
    subgraph AAF["AAF Application"]
        BE["Backend"]
    end

    subgraph MSSQL["MS SQL Server"]
        SP["Stored Procedures"]
    end

    subgraph Oracle["Oracle via Linked Server"]
        subgraph GL["GL Staging"]
            GLH["XXGL_JOURNAL_HEADERS_STAGING"]
            GLD["XXGL_JOURNAL_DETAILS_STAGING"]
        end

        subgraph AP["AP Staging"]
            APH["XXAP_INVOICES_HDR_STG"]
            APL["XXAP_INVOICES_LINE_STG"]
        end

        CP["Concurrent Program"]
    end

    BE -->|Execute| SP
    SP -->|OPENQUERY INSERT| GLH
    SP -->|OPENQUERY INSERT| GLD
    SP -->|OPENQUERY INSERT| APH
    SP -->|OPENQUERY INSERT| APL

    GLH -->|Trigger| CP
    GLD -->|Trigger| CP
    APH -->|Trigger| CP
    APL -->|Trigger| CP
```

---

## 6. Ringkasan Stored Procedures

### 6.1. By Module

| Module             | Jumlah SP | Prefix                                        |
| ------------------ | --------- | --------------------------------------------- |
| Accrued            | 15+       | SP*APIHUB_ERPORACLE_AAF_ACCRUED*\*            |
| Actual Accrued     | 20+       | SP*APIHUB_ERPORACLE_AAF_ACTUAL*\*             |
| Actual Non Accrued | 15+       | SP*APIHUB_ERPORACLE_AAF_ACTUAL_NON_ACCRUED*\* |
| Dashboard          | 5+        | SP*APIHUB_ERPORACLE_AAF_DASHBOARD*\*          |
| Master             | 5+        | SP*APIHUB_ERPORACLE_AAF_INDEX*\*              |

### 6.2. Naming Convention

```
SP_APIHUB_ERPORACLE_AAF_[MODULE]_[ACTION]_[VERSION]

Contoh:
- SP_APIHUB_ERPORACLE_AAF_ACCRUED_CREATE_HEADER
- SP_APIHUB_ERPORACLE_AAF_ACTUAL_ACCRUED_INDEX_V3
- SP_APIHUB_ERPORACLE_AAF_CHECK_KLICI_ACTF
```

---

## 7. Referensi File

| Handler             | Path                                                                     |
| ------------------- | ------------------------------------------------------------------------ |
| accrued.js          | `saski-aaf-backend/src/routes/v1/handlers/transaction/accrued/`          |
| actual.js           | `saski-aaf-backend/src/routes/v1/handlers/transaction/actual/`           |
| actualNonAccrued.js | `saski-aaf-backend/src/routes/v1/handlers/transaction/actualNonAccrued/` |
| invoice.js          | `saski-aaf-backend/src/routes/v1/handlers/transaction/report/`           |
| freightReport.js    | `saski-aaf-backend/src/routes/v1/handlers/transaction/report/`           |
| supplier.js         | `saski-aaf-backend/src/routes/v1/handlers/master/`                       |
| user.js             | `saski-aaf-backend/src/routes/v1/handlers/master/`                       |
| dashboardAAF.js     | `saski-aaf-backend/src/routes/v1/handlers/dashboard/`                    |
| helpers.js          | `saski-aaf-backend/src/routes/v1/helpers/`                               |
