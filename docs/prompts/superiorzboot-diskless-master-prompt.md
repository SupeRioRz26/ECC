# SupeRioRzBoot — Master Prompt Bertahap: Diskless Server & Client (Rust, Windows 11 x64)

> Dokumen ini adalah **prompt induk + prompt per tahap** untuk membangun aplikasi diskless boot
> (kelas CCBoot) berbasis Rust dengan UI/UX modern-minimalis, rantai **Secure Boot penuh tanpa
> inject sertifikat custom ke BIOS**, dukungan **VBS / Core Isolation (HVCI) / TPM 2.0** agar game
> dengan anti-cheat kernel (Valorant/Vanguard, dll.) bisa berjalan, **GPU PnP multi-vendor NVIDIA +
> AMD dalam satu VHDX**, **monitoring klien yang bisa dicentang**, **remote view/control**, dan
> **mode Super User** dengan versioning image. Keluaran tahap terakhir: **`.zip` berisi
> `SupeRioRzBoot-Server.exe` dan `SupeRioRzBoot-Client.exe`**.

---

## 0. Cara Memakai Dokumen Ini

1. Buat repo kosong baru untuk proyek ini (jangan dicampur dengan repo lain).
2. Salin **Bagian A (Konteks Induk)** ke `CLAUDE.md` / `AGENTS.md` di root repo baru, atau tempel
   sebagai pesan pertama sesi AI coding Anda. Konteks ini berlaku untuk **semua** tahap.
3. Jalankan **Bagian B** tahap demi tahap. Setiap tahap punya: *Tujuan*, *Prompt* (blok siap
   salin), *Deliverable*, dan *Gerbang Lulus (Definition of Done)*. Urutan:
   0 → 1 → 2 → **2A (Disk Manager)** → 3 → 4 → 5 → **5A (Image Manager)** → 6 → … → 13.
4. **Jangan lanjut ke tahap berikutnya** sebelum gerbang lulus terpenuhi. Setelah tiap tahap,
   jalankan **Prompt Review Loop** (Bagian C.1) sampai tidak ada temuan kritis/tinggi.
5. Tahap 0 adalah *spike* verifikasi. Hasilnya (ADR) boleh mengubah keputusan di tahap lain —
   dan itu memang tujuannya.

---

## Bagian A — Konteks Induk (tempel sekali, berlaku untuk semua tahap)

```text
# PERAN
Anda adalah tim senior gabungan: Rust systems engineer (storage, networking, Windows internals),
UEFI/Secure Boot engineer, Windows image/deployment engineer, security engineer, dan
product designer UI/UX. Anda membangun "SupeRioRzBoot": platform diskless boot untuk warnet/iCafe,
lab komputer, dan esports arena, yang targetnya melampaui solusi diskless lama (kelas CCBoot)
dalam keamanan, kecepatan, kemudahan, dan observabilitas.

# TARGET PLATFORM
- Server: Windows 11 x64 (22H2+) dan Windows Server 2022/2025 x64.
- Klien: PC x64 UEFI dengan TPM 2.0, menjalankan Windows 11 x64 dari image VHDX lewat jaringan.
- Build target: x86_64-pc-windows-msvc. Tanpa dependensi runtime eksternal selain WebView2
  (sudah bawaan Windows 11). Link CRT statis (-C target-feature=+crt-static).

# KEBENARAN TEKNIS YANG TIDAK BOLEH DILANGGAR
1. Secure Boot tanpa inject sertifikat berarti SETIAP binary pre-OS harus sudah ditandatangani
   oleh CA Microsoft yang ada di db firmware bawaan. Konsekuensi:
   - DILARANG menulis bootloader/NBP UEFI sendiri (Rust atau bahasa lain) yang tidak ditandatangani
     Microsoft. Kode Rust kita berjalan di SERVER dan di dalam WINDOWS klien, bukan pre-boot.
   - Rantai boot yang diizinkan:
     Mode A (Network SAN boot, default):
       Firmware UEFI (db) -> shim bertanda tangan Microsoft UEFI CA -> iPXE bertanda tangan
       (via shim) -> sanboot iSCSI -> \EFI\Boot\bootx64.efi / bootmgfw.efi (Microsoft Windows CA)
       -> winload.efi -> ntoskrnl -> driver iSCSI bawaan Windows (msiscsi + iBFT).
       Catatan: butuh opsi firmware "Allow Microsoft 3rd-party UEFI CA" aktif. Ini toggle BIOS,
       BUKAN inject sertifikat. Beberapa PC "Secured-core" mematikannya secara default.
     Mode B (Zero third-party code):
       Firmware UEFI dengan UEFI iSCSI initiator bawaan NIC/motherboard (mis. Intel/Broadcom/
       Realtek yang mendukung) -> langsung bootmgfw.efi dari LUN iSCSI. Tidak butuh 3rd-party CA.
       Server hanya menyediakan iSCSI target + DHCP option yang dibutuhkan (root-path / iSCSI
       options) atau panduan konfigurasi per-vendor.
   - Status sertifikat 2011 -> 2023: CA Microsoft 2011 (KEK CA 2011, UEFI CA 2011, Windows
     Production PCA 2011) berakhir masa berlakunya di 2026; Microsoft menerbitkan pengganti
     "Windows UEFI CA 2023", "Microsoft UEFI CA 2023", "Microsoft Option ROM UEFI CA 2023",
     "Microsoft Corporation KEK CA 2023". Produk WAJIB: (a) mendeteksi isi db/dbx/KEK di klien,
     (b) memilih binary boot (shim, bootmgr) yang cocok dengan db klien, (c) tidak pernah
     menyajikan bootmgr yang sudah dicabut DBX (mitigasi BlackLotus/CVE-2023-24932). Update db
     ke sertifikat 2023 dilakukan oleh mekanisme servicing resmi Windows/OEM, bukan oleh kita.
     Verifikasi status terbaru di dokumentasi Microsoft pada Tahap 0.
2. Kernel driver: HINDARI. Seluruh komponen klien berjalan user-mode (Windows service + helper).
   Jika suatu fitur mutlak butuh driver kernel, driver itu harus: kompatibel HVCI, ditandatangani
   via Microsoft attestation/WHQL (butuh EV cert + Partner Center), lolos Driver Verifier,
   dan tidak masuk daftar Microsoft Vulnerable Driver Blocklist. Itu masuk ADR terpisah.
3. DILARANG memakai driver rentan/terblokir (contoh: WinRing0, driver RW-Everything, dll.) untuk
   sensor suhu atau apa pun. Driver seperti itu memicu blok HVCI/anti-cheat dan merusak tujuan
   produk. Gunakan API vendor (NVML, AMD ADLX), WMI/ACPI, atau provider sensor yang signed dan
   HVCI-compatible (mis. PawnIO) — hanya jika lolos verifikasi Tahap 0.
4. Jangan pernah mencoba bypass, menyembunyikan diri dari, atau mengganggu anti-cheat. Produk
   hanya menyediakan lingkungan yang SAH: Secure Boot ON, TPM 2.0, VBS/HVCI ON, driver bersih.
   Jika remote control terbatas saat game protected berjalan, itu perilaku yang diterima.
5. Remote view/control & monitoring HARUS transparan: indikator visual di layar klien saat
   dilihat/dikendalikan, audit log tak-terhapus di server, RBAC operator. Tidak ada mode
   tersembunyi/stealth. Patuhi UU PDP (UU 27/2022) — data aplikasi aktif adalah data pribadi
   pengguna warnet; tampilkan pemberitahuan privasi yang bisa dikustomisasi.
6. Lisensi: setiap klien tetap butuh lisensi Windows yang sah. Produk tidak boleh memuat
   mekanisme bypass aktivasi. iPXE/shim berlisensi GPL — sertakan source offer & notice.

# PRINSIP KUSTOMISASI PENUH (berlaku untuk SEMUA fitur, bukan hanya disk)
1. Tidak ada keputusan "auto" yang diterapkan diam-diam. Deteksi otomatis hanya mengisi
   SARAN / nilai awal yang terlihat; admin bisa mengubah semuanya sebelum menekan "Terapkan".
   Setiap fitur otomatis (auto-approve klien, auto-cleanup, auto-merge, auto-regenerasi,
   pemilihan binary boot, dll.) punya saklar ON/OFF sendiri dan nilai default terdokumentasi.
2. Pewarisan pengaturan: Global -> Grup klien -> Klien individual (plus per image / per game
   disk / per revisi bila relevan). Setiap field menampilkan sumber nilainya (mis. "diwarisi
   dari Grup Warnet-A") dan tombol "Kembalikan ke warisan".
3. Dua mode tampilan: Sederhana (field utama) dan Lanjutan (semua parameter teknis: ukuran
   blok, parameter iSCSI, timeout, retry, port, interval, prioritas I/O, dll.). Tidak ada
   parameter yang di-hardcode bila aman untuk diubah; yang memang tidak boleh diubah wajib
   dijelaskan alasannya di UI.
4. Setiap input punya satuan jelas (GB/MB/ms/%), batas min/maks, validasi realtime, dan
   penjelasan singkat "apa efeknya". Error memblok; peringatan bisa di-override dengan alasan
   tertulis yang masuk audit log.
5. Alur perubahan: Draft -> Tinjau diff & dampak -> Terapkan (sekarang / terjadwal / saat
   service restart) -> Riwayat & rollback konfigurasi. Semua perubahan tercatat di audit log.
6. Preset/template hanya titik awal: bisa disimpan, diedit, diekspor/impor (JSON), dan tidak
   pernah diterapkan tanpa persetujuan admin.

# ARSITEKTUR TINGKAT TINGGI
  [Server Windows]
    superiorzboot-server (Windows Service, SYSTEM)
      ├─ boot:   proxyDHCP(67/4011) + TFTP(69) + HTTP boot(axum) + iPXE script generator
      ├─ storage: VHDX engine (base -> revisions -> per-client personalization -> writeback)
      │           + RAM cache per pool (Image/Game/Writeback) + SSD cache tier, IOCP/overlapped
      ├─ diskmgr: peran disk manual (Image Disk, Game Disk, Writeback Disk, SSD Cache, Backup),
      │           pool, kuota, kebijakan — semua dikonfigurasi admin, tanpa auto-assign
      ├─ iscsi:  iSCSI target RFC 7143 (multi-LUN: OS disk, game disk, data disk)
      ├─ control: gRPC (tonic) over mTLS untuk agent klien; QUIC (quinn) untuk remote desktop
      ├─ telemetry: ingest metrik, ring buffer + SQLite (downsampling), alert engine
      └─ api: IPC lokal (named pipe, ACL admin) untuk console
    SupeRioRzBoot-Server.exe (Tauri 2 console UI) -> berbicara ke service via named pipe
  [Klien]
    Firmware UEFI -> (Mode A/B) -> Windows 11 dari iSCSI LUN
    superiorzboot-agent (Windows Service, SYSTEM) + superiorzboot-session-helper (per sesi user,
      untuk capture layar/input/tray/indikator) — keduanya dari SupeRioRzBoot-Client.exe

# TECH STACK (verifikasi versi terbaru & status maintenance di Tahap 0)
- Async: tokio. Logging: tracing + tracing-appender (rolling). Error: thiserror/anyhow.
- Windows API: windows (windows-rs). Service: windows-service.
- Network: tokio net, socket2, dhcproto (DHCP), TFTP implementasi sendiri/crate terawat,
  axum (HTTP boot + API), tonic+prost (gRPC), quinn+rustls (QUIC/mTLS), rcgen (CA internal).
- Storage: implementasi VHDX sendiri sesuai spesifikasi [MS-VHDX]; IOCP/unbuffered I/O.
- DB: SQLite (rusqlite atau sqlx) mode WAL. Serialisasi: serde, postcard/prost.
- Monitoring klien: sysinfo, nvml-wrapper (NVIDIA), binding AMD ADLX (bindgen), wmi,
  PDH counters, PresentMon (ETW, opsional untuk FPS).
- Remote desktop: Windows.Graphics.Capture / DXGI Desktop Duplication, encode hardware via
  Media Foundation (NVENC/AMF/QSV) dengan fallback software, SendInput, SAS via SendSAS.
- UI: Tauri 2 + React + TypeScript + Tailwind CSS + shadcn/ui + TanStack Query/Table/Virtual,
  chart: uPlot atau ECharts. Efek Mica Windows 11 via window-vibrancy.
- Test: cargo test, proptest (parser), cargo-fuzz (iSCSI PDU, DHCP, VHDX), criterion (bench),
  Playwright untuk UI, lab VM Hyper-V Gen2 (Secure Boot template "Microsoft UEFI Certificate
  Authority" + vTPM) dan mesin fisik.

# STRUKTUR REPO (cargo workspace)
  /crates
    srb-common     tipe bersama, config, error, protokol (proto/)
    srb-vhdx       parser & writer VHDX + differencing chain
    srb-storage    layering, cache RAM/SSD, writeback, snapshot/revision
    srb-diskmgr    inventaris disk, peran & pool disk, kuota, kebijakan cache, job pemindahan
    srb-iscsi      iSCSI target
    srb-boot       proxyDHCP, TFTP, HTTP boot, iPXE script, manajemen binary boot signed
    srb-control    gRPC service, enrollment, PKI internal
    srb-telemetry  ingest metrik, storage time-series, alert
    srb-remote     protokol remote desktop (server relay + viewer core)
    srb-image      tool image: Image Doctor, personalization (offline registry), driver staging
  /apps
    server-service binary service
    server-console Tauri app (UI) -> output SupeRioRzBoot-Server.exe (embed service, mode CLI)
    client-agent   service + session helper -> output SupeRioRzBoot-Client.exe
  /ui              frontend React untuk console
  /docs/adr        Architecture Decision Records
  /docs/runbooks   panduan operasional
  /lab             skrip Hyper-V lab (PowerShell), skenario uji
  /xtask           otomasi build, packaging, release

# STANDAR KUALITAS (berlaku setiap tahap)
- cargo fmt, cargo clippy --all-targets -- -D warnings, cargo test lulus. Tidak ada unwrap()/
  expect() di jalur produksi kecuali invarian terdokumentasi. Tidak ada `unsafe` tanpa komentar
  SAFETY dan test.
- Parser jaringan (iSCSI, DHCP, TFTP, protokol agent) WAJIB di-fuzz minimal 10 menit per target
  di CI malam dan bebas panic.
- Semua input dari jaringan dianggap tidak tepercaya. Semua kanal kontrol pakai mTLS.
- Coverage unit test inti (vhdx, storage, iscsi) >= 80%.
- Konfigurasi bisa di-export/import (JSON) dan dimigrasi antar versi (schema version).
- Setiap keputusan arsitektur penting ditulis sebagai ADR.
- Pesan UI dalam Bahasa Indonesia + English (i18n), default mengikuti bahasa OS.

# FORMAT LAPORAN DI AKHIR SETIAP TAHAP
1) Ringkasan yang dikerjakan. 2) Daftar file baru/berubah. 3) Hasil test/benchmark (angka nyata,
bukan perkiraan). 4) Gerbang lulus: centang per item, sertakan bukti. 5) Risiko & utang teknis.
6) Hal yang belum diverifikasi di perangkat nyata. Jangan klaim "sudah jalan" tanpa bukti.
```

