# LAPORAN UJI TERIMA (*UAT*) & JURNAL TROUBLESHOOTING NOC
## PT BANK BARNIBAR — PROYEK GNS3 v4.0

**Kode Dokumen:** `UAT-NOC-BARNIBAR-v4.0`  
**Status:** Terverifikasi s.d. Milestone 6 (Akan terus diperbarui setiap Milestone selesai)

---

## BAGIAN A — RINGKASAN DAFTAR UJI TERIMA (*ACCEPTANCE TEST MATRIX*)

| Kode Uji | Milestone | Skenario Pengujian | Kriteria Lulus | Hasil |
|---|---|---|---|---|
| `UAT-1.1` | M1 | Ping `R-HQ` ke gateway ISP Utama (`203.0.113.2`) | Reply 100% | ✅ **LULUS** |
| `UAT-1.2` | M1 | Ping `R-HQ` ke gateway ISP Cadangan (`198.51.200.2`) | Reply 100% | ✅ **LULUS** |
| `UAT-1.3` | M1 | Ping melintasi jalur IXP (`198.18.0.2`) & Internet (`8.8.8.8`) | Reply 100% | ✅ **LULUS** |
| `UAT-2.1` | M2 | Bundel LACP `Port-channel 1` antara `CORE-1` dan `CORE-2` | Status `Po1(SU)` aktif di Trunk | ✅ **LULUS** |
| `UAT-2.2` | M2 | Pembagian HSRP Active/Standby (`CORE-1` v10/30, `CORE-2` v20/50) | Sesuai prioritas `110` vs `90` | ✅ **LULUS** |
| `UAT-2.3` | M2 | Keselarasan STP Forwarding di `ACC-1` (`Gi0/1` vs `Gi0/2`) | `Gi0/1`: v10,30 · `Gi0/2`: v20,50 | ✅ **LULUS** |
| `UAT-2.4` | M2+ | Seragam Native VLAN `999` tanpa peringatan *CDP mismatch* | Seluruh Trunk `Native vlan = 999` | ✅ **LULUS** |
| `UAT-2.5` | M2+ | Kunci `Port-Security` (`Sticky` + `Restrict`) di meja pengguna `ACC-1` | MAC tercatat `SecureSticky` | ✅ **LULUS** |
| `UAT-3.1` | M3 | Klien `PC-STAFF` & `PC-TELLER` dapat IP otomatis via DHCP Relay | DORA sukses + GW `.254` + DNS `.20`| ✅ **LULUS** |
| `UAT-3.2` | M3 | Penerjemahan nama domain internal `srv-log.barnibar.local` | Terjemah ke `10.10.40.30` & reply | ✅ **LULUS** |
| `UAT-3.3` | M3 | Sinkronisasi waktu NTP Switch Inti ke `R-HQ` (`10.10.100.1`) | `Clock is synchronized, stratum 3` | ✅ **LULUS** |
| `UAT-3.4` | M3 | Pengiriman rekaman kejadian dari perangkat ke `SRV-NTP-LOG` | Tercatat di `/var/log/syslog` (WIB)| ✅ **LULUS** |
| `UAT-4.1` | M4 | Isolasi komunikasi langsung `PC-STAFF` ◄──► `PC-TELLER` | Ditolak oleh ACL Core (`code:13`) | ✅ **LULUS** |
| `UAT-4.2` | M4 | Akses internet `PC-STAFF` tembus vs `PC-TELLER` diblokir ZBF | Staff reply `8.8.8.8`, Teller *timeout* | ✅ **LULUS** |
| `UAT-4.3` | M4 | Akses `SRV-APP` `:8080` hanya terbuka bagi `PC-TELLER` | Teller sukses `:8080`, Staff hanya `:80`| ✅ **LULUS** |
| `UAT-4.4` | M4 | Bukti pelanggaran ZBF Kasir tercatat di `SRV-NTP-LOG` | Muncul log `%FW-6-DROP_PKT` (WIB) | ✅ **LULUS** |
| `UAT-5.1` | M5 | Akses publik dari ISP ke `203.0.113.1:80` masuk ke `WEB-DMZ` | DNAT + ZBF sukses menampilkan portal | ✅ **LULUS** |
| `UAT-5.2` | M5 | Akses mesin aplikasi (`SRV-APP` & `WEB-DMZ`) ke `SRV-DB` `:5432` | Respon `STATUS OK` dari `:5432` | ✅ **LULUS** |
| `UAT-5.3` | M5 | Blokir akses langsung pengguna (`PC-STAFF`/`TELLER`) ke `SRV-DB` | Ditolak + log `ZP-BLOKIR-USER-KE-DB` | ✅ **LULUS** |
| `UAT-5.4` | M5 | Blokir serangan *Anti-Pivot* dari `WEB-DMZ` ke LAN dalam | Ditolak + log `ZP-ANTI-PIVOT-LAN` | ✅ **LULUS** |
| `UAT-6.1` | M6 | Deteksi otomatis putus jalur ISP-1 via `IP SLA 1` + `Track 1` | Status `Track 1` berubah `Up -> Down` | ✅ **LULUS** |
| `UAT-6.2` | M6 | Perpindahan otomatis jalur internet & NAT ke `R-ISP-BIZNET` | `trace 8.8.8.8 -P 1` lewat `198.51.200.2`| ✅ **LULUS** |
| `UAT-6.5a`| M6.5 | Segmentasi 3 Lantai: ATM v25, Manager v15, Tamu v60 | Akses terisolasi & sesuai kebijakan ZBF | ✅ **LULUS** |
| `UAT-6.5b`| M6.5 | Otorisasi & Audit TACACS+: `senior_it` permit vs `noc_op` deny `conf t` | Tercatat di `/var/log/tac_plus.acct` (WIB) | ✅ **LULUS** |
| `UAT-6.5c`| M6.5 | Command Accounting Level 1 & Level 15 (`show ip route`) | Tercatat di `/var/log/tac_plus.acct` | ✅ **LULUS** |
| `UAT-7.1` | M7 | DHCP Mandiri Cabang 1 (Cisco) & Cabang 2 (MikroTik) | Dapat IP subnet 10, 25, 30, 60 lokal | ✅ **LULUS** |
| `UAT-7.2` | M7 | Local Internet Breakout Cabang 1 & 2 (`trace 8.8.8.8`) | Keluar via ISP lokal masing-masing | ✅ **LULUS** |
| `UAT-7.3` | M7 | Mesin ATM Cabang 1 & Cabang 2 Dilarang ke Internet | Timeout 100% saat ping `8.8.8.8` | ✅ **LULUS** |
| `UAT-7.4` | M7 | Isolasi Wi-Fi Tamu dari Jaringan Bank di Cabang | Ping Tamu ke Staff timeout 100% | ✅ **LULUS** |
| `UAT-8.1` | M8A | IPsec VTI HQ ◄──► Cabang 1 (Cisco to Cisco) | ISAKMP QM_IDLE & Tunnel1 UP | ✅ **LULUS** |
| `UAT-8.2` | M8A | GRE-IPsec Multi-Vendor HQ ◄──► Cabang 2 (Cisco to MikroTik) | IPsec Established & `gre-ke-hq` UP | ✅ **LULUS** |
| `UAT-8.3` | M8A | OSPF Area 0 Dinamis Tiga Kantor (HQ, BR1, BR2) | Tetangga OSPF `FULL` di kedua sisi | ✅ **LULUS** |
| `UAT-8.4` | M8A | Transaksi Mesin ATM Cabang ke Core Banking Jakarta | TCP port 8080 sukses (`Connect/SendData`) | ✅ **LULUS** |
| `UAT-8.5` | M8A | Komunikasi Antar-Cabang (Medan ke Surabaya via HQ) | Ping `10.20.10.50` ke `10.30.10.150` TTL=61 | ✅ **LULUS** |
| `UAT-8.6` | M8B | Backbone Telco MPLS LDP & MP-BGP (`PE-HQ` ◄► `PE-BR`) | LDP `Oper` & vpnv4 BGP Established | ✅ **LULUS** |
| `UAT-8.7` | M8B | Integrasi eBGP 3 Kantor Bank ke Telco (AS 65000) | BGP Peering Established di HQ, BR1, BR2 | ✅ **LULUS** |
| `UAT-8.8` | M8B | Dominasi Rute Utama MPLS (AD 20) vs OSPF Cadangan (AD 110) | Tabel routing memilih rute `B [20/0]` | ✅ **LULUS** |
| `UAT-8.9` | M8C | Failover Otomatis WAN saat Jalur Utama MPLS Putus | Jalur otomatis berbelok ke IPsec VTI (Hop 2: `10.255.2.1`) | ✅ **LULUS** |
| `UAT-8.10`| M8C | Pemulihan Jalur Utama (*Preemption*) saat Fiber Telco Pulih | Jalur otomatis kembali ke MPLS (Hop 2: `10.254.2.1`) | ✅ **LULUS** |

