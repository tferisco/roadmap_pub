# MRO ERP Implementation Plan

> **Project**: TC ERP — Engineering & Production MRO System
> **Platform**: ERPNext (Frappe Docker) + Apache Airflow
> **Last Updated**: 2026-02-24

Dokumen ini adalah rencana implementasi lengkap untuk sistem MRO ERP, mencakup modul, integrasi, roles & permissions, serta langkah verifikasi.

---

## Phase 1 — Engineering Module ✅ `COMPLETE`

Modul fondasi untuk master data capability dan spesifikasi teknis.

### 1.1 Deliverables

| # | Item | DocType | Status |
|:--|:-----|:--------|:-------|
| 1 | Capability Master | `Capability Master` | ✅ Done |
| 2 | Aircraft Type | `MRO Aircraft Type` | ✅ Done |
| 3 | Manufacturer | `MRO Manufacturer` | ✅ Done |
| 4 | Test Equipment Master | `MRO Test Equipment Master` | ✅ Done |
| 5 | LRU Part Number Master | `LRU Part Number Master` | ✅ Done |
| 6 | ATA Chapter | `MRO ATA Chapter` | ✅ Done |
| 7 | Chemicals Master | `MRO Chemicals Master` | ✅ Done |
| 8 | BDP Materials | `MRO BDP Materials` | ✅ Done |
| 9 | Workshop | `MRO Workshop` | ✅ Done |
| 10 | Personnel | `MRO Personnel` | ✅ Done |
| 11 | Document Control | `MRO Document` | ✅ Done |
| 12 | Revision History | `MRO Document Revision` | ✅ Done |
| 13 | Reports | Custom Print Format (Q-183) | ✅ Done |

### 1.2 Dependencies
- None (foundational module)

---

## Phase 2 — Production Planning & Control (PPC) 🚧 `IN PROGRESS`

Integrasi order data dari SAP dan generasi dokumen Maintenance Instructions (MI).

### 2.1 Order Management (Manual Input)

| Item | Detail |
|:-----|:-------|
| **Approach** | Manual Data Entry / **Batch Import** |
| **Source** | Email / Hardcopy Order |
| **Destination** | ERPNext (`MRO Order Input`) |
| **Data Fields** | `Order No`, `Part No`, `Serial No`, `Customer`, `Date In` |

**Tasks:**
- [x] Setup permissions supaya PPC bisa create `Work Order` / `MRO Order Input`
- [x] Validasi field `Serial No` dan `Part No` agar sinkron dengan Master Data
- [x] Testing manual creation flow (via list view & batch import)

### 2.2 Job Card / PD Sheet Generation

| Item | Detail |
|:-----|:-------|
| **Concept** | `MRO Maintenance Instruction` attachment di `Capability Master` → sistem merge dengan order data |
| **Template Engine** | `docxtpl` (Jinja2 for DOCX) |
| **Template Fields** | `{{ order_number }}`, `{{ serial_number }}`, `{{ material }}`, `{{ customer }}` |
| **Trigger** | "Generate" button di MRO Order Input (per-row / batch) |
| **Output** | `.docx` dan auto-convert `.pdf` (via `DocConverter` service) |
| **Scalability** | Microservice architecture for PDF conversion |

**Data Flow:**
```
Manual/Batch Input → MRO Order Input
                          ↓
Capability Master (DOCX Template) + Order Data → DocConverter → Final PDF
```

**Tasks:**
- [x] Add Template fields ke `Capability Master` (Maintenance Instructions)
- [x] Install `docxtpl` di container framework
- [x] Build render logic (`mro_core/doc_generator.py`)
- [x] Add "Generate" button di MRO Order Input UI
- [x] Setup LibreOffice headless (Dockerized `docconverter` service)
- [x] Testing dengan sample orders (Header/Footer supported)

### 2.3 MinIO Object Storage ✅ `COMPLETE`