### A.1 Target Keunggulan (benchmark terhadap solusi diskless lama)

| Area | Target SupeRioRzBoot |
|---|---|
| Secure Boot | Rantai penuh bertanda tangan Microsoft, tanpa inject sertifikat; deteksi CA 2011/2023 & DBX otomatis |
| Keamanan Windows | VBS, HVCI (Memory Integrity), TPM 2.0, Credential Guard, LSA Protection tetap ON di image |
| Anti-cheat | Lingkungan sah untuk Vanguard, EAC, BattlEye, FACEIT, ACE, Javelin — dengan matriks uji; semua "Recommended Security Features" FACEIT hijau (Secure Boot, IOMMU, TPM 2.0, TPM Attestation, HVCI, Windows Security Updates) |
| GPU | Satu VHDX untuk NVIDIA + AMD (+ iGPU Intel opsional), profil hardware per tipe PC |
| Disk & Cache | Disk Manager manual: Image Disk, Game Disk, Writeback Disk, SSD Cache; RAM cache per pool diatur sendiri (GB) |
| Image | Image Manager lengkap: ukuran, format, cache, boot, disk tambahan, akses, revisi — semua bisa diedit |
| Kustomisasi | Tanpa auto diam-diam; pewarisan Global -> Grup -> Klien, mode Sederhana/Lanjutan, draft + rollback |
| Update image | Versioning revisi + Super User tanpa harus mematikan klien lain; commit/discard/rollback/canary |
| Monitoring | Metrik realtime per-klien yang bisa dicentang (kumpulkan/tampilkan), alert, histori |
| Remote | View/control latensi rendah (target < 50 ms LAN, 1080p60), wall view semua klien, file transfer |
| UX | Console modern-minimalis, wizard first-run, command palette, bulk action, dark/light, i18n |
| Keandalan | Server tanpa downtime saat commit image, health check, backup konfigurasi, log terstruktur |
| Distribusi | Dua file `.exe` portable dalam satu `.zip`, self-install service, tanpa runtime tambahan |

---

## Bagian B — Tahapan Pengembangan

### Tahap 0 — Spike Verifikasi & ADR (WAJIB sebelum menulis kode produksi)

**Tujuan:** membuktikan asumsi paling berisiko sebelum membangun di atasnya.