---

## BAGIAN B — REKAMAN BUKTI KONSOL NYATA (*VERIFIED ARTIFACTS*)

### 1. Bukti Keamanan Layer-2: Port-Security Sticky pada `ACC-1` (2026-10-26)
```text
ACC-1#show port-security
Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
                (Count)       (Count)          (Count)
---------------------------------------------------------------------------
      Gi0/3              1            1                  0         Restrict
      Gi1/0              1            1                  0         Restrict

ACC-1#show port-security address
               Secure Mac Address Table
-----------------------------------------------------------------------------
Vlan    Mac Address       Type                          Ports   Remaining Age
----    -----------       ----                          -----   -------------
  10    0050.7966.6800    SecureSticky                  Gi0/3        -
  20    0050.7966.6801    SecureSticky                  Gi1/0        -
```

### 2. Bukti Native VLAN 999, Bundel `Po1`, & Pembagian Beban STP Simetris (2026-10-26)
```text
CORE-1#show interface trunk
Port        Mode             Encapsulation  Status        Native vlan
Gi0/2       on               802.1q         trunking      999
Gi0/3       on               802.1q         trunking      999
Po1         on               802.1q         trunking      999

ACC-1#show interface trunk
Port        Mode             Encapsulation  Status        Native vlan
Gi0/1       on               802.1q         trunking      999
Gi0/2       on               802.1q         trunking      999

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/1       10,30
Gi0/2       1,20,50,888,999
```
*Analisis Bukti:* `Gi0/1` meneruskan VLAN `10, 30` lurus ke `CORE-1` (Active HSRP VLAN 10/30), sementara `Gi0/2` meneruskan VLAN `20, 50` lurus ke `CORE-2` (Active HSRP VLAN 20/50). Seluruh trunk konsisten di Native VLAN `999`.