| Item | Detail |
|:-----|:-------|
| **Service** | MinIO (S3-compatible, self-hosted) |
| **Docker** | `minio/minio:latest` — port 9000 (API), 9001 (Console) |
| **Bucket** | `mro` |
| **Persistence** | Host bind mount `./minio-data` |
| **Access** | Proxied via ERPNext API (no direct MinIO URL exposure) |

**Folder Structure:**
```
mro/
├── capability-master/{CAP-name}/maintenance-instructions/{template}.docx
├── mro-order-input/{order-number}/generated/{output}.pdf
└── mro-document/{doc-number}/revisions/{rev}.pdf
```

**Features:**
- [x] MinIO service in `docker-compose.yml` dengan healthcheck
- [x] `minio_storage.py` — upload, download, delete, list dengan structured paths
- [x] `minio_file_hook.py` — auto-sync Capability Master MI uploads ke MinIO via doc_events
- [x] `doc_generator.py` — generated PDF/DOCX langsung upload ke MinIO
- [x] Proxy endpoint `serve_file` — PDF terbuka di browser, login required
- [x] Migration script (`minio_migrate.py`) untuk file lama
- [x] Fallback ke local filesystem jika MinIO unavailable

### 2.4 Dependencies
- SAP access credentials
- Airflow server provisioning

---

## Phase 3 — Tool Management System (TMS) 🛠️ `PLANNED`

Sinkronisasi equipment data dan manajemen jadwal kalibrasi (PMI/IMTE).

### 3.1 SQL Data Synchronization

| Item | Detail |
|:-----|:-------|
| **Direction** | One-way sync (TMS → ERPNext) |
| **Method** | Scheduled job (Airflow / Cron) pull dari TMS SQL Database |
| **Target** | `MRO Test Equipment Master` |

**Tasks:**
- [ ] Request SQL read-only access dari IT Admin
- [ ] Map TMS fields ke `MRO Test Equipment Master`
- [ ] Build sync job
- [ ] Testing & validation

### 3.2 PMI & IMTE Management (Calibration)

| Item | Detail |
|:-----|:-------|
| **DocType** | Extend `MRO Test Equipment Master` atau gunakan Asset module |
| **Fields** | `Last Calibration Date`, `Next Calibration Date`, `Calibration Interval`, `Certificate No` |
| **Automation** | Email alert 30 hari sebelum expiry, auto-set status "Expired" |

**Tasks:**
- [ ] Add calibration fields ke DocType
- [ ] Setup email notification rules
- [ ] Build auto-status-update script
- [ ] Testing alert & status automation

### 3.3 Dependencies
- SQL access ke TMS database dari IT Admin

---

## Phase 4 — Production Module ⚙️ `PLANNED`

Eksekusi kerja oleh teknisi dan visibility ketersediaan tool.

### 4.1 Tool & Equipment Visibility

| Item | Detail |
|:-----|:-------|
| **Linkage** | `Capability Master` → `MRO Test Equipment Master` |
| **UI** | "Required Tools" table di Work Order view |
| **Logic** | Saat Work Order dibuat untuk Part Number, fetch required tools dari Capability |

**Tasks:**
- [ ] Build "Required Tools" child table di Work Order
- [ ] Auto-populate tools dari Capability Master
- [ ] UI implementation

### 4.2 Tool Status Monitoring

| Item | Detail |
|:-----|:-------|
| **Feature** | Real-time status check di "Required Tools" table |
| **Statuses** | ✅ Serviceable, ❌ Calibrating, ❌ Expired |
| **Validation** | Block Work Order release jika critical tools expired/unavailable |

**Tasks:**
- [ ] Add status indicator ke Required Tools table
- [ ] Build validation rule untuk Work Order release
- [ ] Testing edge cases

### 4.3 Personal Ability List (PAL) Management

| Item | Detail |
|:-----|:-------|
| **DocType** | `Personal Ability List` (link ke `Employee` / `MRO Personnel`) |
| **Fields** | `License No`, `Aircraft Type Rating`, `Component Rating`, `Expiry Date` |
| **Validation** | Pastikan assigned technician punya valid PAL untuk Part Number/Task |

