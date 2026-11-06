# DOKUMEN SPESIFIKASI & DESAIN JARINGAN (SRS / HLD & LLD)
## PT BANK BARNIBAR — ENTERPRISE MULTI-SITE NETWORK v4.0

**Kode Dokumen:** `NDD-BARNIBAR-v4.0`  
**Tanggal Pembuatan:** 2026-10-25  
**Pembaruan Terakhir:** 2026-11-04  
**Penyusun:** Edi Suranta Ginting  
**Status Dokumen:** *As-Built* — seluruh 8 milestone selesai diimplementasikan & diuji (lihat [`03-LAPORAN-UJI-DAN-JURNAL.md`](./03-LAPORAN-UJI-DAN-JURNAL.md))

> **Ringkasan status:** Dokumen ini sekarang mencerminkan **kondisi akhir** jaringan (as-built), bukan lagi rencana. Seluruh tabel inventaris, alokasi IP, dan kebijakan firewall sudah diverifikasi lewat pengujian di GNS3.

---

## DAFTAR ISI
1. Ringkasan Eksekutif & Kebutuhan Bisnis
2. Arsitektur Jaringan & Keputusan Desain Teknis
3. Inventaris Perangkat, Penamaan Node & Alokasi Sumber Daya Lab
4. Skema Pengalamatan IP & VLAN (*Master IP & VLAN Plan*)
5. Desain Switching, Redundansi Gateway, & Keamanan Layer-2
6. Matriks Kebijakan Keamanan Firewall (*Zone-Based Firewall*)
7. Desain WAN: Dual ISP, MPLS L3VPN, & IPsec VTI Backup
8. Desain Monitoring & Observability
9. Riwayat Revisi Desain (*Changelog*)


---

## 1. RINGKASAN EKSEKUTIF & KEBUTUHAN BISNIS

**PT Bank Barnibar** membutuhkan infrastruktur jaringan tingkat perbankan yang menghubungkan **Kantor Pusat (HQ)** dengan **2 Kantor Cabang** (Cabang 1 berbasis Cisco dan Cabang 2 berbasis MikroTik). Sebagai institusi keuangan, jaringan wajib memenuhi empat pilar utama:

1. **Ketersediaan Tinggi (*High Availability*):** Kerusakan satu switch inti di pusat, putusnya satu penyedia internet (ISP), atau matinya jalur pribadi MPLS tidak boleh menghentikan transaksi perbankan.
2. **Kerahasiaan & Isolasi Berlapis (*Defense-in-Depth*):** Komputer Kasir (*Teller*) tidak boleh menyentuh internet bebas maupun diganggu oleh komputer Karyawan biasa (*Staff*). Database nasabah wajib dikurung di segmen khusus yang hanya boleh diakses oleh mesin aplikasi transaksi, bukan oleh pengguna langsung.
3. **Kemandirian Cabang (*Branch Autonomy & Local Breakout*):** Masing-masing cabang memiliki firewall sendiri, pembagi IP lokal, serta jalur keluar internet lokal untuk Wi-Fi tamu tanpa membebani pipa pusat.
4. **Kepatuhan Audit & Jejak Forensik (*Centralized Logging & Time Sync*):** Seluruh perangkat wajib memiliki waktu detik yang seragam (NTP) dan mengirim seluruh rekaman perubahan konfigurasi serta pelanggaran keamanan ke satu server pencatat terpusat (Syslog).

---

## 2. ARSITEKTUR JARINGAN & KEPUTUSAN DESAIN TEKNIS

Diagram visual resolusi tinggi tersimpan pada: `CISCO/00-roadmap/topologi-barnibar-v4.png`.

```
                  ┌────────────────────────────────────────────────────┐
                  │          JARINGAN PENYEDIA LAYANAN (WAN)           │
                  │  R-ISP-TELKOMSEL (ISP-1) ◄── IXP ──► R-ISP-BIZNET  │
                  │         ▲         Loopback: 8.8.8.8         ▲      │
                  │         │      MPLS L3VPN (AS 65000)        │      │
                  │         └────── [PE]──[P1]──[P2]──[PE] ─────┘      │
                  └───────────────────┬──────────────────┬─────────────┘
                                      │                  │
                     ┌────────────────┴───────┐          │
                     │   KANTOR PUSAT (HQ)    │          │
                     │ • R-HQ (Dual WAN + ZBF)│          │
                     │ • CORE-1 & CORE-2      │          │
                     │   (LACP + HSRP + STP)  │          │
                     │ • ACC-1 (Port-Security)│          │
                     │ • Server Docker 3-Tier │          │
                     └────────────────────────┘          │
                                  │                      │
                    ┌─────────────┴──────────┐  ┌────────┴───────────────┐
                    │ CABANG 1 (Cisco IOSv)  │  │ CABANG 2 (MikroTik CHR)│
                    │ • BR1-FW (Router + ZBF)│  │ • BR2-FW (RouterOS FW) │
                    │ • 1 ISP + MPLS + VTI   │  │ • 1 ISP + MPLS + VTI   │
                    └────────────────────────┘  └────────────────────────┘
```