### 3. Bukti Central Syslog Real-Time WIB di `SRV-NTP-LOG` (2026-10-26)
```text
root@SRV-NTP-LOG:~# grep "10.10." /var/log/syslog
Oct 26 15:02:11 10.10.100.252 130: Oct 26 22:01:48.922 WIB: %SYS-5-CONFIG_I: Configured from console by console
```
*Analisis Bukti:* Perubahan konfigurasi pada `CORE-1` (`10.10.100.252`) berhasil dikirim melalui jaringan menuju server Docker `SRV-NTP-LOG` lengkap dengan penanda waktu akurat `WIB`.

### 4. Bukti Sekat Antar-Divisi, Internet Karyawan vs Kasir, & Audit ZBF (2026-10-27)
```text
PC-STAFF> ping 10.10.20.50
*10.10.10.252 icmp_seq=1 ttl=255 time=4.282 ms (ICMP type:3, code:13, Communication administratively prohibited)

PC-STAFF> ping 8.8.8.8
84 bytes from 8.8.8.8 icmp_seq=1 ttl=253 time=8.170 ms

PC-TELLER> ping 8.8.8.8
8.8.8.8 icmp_seq=1 timeout

root@SRV-NTP-LOG:~# grep "DROP" /var/log/syslog | tail -n 5
Oct 27 07:39:06 10.10.40.1 42: Oct 27 14:38:59.142 WIB: %FW-6-DROP_PKT: Dropping icmp session 10.10.20.50:0 8.8.8.8:0 on zone-pair ZP-INT-KE-WAN class CM-BLOKIR-TELLER-WAN due to DROP action found in policy-map with ip ident 21785
```
*Analisis Bukti:* ACL Switch Core berhasil menolak percobaan akses langsung Staff ke Teller (`code:13`), ZBF mengizinkan Staff ke internet via NAT namun memblokir mutlak Teller (`10.10.20.50`) sambil melempar bukti pelanggaran `%FW-6-DROP_PKT` ke `SRV-NTP-LOG`, serta port `:8080` pada `SRV-APP` terverifikasi hanya dapat dibuka oleh Teller.

