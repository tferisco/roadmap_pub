# Laporan Progress Sistem MRO ERP
## TC ERP — Engineering & Production MRO System

> **Platform**: ERPNext (Frappe Docker) + Apache Airflow
> **Tanggal Laporan**: 7 April 2026
> **Dibuat oleh**: Tim Implementasi MRO ERP

---

## Ringkasan Eksekutif

Sistem MRO ERP telah berhasil dibangun di atas platform **ERPNext** (open-source ERP berbasis web) yang berjalan di infrastruktur **Docker container** internal. Sistem ini dirancang untuk mendigitalisasi dan mengintegrasikan seluruh proses kerja divisi MRO — mulai dari manajemen capability engineering, perencanaan produksi, pengelolaan dokumen teknis, hingga monitoring tool kalibrasi.

### Status Keseluruhan

| Phase | Modul | Status | % Selesai |
|:------|:------|:------:|:---------:|
| Phase 1 | Engineering Module | ✅ **SELESAI** | 100% |
| Phase 2 | Production Planning & Control (PPC) | ✅ **SELESAI** | 100% |
| Phase 3 | Tool Management System (TMS) | 🚧 **SEBAGIAN** | 40% |
| Phase 4 | Production Module | 🔲 Belum Dimulai | 0% |
| Phase 5 | Quality Module | 🔲 Belum Dimulai | 0% |
| Phase 6 | Reporting Module | 🔲 Belum Dimulai | 0% |
| Phase 7 | Operational Dashboard | 🔲 Belum Dimulai | 0% |

---

## Phase 1 — Engineering Module ✅ SELESAI

### Fungsi & Tujuan
Modul ini adalah **fondasi utama** seluruh sistem. Menyimpan semua master data teknis yang dibutuhkan oleh departemen engineering, seperti daftar kemampuan (capability) untuk setiap komponen pesawat, spesifikasi tools, material, bahan kimia, dan dokumen teknis resmi.

### Cara Kerja

```
Engineer membuka ERPNext
    ↓
Membuat/update Capability Master (per Part Number LRU)
    ├── Mengisi data identitas komponen (ATA Chapter, Aircraft Type, Manufacturer)
    ├── Mendaftarkan Test Equipment & Special Tools yang dibutuhkan
    ├── Mendaftarkan Material & Chemicals yang digunakan
    ├── Mendaftarkan personil (Man Power Plan)
    ├── Melampirkan dokumen teknis (CMM, SB, AD, AMM, dll.)
    └── Mengisi checklist readiness → sistem otomatis menghitung Readiness Score (%)
```

### DocType yang Sudah Berjalan

| # | Nama | Fungsi |
|:--|:-----|:-------|
| 1 | **Capability Master** | Dokumen utama kemampuan per komponen (CAP-XXXXX) |
| 2 | **LRU Part Number Master** | Daftar nomor part komponen yang dikerjakan |
| 3 | **MRO Aircraft Type** | Master tipe pesawat (B737, A320, dll.) |
| 4 | **MRO Manufacturer** | Master pabrik/manufaktur komponen |
| 5 | **MRO ATA Chapter** | Klasifikasi sistem pesawat (ATA 34, ATA 24, dll.) |
| 6 | **MRO Test Equipment Master** | Daftar alat uji (IMTE/PMI) |
| 7 | **MRO Chemicals Master** | Daftar bahan kimia yang digunakan |
| 8 | **MRO BDP Materials** | Daftar material/suku cadang BDP |
| 9 | **MRO Workshop** | Struktur workshop/bengkel (hierarki Division → Department → Workshop) |
| 10 | **MRO Personnel** | Daftar personil (ID Number + Nama) |
| 11 | **MRO Document** | Dokumen teknis (CMM, SB, AD, SRM, dll.) dengan riwayat revisi |
| 12 | **MRO Document Revision** | Histori revisi dokumen (child table) |
| 13 | **Print Format Q-183 R7** | Laporan kemampuan resmi dalam format PDF (Shop Ability Form) |

### Fitur Unggulan