### Alasan Keputusan Desain (*Design Rationale*):
* **Mengapa Wajib GNS3 (Bukan Packet Tracer)?** Packet Tracer tidak mendukung MPLS L3VPN (`vrf`, MP-BGP), antarmuka tunnel IPsec VTI untuk OSPF, router multi-vendor MikroTik CHR, maupun server Linux asli.
* **Mengapa Router & Firewall Digabung per Lokasi (`R-HQ`, `BR1-FW`, `BR2-FW`)?** Memisahkan kotak router dan VM firewall pihak ketiga membutuhkan tambahan >4,5 GB RAM. Menggabungkan fungsi routing dan *Zone-Based Firewall* (Cisco IOSv) / *RouterOS Firewall* (MikroTik CHR) menghemat RAM secara drastis pada laptop host 16 GB dengan kapabilitas keamanan yang tetap setara produksi.
* **Mengapa Server Menggunakan Container Docker?** Menjalankan 7 fungsi server terpisah dengan VM penuh akan menghabiskan >4 GB RAM. Container Docker di GNS3 VM menjalankan paket aplikasi Linux produksi asli (`isc-dhcp-server`, `dnsmasq`, `rsyslog`, `nginx`, `postgresql`) hanya dengan total beban ~150 MB RAM.
* **Mengapa Gerbang VLAN Server Ditaruh di `R-HQ` (Bukan di Switch Core)?** Agar lalu lintas dari komputer pengguna menuju Ruang Server tidak langsung berbelok di Switch L3, melainkan dipaksa naik melewati pemeriksaan *stateful* Firewall ZBF di `R-HQ`.

---

## 3. INVENTARIS PERANGKAT, PENAMAAN NODE & ALOKASI SUMBER DAYA LAB

Host Fisik: Acer Nitro AN515-58 (Intel i7-12700H, RAM 16 GB, Windows 11 + VMware GNS3 VM).  
Nama Project GNS3 Aktif: **`BANK`** (`C:\Users\Acer\GNS3\projects\BANK`) — **41 node**.

### 3.1 Konvensi Penamaan

| Awalan | Arti | Contoh |
|---|---|---|
| `R-` / `PE-` | Router / Provider Edge | `R-ISP-TELKOMSEL`, `PE-HQ` |
| `HQ-` | Perangkat kantor pusat | `HQ-FW` |
| `CORE-` | Switch inti (*collapsed core*) | `CORE-1`, `CORE-2` |
| `SW-LT` | Switch akses per lantai | `SW-LT1`, `SW-LT2`, `SW-LT3` |
| `BR1-` / `BR2-` | Perangkat Cabang 1 (Medan) / Cabang 2 (Surabaya) | `BR1-FW`, `BR2-SW` |
| `SRV-` | Server (container Linux) | `SRV-DHCP`, `SRV-DB` |
| `PC-` / `ATM-` / `TAMU` | End device (VPCS) | `PC-TELLER-1`, `ATM-1` |

> **Alias dokumen lama:** sebagian runbook awal memakai nama sebelum standar ini diterapkan. Pemetaannya:
> `R-HQ` → **`HQ-FW`** · `ACC-1` → **`SW-LT1`** · `PC-STAFF` → **`PC-STAFF-1`** · `PC-TELLER` → **`PC-TELLER-1`** · `PC-MANAGER-1` → **`PC-MANAGER`** · `PC-SENIOR-IT-1` → **`PC-SENIOR-IT`**.
> Konfigurasi perangkat tidak berubah karena pergantian nama ini — hanya label node di kanvas GNS3.

### 3.2 Tabel Inventaris Lengkap (As-Built)

**A. Jaringan Penyedia Layanan (WAN) & Backbone MPLS**

| Nama Node | Peran Arsitektur | Tipe / Image | RAM | Status |
|---|---|---|---|---|
| `R-ISP-TELKOMSEL` | ISP-1: WAN utama HQ, WAN Cabang 1, simulasi `8.8.8.8`, peering IXP | QEMU Cisco IOSv 15.9(3)M6 | 512 MB | ✅ Aktif |
| `R-ISP-BIZNET` | ISP-2: WAN cadangan HQ, WAN Cabang 2, peering IXP | QEMU Cisco IOSv 15.9(3)M6 | 512 MB | ✅ Aktif |
| `PE-HQ` | Provider Edge Telco sisi Jakarta — VRF `BANK-BARNIBAR`, MP-BGP `vpnv4`, MPLS LDP | QEMU Cisco IOSv | 512 MB | ✅ Aktif |
| `PE-BR` | Provider Edge Telco sisi cabang — meneruskan VRF ke Medan & Surabaya | QEMU Cisco IOSv | 512 MB | ✅ Aktif |