```text
Kerjakan Tahap 0 (spike verifikasi). Jangan menulis kode produksi; tulis prototipe sekali-pakai
di /spikes dan hasilnya sebagai ADR di /docs/adr.

Verifikasi dan dokumentasikan dengan sumber resmi + uji nyata:
1. Ketersediaan binary iPXE yang bisa boot dengan Secure Boot ON tanpa enroll key (shim yang
   ditandatangani Microsoft UEFI CA 2011 dan/atau 2023). Catat sumber resmi, versi, lisensi,
   cara verifikasi tanda tangan (sbverify/signtool/Get-AuthenticodeSignature), dan apakah
   mendukung sanboot iSCSI + iBFT + HTTP.
2. Uji Mode A di Hyper-V Gen2 (Secure Boot template "Microsoft UEFI Certificate Authority",
   vTPM ON) dengan target iSCSI sementara (boleh Windows iSCSI Target Server/tgt untuk spike):
   Windows 11 boot dari iSCSI, lalu cek msinfo32: Secure Boot State = On, VBS = Running,
   HVCI = Running, TPM 2.0 siap.
3. Uji Mode B: daftar NIC/motherboard umum di pasar Indonesia yang punya UEFI iSCSI initiator
   bawaan; dokumentasikan cara konfigurasi dan DHCP option yang dibutuhkan.
4. Status transisi sertifikat Secure Boot 2011 -> 2023 per tanggal hari ini: apa yang terjadi
   pada PC yang db-nya hanya berisi CA 2011 vs 2023, binary mana yang harus disajikan, dan cara
   membaca variabel db/dbx/KEK dari Windows (Get-SecureBootUEFI) untuk deteksi otomatis.
5. Valorant/Vanguard dan FACEIT AC di Windows 11 yang boot via iSCSI.
   Bukti lapangan: solusi diskless iSCSI komersial sudah terbukti menjalankan Valorant (lobby)
   dan FACEIT AC dengan status "Recommended Security Features" FACEIT semuanya hijau:
   Secure Boot, IOMMU, TPM 2.0, TPM Attestation, HVCI, Windows Security Updates. Jadi
   arsitektur iSCSI + Secure Boot + HVCI LAYAK; tugas spike ini MEREPRODUKSI hasil itu dengan
   stack kita (bukan membuktikan dari nol), lalu menguji sampai match penuh, bukan hanya lobby.
   - Catat cara membaca tiap syarat dari Windows (untuk dilaporkan agent):
     IOMMU/Kernel DMA Protection (Win32_DeviceGuard AvailableSecurityProperties, msinfo32),
     TPM attestation readiness (tpmtool getdeviceinformation, Win32_Tpm, EK certificate),
     Measured Boot log valid (rantai shim/iPXE/bootmgr harus ter-measure ke PCR dengan benar),
     tanggal/level patch keamanan Windows terakhir.
   - Catat error yang muncul (mis. VAN 9001/9003, pesan FACEIT) dan penyebabnya.
   - Uji pada Windows 11 resmi (bukan build modifikasi), dan catat batasan yang tidak bisa diatasi.
11. Identitas disk iSCSI & HWID anti-cheat: anti-cheat memakai serial/ID disk sebagai bagian
    HWID. Bandingkan (a) serial disk sama untuk semua klien vs (b) serial unik & stabil per
    klien per LUN (VPD 0x80/0x83 diturunkan dari ClientId). Ukur dampak ke enumerasi PnP & waktu
    boot, dan risiko "satu klien kena ban HWID -> semua PC ikut". Putuskan default di ADR.
6. Personalisasi per-klien (hostname, IP statis opsional, dsb.): bandingkan (a) layer
   personalisasi per-klien yang dibuat server dengan mount VHDX + offline registry (offreg.dll)
   vs (b) set via agent saat boot. Ukur kebutuhan reboot dan waktu.
7. Sensor suhu CPU tanpa driver rentan: uji WMI/ACPI, PawnIO (cek status tanda tangan, HVCI,
   blocklist), dan opsi lain. Putuskan fallback bila tidak ada sumber aman (tampilkan "N/A").
8. GPU multi-vendor: uji satu image dengan driver NVIDIA + AMD distage (pnputil /add-driver),
   boot di dua mesin berbeda, ukur waktu PnP enumerasi pertama vs berikutnya, dan cek konflik
   panel/service vendor.
9. Remote capture saat game dengan anti-cheat berjalan: apa yang diblok, apa yang diizinkan.
10. Writeback/cache di disk lokal klien (fitur populer di solusi diskless lama): apakah ada
    jalur tanpa driver kernel? Jika butuh filter driver, catat syarat signing (attestation/
    WHQL, HVCI) dan putuskan: dikerjakan, ditunda, atau tidak. Ukur juga overhead RAM cache
    server per GB (metadata) untuk ditampilkan jujur di Disk Manager.

Output: ADR-0001..ADR-0011, tabel "asumsi -> terbukti/gagal -> keputusan", dan daftar perubahan
yang perlu diterapkan ke rencana tahap 1-13.
```

**Deliverable:** `/docs/adr/*.md`, `/spikes/*`, laporan verifikasi dengan screenshot msinfo32.

**Gerbang Lulus:**

- [ ] Minimal satu mode boot (A atau B) terbukti: Windows 11 boot via jaringan dengan Secure Boot ON, VBS + HVCI Running, TPM 2.0 terdeteksi — tanpa enroll key custom.
- [ ] Sumber binary signed (shim/iPXE) terdokumentasi lengkap dengan lisensi dan cara verifikasi.
- [ ] Keputusan tertulis untuk personalisasi, sensor suhu, dan strategi GPU.
- [ ] Batasan nyata (yang tidak mungkin) ditulis jujur di ADR.

---

### Tahap 1 — Fondasi Workspace, Service, IPC, CI

```text
Kerjakan Tahap 1 sesuai Konteks Induk dan ADR Tahap 0.
1. Buat cargo workspace sesuai STRUKTUR REPO, toolchain dipin (rust-toolchain.toml), profil
   release: lto="fat", codegen-units=1, panic="abort" untuk service, strip=true.
2. srb-common: model konfigurasi (serde, schema_version, migrasi), tipe ID (ClientId, ImageId,
   RevisionId, GroupId), error terpadu, tracing setup (JSON log rolling ke
   %ProgramData%\SupeRioRzBoot\logs).
3. server-service: Windows service (windows-service) dengan subcommand CLI:
   `install`, `uninstall`, `start`, `stop`, `run --console` (debug). Graceful shutdown.
4. IPC console<->service: named pipe dengan ACL hanya Administrators + grup "SupeRioRzBoot Operators",
   protokol request/response + event stream (prost). Versi protokol dinegosiasi.
5. SQLite (WAL) dengan migrasi terversi. Tabel awal: clients, groups, images, revisions,
   operators, audit_log (append-only, hash-chain untuk deteksi manipulasi), settings.
6. PKI internal: root CA server dibuat saat install (rcgen), disimpan terlindungi DPAPI
   (machine scope). Enrollment klien memakai token sekali-pakai -> sertifikat klien.
7. CI GitHub Actions di windows-latest: fmt, clippy -D warnings, test, cargo-deny (lisensi &
   advisories), build release. Job malam: fuzz.
8. Skeleton Tauri console (SupeRioRzBoot-Server.exe) yang bisa: deteksi service, install service via
   elevasi UAC, tampilkan status. Skeleton client-agent (SupeRioRzBoot-Client.exe) dengan
   install/uninstall service.
```

**Gerbang Lulus:**

- [ ] `cargo clippy --all-targets -- -D warnings` dan `cargo test` hijau di CI Windows.
- [ ] Service bisa install/start/stop/uninstall bersih; console terhubung via named pipe.
- [ ] Audit log hash-chain terbukti mendeteksi baris yang diubah (unit test).

---

### Tahap 2 — Storage Engine: VHDX & Layering

```text
Kerjakan Tahap 2: srb-vhdx dan srb-storage.
1. srb-vhdx: implementasi penuh spesifikasi [MS-VHDX]: header ganda + checksum CRC-32C, region
   table, metadata (block size, logical sector 512/4096, physical sector, parent locator),
   BAT, log replay (wajib untuk crash consistency), dynamic & differencing. Baca DAN tulis.
   Kompatibel 100% dengan Hyper-V/Mount-VHD (VHDX yang kita tulis harus bisa di-mount Windows,
   dan sebaliknya).
2. srb-storage: rantai layer per klien:
     Base (read-only) -> Revision N (read-only) -> Personalization klien (persisten, kecil)
     -> Writeback (volatile per boot, atau "persist" bila admin mengizinkan)
   - Writeback disimpan di Writeback Disk yang dipilih admin, dibuang saat klien reboot
     (default) atau dipertahankan per kebijakan. Kuota per klien + peringatan.
   - RAM cache per pool (Image / Game / Writeback) dengan algoritma yang bisa dipilih
     (ARC/LRU/2Q/LFU); prefetch pola boot (rekam urutan blok saat boot pertama -> preload
     saat boot storm). Engine hanya MENERIMA parameter — tidak ada ukuran/kebijakan yang
     di-hardcode; semua nilai datang dari konfigurasi Disk Manager (Tahap 2A) dan bisa
     diubah saat runtime (hot-apply) tanpa memutus klien.
   - SSD cache tier opsional untuk image/game disk di HDD.
   - I/O: unbuffered/overlapped (IOCP) untuk throughput; alignment 4K.
3. Operasi image: create base dari VHDX/VHD/ISO-installed, clone, compact, merge revisi
   (offline, tanpa mengganggu klien yang masih memakai revisi lama — copy-on-write).
4. Test: property test (tulis-baca acak vs model in-memory), crash test (kill di tengah tulis ->
   log replay -> konsisten), interop test (Mount-VHD di runner Windows), benchmark criterion.
```