**Tasks:**
- [ ] Create `Personal Ability List` DocType
- [ ] Build validation logic
- [ ] Link ke Work Order assignment flow

### 4.4 Dependencies
- Phase 2 (Work Order dari SAP)
- Phase 3 (Tool data sync)

---

## Phase 5 — Quality Module 🛡️ `PLANNED`

Capability development workflow dan monitoring.

### 5.1 Capability Development Workflow

| State | Action | Role |
|:------|:-------|:-----|
| `Draft` | Submit | Engineering |
| `Pending Review` | Review & Approve/Reject | Quality Manager |
| `Approved` | Final Approval | Management / Director |
| `Rejected` | Return to Draft | Quality Manager |

**Tasks:**
- [ ] Configure ERPNext Workflow engine pada `Capability Master`
- [ ] Setup state transitions & role-based buttons
- [ ] Testing workflow end-to-end

### 5.2 Capability Monitoring

| Item | Detail |
|:-----|:-------|
| **Report** | "Active Capability List" — filter by Aircraft, ATA, Customer |
| **Dashboard** | Count Capabilities by Status (Developed vs Approved) |

**Tasks:**
- [ ] Build script report
- [ ] Build dashboard widget

### 5.3 Dependencies
- Phase 1 (Capability Master)

---

## Phase 6 — Reporting Module 📊 `PLANNED`

### 6.1 Custom Reports

| Type | Use Case |
|:-----|:---------|
| **Script Reports** | Complex logic (TAT excluding weekends/holidays, cost analysis) |
| **Query Reports** | Direct SQL data extraction |
| **Report Builder** | Ad-hoc user reports |

**Tasks:**
- [ ] Identify top 5 priority reports
- [ ] Build script reports (TAT, Defect Rate, Cost)
- [ ] Build query reports
- [ ] User training untuk Report Builder

### 6.2 Dependencies
- Phase 2, 3, 4 (data availability)

---

## Phase 7 — Operational Dashboard 📈 `PLANNED`

### 7.1 Management Dashboard

| Metric | Visualization |
|:-------|:-------------|
| Orders in Progress vs Completed | Bar/Line Chart |
| Tools approaching calibration expiry | Donut Chart |
| Pending Capability Approvals | Counter |
| Daily SAP Orders fetched | Line Chart |

**Tasks:**
- [ ] Configure ERPNext Dashboard charts
- [ ] Link ke relevant DocTypes
- [ ] Setup auto-refresh

### 7.2 Dependencies
- Phase 2, 3, 5 (data sources)

---

## Roles & Permissions Matrix

### Role Definitions

| # | Role | Deskripsi | Estimasi Users |
|:--|:-----|:----------|:---------------|
| 1 | **Engineering** | Manage master data, capability development, template upload | 3–5 |
| 2 | **PPC** | Order management, Job Card/PD Sheet generation | 2–4 |
| 3 | **Technician** | Work execution, tool checks, PAL | 10–30+ |
| 4 | **Quality Manager** | Capability approval workflow, quality monitoring | 1–3 |
| 5 | **Management / Director** | Final approval, dashboard, reports | 1–3 |
| 6 | **Tool Store** | Equipment & calibration management | 1–2 |
| 7 | **System Administrator** | System config, integrations, user management | 1–2 |

> **Total estimasi: ~19–49+ user accounts** dengan **7 distinct roles**.
> Satu user bisa memiliki multiple roles via ERPNext Role Permission Manager.

### Permission Matrix

#### 1. Engineering

| DocType | Read | Create | Write | Submit | Delete |
|:--------|:----:|:------:|:-----:|:------:|:------:|
| `Capability Master` | ✅ | ✅ | ✅ | ✅ | ❌ |
| `MRO Document` / `MRO Document Revision` | ✅ | ✅ | ✅ | — | ❌ |
| All Master Data* | ✅ | ✅ | ✅ | — | ❌ |
| Work Order / MRO Order | ✅ | ❌ | ❌ | — | ❌ |