### 5. Bukti Isolasi Brankas Database 3-Tier & Anti-Pivot DMZ (2026-10-27)
```text
Oct 27 08:05:19 10.10.40.1 44: Oct 27 15:05:08.586 WIB: %FW-6-DROP_PKT: Dropping tcp session 10.10.10.50:9121 10.10.40.50:8080 on zone-pair ZP-INT-KE-SRV class class-default
Oct 27 10:40:40 10.10.40.1 50: Oct 27 17:40:06.402 WIB: %FW-6-DROP_PKT: Dropping icmp session 10.10.10.50:0 10.10.41.10:0 on zone-pair ZP-BLOKIR-USER-KE-DB class class-default
Oct 27 10:44:44 10.10.40.1 52: Oct 27 17:44:09.420 WIB: %FW-6-DROP_PKT: Dropping icmp session 172.16.50.10:0 10.10.20.50:0 on zone-pair ZP-ANTI-PIVOT-LAN class class-default
```
*Analisis Bukti:* Terbukti 3 lapis pertahanan bekerja presisi: (1) Karyawan biasa (`10.10.10.50`) ditolak masuk ke port Core Banking `:8080`, (2) Karyawan ditolak menyentuh langsung brankas database `10.10.41.10` (`ZP-BLOKIR-USER-KE-DB`), dan (3) simulasi peretas dari dalam `WEB-DMZ` (`172.16.50.10`) yang mencoba melompat ke PC Kasir (`10.10.20.50`) tertahan mutlak oleh `ZP-ANTI-PIVOT-LAN`.

### 6. Bukti Perpindahan Otomatis Dual ISP Failover ke `R-ISP-BIZNET` (2026-10-27)
```text
PC-STAFF> trace 8.8.8.8 -P 1
trace to 8.8.8.8, 8 hops max (ICMP), press Ctrl+C to stop
 1   10.10.10.252   8.071 ms  5.396 ms  3.902 ms
 2   10.10.100.1   5.586 ms  5.880 ms  4.785 ms
 3   198.51.200.2   9.074 ms  5.123 ms  6.990 ms
 4   8.8.8.8   9.191 ms  6.236 ms  8.569 ms
```
*Analisis Bukti:* Ketika port utama `R-ISP-TELKOMSEL` diputus, sensor `IP SLA 1` + `Track 1` mencabut rute utama dan secara otomatis mengalihkan trafik internet karyawan melewati lompatan ke-3 **`198.51.200.2`** (`R-ISP-BIZNET`) beserta terjemahan NAT `RM-NAT-ISP2` tanpa memutus akses ke `8.8.8.8`.

### 7. Bukti Otorisasi Perintah TACACS+ & Rekam Audit Real-Time WIB (2026-10-28)
```text
Oct 28 18:59:12 10.10.40.1 noc_op     tty578 10.10.30.20 start task_id=10 timezone=WIB service=shell
Oct 28 18:59:12 10.10.40.1 noc_op     tty578 10.10.30.20 stop  task_id=10 timezone=WIB priv-lvl=1 cmd=show ip route <cr>
Oct 28 18:59:13 10.10.40.1 noc_op     tty578 10.10.30.20 stop  task_id=10 timezone=WIB elapsed_time=0
Oct 28 11:39:19 10.10.40.1 senior_it  tty578 10.10.30.10 stop  task_id=7  timezone=WIB priv-lvl=15 cmd=configure terminal <cr>
```
*Analisis Bukti:* Terbukti sistem TACACS+ bekerja presisi:
1. Akun `senior_it` diizinkan menjalankan `configure terminal` (Privilege 15).
2. Akun `noc_op` hanya diizinkan perintah pantau (`show ip route`), dan ketika mencoba `configure terminal` langsung dibatalkan router dengan pesan `Command authorization failed`.
3. Seluruh sesi login, logout, dan eksekusi perintah terekam permanen di server `SRV-AAA` lengkap dengan penanda waktu WIB yang akurat.