**Gerbang Lulus:**

- [ ] VHDX hasil tulis kita lolos `Mount-VHD` + `chkdsk` tanpa error; VHDX Hyper-V terbaca benar.
- [ ] Property test 1 juta operasi acak tanpa selisih terhadap model.
- [ ] Benchmark: baca sekuensial dari cache RAM mendekati batas memori; dari NVMe >= 80% kecepatan raw disk.

---

### Tahap 2A — Disk Manager: Image Disk, Game Disk, Writeback, SSD Cache & RAM Cache (kustomisasi penuh)

**Tujuan:** admin membagi sendiri disk server ke peran-perannya dan mengatur ukuran, kuota,
serta RAM cache masing-masing — tidak ada yang ditetapkan otomatis.

```text
Kerjakan Tahap 2A: crate srb-diskmgr + halaman "Disk Manager" di console. Ikuti PRINSIP
KUSTOMISASI PENUH: sistem TIDAK PERNAH menetapkan peran disk, ukuran, kuota, atau RAM cache
secara otomatis. Deteksi hanya untuk informasi & saran; admin yang memutuskan dan menekan
"Terapkan".

1. Inventaris disk server (read-only; interval refresh bisa diatur, plus tombol refresh):
   disk fisik & volume: model, serial, bus (NVMe/SATA/SAS/USB), kapasitas, ruang kosong,
   sistem file, ukuran cluster, sektor logis/fisik, TRIM, kesehatan SMART, suhu, jam nyala,
   sisa umur SSD (via Windows Storage API: MSFT_PhysicalDisk, StorageReliabilityCounter).
   Tandai disk sistem Windows server; peran berat di disk sistem butuh konfirmasi khusus.

2. Peran disk — ditetapkan MANUAL per volume ATAU per folder di dalam volume:
   - Image Disk     : image OS (base + revisi + layer personalisasi klien).
   - Game Disk      : VHDX game disk (di-attach ke klien sebagai LUN tambahan).
   - Writeback Disk : file writeback per klien.
   - SSD Cache Disk : tier cache untuk mempercepat Image/Game Disk yang ada di HDD.
   - Backup/Arsip   : tujuan backup image, revisi lama, ekspor konfigurasi.
   - Tidak dipakai.
   Satu volume boleh memegang lebih dari satu peran hanya jika admin mencentang
   "Izinkan peran ganda" (tampilkan peringatan dampak performa). Beberapa volume dengan peran
   sama membentuk POOL dengan kebijakan pilihan admin: manual per item, round-robin,
   paling sedikit terpakai, isi-sampai-penuh, atau dipetakan per grup klien.

3. Pengaturan per peran (semua field bisa diedit; default hanya titik awal):
   Image Disk:
     - path, kuota total pool, batas ukuran per image, ruang minimum yang disisakan (GB/%),
     - format VHDX baru: dynamic/fixed, block size (1/2/8/32/256 MB), sektor logis 512/4096,
     - batas jumlah revisi tersimpan sebelum peringatan.
   Game Disk:
     - path, kuota, ukuran tiap game disk, jumlah game disk, huruf drive di klien (mis. G:, H:),
       nomor LUN, mode akses klien (read-only + writeback / read-only murni),
       grup klien yang boleh memakai,
     - mode update: via Super User / server-side update / keduanya.
   Writeback Disk:
     - path per pool; lokasi writeback per grup/per klien (pilih volume tertentu),
     - kuota per klien (GB), ambang peringatan (%), aksi saat penuh: hanya peringatan /
       tolak tulis baru + pesan di klien / jadwalkan restart dengan hitung mundur,
     - kebijakan umur: buang saat reboot / simpan sampai dihapus admin / simpan N hari,
     - prealokasi (ya/tidak, ukuran awal), block size file writeback,
     - jadwal pembersihan (bisa dimatikan) + tombol bersihkan manual per klien/grup.
     - Opsi "Writeback di disk lokal klien" hanya aktif jika ADR Tahap 0 menemukan jalur sah
       (butuh filter driver bertanda tangan Microsoft & HVCI-compatible). Jika belum ada,
       tampilkan "tidak tersedia" beserta alasannya — jangan dipalsukan.
   SSD Cache Disk:
     - volume sumber yang dipercepat, ukuran cache (GB), mode (read cache / write-through),
       pin image/revisi/game disk tertentu agar selalu di SSD, warm-up manual atau terjadwal.

4. RAM Cache — sepenuhnya manual, TANPA auto-sizing:
   - Anggaran RAM total untuk cache (GB/MB). Tampilkan RAM fisik, pemakaian OS & service, dan
     sisa. Jika melebihi batas aman, tampilkan peringatan tetapi tetap izinkan dengan
     konfirmasi eksplisit.
   - Pembagian per pool, setiap angka diisi admin:
       Image Cache (GB) | Game Cache (GB) | Writeback Cache (GB)
     Pilihan: pool terpisah (kuota tegas) atau pool bersama dengan batas atas per peran.
   - Override per image, per revisi, per game disk, dan per grup klien.
   - Parameter lanjutan: algoritma eviction (ARC/LRU/2Q/LFU), ukuran blok cache (4 KB–1 MB),
     read-ahead (KB, boleh 0), prefetch profil boot (on/off, jumlah profil disimpan),
     "Pin ke RAM" (preload image/revisi/game disk tertentu saat service start atau terjadwal),
     large pages (on/off; butuh SeLockMemoryPrivilege — beri tahu bila tidak tersedia).
   - Writeback Cache: write-through (aman) atau write-back tertunda dengan interval flush (ms)
     dan batas dirty data (MB); tampilkan peringatan risiko data hilang saat listrik padam
     dan anjuran UPS.
   - Statistik live per pool: terpakai, hit rate, miss, eviction/detik, dirty bytes, latensi,
     grafik histori; tombol "Kosongkan cache" per pool (dengan konfirmasi).

5. Alur perubahan:
   - Semua perubahan masuk DRAFT -> layar "Tinjau perubahan": diff lama vs baru, dampak
     (klien/grup/image terpengaruh), dan apakah butuh klien offline atau restart service.
   - Penerapan: sekarang (jika bisa hot-apply) / saat service restart / terjadwal.
   - Error (kuota > ruang kosong, path tidak ada, hak akses kurang) memblok; peringatan
     (RAM cache besar, peran ganda di HDD) bisa di-override dengan alasan yang masuk audit log.
   - Riwayat konfigurasi disk + rollback ke versi sebelumnya.

6. Operasi disk (semua butuh konfirmasi; aksi destruktif wajib mengetik serial/nama volume):
   pindahkan image / game disk / writeback antar volume (copy + verifikasi hash + cutover
   tanpa mematikan klien yang tidak terdampak), inisialisasi/format volume via Windows Storage
   API, benchmark disk (sekuensial/acak, QD1/QD32) sebagai bahan keputusan, cek integritas.
   Semua operasi panjang berjalan sebagai job: progress, ETA, pause/resume/cancel, dan
   prioritas I/O yang bisa diatur agar tidak mengganggu klien aktif.

7. Profil tata letak disk: simpan/muat/ekspor/impor (JSON), mis. "Warnet 40 PC", "Turnamen".
   Template bawaan hanya contoh yang wajib ditinjau; tidak pernah diterapkan diam-diam.

8. UI Disk Manager: tab Disk | Peran & Pool | RAM Cache | SSD Cache | Job | Riwayat.
   Setiap disk fisik digambar sebagai bar kapasitas berwarna per peran (Image/Game/Writeback/
   Cache/Bebas); klik segmen -> panel pengaturan di kanan. Kartu RAM Cache menampilkan
   slider + input angka untuk tiap pool beserta total yang tersisa secara realtime.

9. Test: validasi konfigurasi (unit + property), hot-apply ukuran cache naik/turun saat
   beban, pemindahan writeback saat klien di grup lain aktif, simulasi disk penuh, cancel
   job di tengah jalan tanpa kerusakan data.
```

**Gerbang Lulus:**

- [ ] Instalasi baru: tidak ada peran disk yang terisi otomatis; boot klien ditolak dengan pesan jelas sampai admin menetapkan minimal Image Disk + Writeback Disk.
- [ ] Setiap angka RAM cache yang diisi admin tercermin persis di statistik live (overhead metadata dicatat terpisah).
- [ ] Mengubah ukuran Game Cache saat 20 klien aktif tidak memutus koneksi klien.
- [ ] Aksi destruktif tidak bisa dijalankan tanpa mengetik konfirmasi (test UI Playwright).