> *Master Data = Aircraft Type, Manufacturer, Test Equipment, LRU Part Number, ATA Chapter, Chemicals, BDP Materials, Workshop, Personnel

**Special:** Upload `.docx` templates ke `Capability Master`

---

#### 2. PPC (Production Planning & Control)

| DocType | Read | Create | Write | Submit | Delete |
|:--------|:----:|:------:|:-----:|:------:|:------:|
| Work Order / MRO Order | ✅ | ❌ | ✅ | — | ❌ |
| `Capability Master` | ✅ | ❌ | ❌ | — | ❌ |
| All Master Data | ✅ | ❌ | ❌ | — | ❌ |

**Special:** Trigger "Generate Job Card" & "Generate PD Sheet", Print/Export .docx/.pdf

---

#### 3. Technician (Production)

| DocType | Read | Create | Write | Submit | Delete |
|:--------|:----:|:------:|:-----:|:------:|:------:|
| Work Order (assigned) | ✅ | ❌ | ✅* | — | ❌ |
| `MRO Test Equipment Master` | ✅ | ❌ | ❌ | — | ❌ |
| `Capability Master` | ✅ | ❌ | ❌ | — | ❌ |
| `Personal Ability List` (own) | ✅ | ❌ | ❌ | — | ❌ |

> *Write terbatas pada status update (mulai/selesai kerja)

---

#### 4. Quality Manager

| DocType | Read | Create | Write | Submit | Delete |
|:--------|:----:|:------:|:-----:|:------:|:------:|
| `Capability Master` | ✅ | ❌ | ✅ | ✅** | ❌ |
| All Master Data | ✅ | ❌ | ❌ | — | ❌ |
| `MRO Test Equipment Master` | ✅ | ❌ | ❌ | — | ❌ |
| Reports & Dashboard | ✅ | — | — | — | — |

> **Workflow action: Approve / Reject capability submissions

**Special:** Receive email alerts untuk tool calibration expiry

---

#### 5. Management / Director

| DocType | Read | Create | Write | Submit | Delete |
|:--------|:----:|:------:|:-----:|:------:|:------:|
| `Capability Master` | ✅ | ❌ | ❌ | ✅** | ❌ |
| All DocTypes | ✅ | ❌ | ❌ | — | ❌ |
| Reports & Dashboard | ✅ | — | — | — | — |

> **Workflow action: Final Approval pada Capability Master

---

#### 6. Tool Store / Equipment Manager

| DocType | Read | Create | Write | Submit | Delete |
|:--------|:----:|:------:|:-----:|:------:|:------:|
| `MRO Test Equipment Master` | ✅ | ✅ | ✅ | — | ❌ |
| Work Order | ✅ | ❌ | ❌ | — | ❌ |

**Special:** Receive email alerts untuk calibration expiry, update calibration dates & certificates

---

#### 7. System Administrator

| DocType | Read | Create | Write | Submit | Delete |
|:--------|:----:|:------:|:-----:|:------:|:------:|
| All DocTypes | ✅ | ✅ | ✅ | ✅ | ✅ |

**Special:** Full system access — Airflow DAGs, user management, workflow config, Report Builder, TMS SQL config

---

## Immediate Next Steps

| # | Action | Phase | Priority |
|:--|:-------|:------|:---------|
| 1 | Request SQL access untuk TMS | Phase 3 | 🔴 High |
| 2 | Define Airflow server specs | Phase 2 | 🔴 High |
| 3 | Production MinIO credentials (ganti default) | Phase 2 | 🔴 High |
| 4 | Configure Workflow pada Capability Master | Phase 5 | 🟡 Medium |
| 5 | Setup user roles & permissions di ERPNext | All | 🟡 Medium |