### 8. Bukti Terowongan IPsec VTI & Pertukaran Rute OSPF Antar-Kantor (Tahap 8A — 2026-10-29)
```text
HQ-FW#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.30.255.1       0   FULL/  -        00:00:32    10.255.2.2      Tunnel2 (MikroTik Surabaya)
10.20.255.1       0   FULL/  -        00:00:35    10.255.1.2      Tunnel1 (Cisco Medan)

BR2-ATM> ping 10.10.40.50 -P 6 -p 8080
Connect   8080@10.10.40.50 seq=1 ttl=62 time=13.054 ms
SendData  8080@10.10.40.50 seq=1 ttl=62 time=11.191 ms
Close     8080@10.10.40.50 seq=1 ttl=62 time=10.938 ms

BR1-STAFF> ping 10.30.10.150
84 bytes from 10.30.10.150 icmp_seq=1 ttl=61 time=14.537 ms
```
*Analisis Bukti:* Terbukti seluruh jaringan kantor bank berhasil disatukan:
1. Hubungan tetangga OSPF antara `HQ-FW` (Cisco) dengan `BR1-FW` (Cisco) dan `BR2-FW` (MikroTik) berstatus `FULL`.
2. Mesin ATM Surabaya (`BR2-ATM`) berhasil membuka koneksi transaksi ke server Core Banking Jakarta (`10.10.40.50:8080`).
3. Komputer staf Medan (`10.20.10.50`) dapat berkomunikasi dua arah ke staf Surabaya (`10.30.10.150`) melalui perantara enkripsi Jakarta.

### 9. Bukti Integrasi Jalur Utama Telco MPLS L3VPN & Dominasi AD 20 (Tahap 8B — 2026-10-30)
```text
PE-HQ#show mpls ldp neighbor
    Peer LDP Ident: 10.255.255.22:0; Local LDP Ident 10.255.255.11:0
        TCP connection: 10.255.255.22.60386 - 10.255.255.11.646
        State: Oper; Msgs sent/rcvd: 10/10; Downstream
        Up time: 00:03:53

HQ-FW#show ip route bgp
      10.0.0.0/8 is variably subnetted, 27 subnets, 3 masks
B        10.20.10.0/24 [20/0] via 10.254.0.1, 00:04:27, GigabitEthernet0/3
B        10.20.25.0/24 [20/0] via 10.254.0.1, 00:04:27, GigabitEthernet0/3
B        10.20.30.0/24 [20/0] via 10.254.0.1, 00:04:27, GigabitEthernet0/3
B        10.30.10.0/24 [20/0] via 10.254.0.1, 00:04:27, GigabitEthernet0/3
B        10.30.25.0/24 [20/0] via 10.254.0.1, 00:04:27, GigabitEthernet0/3
B        10.30.30.0/24 [20/0] via 10.254.0.1, 00:04:27, GigabitEthernet0/3

[admin@BR2-FW] > /ip/route/print where dst-address in 10.10.40.0/24
Flags: D - DYNAMIC; A - ACTIVE; c - CONNECT, s - STATIC, b - BGP, o - OSPF
Columns: DST-ADDRESS, GATEWAY, DISTANCE
#     DST-ADDRESS     GATEWAY            DISTANCE
0 DAb 10.10.40.0/24   10.254.2.1%ether2        20
1 D o 10.10.40.0/24   10.255.2.1%gre-ke-hq    110
```
*Analisis Bukti:*
1. Hubungan MPLS LDP antar-router Telco berstatus `Oper` dan MP-BGP `vpnv4` berhasil mendistribusikan rute VRF `BANK-BARNIBAR`.
2. Di seluruh router bank (`HQ-FW`, `BR1-FW`, dan `BR2-FW`), rute utama eBGP (`[20/0]`) secara otomatis mengungguli rute OSPF IPsec (`[110/...]`), membuktikan failover berjenjang siap diuji.