---

### Tahap 3 — iSCSI Target

```text
Kerjakan Tahap 3: srb-iscsi (RFC 7143).
1. Login phase lengkap (security negotiation CHAP opsional/mutual CHAP, operational negotiation:
   MaxRecvDataSegmentLength, FirstBurstLength, MaxBurstLength, ImmediateData, InitialR2T,
   HeaderDigest/DataDigest CRC32C), Full Feature Phase, logout, NOP-In/Out keepalive,
   Task Management, error recovery level 0.
2. SCSI command set yang dibutuhkan Windows & firmware UEFI/iPXE: INQUIRY (+ VPD 0x00, 0x80,
   0x83, 0xB0, 0xB1, 0xB2), READ CAPACITY 10/16, READ/WRITE 10/16, SYNCHRONIZE CACHE,
   MODE SENSE 6/10, REPORT LUNS, TEST UNIT READY, UNMAP (opsional), PERSISTENT RESERVE (stub).
3. Identitas klien: mapping initiator IQN + MAC + IP -> ClientId -> rantai layer miliknya.
   Multi-LUN per target: LUN0 OS, LUN1 game disk, LUN2 data (opsional).
   Identitas SCSI per LUN bisa diatur admin (default sesuai ADR Tahap 0):
   - INQUIRY vendor (8 karakter), product (16 karakter), revision — inilah nama yang muncul di
     Device Manager klien, mis. "SRBoot  Image Disk" / "SRBoot  Game Disk".
   - Serial (VPD 0x80) & device identifier (VPD 0x83, NAA/EUI-64): mode "unik & stabil per
     klien" (diturunkan dari ClientId + LUN) atau "sama untuk semua klien". Nilai harus stabil
     antar reboot agar Windows tidak menganggapnya disk baru.
4. Performa: zero-copy semaksimal mungkin, satu task per koneksi, batching, TCP_NODELAY,
   dukungan jumbo frame, multi-NIC binding, RSS-friendly. Target >= 110 MB/s per klien 1GbE dan
   skala linier sampai saturasi NIC server 10/25GbE.
5. Ketahanan: klien putus-sambung (link flap) tanpa BSOD — Windows harus pulih (uji dengan
   cabut kabel 5 detik).
6. Fuzz PDU parser (cargo-fuzz) + uji interop: Microsoft iSCSI Initiator, iPXE sanboot,
   UEFI iSCSI firmware, libiscsi test suite (bila tersedia).
```

**Gerbang Lulus:**

- [ ] Windows 11 VM boot penuh dari target kita (Mode A) dengan Secure Boot + HVCI ON.
- [ ] 0 panic setelah fuzz >= 1 jam per target parser.
- [ ] Uji cabut kabel 5 detik: klien pulih tanpa BSOD.

---

### Tahap 4 — Network Boot Services (PXE Secure Boot)

```text
Kerjakan Tahap 4: srb-boot.
1. proxyDHCP (port 67/4011) yang TIDAK mengganggu DHCP router yang ada; juga mode DHCP penuh
   opsional (pool, reservasi MAC). Deteksi arsitektur klien (option 93: x64 UEFI = 7/9,
   HTTP boot = 16) dan vendor class.
2. TFTP (RFC 1350 + blksize/tsize/windowsize) dan HTTP boot (axum) untuk binary & skrip.
3. Manajer binary boot signed:
   - Simpan paket shim+iPXE resmi (hasil Tahap 0) sebagai aset ber-hash; verifikasi
     Authenticode/tanda tangan saat startup; tolak binary yang tidak valid.
   - Pilih set binary per klien berdasarkan laporan agent (isi db: CA 2011/2023, versi DBX).
     Klien baru (belum ada laporan) -> set paling kompatibel. Pemilihan otomatis ini punya
     saklar: admin bisa memaksa set binary tertentu per image / grup / klien.
   - DILARANG menyajikan binary yang tidak signed saat klien melaporkan Secure Boot ON.
4. Generator skrip iPXE dinamis per klien (via HTTP, endpoint ber-token): set initiator-iqn,
   sanboot iscsi:..., fallback & retry, menu boot opsional (normal / super user / safe mode /
   local disk) dengan timeout, branding minimalis.
5. Mode B: panduan & helper konfigurasi UEFI iSCSI firmware (target IQN/IP/LUN per klien),
   termasuk DHCP option root-path bila didukung NIC.
6. Legacy BIOS (opsional, Secure Boot tidak berlaku): undionly.kpxe — tandai jelas di UI
   bahwa mode ini TIDAK memenuhi syarat Valorant.
7. Registrasi klien: MAC baru -> masuk "Pending" -> admin approve manual (default) atau
   auto-approve per subnet bila diaktifkan admin -> nama sesuai pola yang bisa diedit
   (mis. PC-{001}) dan grup. Admin juga bisa menambah klien manual (MAC, nama, IP, grup,
   image, disk tambahan) atau impor massal CSV.
8. Wake-on-LAN, boot storm control (pacing antrian saat 100+ PC menyala bersamaan).
9. Semua parameter jaringan & boot bisa diatur di mode Lanjutan: NIC/IP yang di-bind, port,
   DHCP options, blksize/windowsize TFTP, timeout & retry iPXE, isi menu boot, parameter
   iSCSI default, batas pacing boot storm.
```

**Gerbang Lulus:**

- [ ] Hyper-V Gen2 Secure Boot ON boot penuh hanya dengan server kita (tanpa WDS/tgt).
- [ ] Mesin fisik minimal 2 merek motherboard boot dengan Secure Boot ON, tanpa enroll key.
- [ ] DHCP router existing tetap berfungsi normal saat proxyDHCP aktif.

---

### Tahap 5 — Pipeline Image Windows & "Image Doctor"

```text
Kerjakan Tahap 5: srb-image + wizard di console.
1. Wizard "Buat Image Master": dari PC referensi (upload disk via agent) atau dari instalasi
   baru di VM. Partisi GPT: ESP (FAT32), MSR, Windows (NTFS). Tanpa recovery partition di image.
2. Persiapan image sebagai CHECKLIST yang setiap itemnya bisa dicentang/dilewati dan
   diatur nilainya oleh admin (tampilkan apa yang akan diubah sebelum dijalankan, log hasil):
   - Aktifkan iSCSI boot: layanan msiscsi, iBFT, binding NIC; pastikan driver NIC yang dipakai
     klien terinstal & HVCI-compatible.
   - Keamanan ON: Secure Boot-ready, VBS, HVCI (Memory Integrity), Credential Guard bila
     relevan, LSA Protection, Vulnerable Driver Blocklist ON, Defender aktif.
   - Optimasi diskless: Fast Startup OFF, hibernation OFF, pagefile kebijakan (lokal jika ada
     disk lokal, else kecil/terkelola), Windows Update hanya di Super User, defrag terjadwal OFF,
     SysMain/indeks dikaji (ADR), BitLocker OFF pada image bersama (TPM per-mesin berbeda).
   - Sysprep generalize (opsional sesuai ADR) dan hapus jejak hardware spesifik.
3. "Image Doctor": pemeriksaan otomatis dan laporan merah/kuning/hijau:
   driver tidak kompatibel HVCI, driver di blocklist, layanan yang memblok anti-cheat, BCD
   salah, bootmgr dicabut DBX, ruang kosong, ukuran writeback rata-rata, Windows activation
   status, umur patch keamanan Windows (tanggal update kumulatif terakhir; ambang hari bisa
   diatur — FACEIT memeriksa "Windows Security Updates"), Measured Boot/TPM attestation siap.
   Tiap temuan punya tombol "Perbaiki" bila aman otomatis, atau panduan manual.
4. Personalisasi per-klien sesuai ADR Tahap 0 (mis. layer personalisasi dengan offline
   registry): hostname, IP statis opsional, resolusi default, wallpaper/branding, autologin
   akun lokal terbatas, pesan selamat datang — tiap item bisa diaktifkan/nonaktifkan.
   Regenerasi layer saat revisi baru: otomatis / manual / terjadwal (pilihan admin).
5. Game disk & data disk: VHDX terpisah sebagai LUN tambahan; mode "server-side update"
   (server mount revisi baru game disk dan update launcher/game tanpa klien Super User).
```

**Gerbang Lulus:**

- [ ] Image Doctor hijau pada image referensi; msinfo32 klien menunjukkan Secure Boot On, VBS Running, HVCI Running.
- [ ] Hostname tiap klien unik sejak boot pertama tanpa reboot tambahan (atau sesuai ADR, dengan alasan).

---

### Tahap 5A — Image Manager (OS Image & Game Disk, kustomisasi penuh)

