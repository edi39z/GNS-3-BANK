# Changelog — Dokumentasi Proyek PT Bank Barnibar v4.0

Seluruh perubahan penting pada **dokumentasi** proyek dicatat di sini.
Format mengikuti [Keep a Changelog](https://keepachangelog.com/id/1.1.0/).

---

## [4.0.0] — 2026-11-04 — Rilis Final (Siap Publikasi)

### Ditambahkan
- **`04-MONITORING-NOC-ZABBIX.md`** — dokumen desain monitoring & observability: arsitektur Zabbix 7.0 terdistribusi, alasan pemilihan proxy, checklist metrik per lapisan, prosedur *drill*, otomasi JSON-RPC, dan troubleshooting widget.
- **`configs/`** — folder *configuration baseline* berisi berkas konfigurasi per perangkat:
  `HQ-FW.cfg`, `CORE-1.cfg`, `CORE-2.cfg`, `SW-LT1.cfg`, `R-ISP-TELKOMSEL.cfg`, `R-ISP-BIZNET.cfg`, `SRV-DHCP.md`, `SRV-DNS-INT.md`, beserta indeks `README.md`.
- **`diagrams/`** — diagram topologi resolusi tinggi (`topologi-bank-barnibar-v4.png`, 2492×1382 px).
- **`CHANGELOG.md`** — berkas ini.
- **`.gitignore`** dan **`LICENSE`** (MIT).

### Diubah
- **`README.md`** — ditulis ulang total menjadi halaman *showcase*: badge status, ringkasan angka kunci, diagram topologi tersemat, tabel rekapitulasi 10 milestone, enam sorotan rekayasa beserta bukti konsol, ringkasan observability, tabel teknologi, peta struktur repositori, dan panduan "cara membaca repositori" untuk tiap jenis pembaca.
- **`01-SRS-ARSITEKTUR-BARNIBAR.md`**
  - Status dokumen diubah dari *Living Document* menjadi **As-Built**.
  - Bagian 3 ditulis ulang: konvensi penamaan node, tabel alias dokumen lama (`R-HQ` → `HQ-FW`, `ACC-1` → `SW-LT1`), inventaris lengkap 6 kelompok perangkat (41 node), dan tabel *capacity planning* RAM.
  - **Bagian 7 baru** — Desain WAN: tabel peering MPLS (AS 65000/65100/65200/65300), VRF & RD/RT, terowongan IPsec VTI, mekanisme *Administrative Distance*, tuning timer BGP, dan perbandingan proposal IPsec Cisco↔MikroTik.
  - **Bagian 8 baru** — ringkasan desain monitoring & observability.
  - Changelog desain ditambah 5 revisi (`rev5`–`rev9`).
- **`02-BUKU-KERJA-IMPLEMENTASI.md`**
  - Header status diperbarui; ditambah navigasi cepat antar-milestone.
  - **Milestone 7 diisi lengkap**: tabel perbandingan cabang, konfigurasi `BR1-FW` (sub-interface, DHCP lokal, NAT `ACL-NAT-BR1`, ACL isolasi tamu) dan `BR2-FW` (bridge, VLAN, IP pool, DHCP server, *masquerade*, firewall filter).
  - **Milestone 8 diisi lengkap**: proposal IKEv2, keyring, profil IPsec, `Tunnel1`/`Tunnel2`, OSPF, GRE-IPsec MikroTik, VRF `BANK-BARNIBAR`, MP-BGP `vpnv4`, eBGP CE, `timers bgp 3 9`, dan tabel uji bencana.
  - **Milestone 9 baru**: ringkasan monitoring NOC.
  - **Lampiran baru**: urutan eksekusi yang disarankan bila topologi dibangun ulang dari nol.
- **`03-LAPORAN-UJI-DAN-JURNAL.md`**
  - Status diubah menjadi **FINAL — 45 test case LULUS (100%)**, dengan tabel rekapitulasi per milestone.
  - Ditambah 6 test case monitoring (`UAT-9.1` s.d. `UAT-9.6`).
  - Ditambah bukti #11: tabel ketersediaan 13 host di Zabbix.
  - Ditambah **3 insiden baru**: #10 (ZBF memblokir SNMP), #11 (rute balik GNS3 VM), #12 (error `base_color` pada widget).
  - Ditambah **Bagian D — Serah Terima & Kesimpulan**: pernyataan kelulusan, tabel status per area, **keterbatasan yang diakui**, dan rekomendasi lanjutan.

### Catatan
- Seluruh nilai sensitif (password, *pre-shared key*, SNMP community) disamarkan menjadi `[REDACTED]` agar aman dipublikasikan.
- Berkas proyek GNS3 (`.gns3`) dan image QEMU tidak disertakan karena berukuran besar; repositori fokus pada dokumentasi dan konfigurasi.

---

## [3.0.0] — 2026-10-30 — Penutupan Milestone 8

### Ditambahkan
- Milestone 8: backbone MPLS L3VPN (AS 65000) + IPsec VTI *backup* + uji failover total.
- Bukti #9 dan #10: LDP `Oper`, MP-BGP `vpnv4` *established*, dominasi AD 20 vs AD 110, dan *trace* tiga kondisi (normal → bencana → pemulihan).

### Diubah
- Status proyek: **8 dari 8 milestone selesai**.
- `README.md`, `01-SRS`, `02-BUKU-KERJA`, `03-LAPORAN-UJI` ditandai lunas.

---

## [2.0.0] — 2026-10-29 — Cabang Multi-Vendor

### Ditambahkan
- Milestone 7: `BR1-FW` (Cisco IOSv) dan `BR2-FW` (MikroTik CHR) dengan DHCP lokal, NAT *local breakout*, isolasi ATM, dan isolasi tamu.
- Insiden #08: salah ketik nama ACL `ACL-NAT-BRI` dan tag `dot1Q 20` pada sub-interface ATM.

---

## [1.5.0] — 2026-10-28 — Gedung 3 Lantai & TACACS+

### Ditambahkan
- Milestone 6.5: VLAN 15 (Manager), VLAN 25 (ATM), VLAN 60 (Tamu), dan server AAA TACACS+.
- Bukti #7: otorisasi perintah `senior_it` vs `noc_op` beserta rekam audit WIB.

---

## [1.0.0] — 2026-10-27 — Fondasi Jaringan HQ

### Ditambahkan
- Milestone 1–6: WAN dual ISP, collapsed core (LACP + HSRP + Rapid-PVST+), layanan DHCP/DNS/NTP/Syslog, ZBF, DMZ, isolasi DB 3-tier, dan failover ISP berbasis IP SLA.
- Struktur dokumen awal: `README.md`, `01-SRS`, `02-BUKU-KERJA`, `03-LAPORAN-UJI`.