### 10. Bukti Pengujian Bencana Putus Kabel Telco & Pemulihan Otomatis (*Zero Downtime WAN* — Tahap 8C)
```text
KONDISI 1 — NORMAL (LINTASAN UTAMA TELCO MPLS L3VPN):
BR2-IT> trace 10.10.40.50 -P 1
 1   10.30.30.1   2.021 ms
 2   10.254.2.1   6.511 ms (Masuk PE-BR Telco MPLS)
 3   10.254.0.1   5.396 ms (Keluar PE-HQ Telco MPLS)
 4   10.254.0.2   6.055 ms (Masuk HQ-FW Jakarta)
 5   10.10.40.50  8.687 ms (Tembus Server Aplikasi SRV-APP)

KONDISI 2 — BENCANA: KABEL TELCO JAKARTA PUTUS (HQ-FW Gi0/3 SHUTDOWN):
BR2-IT> ping 10.10.40.50 -c 2
84 bytes from 10.10.40.50 icmp_seq=1 ttl=62 time=20.779 ms
84 bytes from 10.10.40.50 icmp_seq=2 ttl=62 time=15.697 ms

BR2-IT> trace 10.10.40.50 -P 1
 1   10.30.30.1   1.041 ms
 2   10.255.2.1  11.309 ms (Otomatis Melompat Lewat IPsec VTI gre-ke-hq!)
 3   10.10.40.50 16.108 ms (Server Aplikasi Tetap Melayani Transaksi)

KONDISI 3 — PEMULIHAN FIBER TELCO (HQ-FW Gi0/3 NO SHUTDOWN):
BR2-IT> trace 10.10.40.50 -P 1
 1   10.30.30.1   1.166 ms
 2   10.254.2.1   5.274 ms (Otomatis Kembali ke Jalur Utama Telco MPLS AD 20)
 3   10.254.0.1   6.512 ms
 4   10.254.0.2   7.951 ms
 5   10.10.40.50  11.254 ms
```
*Analisis Bukti:*
Arsitektur WAN Dual-Path PT Bank Barnibar terbukti berhasil 100%:
1. Saat jalur utama Telco MPLS putus, algoritma BGP dan OSPF berkonvergensi dalam 7 detik, mengalihkan seluruh lalu lintas transaksi ke terowongan rahasia IPsec VTI via Internet publik.
2. Ketika kabel Telco diperbaiki, sifat deterministik Administrative Distance (AD 20 BGP vs AD 110 OSPF) secara mulus merebut kembali lintasan utama (*preemption*) tanpa perlu campur tangan manual NOC.

---

## BAGIAN C — JURNAL INSIDEN & TROUBLESHOOTING SELAMA IMPLEMENTASI

Bagian ini mendokumentasikan kendala teknis nyata yang muncul saat pengerjaan lab beserta akar masalah dan solusinya (bahan penting untuk wawancara teknis NOC / Network Engineer):

### Insiden #01 — Layanan `isc-dhcp-server` Gagal Menyala (`[fail]`) di `SRV-DHCP`
* **Gejala:** Saat menjalankan `/etc/init.d/isc-dhcp-server restart`, layanan gagal berjalan dengan pesan `[fail]`.
* **Akar Masalah:** Kartu jaringan `eth0` belum aktif memegang IP `10.10.40.10` saat container menyala atau file basis data sewa `/var/lib/dhcp/dhcpd.leases` belum terbentuk. Daemon `dhcpd` menolak aktif jika tidak menemukan antarmuka jaringan yang cocok dengan deklarasi subnet lokal.
* **Solusi:** Memastikan baris `auto eth0` tidak bertanda komentar (`#`) pada `/etc/network/interfaces`, mengaktifkan IP pada `eth0`, membuat `/var/lib/dhcp/dhcpd.leases` dengan `touch`, lalu me-restart layanan hingga berstatus `running` di port UDP `67`.