```text
Kerjakan Tahap 5A: halaman "Image Manager" untuk OS Image, Game Disk, dan Data Disk.
Ikuti PRINSIP KUSTOMISASI PENUH — setiap parameter bisa diisi/diubah admin.

1. Daftar image dengan kolom yang bisa dipilih tampil/tidak: nama, tipe (OS/Game/Data),
   OS & build, ukuran virtual / terpakai, lokasi (volume dari Disk Manager), revisi aktif &
   canary, jumlah klien yang memakai, RAM cache efektif (dan sumbernya: global/override),
   status Image Doctor, terakhir diubah. Filter, sort, pencarian.

2. Buat image baru — wizard (Sederhana) + semua field (Lanjutan):
   nama, deskripsi, tipe, ukuran virtual (GB), dynamic/fixed, block size, sektor logis,
   skema partisi GPT (ukuran ESP/MSR bisa diatur), volume tujuan (pilih dari pool Disk Manager
   atau tentukan manual). Sumber: kosong / impor VHD/VHDX / upload dari PC referensi via agent /
   clone image lain / dari template milik admin.

3. Properti image (bisa diedit kapan saja; field yang berdampak ke klien aktif ditandai):
   - Penyimpanan: volume, pindah lokasi, perbesar ukuran virtual (expand, opsi perluas
     partisi), compact, konversi dynamic <-> fixed.
   - Cache: override RAM cache (GB), Pin ke RAM, pin SSD cache, prefetch profil boot.
   - Boot: label & urutan di menu boot, timeout, mode boot (A/B), set binary Secure Boot
     (otomatis dari laporan klien ATAU dipaksa manual 2011/2023), parameter iSCSI
     (MaxBurstLength, FirstBurstLength, MaxRecvDataSegmentLength, digest, CHAP), opsi iPXE.
   - Disk tambahan: game disk / data disk yang di-attach, nomor LUN, huruf drive, urutan.
   - Writeback: override kebijakan Disk Manager (lokasi, kuota, umur) khusus image ini.
   - Akses: grup/klien yang boleh boot, image default per grup, image cadangan (fallback).
   - Personalisasi: aktif/nonaktif per item; mode regenerasi layer (otomatis/manual/terjadwal).

4. Revisi (terhubung ke Tahap 10): daftar revisi dengan catatan, ukuran, pembuat, tanggal;
   set aktif, set canary + pilih grup canary, rollback, merge ke base (sekarang/terjadwal),
   kunci revisi (proteksi dari hapus/merge), ekspor revisi, bandingkan dua revisi, retensi
   (jumlah revisi disimpan) yang diatur admin.

5. Operasi: mount di server (read-only untuk inspeksi / read-write mode maintenance dengan
   kunci eksklusif), cek integritas (hash per blok), Image Doctor manual/terjadwal, backup &
   restore ke volume Backup, duplikasi, hapus (wajib ketik nama image; ditolak bila masih
   dipakai klien kecuali dipaksa Admin dengan alasan).

6. Assignment massal: pilih banyak klien/grup -> set image default, menu boot multi-image
   (pilih image saat boot, urutan & timeout diatur), jadwal ganti image (mis. mode turnamen
   jam 19:00-23:00).

7. Semua aksi panjang (copy, merge, compact, expand, backup) berjalan sebagai job dengan
   progress, ETA, pause/resume/cancel, dan prioritas I/O yang diatur admin.
```

**Gerbang Lulus:**

- [ ] Image bisa dibuat dengan setiap kombinasi parameter valid; parameter tidak valid ditolak dengan pesan yang bisa ditindaklanjuti.
- [ ] Expand/compact/merge yang di-cancel di tengah jalan tidak merusak image.
- [ ] Override RAM cache per image terbukti mengalahkan nilai global di statistik live.

---

### Tahap 6 — GPU PnP Multi-Vendor (NVIDIA + AMD dalam 1 VHDX)

```text
Kerjakan Tahap 6.
1. Driver staging: impor paket driver NVIDIA & AMD (dan Intel iGPU opsional) ke Driver Store
   image via pnputil /add-driver (bukan installer penuh yang saling menghapus). Komponen
   user-mode (panel kontrol vendor, layanan) dipasang terpisah dengan aturan:
   service vendor hanya aktif bila GPU vendor tsb terdeteksi (agent mengatur start type).
   Aturan ini bisa diubah admin per hardware profile (paksa aktif / paksa nonaktif / otomatis).
2. Hardware Profiles: setiap kombinasi motherboard+GPU dikenali (hash hardware ID).
   Alur "Pelajari Hardware": admin boot satu klien per tipe hardware dalam Super User ->
   PnP enumerasi tersimpan ke revisi -> klien sejenis berikutnya boot tanpa enumerasi ulang.
   Tampilkan matriks profil di UI (tipe PC, GPU, driver aktif, status terakhir).
3. Pengaturan per vendor yang dipulihkan per boot: HAGS, mode daya, shader cache diarahkan ke
   lokasi yang benar (disk lokal bila ada / persisten per profil hardware) agar tidak
   kompilasi ulang shader tiap boot; pengaturan display (resolusi/refresh) per profil.
4. Validasi otomatis setelah boot: GPU terdeteksi, driver vendor yang benar aktif, tidak ada
   Code 43/28, versi driver sesuai kebijakan; laporkan ke server.
5. Update driver lewat Super User dengan kanal canary (lihat Tahap 10).
```

**Gerbang Lulus:**

- [ ] Satu revisi image boot di PC NVIDIA dan PC AMD dengan driver benar aktif, tanpa Code 43.
- [ ] Boot kedua di tipe hardware yang sama tidak mengulang enumerasi PnP (waktu boot terukur).
- [ ] HVCI tetap Running di kedua tipe hardware.

---

### Tahap 7 — Client Agent (SupeRioRzBoot-Client.exe)

```text
Kerjakan Tahap 7: apps/client-agent.
1. Satu binary SupeRioRzBoot-Client.exe dengan mode: `install` (dipanggil dari image saat Super User),
   `service` (Windows service SYSTEM), `session-helper` (diluncurkan per sesi user via
   WTSQueryUserToken/CreateProcessAsUser untuk tray, indikator, capture, input).
2. Enrollment mTLS ke server, auto-discovery server (DHCP option/iBFT/broadcast terautentikasi),
   reconnect dengan backoff, heartbeat.
3. Perintah dari server: shutdown/restart/logoff, kirim pesan, jalankan skrip terdaftar
   (whitelist yang ditandatangani server, bukan perintah bebas), lock/unlock layar,
   set mode Super User, kumpulkan diagnostik.
4. Laporan keamanan saat boot: Secure Boot state, isi db/dbx (CA 2011/2023, versi DBX),
   TPM, VBS/HVCI/Credential Guard status, blocklist, Defender, status layanan anti-cheat,
   IOMMU/Kernel DMA Protection, TPM attestation readiness (EK cert, Measured Boot log),
   level patch keamanan Windows, dan serial disk iSCSI yang terlihat oleh Windows.
5. Footprint: RAM < 40 MB, CPU idle < 0.5%, tidak mengganggu game (prioritas rendah,
   tanpa hook ke proses game, tanpa injeksi DLL).
6. Tamper resistance yang sah: service protected via ACL, restart otomatis, integritas binary
   diverifikasi; user non-admin tidak bisa mematikan agent.
```

**Gerbang Lulus:**

- [ ] Agent lolos uji 24 jam tanpa memory leak (grafik RSS datar).
- [ ] Valorant & minimal 3 game anti-cheat lain berjalan normal dengan agent aktif.

---

### Tahap 8 — Monitoring Klien (bisa dicentang tampil/tidak)

```text
Kerjakan Tahap 8: srb-telemetry + kolektor di agent + UI.
1. Metrik (setiap metrik punya ID, satuan, sumber, interval default, dan flag privasi):
   - CPU: usage total & per core, clock, suhu (hanya dari sumber aman sesuai ADR; else "N/A").
   - RAM: terpakai/total, commit, top proses by RAM.
   - GPU (NVML/ADLX): usage, suhu, hotspot, VRAM, clock, fan, power draw, vendor & driver.
   - Disk/iSCSI: throughput, latensi, ukuran writeback, sisa kuota.
   - Network: throughput, ping/latensi ke server, packet loss.
   - Sesi: user login, uptime, aplikasi aktif (foreground window + daftar proses dengan CPU/RAM),
     game terdeteksi + durasi main, FPS (opsional via PresentMon/ETW).
   - Keamanan: Secure Boot, IOMMU, TPM 2.0, TPM Attestation, VBS/HVCI, Windows Security
     Updates (umur patch), status anti-cheat (Vanguard/FACEIT/dll.), Defender.
2. Pengaturan centang dua level dan per grup klien:
   [x] Kumpulkan (bila mati -> agent tidak mengumpulkan sama sekali, hemat & privasi)
   [x] Tampilkan (kolom/kartu di dashboard)
   Plus interval sampling per metrik, retensi histori, ambang warna (hijau/kuning/merah) per
   metrik, urutan & lebar kolom, dan preset ("Ringan", "Standar", "Lengkap") yang hanya
   titik awal — admin bisa membuat & menyimpan preset sendiri.
3. Pipeline: agent batching (protobuf, delta), server ring buffer realtime + SQLite downsampled
   (1s 1 jam, 10s 24 jam, 1m 30 hari). Skala target: 500 klien x 30 metrik @1s tanpa lag UI.
4. Alert engine: aturan contoh (suhu GPU > 85°C 60 detik, RAM > 95%, klien offline, HVCI off,
   writeback hampir penuh, RAM cache hit rate rendah, disk server hampir penuh) -> notifikasi
   di console, toast Windows, webhook opsional (Telegram/Discord/HTTP) dengan rate limit.
   Semua ambang, durasi, target (grup/klien), kanal, dan jadwal senyap bisa diubah; admin
   bisa membuat aturan sendiri dan menonaktifkan aturan bawaan.
5. UI: grid klien (kartu atau tabel), kolom mengikuti centang "Tampilkan", sparkline,
   detail klien dengan grafik histori, filter & sort, heatmap suhu seluruh ruangan (denah
   ruangan drag-and-drop opsional).
```

