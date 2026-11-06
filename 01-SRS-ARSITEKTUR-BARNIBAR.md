# DOKUMEN SPESIFIKASI & DESAIN JARINGAN (SRS / HLD & LLD)
## PT BANK BARNIBAR — ENTERPRISE MULTI-SITE NETWORK v4.0

**Kode Dokumen:** `NDD-BARNIBAR-v4.0`  
**Tanggal Pembuatan:** 2026-10-25  
**Pembaruan Terakhir:** 2026-10-26  
**Penyusun:** Edi Suranta Ginting  
**Status Dokumen:** *Living Document* (Diperbarui seiring implementasi lab)

---

## DAFTAR ISI
1. Ringkasan Eksekutif & Kebutuhan Bisnis
2. Arsitektur Jaringan & Keputusan Desain Teknis
3. Inventaris Perangkat & Alokasi Sumber Daya Lab
4. Skema Pengalamatan IP & VLAN (*Master IP & VLAN Plan*)
5. Desain Switching, Redundansi Gateway, & Keamanan Layer-2
6. Desain Layanan Infrastruktur & Pemisahan Server 3-Tier
7. Matriks Kebijakan Keamanan Firewall (*Zone-Based Firewall*)
8. Desain WAN, Dual ISP, MPLS L3VPN, & IPsec VTI Backup
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

## 3. INVENTARIS PERANGKAT & ALOKASI SUMBER DAYA LAB

Host Fisik: Acer Nitro AN515-58 (Intel i7-12700H, RAM 16 GB, Windows 11 + VMware GNS3 VM).  
Nama Project GNS3 Aktif: **`BANK`** (`C:\Users\Acer\GNS3\projects\BANK`).

| Nama Node (Aktual di GNS3) | Peran Arsitektur | Tipe / Image | RAM | Status Implementasi |
|---|---|---|---|---|
| `R-ISP-TELKOMSEL` | Penyedia Internet Utama HQ (ISP-1) & Cabang 1 + Simulasi `8.8.8.8` | QEMU Cisco IOSv 15.9(3)M6 | 512 MB | ✅ Terpasang |
| `R-ISP-BIZNET` | Penyedia Internet Cadangan HQ (ISP-2) & Cabang 2 | QEMU Cisco IOSv 15.9(3)M6 | 512 MB | ✅ Terpasang |
| `R-HQ` | Perimeter Router, NTP Master, NAT, VPN Hub, & Firewall ZBF Pusat | QEMU Cisco IOSv 15.9(3)M6 (6 Port) | 512 MB | ✅ Aktif |
| `CORE-1` | Collapsed Core Switch 1 (HSRP Active VLAN 10, 30, 100) | QEMU Cisco IOSvL2 | 1024 MB | ✅ Aktif |
| `CORE-2` | Collapsed Core Switch 2 (HSRP Active VLAN 20, 50) | QEMU Cisco IOSvL2 | 1024 MB | ✅ Aktif |
| `ACC-1` | Access Switch Pengguna HQ (PortFast, BPDU Guard, Port-Security) | QEMU Cisco IOSvL2 | 768 MB | ✅ Aktif |
| `SRV-DHCP` | Server Pembagi IP Terpusat (`isc-dhcp-server`) | Docker `adosztal/net_toolbox` | ~25 MB | ✅ Aktif |
| `SRV-DNS-INT` | Server Penerjemah Domain Internal (`dnsmasq` `*.barnibar.local`) | Docker `adosztal/dns` | ~20 MB | ✅ Aktif |
| `SRV-NTP-LOG` | Server Pencatat Kejadian Terpusat (`rsyslog` UDP/TCP 514) | Docker `adosztal/net_toolbox` | ~25 MB | ✅ Aktif |
| `SRV-APP` | Server Aplikasi Transaksi Kasir (`:8080`) & Web Intranet (`:80`) | Docker `adosztal/net_toolbox` | ~25 MB | ✅ Aktif |
| `SRV-AAA` | Server Keamanan AAA Terpusat (TACACS+ TCP `:49`) | Docker `adosztal/aaa` | ~25 MB | ✅ Aktif |
| `SRV-DB` | Server Basis Data Terisolasi (PostgreSQL `:5432`) | Docker `adosztal/net_toolbox` | ~25 MB | ✅ Aktif |
| `WEB-DMZ` | Server Portal Publik Bank (Nginx `:80/:443`) | Docker `adosztal/net_toolbox` | ~25 MB | ✅ Aktif |
| `PC-STAFF` | Komputer Karyawan HQ (VLAN 10) | VPCS | 8 MB | ✅ Aktif |
| `PC-TELLER` | Komputer Kasir HQ (VLAN 20) | VPCS | 8 MB | ✅ Aktif |
| `BR1-FW` & `BR1-SW` | Router+Firewall ZBF & Switch Cabang 1 | Cisco IOSv + Switch | 512 MB | ⬜ Milestone 7 |
| `BR2-FW` & `BR2-SW` | Router+Firewall Multi-Vendor & Switch Cabang 2 | MikroTik CHR 7.23.8 + Switch | 256 MB | ⬜ Milestone 7 |
| `PE-HQ`, `P-CORE`, `PE-BR` | Jaringan Tulang Punggung MPLS Provider (AS 65000) | Cisco IOSv / c7200 | Bertahap | ⬜ Milestone 8 |

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
| `20` | `LAN-TELLER` | `10.10.20.0/24` | `10.10.20.254` (VIP HSRP) | `CORE-2` Aktif (`.253`), `CORE-1` Siaga (`.252`) | `.50` – `.150` |
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

## 7. RIWAYAT REVISI DESAIN (*CHANGELOG*)

| Tanggal | Versi | Perubahan / Revisi yang Dilakukan | Alasan Teknis |
|---|---|---|---|
| 2026-10-25 | `v4.0-draft` | Penyusunan awal skema 3 lokasi, Dual ISP HQ, MPLS + VTI, dan 3-Tier Docker. | Menyesuaikan proposal bank ke kapasitas RAM host 16 GB di GNS3. |
| 2026-10-26 | `v4.0-rev1` | Penyesuaian nama node aktual GNS3 (`R-HQ`, `R-ISP-TELKOMSEL`, `R-ISP-BIZNET`) & penambahan port `R-HQ` dari 4 menjadi 6 adapter. | Menyediakan port fisik cukup untuk jalur DMZ (`Gi0/4`) dan MPLS (`Gi0/3`). |
| 2026-10-26 | `v4.0-rev2` | Penghapusan `clock set` manual pada `R-HQ`; menggunakan jam otomatis QEMU + zona `WIB 7 0`. | Agar cap waktu pada Syslog berjalan *real-time* sesuai jam asli sistem. |
| 2026-10-26 | `v4.0-rev3` | Penambahan VLAN Transit `100` (`10.10.100.0/24`) dengan HSRP antara Core dan `R-HQ`, serta pemisahan `SRV-NTP-LOG` ke `CORE-2`. | Memaksa trafik VLAN 40 naik ke ZBF `R-HQ` dan menyebar risiko fisik rak server. |
| 2026-10-26 | `v4.0-rev4` | Penambahan penguatan Layer-2: Native VLAN `999`, `nonegotiate`, parkir port mati ke VLAN `888`, dan Port-Security Sticky di `ACC-1`. | Menutup celah *VLAN Hopping*, *DTP Spoofing*, dan penyusupan kabel LAN meja kerja. |