**B. Kantor Pusat (HQ) — Perimeter, Core, Akses**

| Nama Node | Peran Arsitektur | Tipe / Image | RAM | Status |
|---|---|---|---|---|
| `HQ-FW` | Perimeter router + **Zone-Based Firewall**, NAT, VPN hub, NTP master, syslog client | QEMU Cisco IOSv (6 port) | 512 MB | ✅ Aktif |
| `CORE-1` | Switch inti 1 — HSRP Active VLAN 10/30/40/100, STP Root Primary | QEMU Cisco IOSvL2 | 1024 MB | ✅ Aktif |
| `CORE-2` | Switch inti 2 — HSRP Active VLAN 20/50, STP Root Primary | QEMU Cisco IOSvL2 | 1024 MB | ✅ Aktif |
| `SW-LT1` | Switch akses Lantai 1 (Lobi) — Port-Security, BPDU Guard | QEMU Cisco IOSvL2 | 768 MB | ✅ Aktif |
| `SW-LT2` | Switch akses Lantai 2 (Operasional) | Switch bawaan GNS3 | 0 MB | ✅ Aktif |
| `SW-LT3` | Switch akses Lantai 3 (IT Admin) | Switch bawaan GNS3 | 0 MB | ✅ Aktif |

**C. Server Farm (Container Docker di GNS3 VM)**

| Nama Node | Peran Layanan | IP | Image | Status |
|---|---|---|---|---|
| `SRV-DHCP` | DHCP terpusat (`isc-dhcp-server`) | `10.10.40.10` | `adosztal/net_toolbox` | ✅ Aktif |
| `SRV-DNS-INT` | DNS internal `barnibar.local` (`dnsmasq`) | `10.10.40.20` | `adosztal/dns` | ✅ Aktif |
| `SRV-NTP-LOG` | Syslog terpusat (`rsyslog` UDP/TCP 514) | `10.10.40.30` | `adosztal/net_toolbox` | ✅ Aktif |
| `SRV-APP` | Aplikasi Core Banking `:8080` & intranet `:80` | `10.10.40.50` | `adosztal/net_toolbox` | ✅ Aktif |
| `SRV-AAA` | TACACS+ (`tac_plus`, TCP 49) | `10.10.40.40` | `adosztal/aaa` | ✅ Aktif |
| `SRV-DB` | PostgreSQL `:5432` di VLAN 41 terisolasi | `10.10.41.10` | `adosztal/net_toolbox` | ✅ Aktif |
| `WEB-DMZ` | Portal publik Nginx `:80/:443` (zona DMZ) | `172.16.50.10` | `adosztal/net_toolbox` | ✅ Aktif |

**D. End Device HQ (VPCS) & Perangkat Admin**

| Nama Node | Peran | VLAN | Status |
|---|---|---|---|
| `PC-STAFF-1`, `PC-STAFF-2` | Karyawan | 10 | ✅ Aktif |
| `PC-TELLER-1`, `PC-TELLER-2`, `ATM-1` | Kasir & mesin ATM | 20 / 25 | ✅ Aktif |
| `PC-MANAGER` | Manajer (VLAN khusus) | 15 | ✅ Aktif |
| `PC-SENIOR-IT`, `PC-NOC-1`, `PC-NOC-2` | Tim IT & NOC (akses pemeliharaan) | 30 | ✅ Aktif |
| `TAMU` | Wi-Fi tamu | 60 | ✅ Aktif |

**E. Kantor Cabang (Multi-Vendor)**

| Nama Node | Peran | Platform | Status |
|---|---|---|---|
| `BR1-FW` | Router + ZBF + DHCP lokal + NAT Cabang 1 (Medan) | Cisco IOSv 15.9(3)M6 | ✅ Aktif |
| `BR1-SW` | Switch akses cabang (VLAN 10/25/30/60) | Switch bawaan GNS3 | ✅ Aktif |
| `BR1-IT`, `BR1-STAFF`, `BR1-ATM`, `BR1-GUEST` | End device cabang 1 | VPCS | ✅ Aktif |
| `BR2-FW` | Router + firewall + DHCP lokal + NAT Cabang 2 (Surabaya) | **MikroTik CHR RouterOS v7** | ✅ Aktif |
| `BR2-SW` | Switch akses cabang (VLAN 10/25/30/60) | Switch bawaan GNS3 | ✅ Aktif |
| `BR2-IT`, `BR2-STAFF`, `BR2-ATM`, `BR2-GUEST` | End device cabang 2 | VPCS | ✅ Aktif |