- **Readiness Score**: Sistem otomatis menghitung persentase kesiapan berdasarkan 6 checklist verifikasi (Facilities, Special Tools, Equipment, Personnel, Approved Data, Rating).
- **Document Cascade Update**: Saat revisi dokumen diupdate, semua Capability Master yang menggunakan dokumen tersebut otomatis terupdate revisinya.
- **Print PDF Q-183 R7**: Tombol cetak langsung di form Capability Master menghasilkan PDF laporan kemampuan resmi.
- **Filter Guard**: Proteksi tampilan list agar data tidak terbuka tanpa filter — mencegah loading data besar yang lambat.

---

## Phase 2 — Production Planning & Control (PPC) ✅ SELESAI

### Fungsi & Tujuan
Modul ini menangani **alur kerja order produksi** — dari penerimaan order, pembuatan dokumen kerja (PD Sheet / Job Card / Maintenance Instruction), hingga penyimpanan dokumen hasil generate secara digital.

### Cara Kerja

```
PPC menerima order (email/hardcopy)
    ↓
Buat MRO Order Input (manual / batch import)
    ├── Isi: Order No, Part No, Serial No, Customer, Date In, TSN/TSO/TSC
    ├── Sistem otomatis menemukan Capability Master yang sesuai
    └── Sistem otomatis memuat daftar Maintenance Instructions yang tersedia
          ↓
PPC klik tombol "Generate" per baris MI
    ├── Sistem validasi dokumen dependensi (harus status "Released")
    ├── Ambil template DOCX dari MinIO storage
    ├── Render template dengan data order (nama customer, serial no, dll.)
    ├── Konversi DOCX → PDF via Gotenberg (LibreOffice headless)
    └── Simpan PDF ke MinIO → link langsung bisa dibuka di browser
```

### Sub-Modul yang Sudah Berjalan

#### 2.1 — MRO Order Input

- Form input order kerja (manual entry)
- Auto-link ke Capability Master berdasarkan Part Number
- Support batch import via CSV/Excel
- Preservasi file yang sudah digenerate saat order di-save ulang

#### 2.2 — MI Document Generation (PD Sheet / Job Card)

- Template engine berbasis **docxtpl** (Jinja2 untuk file Word)
- Tombol **Generate** per baris Maintenance Instruction
- Tombol **Generate All** untuk generate semua MI sekaligus
- Validasi dependensi: dokumen teknis harus berstatus `"Released"` sebelum generate diproses
- Variabel template: `{{ order_number }}`, `{{ serial_number }}`, `{{ customer }}`, `{{ date_in }}`, `{{ tsn }}`, `{{ tso }}`, `{{ tsc }}`, dll.
- Output: **PDF** (semua tipe) + **DOCX** editable (khusus tipe DOCX)

#### 2.3 — MinIO Object Storage

- Penyimpanan file terpusat yang aman dan terstruktur (S3-compatible, self-hosted)
- Tidak ada URL langsung ke file — semua diakses melalui proxy ERPNext (login required)
- Struktur folder:

```
mro/
├── capability-master/{CAP-name}/maintenance-instructions/{template}.docx
├── mro-order-input/{order-number}/generated/{output}.pdf
└── mro-document/{doc-number}/revisions/{rev}.pdf
```

#### 2.4 — DocConverter Service (Gotenberg)

- Service LibreOffice headless yang berjalan di container terpisah
- Menerima file DOCX → menghasilkan PDF berkualitas tinggi
- Timeout 120 detik per konversi

### Pencapaian Teknis Penting

- ✅ Library `docxtpl` berhasil diinstall di semua container relevan (backend, scheduler, workers)
- ✅ Dependency documents divalidasi sebelum generate (mencegah generate dengan dokumen usang)
- ✅ Word MERGEFIELD dihapus otomatis dari template agar kompatibel dengan LibreOffice
- ✅ Fallback ke local filesystem jika MinIO tidak tersedia

---

## Phase 3 — Tool Management System (TMS) 🚧 SEBAGIAN SELESAI

### Fungsi & Tujuan
Modul ini bertujuan untuk **mensinkronisasi data alat ukur/kalibrasi (IMTE/PMI)** dari sistem TMS internal ke ERPNext, serta memungkinkan engineer melihat status dan nomor tool langsung dari Capability Master.

### Yang Sudah Selesai (~40%)

#### TMS Data Lake Sync
- **Airflow DAG** sudah dikonfigurasi untuk menjalankan sinkronisasi terjadwal
- Background job berjalan **setiap hari pukul 02.00 WIB**
- Penanganan timezone WIB sudah benar
- **Live progress UI** di ERPNext: engineer bisa melihat status sync secara real-time
- `MRO TMS Tool` DocType sudah dibuat sebagai penampung data dari TMS

