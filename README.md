PROYEK FLAGSHIP v4.0 — PT BANK BARNIBAR (GNS3 ENTERPRISE LAB)

<img width="2492" height="1382" alt="TOPOLOGI BANK" src="https://github.com/user-attachments/assets/867474d7-764a-4819-b180-51b406a44249" />


**Lokasi Project GNS3:** `C:\Users\Acer\GNS3\projects\BANK`  
**Platform:** GNS3 2.2.61 + GNS3 VM (VMware Workstation)  
**Status Saat Ini:** 🏆 **SELESAI 100% (8 dari 8 Milestone Tuntas)** — Seluruh Arsitektur Flagship v4.0 Berhasil Diimplementasikan dan Teruji Penuh!

---

## DAFTAR DOKUMEN STANDAR INDUSTRI DI FOLDER INI

| File | Jenis Dokumen Industri | Isi & Kegunaan |
|---|---|---|
| [`01-SRS-ARSITEKTUR-BARNIBAR.md`](./01-SRS-ARSITEKTUR-BARNIBAR.md) | **SRS / HLD & LLD** (*System Requirements & Network Design*) | Kebutuhan bisnis bank, arsitektur jaringan, inventaris perangkat, Master IP & VLAN Plan, desain routing/WAN, dan matriks kebijakan keamanan Firewall ZBF. |
| [`02-BUKU-KERJA-IMPLEMENTASI.md`](./02-BUKU-KERJA-IMPLEMENTASI.md) | **NIP / Runbook** (*Network Implementation Plan*) | Panduan konfigurasi teknis langkah demi langkah dari Milestone 1 s.d. Milestone 8 (Cisco IOSv, IOSvL2, Docker Linux, MikroTik CHR) yang telah bersih dan teruji. |
| [`03-LAPORAN-UJI-DAN-JURNAL.md`](./03-LAPORAN-UJI-DAN-JURNAL.md) | **UAT & NOC Log** (*Acceptance Test Report & Troubleshooting Log*) | Bukti hasil pengujian nyata dari terminal/konsol tiap milestone, daftar cek lulus/gagal, serta catatan insiden nyata dan cara perbaikannya selama pengerjaan. |
| `../../00-roadmap/topologi-barnibar-v4.png` | **Diagram Topologi Visual** | Gambar topologi arsitektur lengkap resolusi tinggi (2000x1400 px). |

---

## RINGKASAN PROGRES EKSEKUSI MILESTONE

| Milestone | Fokus Pekerjaan | Status | Tanggal Selesai |
|---|---|---|---|
| **Milestone 1** | Pondasi WAN & Dual ISP Kantor Pusat (`R-ISP-TELKOMSEL`, `R-ISP-BIZNET`, Peering IXP, `R-HQ`) | ✅ **LULUS** | 2026-10-25 |
| **Milestone 2** | Switching Collapsed Core HQ (LACP `Po1`, HSRP Load-Sharing, Rapid-PVST+, Port-Security, Native VLAN 999) | ✅ **LULUS** | 2026-10-26 |
| **Milestone 3** | Jalur Transit & Ruang Server Docker (`SRV-DHCP`, `SRV-DNS-INT`, NTP Master/Client, Central `SRV-NTP-LOG`) | ✅ **LULUS** | 2026-10-26 |
| **Milestone 4** | Firewall Pusat (ZBF) & Sekat Akses Antar-Divisi (`SRV-APP` Core Banking :8080, Isolasi Teller vs Staff) | ✅ **LULUS** | 2026-10-27 |
| **Milestone 5** | Zona Publik (DMZ) & Isolasi Database 3-Tier (`WEB-DMZ` DNAT + `SRV-DB` PostgreSQL VLAN 41) | ✅ **LULUS** | 2026-10-27 |
| **Milestone 6** | Redundansi Dual ISP Otomatis di HQ (IP SLA Tracking, Floating Static Route, Dynamic NAT Route-Map) | ✅ **LULUS** | 2026-10-27 |
| **Milestone 6.5**| Gedung 3 Lantai (ATM v25, Manager v15, Tamu v60) & Server AAA TACACS+ (Senior IT vs NOC) | ✅ **LULUS** | 2026-10-28 |
| **Milestone 7** | Integrasi 2 Kantor Cabang Multi-Vendor (`BR1-FW` Cisco IOSv + `BR2-FW` MikroTik CHR + Local Breakout) | ✅ **LULUS** | 2026-10-29 |
| **Milestone 8** | Backbone WAN Utama & Cadangan (MPLS L3VPN Primary + IPsec VTI OSPF Backup & Uji Failover Total) | ✅ **LULUS** | 2026-10-30 |

> **Aturan Pemeliharaan Dokumen (*Living Documentation*):**  
> Setiap kali satu milestone selesai diuji di GNS3 atau ada revisi desain di tengah pengerjaan, ketiga dokumen di atas wajib diperbarui agar selalu mencerminkan kondisi jaringan yang sebenarnya (*As-Built Documentation*).