**F. Infrastruktur Pendukung Lab**

| Nama Node | Peran | Status |
|---|---|---|
| `Cloud-Winbox` | Jembatan OOBM (Out-of-Band Management) ke VMnet1 untuk akses WinBox ke `BR2-FW` | ✅ Aktif |
| `ZABBIX` | Node NAT penanda keterhubungan ke server monitoring | ✅ Aktif |

### 3.3 Alokasi Sumber Daya (Capacity Planning)

| Komponen | Alokasi | Catatan |
|---|---|---|
| RAM host fisik | 16 GB | Acer Nitro AN515-58 |
| GNS3 VM | 4–10 GB (dinamis) | Disesuaikan beban; router IOSv ~380 MB masing-masing |
| Router IOSv | 512 MB × 6 | `R-ISP-*`, `HQ-FW`, `PE-HQ`, `PE-BR`, `BR1-FW` |
| Switch IOSvL2 | 768–1024 MB × 3 | `CORE-1`, `CORE-2`, `SW-LT1` |
| Container Docker | ~20–25 MB × 9 | Server farm + `PC-SENIOR-IT`, `PC-NOC-1` |
| Switch bawaan GNS3 | 0 MB | `BR1-SW`, `BR2-SW`, `SW-LT2`, `SW-LT3` |
| VPCS | ~8 MB × 15 | End device |

**Alasan pemilihan tipe node:** GNS3 di host 16 GB tidak sanggup menjalankan 41 node bila semuanya berupa VM penuh. Karena itu dipakai strategi hemat RAM: fungsi server dijalankan sebagai **container Docker** (bukan VM), switch cabang dan lantai memakai **switch bawaan GNS3** (tanpa OS), dan end device memakai **VPCS** (bukan VM). Hasilnya, seluruh topologi tetap berjalan dengan konsumsi RAM yang wajar tanpa mengurangi realisme konfigurasi di sisi router dan switch inti.

---

## 4. SKEMA PENGALAMATAN IP & VLAN (*MASTER IP & VLAN PLAN*)

### A. Jaringan Publik WAN & Peering Antar-ISP
| Segmen | Subnet | Perangkat & Port A | IP A | Perangkat & Port B | IP B |
|---|---|---|---|---|---|
| Link WAN Utama HQ (ISP-1) | `203.0.113.0/29` | `R-HQ` (`Gi0/1`) | `203.0.113.1` | `R-ISP-TELKOMSEL` (`Gi0/0`) | `203.0.113.2` |
| Link WAN Cadangan HQ (ISP-2) | `198.51.200.0/29` | `R-HQ` (`Gi0/2`) | `198.51.200.1` | `R-ISP-BIZNET` (`Gi0/0`) | `198.51.200.2` |
| Link Peering IXP Antar-ISP | `198.18.0.0/30` | `R-ISP-TELKOMSEL` (`Gi0/2`) | `198.18.0.1` | `R-ISP-BIZNET` (`Gi0/2`) | `198.18.0.2` |
| Link WAN Cabang 1 (ISP-1) | `198.51.100.0/29` | `R-ISP-TELKOMSEL` (`Gi0/1`) | `198.51.100.1` | `BR1-FW` (`Gi0/1`) | `198.51.100.2` |
| Link WAN Cabang 2 (ISP-2) | `192.0.2.0/29` | `R-ISP-BIZNET` (`Gi0/1`) | `192.0.2.1` | `BR2-FW` (`ether1`) | `192.0.2.2` |
| Simulasi DNS/Target Publik | `8.8.8.8/32` | `R-ISP-TELKOMSEL` (`Lo0`) | `8.8.8.8` | — | — |