#### TMS Tool Assignment di Capability Master
- Tombol **"Assign Tool"** tersedia di tabel Test Equipment dan Special Tools
- Engineer bisa memilih tool number langsung dari data lake TMS
- Validasi dan tampilan status tag (Serviceable, Expired, dll.) sudah berjalan
- List view dengan filter standard sudah aktif untuk `MRO TMS Tool`

### Yang Belum Selesai

| Item | Status | Blocker |
|:-----|:------:|:--------|
| SQL read-only access ke TMS database | ❌ Belum | Menunggu IT Admin |
| Mapping field TMS → MRO Test Equipment Master | ❌ Belum | Bergantung SQL access |
| Full sync job ke Test Equipment Master | ❌ Belum | Bergantung SQL access |
| Email alert kalibrasi mendekati expired | ❌ Belum | Bergantung data kalibrasi dari TMS |
| Auto-set status "Expired" otomatis | ❌ Belum | Bergantung data kalibrasi dari TMS |

---

## Phase 4 — Production Module 🔲 BELUM DIMULAI

### Fungsi & Tujuan (Rencana)
Modul untuk **eksekusi kerja di lantai produksi** — teknisi bisa melihat work order yang ditugaskan, mengecek ketersediaan tool, dan memastikan PAL (Personal Ability List) mereka valid sebelum mengerjakan komponen.

### Rencana Fitur

| Fitur | Deskripsi |
|:------|:----------|
| Tool & Equipment Visibility | Dari Work Order, teknisi langsung lihat tool yang dibutuhkan dan statusnya |
| Tool Status Monitoring | Indikator real-time: ✅ Serviceable / ❌ Calibrating / ❌ Expired |
| Work Order Release Block | Sistem mencegah release jika tool kritis expired/unavailable |
| Personal Ability List (PAL) | DocType baru untuk menyimpan rating/lisensi teknisi |
| PAL Validation | Validasi teknisi punya PAL valid sebelum ditugaskan di Work Order |

**Dependensi**: Phase 3 harus selesai terlebih dahulu (data tool dari TMS + SQL access).

---

## Phase 5 — Quality Module 🔲 BELUM DIMULAI

### Fungsi & Tujuan (Rencana)
Modul untuk **proses approval Capability Master** secara formal dengan jejak digital yang lengkap.

### Rencana Alur Approval

```
Engineering mengajukan Capability Master
    ↓ (Submit)
Status: "Pending Review"
    ↓ (Quality Manager Review)
Approve → Status: "Approved" (butuh final approval Management)
Reject  → Status: "Draft"   (dikembalikan ke Engineering)
    ↓ (Final Approval Management/Director)
Status: "Active" ← Capability siap digunakan untuk produksi
```

---

## Phase 6 — Reporting Module 🔲 BELUM DIMULAI

### Rencana Laporan

| Tipe Laporan | Deskripsi |
|:-------------|:----------|
| **TAT (Turnaround Time)** | Waktu pengerjaan per order, dikecualikan hari libur/weekend |
| **Defect Rate** | Rasio reject/rework per part/workshop |
| **Cost Analysis** | Rekapitulasi manhours dan material per order |
| **Tool Calibration Status** | Daftar tool mendekati/sudah expired |
| **Capability Coverage** | Berapa banyak Part Number yang sudah punya capability approved |

---

## Phase 7 — Operational Dashboard 🔲 BELUM DIMULAI

### Rencana Metrik Dashboard

| Metrik | Visualisasi |
|:-------|:-----------|
| Orders in Progress vs Completed | Bar/Line Chart |
| Tool mendekati expiry kalibrasi | Donut Chart |
| Pending Capability Approvals | Counter |
| Daily orders yang masuk | Line Chart |

---

## Infrastruktur Teknis yang Berjalan