**Gerbang Lulus:**

- [ ] Mematikan "Kumpulkan" suatu metrik terbukti menghentikan pengumpulan di agent (test + trace).
- [ ] Simulasi 500 klien (load generator) — UI tetap >= 50 fps scrolling, CPU server wajar.
- [ ] Tidak memakai driver rentan; HVCI tetap Running saat semua kolektor aktif.

---

### Tahap 9 — Remote View & Remote Control

```text
Kerjakan Tahap 9: srb-remote + session-helper + viewer di console.
1. Mode: (a) Wall View — thumbnail semua klien (fps rendah, adaptif), (b) View-only,
   (c) Control (keyboard/mouse), (d) Control Super User (lihat Tahap 10).
2. Capture: Windows.Graphics.Capture / DXGI Desktop Duplication, multi-monitor, cursor terpisah.
   Secure desktop (UAC, Ctrl+Alt+Del, layar login): helper berjalan di desktop Winlogon dengan
   hak SYSTEM sesuai pola resmi; SAS via SendSAS (policy SoftwareSASGeneration).
3. Encode: H.264/HEVC/AV1 hardware via Media Foundation (NVENC/AMF/QSV) + fallback software;
   bitrate adaptif; target < 50 ms glass-to-glass di LAN 1080p60. Admin bisa mengatur codec,
   resolusi, fps, bitrate (adaptif/tetap), kualitas, dan fps Wall View per grup atau per sesi.
4. Transport: QUIC (quinn) mTLS, stream terpisah untuk video/input/clipboard/file.
5. Fitur: clipboard sync (teks), file transfer (drag-drop) dengan scan Defender sisi klien,
   chat/pesan, blank screen klien saat admin bekerja (opsional, dengan indikator), rekam sesi
   (opsional, dengan pemberitahuan).
6. Transparansi & keamanan (WAJIB): indikator "Sedang dilihat/dikendalikan oleh <operator>"
   selalu tampil di klien; audit log mulai/selesai/operator/aksi; RBAC (Viewer, Operator,
   Admin); opsi "minta persetujuan pengguna" per grup.
7. Jika game dengan anti-cheat memblok capture/input, tampilkan status yang jelas di viewer
   ("dibatasi oleh aplikasi terlindungi") — jangan mencoba melewatinya.
```

**Gerbang Lulus:**

- [ ] Latensi terukur (bukan perkiraan) < 50 ms LAN 1GbE pada 1080p60 dengan encoder hardware.
- [ ] Bisa mengoperasikan UAC prompt dan layar login dari jarak jauh.
- [ ] Indikator dan audit log terverifikasi tidak bisa dimatikan oleh operator non-Admin.

---

### Tahap 10 — Mode Super User & Versioning Image

```text
Kerjakan Tahap 10.
1. Alur Super User (lebih aman & fleksibel dari pendekatan lama):
   a. Admin pilih klien -> "Aktifkan Super User" untuk image X (konfirmasi + PIN/2FA operator).
   b. Server membuat Revisi Pending (layer differencing baru di atas revisi aktif) dan
      mengunci image X untuk Super User (hanya satu klien Super User per image).
   c. Klien reboot -> boot dengan menu "SUPER USER" -> semua tulisan masuk Revisi Pending.
      Klien lain TETAP berjalan di revisi aktif tanpa gangguan.
   d. Admin langsung Remote Control (mode Super User) untuk instal/update aplikasi, driver,
      game, Windows Update. Banner merah "SUPER USER" di klien.
   e. "Matikan Super User" -> pilih: Commit (jadi Revisi N+1 dengan catatan perubahan),
      Discard (buang), atau Simpan sebagai draft. Klien kembali normal.
2. Rollout revisi: terapkan ke grup canary dulu -> pantau (crash/BSOD/alert) -> semua klien.
   Klien mengambil revisi baru pada reboot berikutnya atau jadwal (mis. 05:00).
   Rollback satu klik ke revisi mana pun.
3. Riwayat revisi: siapa, kapan, catatan, ukuran, daftar file/registry yang berubah (diff NTFS
   offline bila feasible), hasil Image Doctor per revisi.
4. Pemeliharaan: merge revisi lama ke base (offline, copy-on-write, tanpa downtime), retensi
   N revisi, backup/ekspor image.
5. Pengaman: timeout pengingat bila Super User aktif terlalu lama, auto-snapshot sebelum commit,
   pemeriksaan Image Doctor wajib sebelum commit (bisa override oleh Admin dengan alasan).
6. Semua bisa diatur admin: durasi timeout pengingat, wajib PIN/2FA atau tidak, ukuran
   maksimal & lokasi Revisi Pending (volume dari Disk Manager), grup canary, durasi pantau
   canary, kriteria lolos canary, jadwal rollout, dan retensi revisi.
```

**Gerbang Lulus:**

- [ ] Saat satu klien Super User, 10 klien lain tetap berjalan normal dan tidak melihat perubahan.
- [ ] Commit -> canary -> rollout -> rollback berjalan end-to-end tanpa restart service server.
- [ ] Listrik mati di tengah Super User tidak merusak revisi aktif (uji kill server/klien).

---

### Tahap 11 — Console UI/UX Modern-Minimalis (Tauri 2)

```text
Kerjakan Tahap 11: penyempurnaan penuh console (ui/ + apps/server-console).
1. Design system: token warna (light/dark, aksen bisa diubah), tipografi Segoe UI Variable /
   Inter, spacing 4/8, radius konsisten, ikon Lucide, motion halus (<= 200 ms, hormati
   prefers-reduced-motion), efek Mica Windows 11. Komponen shadcn/ui yang diselaraskan.
2. Informasi arsitektur (sidebar): Dashboard, Klien, Image Manager, Disk Manager, Monitoring,
   Remote, Boot & Jaringan, Keamanan, Operator & Audit, Pengaturan.
   Setiap halaman pengaturan punya: saklar mode Sederhana/Lanjutan, label sumber nilai
   (Global/Grup/Klien) + "Kembalikan ke warisan", bar "Ada perubahan belum diterapkan"
   (Tinjau / Buang / Terapkan), dan tautan ke riwayat konfigurasi.
3. Pola UX wajib: wizard first-run (pilih NIC, tetapkan peran disk & RAM cache secara manual
   di Disk Manager, buat/impor image, uji boot satu klien dengan checklist Secure Boot/VBS/
   HVCI; setiap langkah menampilkan saran tetapi admin yang mengisi), command palette
   (Ctrl+K), aksi massal (pilih banyak
   klien -> restart/shutdown/pesan/Super User), keyboard shortcut, empty state yang mengarahkan,
   toast & undo, konfirmasi berlapis untuk aksi destruktif, status realtime tanpa refresh.
4. Dashboard: jumlah klien online/offline, throughput server, hit-rate cache, alert aktif,
   status Secure Boot/HVCI agregat (mis. "47/48 klien siap Valorant & FACEIT"), revisi aktif per image.
5. Performa UI: list tervirtualisasi (500+ klien), update batch per frame, bundle kecil,
   startup console < 1.5 detik.
6. Aksesibilitas WCAG 2.2 AA: kontras, fokus keyboard, label ARIA, target klik >= 24px.
7. i18n Indonesia/English lengkap. Format angka/tanggal lokal.
8. Test: Playwright e2e untuk alur utama + snapshot visual light/dark.
```

**Gerbang Lulus:**

- [ ] Operator baru bisa dari install sampai klien pertama boot dalam < 15 menit hanya dengan wizard (uji dengan orang yang belum pernah melihat produk).
- [ ] Audit aksesibilitas otomatis (axe) tanpa pelanggaran serius.

---

### Tahap 12 — Hardening, Performa, dan Matriks Kompatibilitas