### B. Daftar VLAN & Subnet Kantor Pusat (HQ — `10.10.0.0/16`)
| ID VLAN | Nama VLAN | Subnet | Gerbang Utama (Gateway) | Pemegang Gerbang | Rentang DHCP |
|---|---|---|---|---|---|
| `10` | `LAN-STAFF` | `10.10.10.0/24` | `10.10.10.254` (VIP HSRP) | `CORE-1` Aktif (`.252`), `CORE-2` Siaga (`.253`) | `.50` – `.150` |
| `15` | `LAN-MANAGER` | `10.10.15.0/24` | `10.10.15.254` (VIP HSRP) | `CORE-1` Aktif (`.252`), `CORE-2` Siaga (`.253`) | `.50` – `.150` |
| `20` | `LAN-TELLER` | `10.10.20.0/24` | `10.10.20.254` (VIP HSRP) | `CORE-2` Aktif (`.253`), `CORE-1` Siaga (`.252`) | `.50` – `.150` |
| `25` | `ATM-LOBBY` | `10.10.25.0/24` | `10.10.25.254` (VIP HSRP) | `CORE-1` Aktif — *isolasi ketat, tanpa internet* | `.50` – `.150` |
| `30` | `LAN-IT-MGMT` | `10.10.30.0/24` | `10.10.30.254` (VIP HSRP) | `CORE-1` Aktif (`.252`), `CORE-2` Siaga (`.253`) | `.50` – `.150` |
| `40` | `RUANG-SERVER` | `10.10.40.0/24` | `10.10.40.1` | `R-HQ` (`Gi0/0.40`) — *L2 murni di Core* | IP Statis |
| `41` | `SEC-DB` | `10.10.41.0/24` | `10.10.41.1` | `R-HQ` (`Gi0/0.41`) — *Isolasi Database* | IP Statis |
| `50` | `WLAN-STAFF` | `10.10.50.0/24` | `10.10.50.254` (VIP HSRP) | `CORE-2` Aktif (`.253`), `CORE-1` Siaga (`.252`) | `.50` – `.150` |
| `60` | `WLAN-GUEST` | `10.10.60.0/24` | `10.10.60.1` | `R-HQ` (`Gi0/0.60`) — *Isolasi Tamu* | Lokal |
| `100` | `TRANSIT-FIREWALL`| `10.10.100.0/24` | `10.10.100.1` (`R-HQ`) | Ke bawah: VIP HSRP Core `10.10.100.254` | Statis |
| `888` | `BLACKHOLE-UNUSED`| — | — | Parkir seluruh port switch yang dimatikan | — |
| `999` | `NATIVE-DEAD` | — | — | Native VLAN seragam pada seluruh jalur Trunk | — |
| *DMZ* | `ZONA-DMZ` | `172.16.50.0/24` | `172.16.50.1` | `R-HQ` (`Gi0/4` fisik terpisah) | IP Statis |

### C. Direktori Server & Nama Domain Internal (`barnibar.local`)
| Nama Server | Alamat IP | Lokasi Port Fisik | Nama Domain (FQDN) | Layanan & Port |
|---|---|---|---|---|
| `SRV-DHCP` | `10.10.40.10` | `CORE-1 Gi1/0` (VLAN 40) | `srv-dhcp.barnibar.local` | ISC-DHCP Server (`UDP 67/68`) |
| `SRV-DNS-INT` | `10.10.40.20` | `CORE-1 Gi1/1` (VLAN 40) | `srv-dns.barnibar.local` | Dnsmasq Internal (`UDP/TCP 53`) |
| `SRV-NTP-LOG` | `10.10.40.30` | `CORE-2 Gi1/0` (VLAN 40) | `srv-log.barnibar.local` | Rsyslog Pusat (`UDP/TCP 514`) |
| `SRV-APP` | `10.10.40.50` | `CORE-2 Gi1/1` (VLAN 40) | `app.barnibar.local` | Core Banking (`TCP 8080`) & Intranet (`TCP 80`) |
| `SRV-DB` | `10.10.41.10` | VLAN 41 Terisolasi | `db.barnibar.local` | PostgreSQL Database (`TCP 5432`) |
| `WEB-DMZ` | `172.16.50.10`| `R-HQ Gi0/4` (Zona DMZ) | `www.barnibar.co.id` | Portal Publik Nginx (`TCP 80/443` via DNAT) |

### D. Pengalamatan Kantor Cabang & Pipa WAN Privat
* **Cabang 1 (Cisco — `10.20.0.0/16`):** VLAN 10 Staff (`10.20.10.0/24`), VLAN 20 Teller (`10.20.20.0/24`), VLAN 50 WLAN-Staff (`10.20.50.0/24`), VLAN 60 Guest (`10.20.60.0/24`).
* **Cabang 2 (MikroTik — `10.30.0.0/16`):** VLAN 10 Staff (`10.30.10.0/24`), VLAN 20 Teller (`10.30.20.0/24`), VLAN 50 WLAN-Staff (`10.30.50.0/24`), VLAN 60 Guest (`10.30.60.0/24`).
* **Jalur Utama MPLS CE-PE:** HQ (`10.254.0.0/30`), Cabang 1 (`10.254.1.0/30`), Cabang 2 (`10.254.2.0/30`).
* **Jalur Cadangan IPsec VTI:** Tunnel 1 HQ–BR1 (`10.255.1.0/30`), Tunnel 2 HQ–BR2 (`10.255.2.0/30`).

---

## 5. DESAIN SWITCHING, REDUNDANSI GATEWAY, & KEAMANAN LAYER-2

