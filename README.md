# GameZone — Dokumentasi Teknis & Spesifikasi Sistem

<div align="center">

![Versi Rilis](https://img.shields.io/badge/Release-v3.1.2-FFD800?style=flat-square&logo=git&logoColor=000)
![Platform](https://img.shields.io/badge/Platform-Windows%2010%20%7C%2011%20(x86__64)-00D084?style=flat-square&logo=windows&logoColor=000)
![Runtime](https://img.shields.io/badge/Runtime-Python%203.12%20%7C%20WebView2-5D5FEF?style=flat-square&logo=python&logoColor=fff)
![Arsitektur](https://img.shields.io/badge/Architecture-GPU--Accelerated%2060%20FPS-FF6B6B?style=flat-square&logo=webgl&logoColor=fff)
![Desain UI](https://img.shields.io/badge/UI%20Style-Neo--Brutalism%20v2-000000?style=flat-square&logo=w3c&logoColor=fff)

<p align="center">
  Perangkat lunak desktop Windows terintegrasi untuk automasi injeksi aset Steam, diagnostik integritas sistem (NetFix), manajemen sinkronisasi Cloud Save terenkripsi, pelacakan presensi komunitas real-time, serta orkestrasi distribusi patch mandiri berbasis WebView2.
</p>

</div>

---

## Daftar Isi
1. [Pendahuluan & Gambaran Umum](#1-pendahuluan--gambaran-umum)
2. [Arsitektur Perangkat Lunak & Jembatan IPC](#2-arsitektur-perangkat-lunak--jembatan-ipc)
3. [Spesifikasi Fungsional & Subsistem Inti](#3-spesifikasi-fungsional--subsistem-inti)
4. [Rancangan Antarmuka & Pipeline Rendering Visual](#4-rancangan-antarmuka--pipeline-rendering-visual)
5. [Topologi Direktori Repositori](#5-topologi-direktori-repositori)
6. [Panduan Instalasi & Eksekusi Lingkungan Pengembang](#6-panduan-instalasi--eksekusi-lingkungan-pengembang)
7. [Penjaminan Mutu & Otomasi Pengujian (QA Suite)](#7-penjaminan-mutu--otomasi-pengujian-qa-suite)
8. [Pipeline Rekayasa Kompilasi & Build Rilis](#8-pipeline-rekayasa-kompilasi--build-rilis)
9. [Protokol Keamanan & Integritas Data](#9-protokol-keamanan--integritas-data)
10. [Informasi Pengembang & Pemeliharaan](#10-informasi-pengembang--pemeliharaan)

---

## 1. Pendahuluan & Gambaran Umum

**GameZone Launcher** merupakan platform desktop Windows yang dikembangkan secara spesifik untuk memfasilitasi kebutuhan komunitas game PC modern. Sistem ini menggantikan paradigma arsitektur GUI tradisional (seperti Tkinter/CustomTkinter) dengan mengadopsi **Microsoft Edge WebView2** yang dikendalikan melalui kerangka kerja **PyWebView**.

Pilihan arsitektur ini memungkinkan antarmuka berjalan pada lapisan komposit perangkat keras (*hardware-accelerated GPU compositor*) dengan latensi respons 0 milidetik, konsumsi daya rendah saat diminimalkan ke *System Tray*, serta kemampuan menampilkan antarmuka grafis modern bergaya *Neo-Brutalism* secara konsisten pada ekosistem Windows 10 dan Windows 11 (64-bit).

### Domain Permasalahan yang Diselesaikan
- **Automasi Penempatan Lisensi & Manifest**: Mengeliminasi kebutuhan manipulasi manual berkas `.lua` dan `.manifest` ke direktori internal Steam (`stplug-in`).
- **Penanganan Anomali Klien Steam**: Memitigasi bug perulangan status pembelian (*Purchase Loop*), penumpukan berkas lisensi usang (`appinfo.vdf`), dan kegagalan resolusi konektivitas jaringan Steam.
- **Isolasi Progres Permainan**: Menyediakan sarana pencadangan terstruktur untuk data simpanan game lokal (*save games*) serta mekanisme penguncian berkas `.acf` (*Steam Pinning*) dari penimpaan pembaruan daring.
- **Konektivitas Komunitas Terverifikasi**: Menghubungkan anggota komunitas melalui otentikasi Discord OAuth2 dengan verifikasi status keanggotaan aktif serta pelacakan telemetri presensi real-time.

---

## 2. Arsitektur Perangkat Lunak & Jembatan IPC

Sistem GameZone dibangun di atas pola pemisahan tanggung jawab tunggal (*Single Responsibility Principle*) dengan memisahkan secara tegas lapisan presentasi antarmuka, lapisan jembatan komunikasi (*bridge IPC*), dan lapisan logika komputasi lokal.

### Diagram Alur Komunikasi Sistem

```
┌─────────────────────────────────────────────────────────────┐
│                 Lapisan Presentasi (Frontend)               │
│   DOM / ES6 Vanilla JS • Modern CSS Neo-Brutalism • HLS.js  │
│                   Objek Klien Global: window.App            │
└──────────────────────────────▲──────────────────────────────┘
                               │ Inter-Process Communication
                               │ (pywebview.api asynchronous bridge)
┌──────────────────────────────▼──────────────────────────────┐
│                    Lapisan Jembatan (api.py)                │
│              Class GameZoneAPI (~137 Metode RPC)            │
│         Validasi Boundary • Serialisasi Respons JSON        │
└──────────────┬───────────────────────────────┬──────────────┘
               │                               │
┌──────────────▼──────────────┐ ┌──────────────▼──────────────┐
│       Lapisan Inti (core/)  │ │     Lapisan Layanan (services/)│
│ • discord_auth.py (OAuth2)  │ │ • steam_service.py (WebAPI) │
│ • injector.py (Stplug-in)   │ │ • netfix_service.py (IO/Net)│
│ • database.py (SQLite WAL)  │ │ • hardware_service.py (WMI) │
│ • firebase_sync.py (SSE/DB) │ │ • hotfix_service.py (OTA)   │
│ • tray_manager.py (Win32)   │ │ • save_sync_service.py (IO) │
└─────────────────────────────┘ └─────────────────────────────┘
```

### Karakteristik Jembatan Komunikasi (*IPC Bridge*)
1. **Asynchronous Non-Blocking Execution**: Pemanggilan metode dari JavaScript ke Python dieksekusi secara asinkron menggunakan Promise (`await window.pywebview.api.<method>()`), mencegah pembekuan (*UI freezing*) pada utas utama render Chromium.
2. **Standardized Response Protocol**: Seluruh respons dari backend mematuhi struktur data deterministik seragam:
   ```json
   {
     "success": true,
     "message": "Deskripsi status operasi",
     "data": {}
   }
   ```
3. **Safe Parameter Boundary Validation**: Setiap input dari antarmuka melewati validasi tipe dan batasan nilai sebelum diteruskan ke subsistem kernel/file system.

---

## 3. Spesifikasi Fungsional & Subsistem Inti

### 3.1 Subsistem Injeksi Steam & Resolusi Multi-Server (`core/injector.py`)
- **Pipeline Injeksi Berkas**: Mengunduh payload terkompresi yang memuat manifest dan skrip registrasi `.lua`, memverifikasi integritas payload, lalu mengekstraknya secara terstruktur ke direktori target Steam `stplug-in`.
- **Failover Multi-Mirror Dinamis**: Menerapkan algoritma pencarian sumber alternatif secara hierarkis (Firebase Primary Database -> SkyApi -> Sushi Mirror -> Morrenus Registry -> ManifestHub).
- **Proses Penutupan Aman (*Graceful Steam Shutdown*)**: Mengirimkan sinyal terminasi resmi `-shutdown` ke proses `steam.exe` dengan mekanisme polling maksimal 15 detik, menjamin proses penulisan registry Steam selesai sempurna sebelum berkas disuntikkan.
- **Batch Processing Queue**: Mengelola antrean injeksi multithreaded untuk pemrosesan beberapa judul game secara berurutan tanpa konflik konkurensi berkas.

### 3.2 Modul Diagnostik & Pemulihan Steam / NetFix (`services/netfix_service.py`)
- **Pembersihan Cache Lisensi**: Menginvalidasi berkas `appinfo.vdf` secara otomatis saat proses Steam tidak aktif guna memaksa sinkronisasi ulang lisensi lokal tanpa merusak berkas konfigurasi pengguna.
- **Multi-Drive Library Detection**: Memindai seluruh volume penyimpanan aktif di sistem operasi Windows (C:, D:, E:, dst.) untuk memvalidasi lokasi instalasi `steamapps/common`.
- **Resolusi Purchase Loop**: Memperbaiki anomali metadata instalasi lokal yang menyebabkan klien Steam menampilkan tombol pembelian (*Purchase*) alih-alih eksekusi (*Play*).

### 3.3 Presensi Komunitas Real-time & Protokol Interaksi (`core/firebase_sync.py`)
- **Sinkronisasi Presensi Dua Arah**: Mengirimkan data telemetri aktivitas lokal (status online, idle, dan judul game yang sedang dieksekusi) ke Firebase Realtime Database melalui koneksi aman.
- **Live Search Filtering**: Mesin pencari instan berbasis memori lokal di sisi klien yang menyaring daftar pengguna aktif berdasarkan kombinasi nama panggilan, tag Discord unik (`@username`), status aktivitas, dan hierarki peran pengguna.
- **Protokol Nudge (Senggol)**: Fitur transmisi sinyal interaktif antar pengguna dengan batas frekuensi (*rate-limit cooldown*) 5 detik, memicu visualisasi getaran layar (*spring-decay screen shake*) dan umpan balik audio pada klien target.

### 3.4 Persistensi Data, Cloud Save & Steam Pinning (`services/save_sync_service.py`, `services/steam_pinning_service.py`)
- **Database Engine Lokal**: Menggunakan SQLite 3 dengan mode *Write-Ahead Logging* (WAL) dan sinkronisasi normal untuk menjamin integritas transaksi berkecepatan tinggi tanpa overhead I/O disk.
- **Save State Versioning**: Mengompresi dan mengelola titik pemulihan data simpanan game dari emulator Goldberg dan OnlineFix ke penyimpanan lokal terstruktur.
- **Steam Pinning (`.acf` Protection)**: Mengunci hak akses penulisan pada berkas manifes aplikasi Steam (`appmanifest_<appid>.acf`) untuk mencegah klien Steam memaksakan unduhan pembaruan yang berpotensi membatalkan kompatibilitas berkas injeksi.

### 3.5 Panel Administrasi Tersemat (*Embedded Admin Center*) (`frontend/js/admin_embed.js`, `core/admin_api.py`)
- **Arsitektur Hibrida Terintegrasi**: Pengguna dengan hak akses level `ADMIN` dapat mengoperasikan panel kendali penuh langsung di dalam shell antarmuka utama tanpa perlu meluncurkan proses biner terpisah.
- **Stale-While-Revalidate SSD Caching**: Mengimplementasikan *Local Storage caching* lokal berkecepatan 0ms untuk data tabel pengguna dan konfigurasi server, meminimalisasi latensi jaringan pada saat peralihan tab.
- **Manajemen Operasional Komprehensif**: Fitur mencakup penyesuaian peran pengguna (*Role Management*), validasi dan pembaruan kunci API manifest, penyiaran teks pengumuman *News Ticker*, serta kendali rilis update major dan hotfix OTA.

### 3.6 Profiler Hardware & Evaluasi Kelayakan Komputasi (`services/hardware_service.py`)
- **Ekstraksi Telemetri Perangkat Keras**: Mengambil spesifikasi arsitektur prosesor (CPU), memori fisik (RAM terpasang), kartu grafis (GPU VRAM), dan platform display melalui instrumentasi WMI (*Windows Management Instrumentation*) dan pustaka `psutil`.
- **Mesin Inferensi Kelayakan Game (Can-I-Run-It)**: Membandingkan spesifikasi aktual mesin pengguna dengan batas ambang minimum dan rekomendasi teknis game dari pangkalan data Steam Store.

### 3.7 Mekanisme Distribusi Pembaruan & OTA Hotfix (`services/hotfix_service.py`)
- **Pembaruan Tingkat Biner (Major Update)**: Notifikasi pembaruan wajib (*mandatory update*) yang terhubung langsung ke aset rilis GitHub Releases, dilengkapi penampil progres unduhan terintegrasi.
- **Pembaruan Skrip Instan (Over-The-Air Hotfix)**: Memungkinkan penambalan cepat aset frontend (`frontend/`) dan modul layanan Python melalui arsip `.zip` yang diekstraksi ke direktori `%LOCALAPPDATA%\GameZone\hotfixes\current`, di mana sys.path di-override secara dinamis pada saat inisialisasi aplikasi.

---

## 4. Rancangan Antarmuka & Pipeline Rendering Visual

Antarmuka GameZone dirancang menggunakan pendekatan sistem desain **Neo-Brutalism v2** yang disesuaikan secara presisi untuk karakteristik rendering Chromium pada subsistem WebView2 Windows.

```
       [ Kanvas Utama: #F4EFEB ] ──> [ Border: 2.5px Solid #000 ]
                     │                                │
                     └───> [ Bayangan: 4px 4px 0 #000 ]
                                      │
               [ Koordinat Integer Diskrit (Anti-Blur) ]
```

### Prinsip Teknis Rendering
1. **Eliminasi Subpixel Interpolation Blur**: Menghindari pergeseran pecahan desimal (`translate(-1.5px, -1.5px)` atau bayangan `3.5px`). Seluruh transformasi animasi dan elevasi kursor wajib menggunakan nilai piksel bulat diskrit (`translate(-2px, -2px)` dan bayangan `5px 5px 0 #000`).
2. **Pembersihan Layer 3D Compositor**: Menghindari penempatan elemen teks dan kartu informatif di dalam deklarasi `perspective: 1000px` atau `transform: translateZ(0)` yang tidak perlu. Hal ini memastikan engine Skia/Blink mempertahankan antialiasing subpixel ClearType sehingga teks tetap tajam dan terbaca jelas.
3. **Optimasi CSS Containment**: Menggunakan properti `contain: layout style` pada kartu elemen status pengguna dan library untuk mengisolasi proses reflow layout dari dokumen utama.
4. **Sintesis Audio Web Berlatensi 0ms**: Menggunakan Web Audio API untuk menghasilkan gelombang synthesizer retro secara terprogram tanpa ketergantungan pada pemuatan file audio eksternal berukuran besar.

---

## 5. Topologi Direktori Repositori

```text
GameZone/
├── assets/                     # Berkas biner ikon aplikasi, logo, dan aset grafis statis
├── core/                       # Komponen logika bisnis inti dan infrastruktur sistem
│   ├── admin_api.py            # Protokol komunikasi API administratif dan sinkronisasi data
│   ├── config.py               # Pengelola parameter konfigurasi dan path runtime
│   ├── database.py             # Abstraksi database SQLite 3 dengan mode transaksi WAL
│   ├── discord_auth.py         # Orkestrasi OAuth2 Discord dan verifikasi status keanggotaan
│   ├── firebase_sync.py        # Komunikasi telemetri real-time dan listener presensi
│   ├── game_watcher.py         # Pemantau proses eksekusi game di sistem operasi
│   ├── injector.py             # Logika ekstraksi manifest dan registrasi berkas stplug-in
│   ├── profile_manager.py      # Pengelola cache identitas pengguna dan persistensi nickname
│   ├── single_instance.py      # Mekanisme mutex penjaminan eksekusi proses tunggal
│   └── tray_manager.py         # Penanganan interaksi System Tray Windows API
├── frontend/                   # Paket presentasi antarmuka klien tunggal (SPA)
│   ├── css/
│   │   ├── admin_embed.css     # Lembar gaya khusus modul administrasi terintegrasi
│   │   └── style.css           # Sistem desain terpadu Neo-Brutalism v2 dan animasi
│   ├── js/
│   │   ├── admin_embed.js      # Pengontrol logika administratif dan manipulasi DOM admin
│   │   ├── app.js              # State manager klien, event routing, dan objek global App
│   │   └── hls.min.js          # Library pemutar video HTTP Live Streaming (HLS)
│   └── index.html              # Titik masuk tunggal dokumen HTML5 antarmuka WebView2
├── services/                   # Layanan sistem fungsional modular
│   ├── deals_service.py        # Pengambil data katalog promosi dan potongan harga resmi
│   ├── diagnostic_service.py   # Pemeriksa integritas berkas sistem dan direktori Steam
│   ├── hardware_service.py     # Instrumentasi WMI deteksi spesifikasi CPU/GPU/RAM
│   ├── hotfix_service.py       # Pengevaluasi versi dan ekstraktor patch live OTA
│   ├── netfix_service.py       # Pengelola pembersihan cache lisensi dan perbaikan koneksi
│   ├── save_sync_service.py    # Utilitas pencadangan dan pemulihan berkas save game
│   ├── steam_service.py        # Integrasi Steam Web API, pencarian judul, dan media store
│   └── workshop_downloader_service.py # Pengunduh konten bengkel kerja Steam Workshop
├── tools/                      # Skrip utilitas operasional pengembang
│   ├── key_grabber.py          # Utilitas pengekstraksi kunci API manifest otomatis
│   ├── preflight.py            # Audit statis verifikasi sintaksis dan konsistensi kode
│   └── setup_hooks.py          # Konfigurasi otomatis pengait Git pre-commit
├── installer.iss               # Naskah konfigurasi Inno Setup Compiler 6 (LZMA2 packaging)
├── GameZone.spec               # Spesifikasi kompilasi PyInstaller binary collector
├── main.py                     # Entry point eksekusi launcher PyWebView
├── admin.py                    # Entry point aplikasi administratif mandiri (CustomTkinter)
├── qa_runner.py                # Kerangka kerja verifikasi fungsional dan pengujian UI
├── requirements.txt            # Deklarasi dependensi paket Python
└── AGENTS.md                   # Spesifikasi panduan teknis dan kepatuhan agen perekayasa
```

---

## 6. Panduan Instalasi & Eksekusi Lingkungan Pengembang

### Prasyarat Sistem
- **Sistem Operasi**: Windows 10 (Build 19041+) atau Windows 11 (64-bit).
- **Lingkungan Kompilasi**: Python versi `3.12.x` (64-bit).
- **Komponen Windows**: Microsoft Edge WebView2 Runtime (terpasang secara baku pada Windows 10/11 mutakhir).
- **Sistem Kontrol Versi**: Git client terkonfigurasi.

### Prosedur Setup Lingkungan
1. **Kloning Repositori**:
   ```powershell
   git clone https://github.com/mhmdaryaqadi/GameZone.git
   Set-Location GameZone
   ```

2. **Isolasi Virtual Environment**:
   ```powershell
   python -m venv venv
   .\venv\Scripts\Activate.ps1
   ```

3. **Instalasi Pustaka Dependensi**:
   ```powershell
   pip install -r requirements.txt
   ```

4. **Registrasi Git Pre-Commit Hook**:
   Pastikan skrip audit statis terpasang pada siklus komit lokal:
   ```powershell
   python tools/setup_hooks.py
   ```

### Prosedur Eksekusi
- **Menjalankan Launcher Utama (Mode Pengembang)**:
  ```powershell
  .\venv\Scripts\python.exe main.py
  ```
- **Menjalankan Panel Administrasi Mandiri**:
  ```powershell
  .\venv\Scripts\python.exe admin.py
  ```

---

## 7. Penjaminan Mutu & Otomasi Pengujian (QA Suite)

Proses pengembangan GameZone memberlakukan pengujian deterministik sebelum perubahan kode dapat diintegrasikan ke cabang utama (`main`).

### 1. Audit Statis Kode (`tools/preflight.py`)
Memvalidasi kelayakan kompilasi 60+ berkas Python, sintaksis ES6 `app.js`, keabsahan seluruh referensi metode `App.xxx`, keberadaan implementasi metode `pywebview.api`, serta sinkronisasi nomor versi di 6 titik lokasi kritis:
```powershell
.\venv\Scripts\python.exe tools\preflight.py
```
*Kriteria Lolos: Menghasilkan status `AMAN ✔` dengan exit code `0`.*

### 2. Validasi Integrasi Layanan Backend (`qa_runner.py --api`)
Menguji fungsionalitas class `GameZoneAPI`, konektivitas database lokal, integrasi API Steam, dan endpoint Firebase secara headless:
```powershell
.\venv\Scripts\python.exe qa_runner.py --api
```

### 3. Simulasi Auto-Pilot UI Tanpa Jendela (`qa_runner.py --headless`)
Menginstansiasi Chromium WebView2 secara headless untuk mengevaluasi 40 skenario interaksi DOM, peralihan halaman tab, integritas rendering CSS, serta verifikasi zero horizontal scrollbar:
```powershell
.\venv\Scripts\python.exe qa_runner.py --headless
```

### 4. Evaluasi Visual Interaktif (`qa_runner.py --live-ui`)
Membuka jendela WebView nyata dan mendemonstrasikan navigasi auto-pilot untuk pengamatan langsung rendering visual dan transisi halaman.
```powershell
.\venv\Scripts\python.exe qa_runner.py --live-ui
```

---

## 8. Pipeline Rekayasa Kompilasi & Build Rilis

Pembuatan paket biner mandiri produksi dilaksanakan melalui dua tahapan berurutan:

### Tahap 1: Kompilasi Biner Executable (PyInstaller)
Mengumpulkan interpreter Python, dependensi native C/C++, dan berkas aset frontend ke dalam satu direktori biner menggunakan optimasi bytecode (`optimize=2`):
```powershell
.\venv\Scripts\pyinstaller.exe GameZone.spec --noconfirm
```
*Keluaran: Direktori artefak biner di `dist/GameZone/`.*

### Tahap 2: Pembuatan Paket Installer Mandiri (Inno Setup)
Mengompilasi direktori biner menjadi satu berkas instalasi mandiri (`Setup.exe`) menggunakan algoritma kompresi tingkat tinggi LZMA2/ultra64:
```powershell
iscc installer.iss
```
*Keluaran: File instalasi tunggal di `dist/GameZoneSetup_v3.1.2.exe`.*

---

## 9. Protokol Keamanan & Integritas Data

1. **Otentikasi & Otorisasi Bertingkat**: Otentikasi pengguna diverifikasi menggunakan token Discord OAuth2 yang divalidasi silang terhadap ID server Discord resmi. Pengguna yang meninggalkan server Discord akan dikeluarkan secara otomatis dalam rentang toleransi 30 detik.
2. **Integritas Sistem Klien Steam**: Sistem tidak pernah memodifikasi biner eksekusi `Steam.exe` maupun berkas dinamis `.dll` bawaan Valve. Penambahan lisensi dilakukan secara modular melalui subsistem plugin Steam pihak ketiga (`stplug-in`).
3. **Pemisahan Kredensial & Rahasia Lingkungan**: Berkas repositori publik terlindungi dari pencatatan token otentikasi bot, kunci privat Firebase, maupun kredensial administratif melalui konfigurasi `.gitignore` yang ketat.
4. **Proteksi Kegagalan Jaringan**: Operasi penulisan berkas dan unduhan manifes mengimplementasikan penulisan temporer (*atomic write*) untuk mencegah korupsi data saat terjadi pemutusan jaringan tiba-tiba.

---

## 10. Informasi Pengembang & Pemeliharaan

- **Arsitek & Pengembang Utama**: **Muhammad Arya Alqadi** ([@mhmdaryaqadi](https://github.com/mhmdaryaqadi))
- **Organisasi & Komunitas**: Komunitas GameZone Discord
- **Repositori Distribusi Resmi**: [mhmdaryaqadi/GameZone-Releases](https://github.com/mhmdaryaqadi/GameZone-Releases)
- **Status Basis Kode**: Produksi Aktif (Versi `v3.1.2`)

---

<div align="center">
  <sub>Hak Cipta © 2026 GameZone. Seluruh hak cipta dilindungi undang-undang.</sub>
</div>