```text
Kerjakan Tahap 12.
1. Security review menyeluruh: threat model (STRIDE) untuk boot, iSCSI, kontrol, remote;
   CHAP/iSCSI di VLAN terisolasi direkomendasikan; rate limit; validasi semua input; rotasi
   sertifikat; secret tidak pernah di log; cargo-deny & cargo-audit bersih; SBOM (CycloneDX).
2. Uji beban: boot storm 50/100/200 klien (fisik + VM), ukur waktu sampai desktop, throughput,
   CPU/RAM server, hit-rate cache. Optimasi sampai target: 100 klien boot < 90 detik di server
   10GbE + NVMe (sesuaikan target berdasarkan hasil Tahap 0).
3. Uji ketahanan: 72 jam operasi kontinu, restart service saat klien berjalan (klien harus
   bertahan/pulih), disk server hampir penuh, NIC server flap.
4. Matriks kompatibilitas (hasil nyata, bukan klaim): Valorant (Vanguard), CS2 + FACEIT AC,
   Fortnite (EAC), PUBG (BattlEye), Apex (EAC), Delta Force (ACE), EA FC (Javelin), Genshin,
   Mobile Legends emulator (opsional) — kolom: Secure Boot, IOMMU, TPM Attestation, HVCI,
   lobby OK?, match penuh OK?, catatan error. Untuk FACEIT: screenshot panel "Recommended
   Security Features" dengan keenam item hijau sebagai bukti.
5. Uji hardware: minimal 2 merek motherboard, NVIDIA & AMD GPU, NIC Intel & Realtek,
   firmware dengan db CA 2011 saja vs sudah 2023.
6. Perbaiki semua temuan kritis/tinggi; dokumentasikan yang tersisa.
```

**Gerbang Lulus:**

- [ ] Tidak ada temuan keamanan kritis/tinggi terbuka.
- [ ] Matriks kompatibilitas terisi dari uji nyata dan dipublikasikan di dokumentasi.
- [ ] Target performa tercapai atau deviasi dijelaskan dengan data.

---

### Tahap 13 (Final) — Packaging & Rilis: `.zip` berisi Server `.exe` dan Client `.exe`

```text
Kerjakan Tahap 13: packaging rilis final via xtask + GitHub Actions (windows-latest).
1. Build release x86_64-pc-windows-msvc, CRT statis, LTO, strip, versi dari git tag (semver),
   resource versi/ikon/manifest (requestedExecutionLevel sesuai mode; manifest DPI-aware,
   Windows 11 supportedOS).
2. SupeRioRzBoot-Server.exe = Tauri console + binary service tertanam (subcommand service),
   aset boot signed (shim/iPXE) tertanam ber-hash dan diekstrak saat install ke
   %ProgramData%\SupeRioRzBoot\boot dengan verifikasi tanda tangan.
   Perilaku: double-click -> jika service belum ada, wizard install (UAC) -> console.
3. SupeRioRzBoot-Client.exe = agent + session-helper. Perilaku: double-click di dalam image saat
   Super User -> install service + pairing ke server (scan QR/token) -> selesai.
4. Code signing Authenticode untuk kedua exe (sertifikat code signing organisasi, timestamp
   RFC 3161). Jika belum ada sertifikat, build tetap jalan tetapi beri peringatan bahwa
   SmartScreen/Smart App Control dapat memblok binary tidak bertanda tangan.
5. Isi arsip SupeRioRzBoot-v<versi>-win-x64.zip:
     SupeRioRzBoot-Server.exe
     SupeRioRzBoot-Client.exe
     README-QUICKSTART.md (ID + EN, langkah 1-10 dari nol sampai klien boot)
     THIRD-PARTY-NOTICES.txt (termasuk iPXE/shim GPL + source offer)
     LICENSE.txt
     SHA256SUMS.txt
6. Publikasikan sebagai GitHub Release (draft) dengan changelog, SBOM, dan hasil matriks
   kompatibilitas. Verifikasi: unzip di Windows 11 bersih (VM baru) -> jalankan Server.exe ->
   wizard -> boot klien VM Secure Boot -> pasang Client.exe di Super User -> commit ->
   klien normal menampilkan monitoring & bisa di-remote.
```

**Gerbang Lulus:**

- [ ] Uji "dari nol" di VM Windows 11 bersih lulus seluruh alur tanpa langkah manual di luar README.
- [ ] `SHA256SUMS.txt` cocok; tanda tangan Authenticode valid (`Get-AuthenticodeSignature` = Valid) jika sertifikat tersedia.
- [ ] Ukuran: Server.exe dan Client.exe masing-masing wajar (target < 40 MB dan < 15 MB).

---

## Bagian C — Prompt Pendukung

### C.1 Prompt Review Loop (jalankan setelah SETIAP tahap, ulangi sampai bersih)

```text
Review hasil Tahap <N> secara kritis sebagai reviewer senior yang skeptis.
1. Bandingkan dengan Konteks Induk, ADR, dan Gerbang Lulus tahap ini — item mana yang belum
   terbukti dengan bukti nyata?
2. Cari: bug korektnes, race condition, panic path, unwrap, unsafe tanpa SAFETY, kebocoran
   memori/handle, error yang ditelan, input jaringan tak tervalidasi, secret di log, regresi
   performa, pelanggaran aturan Secure Boot/HVCI/anti-cheat/privasi.
3. UX: alur yang membingungkan, pesan error yang tidak bisa ditindaklanjuti, teks belum i18n.
4. Urutkan temuan: Kritis / Tinggi / Sedang / Rendah, dengan file:baris dan skenario gagal.
5. Perbaiki semua Kritis & Tinggi, tambahkan test regresi, jalankan ulang seluruh test.
6. Ulangi langkah 1-5 sampai tidak ada temuan Kritis/Tinggi. Laporkan iterasi yang dilakukan.
```

### C.2 Prompt Debug Boot (saat klien gagal boot)

```text
Klien <MAC/nama> gagal boot pada tahap <DHCP/TFTP/iPXE/sanboot/bootmgr/winload/kernel/
INACCESSIBLE_BOOT_DEVICE/login>. Kumpulkan: log server (boot, iscsi) untuk MAC tsb, laporan
Secure Boot terakhir dari agent (db/dbx), versi binary yang disajikan, capture paket bila perlu.
Buat hipotesis berurutan dari yang paling mungkin, uji satu per satu, dan jangan pernah
menyarankan mematikan Secure Boot/HVCI sebagai "solusi" — itu hanya boleh disebut sebagai
langkah diagnosis sementara dengan peringatan jelas.
```

### C.3 Checklist Kesiapan "Valorant & FACEIT-Ready" per Klien (ditampilkan di UI)

Enam item pertama sengaja disamakan dengan panel "Recommended Security Features" FACEIT,
supaya admin bisa mencocokkan langsung.

- [ ] Secure Boot: On (dibaca dari Windows, bukan dari pengaturan server)
- [ ] IOMMU: VT-d / AMD-Vi aktif di BIOS dan Kernel DMA Protection On
- [ ] TPM 2.0: Present & Ready
- [ ] TPM Attestation: siap (EK certificate ada, Measured Boot log valid)
- [ ] HVCI (Memory Integrity): Running (VBS Running)
- [ ] Windows Security Updates: image sudah berisi update keamanan terbaru (umur patch di bawah ambang yang diatur admin)
- [ ] Firmware mode UEFI (bukan CSM/Legacy)
- [ ] Tidak ada driver dari Vulnerable Driver Blocklist yang termuat
- [ ] Binary boot sesuai db klien (CA 2011/2023) dan tidak dicabut DBX
- [ ] Layanan anti-cheat terpasang & dapat start
- [ ] Waktu sistem sinkron (NTP) — sertifikat & anti-cheat sensitif terhadap jam
- [ ] Serial disk iSCSI sesuai kebijakan ADR (unik per klien bila dipilih) agar HWID antar PC tidak sama

### C.4 Glosarium Singkat

| Istilah | Arti |
|---|---|
| shim | First-stage UEFI loader yang ditandatangani Microsoft UEFI CA, memverifikasi loader berikutnya |
| iBFT | iSCSI Boot Firmware Table — cara firmware/iPXE memberi tahu Windows parameter iSCSI boot |
| db / dbx / KEK | Database sertifikat yang diizinkan / dicabut / kunci pertukaran pada UEFI Secure Boot |
| VBS / HVCI | Virtualization-Based Security / Hypervisor-protected Code Integrity (Memory Integrity) |
| Writeback | Layer tulis per-klien; dibuang saat reboot sehingga image tetap bersih |
| Revisi | Snapshot image read-only hasil commit Super User; bisa rollback |
| Super User | Mode klien yang tulisannya masuk ke revisi pending image, untuk instal/update |
| Image Disk | Volume/folder server yang ditetapkan admin untuk menyimpan image OS & revisinya |
| Game Disk | VHDX game terpisah yang di-attach ke klien sebagai drive tambahan (mis. G:) |
| Writeback Disk | Volume server tempat file writeback per klien disimpan |
| RAM Cache | Cache baca/tulis di RAM server per pool (Image/Game/Writeback), ukurannya diatur admin |
| SSD Cache | Tier cache di SSD untuk mempercepat image/game disk yang berada di HDD |