1. **Bundel Pipa Antar-Core:** `CORE-1` (`Gi0/0-1`) dan `CORE-2` (`Gi0/0-1`) digabung dalam `Port-channel 1` berprotokol **LACP (`mode active`)**.
2. **Keselarasan STP & HSRP (*Symmetric Load-Balancing*):**
   * `CORE-1` menjadi **STP Root Primary** sekaligus **HSRP Active (Priority 110)** untuk **VLAN 10, 30, 40, dan 100**.
   * `CORE-2` menjadi **STP Root Primary** sekaligus **HSRP Active (Priority 110)** untuk **VLAN 20 dan 50**.
   * Hasil nyata pada `ACC-1`: Uplink `Gi0/1` meneruskan VLAN `10, 30`, sedangkan uplink `Gi0/2` meneruskan VLAN `20, 50` (tanpa ada kabel uplink yang menganggur sia-sia).
3. **Penguatan Keamanan Layer-2 (*L2 Hardening*):**
   * **Native VLAN 999 (`NATIVE-DEAD`)** + **`switchport nonegotiate`** diterapkan di seluruh link Trunk (`Po1`, `Gi0/1`, `Gi0/2`, `Gi0/3`, dan `R-HQ Gi0/0.999`) untuk menutup celah *VLAN Hopping* dan *DTP Spoofing*.
   * **Port-Security (`maximum 1`, `mac-address sticky`, `violation restrict`)** + **PortFast** + **BPDU Guard** aktif di seluruh port meja pengguna pada `ACC-1`.
   * Seluruh 12 port kosong pada `ACC-1` diparkir ke **VLAN 888 (`BLACKHOLE-UNUSED`)** dalam keadaan **`shutdown`**.

---

## 6. MATRIKS KEBIJAKAN KEAMANAN FIREWALL (*ZBF & ACL*)

Prinsip utama: **Tolak Semua Secara Bawaan (*Default Deny*), Izinkan Hanya yang Dibutuhkan (*Least Privilege*)**.

| Asal Lalu Lintas | Tujuan | Port / Protokol | Tindakan | Alasan Keamanan Bisnis |
|---|---|---|---|---|
| **Kasir (VLAN 20)** | Karyawan (VLAN 10/50) | Semua IP | **TOLAK + LOG** (ACL Core) | Mencegah gangguan lateral dari meja umum ke komputer kasir. |
| **Semua Divisi LAN** | `SRV-DHCP` & `SRV-DNS-INT`| DHCP (`67/68`), DNS (`53`) | **INSPECT** (ZBF) | Kebutuhan dasar mendapatkan IP dan penerjemahan nama. |
| **Kasir (VLAN 20)** | `SRV-APP` (`10.10.40.50`) | `TCP 8080` | **INSPECT** (ZBF) | Kasir hanya boleh membuka sistem transaksi Core Banking. |
| **Kasir (VLAN 20)** | Internet (`ZONE-WAN`) | Semua IP | **DROP + LOG** (ZBF) | Komputer kasir dilarang keras terhubung ke internet luar. |
| **Karyawan (VLAN 10)**| `ZONE-SERVER` | `TCP 80` (Intranet), `ICMP` | **INSPECT** (ZBF) | Membuka dokumen kerja internal; dilarang buka port `:8080` kasir. |
| **Karyawan (VLAN 10)**| Internet (`ZONE-WAN`) | `HTTP, HTTPS, DNS, ICMP` | **INSPECT** (ZBF) | Kebutuhan kerja karyawan ke internet luar (via NAT). |
| **Tim IT (VLAN 30)** | Semua Zona | Semua IP (`SSH, ICMP, Web`) | **INSPECT** (ZBF) | Hak pemeliharaan dan pemantauan perangkat oleh tim NOC. |
| **Semua Pengguna** | `SRV-DB` (VLAN 41) | Semua IP | **DROP + LOG** (ZBF) | Pengguna dilarang menyentuh tabel database secara langsung. |
| **`SRV-APP` / `WEB-DMZ`**| `SRV-DB` (VLAN 41) | `TCP 5432` (PostgreSQL) | **INSPECT** (ZBF) | Hanya mesin aplikasi resmi yang boleh mengambil data nasabah. |
| **`ZONE-DMZ`** | `ZONE-INTERNAL` / `SERVER`| Semua IP | **DROP + LOG** (ZBF) | *Anti-Pivot*: jika web publik diretas, peretas tertahan di DMZ. |
| **Internet (`ZONE-WAN`)**| `WEB-DMZ` | `TCP 80, 443` | **PASS + DNAT** | Nasabah luar membuka portal resmi bank. |

---

## 7. DESAIN WAN: DUAL ISP, MPLS L3VPN, & IPsec VTI BACKUP

Jaringan bank tidak boleh bergantung pada satu jalur. Desain WAN dibuat **dua lapis** dengan teknologi yang berbeda, sehingga kegagalan satu lapis tidak menyentuh lapis lainnya.