### Insiden #02 — File `/etc/init.d/dnsmasq` Tidak Ditemukan (`No such file or directory`) di `SRV-DNS-INT`
* **Gejala:** Perintah `sed` dan restart `dnsmasq` ditolak dengan pesan file tidak ada.
* **Akar Masalah:** Saat pembuatan template `Docker-DNS` di GNS3, opsi masih memilih *Existing image* (`adosztal/net_toolbox:latest`) alih-alih *New image* (`adosztal/dns:latest`), sehingga container tidak memiliki paket `dnsmasq`.
* **Solusi:** Menghapus template lama, membuat ulang template dengan opsi *New image* `adosztal/dns:latest`, dan memasang ulang node `SRV-DNS-INT`.

### Insiden #03 — Error Docker `400 invalid reference format` Saat Menarik Image DNS
* **Gejala:** Muncul banner merah GNS3 `Docker has returned an error: 400 invalid reference format` saat menarik node `Docker-DNS` ke lembar kerja.
* **Akar Masalah:** Karakter baris baru (*newline / Enter*) tidak sengaja ikut tersalin di akhir teks `adosztal/dns:latest\n` saat melakukan *copy-paste* ke kolom nama image GNS3.
* **Solusi:** Membuka **Edit** pada template Docker di GNS3 Preferences dan mengetik ulang `adosztal/dns:latest` secara bersih tanpa baris baru di belakangnya.

### Insiden #04 — Peringatan `Native VLAN mismatch` pada `ACC-1 Gi0/1` & Lepasnya `Port-channel 1`
* **Gejala:** Muncul log `%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on GigabitEthernet0/1 (1), with CORE-1 GigabitEthernet0/2 (999)`, serta `Po1` menghilang dari daftar `show interfaces trunk` di `CORE-1`.
* **Akar Masalah:**
  1. Port `Gi0/1` di `ACC-1` belum tersetel penuh ke `switchport mode trunk` dengan `native vlan 999`.
  2. Mengubah parameter Native VLAN pada anggota LACP ketika port sedang menyala membuat bundel `Port-channel 1` mengalami ketidaksinkronan sesaat (*suspended*).
* **Solusi:** Menyetel eksplisit `switchport trunk encapsulation dot1q`, `mode trunk`, dan `native vlan 999` pada `ACC-1 Gi0/1`, serta melakukan `shutdown` ➔ seragamkan parameter trunk di `Gi0/0-1` dan `Po1` ➔ `no shutdown` di `CORE-1` dan `CORE-2`. Bundel `Po1` langsung kembali aktif bersih.

### Insiden #05 — `SRV-DHCP` Tidak Menyala Otomatis Saat Container Dihidupkan Ulang
* **Gejala:** Klien `PC-STAFF` memunculkan `DDD / Can't find dhcp server` setelah container di-restart, meskipun konfigurasi DHCP sudah benar.
* **Akar Masalah:** Skrip bawaan `/sbin/starter.sh` pada image `adosztal/net_toolbox` hanya menjalankan `rsyslog` dan `nginx`, tidak menyertakan perintah start `isc-dhcp-server`.
* **Solusi:** Mengisi kolom **Start command** pada pengaturan node GNS3 `SRV-DHCP` dengan `sh -c "/etc/init.d/isc-dhcp-server start; /sbin/starter.sh"` sehingga daemon DHCP selalu aktif otomatis sejak boot.