| Komponen | Teknologi | Deskripsi |
|:---------|:----------|:----------|
| **ERP Platform** | ERPNext v15 (Frappe Framework) | Web-based ERP, open-source |
| **Custom App** | `mro_core` (Python + JavaScript) | Semua customization berada di app ini |
| **Containerisasi** | Docker Compose | Semua service berjalan dalam container |
| **Database** | MariaDB | Database utama ERPNext |
| **Cache/Queue** | Redis | Session, cache, background jobs |
| **Object Storage** | MinIO (S3-compatible) | Penyimpanan file DOCX/PDF terstruktur |
| **PDF Converter** | Gotenberg (LibreOffice headless) | Konversi DOCX ke PDF |
| **Scheduler** | Apache Airflow + ERPNext Scheduler | Job terjadwal (TMS sync, alerts) |

### Roles & Akses yang Sudah Aktif

| Role | Portal | Jumlah Shortcut | Status |
|:-----|:-------|:---------------:|:------:|
| **MRO Engineering** | `/mro-engineering` | 11 card | ✅ Aktif |
| **MRO PPC** | `/mro-ppc` | 2 card | ✅ Aktif |
| **System Administrator** | `/app` (desk penuh) | Semua | ✅ Aktif |

> Setiap role mendapat halaman portal khusus setelah login — user tidak perlu familiar dengan navigasi ERPNext standar.

---

## Pencapaian Signifikan vs Roadmap Awal

| Pencapaian | Keterangan |
|:-----------|:-----------|
| ✅ Custom Login Page | Halaman login dengan branding MRO (glassmorphism design) |
| ✅ Role-Based Portal | Engineering & PPC punya landing page sendiri setelah login |
| ✅ MI Document Generation | Generate PD Sheet/Job Card dari template DOCX otomatis |
| ✅ MinIO Integration | File management terpusat, aman, akses via proxy ERPNext |
| ✅ Workshop Hierarchy | Struktur tree: Division → Department → Workshop |
| ✅ TMS Assignment UI | Engineer bisa assign tool number dari data lake di Capability Master |
| ✅ Dependency Validation | Sistem mencegah generate dokumen jika referensi belum "Released" |
| ✅ Batch Import Order | PPC bisa import order dalam jumlah besar sekaligus |

---

## Hambatan & Risiko Aktif

| # | Hambatan | Dampak | Solusi yang Dibutuhkan |
|:--|:---------|:------:|:----------------------|
| 1 | **SQL access TMS belum tersedia** | Phase 3 tidak bisa selesai penuh | Request resmi ke IT Admin |
| 2 | **MinIO credentials masih default** | Risiko keamanan di environment production | Ganti credentials sebelum go-live |
| 3 | **Workflow Approval belum dikonfigurasi** | Phase 5 (Quality) belum bisa dimulai | Alokasi waktu konfigurasi |
| 4 | **User roles belum di-setup untuk semua user nyata** | User belum bisa onboarding | Setup roles per departemen |
| 5 | **Airflow server provisioning** | Sinkronisasi TMS belum production-ready | Definisi spesifikasi server Airflow |

---

## Prioritas Immediate (30 Hari Ke Depan)

| # | Action Item | Penanggung Jawab | Prioritas |
|:--|:-----------|:-----------------|:---------:|
| 1 | Request SQL read-only access TMS ke IT Admin | Manager MRO / IT | 🔴 Tinggi |
| 2 | Ganti MinIO credentials (production security) | Tim Implementasi | 🔴 Tinggi |
| 3 | Setup user accounts + roles untuk semua departemen | Tim Implementasi | 🟡 Sedang |
| 4 | Konfigurasi Workflow approval Capability Master | Tim Implementasi + Quality | 🟡 Sedang |
| 5 | Provisioning server Airflow (definisi specs) | IT Admin | 🟡 Sedang |
| 6 | Training user Engineering & PPC (pilot) | Tim Implementasi | 🟢 Bisa segera |

---

## Estimasi Timeline ke Depan

```
April 2026   : Selesaikan Phase 3 (TMS) — bergantung SQL access
               Setup user accounts & permissions semua departemen

Mei 2026     : Mulai Phase 4 (Production Module)
               Konfigurasi Phase 5 (Quality Workflow)

Juni 2026    : Phase 4 & 5 selesai
               Mulai Phase 6 (Reporting) & Phase 7 (Dashboard)

Juli 2026    : Go-live penuh semua modul
               Training menyeluruh ke semua user
               Monitoring & stabilisasi sistem
```

---

*Laporan ini dibuat berdasarkan kondisi sistem per 7 April 2026.*
*Untuk pertanyaan teknis lebih lanjut, silakan hubungi Tim Implementasi MRO ERP.*