### 7.1 Topologi & Alamat Peering

**Jalur utama — MPLS L3VPN milik operator (AS 65000):**

| Segmen | Subnet | Sisi Bank | Sisi Operator | AS |
|---|---|---|---|---|
| CE–PE Jakarta | `10.254.0.0/30` | `HQ-FW Gi0/3` → `10.254.0.2` | `PE-HQ Gi0/1` → `10.254.0.1` | bank `65100` → telco `65000` |
| CE–PE Medan | `10.254.1.0/30` | `BR1-FW Gi0/2` → `10.254.1.2` | `PE-BR Gi0/1` → `10.254.1.1` | bank `65200` → telco `65000` |
| CE–PE Surabaya | `10.254.2.0/30` | `BR2-FW ether2` → `10.254.2.2` | `PE-BR Gi0/2` → `10.254.2.1` | bank `65300` → telco `65000` |
| Backbone antar-PE | `10.255.255.0/30` | — | `PE-HQ Gi0/0` ↔ `PE-BR Gi0/0` | telco `65000` |

Di sisi operator dibangun **VRF `BANK-BARNIBAR`** dengan **RD `65000:100`** dan **RT import/export `65000:100`**, lalu didistribusikan antar-PE memakai **MP-BGP `address-family vpnv4`** di atas **MPLS LDP**. Inilah yang membuat jaringan bank terasa seperti tersambung kabel langsung, padahal melewati jaringan operator bersama pelanggan lain.

**Jalur cadangan — IPsec di atas internet publik:**

| Terowongan | Perangkat A | Perangkat B | Subnet Tunnel |
|---|---|---|---|
| Jakarta ↔ Medan | `HQ-FW Tunnel1` → `10.255.1.1` | `BR1-FW Tunnel1` → `10.255.1.2` | `10.255.1.0/30` |
| Jakarta ↔ Surabaya | `HQ-FW Tunnel2` → `10.255.2.1` | `BR2-FW gre-ke-hq` → `10.255.2.2` | `10.255.2.0/30` |

Di atas kedua terowongan dijalankan **OSPF Area 0** untuk bertukar rute secara dinamis. Router ID: Jakarta `10.255.255.1`, Medan `10.20.255.1`, Surabaya `10.30.255.1`.

### 7.2 Mengapa Jalur Utama Selalu Menang: *Administrative Distance*

Tidak ada route-map, tidak ada *policy routing* yang rumit. Yang dipakai hanya sifat dasar router:

| Sumber Rute | Administrative Distance | Posisi |
|---|---|---|
| eBGP (dari MPLS operator) | **20** | Selalu dipilih |
| OSPF (dari terowongan IPsec) | **110** | Hanya dipakai bila BGP hilang |

Alurnya:

1. Kondisi normal → rute BGP `[20/0]` ada di tabel → dipakai. Rute OSPF tetap terhitung tetapi kalah prioritas.
2. Kabel Telco putus → *neighbor* BGP mati → rute BGP **ditarik** → rute OSPF `[110/...]` otomatis naik jadi jalur aktif.
3. Kabel Telco pulih → BGP membentuk *neighbor* lagi → rute `[20/0]` kembali → **preemption** otomatis, tanpa campur tangan NOC.

### 7.3 Tuning Timer untuk Lingkungan Finansial

| Parameter | Standar BGP | Nilai di Lab | Alasan |
|---|---|---|---|
| `timers bgp` | `60 180` | **`3 9`** | Deteksi link putus ≈ 9 detik, bukan 3 menit |
| `neighbor advertisement-interval` | 30 detik | **`0`** | Perubahan rute langsung diiklankan |
| Hasil uji nyata | — | **≈ 7 detik** peralihan penuh | Diukur pada UAT-8.9 |

> Konsekuensi yang harus disadari: timer agresif menambah beban CPU dan memperbesar risiko *flapping* bila link tidak stabil. Pada jaringan produksi, nilai ini harus disesuaikan dengan kualitas link masing-masing operator.

### 7.4 Keamanan Jalur Cadangan

Terowongan IPsec memakai **IKEv2** dengan proposal yang disamakan di kedua vendor:

| Parameter | Cisco IOS | MikroTik RouterOS v7 |
|---|---|---|
| Enkripsi | `aes 256` | `aes-256-cbc` |
| Integritas | `sha256` | `sha256` |
| Group (DH) | `14` (modp2048) | `modp2048` |
| Umur SA | 3600 detik | 3600 detik |