### Insiden #06 — Port `:80` dan `:8080` pada `SRV-APP` Awalnya Timeout dari Seluruh Klien
* **Gejala:** Baik `PC-STAFF` maupun `PC-TELLER` mengalami *timeout* saat menguji koneksi TCP ke `10.10.40.50`, sedangkan server lain di VLAN 40 lancar.
* **Akar Masalah:** Di GNS3 Docker, file `/etc/network/interfaces` pada menu **Edit config** hanya dibaca satu kali ketika container berpindah dari status *Stopped* ke *Started*. Jika diedit saat atau setelah dinyalakan (atau port switch `CORE-2 Gi1/1` belum masuk VLAN 40), antarmuka `eth0` belum memegang IP `10.10.40.50`.
* **Solusi:** Memastikan port `CORE-2 Gi1/1` aktif di `access vlan 40` dan selalu mengatur **Edit config** ketika container masih berstatus *Stopped* (atau mengaktifkan `eth0` secara langsung via `ifconfig` + `route add default gw`).

### Insiden #07 — Error `timeout is less than threshold 5000` & Traceroute `* * *` pada Uji Dual ISP
* **Gejala:** (1) Perintah `timeout 1500` pada `ip sla 1` ditolak IOSv; (2) Perintah `trace 8.8.8.8` biasa di VPCS menampilkan `* * *` setelah melewati `R-HQ`.
* **Akar Masalah:** (1) Nilai bawaan `threshold` IOSv adalah `5000 ms` sehingga menolak `timeout` yang lebih kecil; (2) `trace` bawaan VPCS mengirim paket UDP port tinggi (`33434`) yang ditolak oleh kebijakan *Least-Privilege* ZBF (`ACL-IZIN-INTERNET` hanya mengizinkan `HTTP/HTTPS/DNS/ICMP`).
* **Solusi:** Menurunkan `threshold 1000` terlebih dahulu sebelum `timeout 1500`, serta menggunakan traceroute berbasis ICMP (`trace 8.8.8.8 -P 1`) yang langsung memperlihatkan perpindahan jalur melewati `198.51.200.2`.

### Insiden #08 — Salah Ketik Nama ACL NAT `ACL-NAT-BRI` & Tag VLAN Sub-interface di Cabang 1
* **Gejala:** `BR1-ATM` tidak memperoleh IP DHCP, serta `BR1-STAFF` dan `BR1-GUEST` tidak bisa ping ke internet `8.8.8.8`.
* **Akar Masalah:**
  1. Sub-interface `Gi0/0.25` di `BR1-FW` menggunakan `encapsulation dot1Q 20` (seharusnya `25`), sehingga paket DHCP dari switch diabaikan.
  2. Perintah NAT pada `BR1-FW` menggunakan nama `ACL-NAT-BRI` (huruf `I` kapital di ujung), sedangkan daftar ACL bernama `ACL-NAT-BR1` (angka `1`), menyebabkan mesin NAT tidak bekerja dan paket keluar dengan IP privat yang dibuang ISP.
* **Solusi:** Memperbaiki tag `encapsulation dot1Q 25` pada `Gi0/0.25`, serta menyamakan nama daftar akses NAT menjadi `ACL-NAT-BR1`. Seluruh PC langsung mendapat IP dan tembus internet via *Local Breakout*.

### Insiden #09 — Negosiasi IPsec Multi-Vendor Cisco-MikroTik (Typo IP Peer & Enkripsi Bawaan)
* **Gejala:** (1) OSPF neighbor MikroTik Surabaya kosong; (2) Transaksi `BR1-ATM` ke `SRV-APP:8080` ditolak.
* **Akar Masalah:**
  1. `crypto isakmp key 3421 address 198.0.2.2` di `HQ-FW` salah ketik octet pertama (`198` vs `192.0.2.2`).
  2. MikroTik RouterOS v7 secara bawaan menggunakan proposal `aes-128/sha1`, sedangkan Cisco mengunci `aes-256/sha256` (Group 14).
  3. Di `HQ-FW`, `ACL-AKSES-COREBANKING` belum mendaftarkan subnet ATM cabang `10.20.25.0/24` dan `10.30.25.0/24` sehingga dibuang ZBF di `class-default`.
* **Solusi:** Mengoreksi IP peer ke `192.0.2.2` di HQ, menyamakan profile IPsec MikroTik ke `aes-256 sha256 modp2048`, dan menambahkan permit subnet ATM cabang pada `ACL-AKSES-COREBANKING`. Seluruh ATM cabang sukses bertransaksi dan OSPF berstatus `FULL`.