Perbedaan default inilah yang sempat membuat terowongan Cisco–MikroTik gagal terbentuk (tercatat sebagai Insiden #09 pada laporan UAT).

---

## 8. DESAIN MONITORING & OBSERVABILITY

Monitoring dirancang sebagai bagian dari arsitektur, bukan aktivitas tambahan. Ringkasnya:

| Aspek | Keputusan |
|---|---|
| Platform | **Zabbix 7.0 LTS** (server di Proxmox LXC) |
| Pola | **Distributed monitoring** — Zabbix Proxy di jaringan lab, satu koneksi keluar TCP 10051 |
| Protokol pengumpulan | SNMPv2c (perangkat jaringan) + ICMP/*service check* (server) |
| Cakupan | 5 perangkat jaringan + 8 host server/link = **13 host** |
| Output | Dashboard NOC 2 halaman + peta topologi interaktif + papan insiden |
| Otomasi | Registrasi host & pembuatan widget via **Zabbix JSON-RPC API** |

Detail lengkap (termasuk alasan pemilihan proxy, checklist metrik per lapisan, prosedur *drill*, dan troubleshooting widget) ada di dokumen terpisah: **[`04-MONITORING-NOC-ZABBIX.md`](./04-MONITORING-NOC-ZABBIX.md)**.

---

## 9. RIWAYAT REVISI DESAIN (*CHANGELOG*)

| Tanggal | Versi | Perubahan / Revisi yang Dilakukan | Alasan Teknis |
|---|---|---|---|
| 2026-10-25 | `v4.0-draft` | Penyusunan awal skema 3 lokasi, Dual ISP HQ, MPLS + VTI, dan 3-Tier Docker. | Menyesuaikan proposal bank ke kapasitas RAM host 16 GB di GNS3. |
| 2026-10-26 | `v4.0-rev1` | Penyesuaian nama node aktual GNS3 (`R-HQ`, `R-ISP-TELKOMSEL`, `R-ISP-BIZNET`) & penambahan port `R-HQ` dari 4 menjadi 6 adapter. | Menyediakan port fisik cukup untuk jalur DMZ (`Gi0/4`) dan MPLS (`Gi0/3`). |
| 2026-10-26 | `v4.0-rev2` | Penghapusan `clock set` manual pada `R-HQ`; menggunakan jam otomatis QEMU + zona `WIB 7 0`. | Agar cap waktu pada Syslog berjalan *real-time* sesuai jam asli sistem. |
| 2026-10-26 | `v4.0-rev3` | Penambahan VLAN Transit `100` (`10.10.100.0/24`) dengan HSRP antara Core dan `R-HQ`, serta pemisahan `SRV-NTP-LOG` ke `CORE-2`. | Memaksa trafik VLAN 40 naik ke ZBF `R-HQ` dan menyebar risiko fisik rak server. |
| 2026-10-26 | `v4.0-rev4` | Penambahan penguatan Layer-2: Native VLAN `999`, `nonegotiate`, parkir port mati ke VLAN `888`, dan Port-Security Sticky di `ACC-1`. | Menutup celah *VLAN Hopping*, *DTP Spoofing*, dan penyusupan kabel LAN meja kerja. |
| 2026-10-27 | `v4.0-rev5` | Pemisahan zona DMZ ke antarmuka fisik tersendiri (`Gi0/4`) dan pembentukan VLAN 41 khusus database. | Mencegah peretas yang berhasil masuk DMZ bergerak lateral ke jaringan internal (*anti-pivot*). |
| 2026-10-28 | `v4.0-rev6` | Penambahan VLAN 15 (Manager) & VLAN 25 (ATM) pada gedung 3 lantai, serta server AAA TACACS+ `10.10.40.40`. | Memisahkan jalur manajemen dari jalur operasional + mewajibkan otorisasi per perintah untuk perangkat jaringan. |
| 2026-10-29 | `v4.0-rev7` | Standarisasi nama node di kanvas GNS3 (`R-HQ` → `HQ-FW`, `ACC-1` → `SW-LT1`, dsb.) dan penambahan cabang multi-vendor (`BR2-FW` MikroTik). | Menyamakan nama perangkat dengan perannya sehingga dokumentasi dan konsol tidak lagi membingungkan saat operasional. |
| 2026-10-30 | `v4.0-rev8` | Pembangunan backbone MPLS L3VPN (AS 65000) + terowongan IPsec VTI sebagai cadangan, dengan `timers bgp 3 9`. | Memenuhi syarat *high availability* perbankan: dua jalur WAN berbeda teknologi, failover ≈ 7 detik tanpa intervensi manual. |
| 2026-11-04 | `v4.0-rev9` | Penambahan lapisan monitoring terdistribusi (Zabbix 7.0 + proxy), registrasi 13 host via API, dan dashboard NOC. | Jaringan tanpa pengukuran tidak dapat diklaim sehat — monitoring menjadi bukti operasional sekaligus materi audit SLA. |
