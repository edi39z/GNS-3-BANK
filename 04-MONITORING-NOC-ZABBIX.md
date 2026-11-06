# 📊 DESAIN MONITORING & OBSERVABILITY (NMS / NOC)
## PT BANK BARNIBAR — ZABBIX 7.0 LTS TERDISTRIBUSI

**Kode Dokumen:** `NMS-BARNIBAR-v4.0`
**Tanggal:** 2026-11-04
**Penyusun:** Edi Suranta Ginting
**Status:** *As-Built* — terpasang dan terverifikasi

---

## 1. TUJUAN & RUANG LINGKUP

Dokumen ini menjelaskan bagaimana jaringan PT Bank Barnibar **diawasi secara terus-menerus** (*continuous monitoring*) setelah seluruh milestone implementasi selesai. Monitoring bukan pelengkap, melainkan bagian dari operasional: jaringan yang tidak terukur tidak bisa diklaim sehat.

Ruang lingkup:

1. Memantau **ketersediaan** (*availability*) seluruh perangkat jaringan dan server.
2. Memantau **kesehatan** (*health*) — CPU, memori, suhu, dan *uptime* perangkat.
3. Memantau **kinerja** (*performance*) — trafik per antarmuka WAN dan terowongan VPN.
4. Menyediakan **satu layar** (*single pane of glass*) untuk tim NOC.

Di luar ruang lingkup: manajemen *ticket*, manajemen aset, dan analisis paket (*packet capture*) — ketiganya bisa ditambahkan pada fase berikutnya.

---

## 2. MENGAPA ARSITEKTUR *PROXY* (DISTRIBUTED MONITORING)?

Ada dua cara menghubungkan Zabbix Server (di Proxmox, jaringan `192.168.18.0/24`) dengan perangkat lab GNS3 (jaringan internal `10.0.0.0/8` dan `192.168.42.0/24`):

| Aspek | Opsi 1 — Rute Langsung | Opsi 2 — Zabbix Proxy ✅ **Dipilih** |
|---|---|---|
| Cara kerja | Server Zabbix diberi rute statis ke subnet lab lewat host Windows | Proxy berjalan **di dalam** jaringan lab, lalu menghubungi server lewat satu koneksi keluar |
| Arah koneksi | Server harus bisa "menjangkau masuk" ke lab | Proxy yang **menjangkau keluar** ke server (TCP 10051) |
| Ketergantungan | IP *routing* Windows + aturan *firewall* host harus dibuka | Tidak perlu mengubah routing maupun firewall host |
| Keamanan | Port SNMP internal terbuka ke jaringan kantor | Port SNMP tetap tertutup; hanya satu port keluar yang dipakai |
| Skalabilitas | Tiap site baru = tambah rute | Tiap site baru = tambah proxy (pola yang sama) |
| Kesamaan dengan produksi | Tidak lazim | **Standar** pada bank/enterprise multi-site |

**Kesimpulan:** Opsi 2 dipilih karena mencerminkan arsitektur nyata di industri: setiap lokasi (kantor pusat, cabang, atau *data center* terpisah) memiliki **proxy** yang mengumpulkan metrik secara lokal, lalu mengirim ringkasannya ke server pusat. Jika koneksi antar-site putus, proxy tetap menyimpan data (*buffering*) dan mengirimkannya saat koneksi pulih.

```text
┌──────────────────────────────────────────────────────────────────────────┐
│              ZABBIX SERVER 7.0.32 LTS — Proxmox LXC (CT 101)             │
│                        192.168.18.50                                     │
│   PostgreSQL 17 · Web GUI :80/zabbix · Trapper :10051 · Alerting         │
└───────────────────────────────▲──────────────────────────────────────────┘
                                │  Satu koneksi keluar: TCP 10051
                                │  (proxy → server, selalu inisiatif proxy)
┌───────────────────────────────┴──────────────────────────────────────────┐
│        ZABBIX PROXY "Bank-Barnibar-Proxy" — Docker di GNS3 VM            │
│                        192.168.65.128                                    │
│              SQLite3 · menyimpan buffer saat server tak terjangkau       │
└──────┬──────────────────────────┬───────────────────────────┬────────────┘
       │ SNMPv2c UDP/161          │ SNMPv2c UDP/161           │ ICMP + service check
       ▼                          ▼                           ▼
   HQ-FW, CORE-1,             BR1-FW (Cisco)            13 host server farm:
   CORE-2 (Cisco IOS)         BR2-FW (MikroTik)         SRV-APP, SRV-DB, SRV-AAA,
                                                        SRV-DNS-INT, SRV-DHCP,
                                                        SRV-NTP-LOG, WEB-DMZ,
                                                        WAN-INTERNET-DNS
```

### Prasyarat Jaringan yang Harus Disiapkan

Proxy harus bisa mencapai subnet bank yang berada di belakang GNS3 VM. Karena GNS3 VM memakai `virbr0` sebagai jembatan ke topologi lab, ditambahkan rute statis permanen di dalam GNS3 VM:

```bash
# /etc/systemd/system/bank-route.service  (rute permanen ke subnet lab)
[Unit]
Description=Static route ke subnet lab Bank Barnibar
After=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/ip route add 10.0.0.0/8 via 192.168.42.1 dev virbr0
ExecStart=/sbin/ip route add 172.16.0.0/16 via 192.168.42.1 dev virbr0
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Selain itu, *firewall* `HQ-FW` harus mengizinkan subnet monitoring `192.168.42.0/24` menyentuh zona server dan DMZ. Tanpa langkah ini, host akan tampil **merah** di Zabbix meski perangkatnya hidup — kasus nyata yang tercatat sebagai Insiden #10.

---

## 3. PERANGKAT YANG DIMONITOR

### 3.1 Grup `Bank Barnibar Network` — protokol SNMPv2c

| Host Zabbix | Alamat | Peran | Metrik Utama |
|---|---|---|---|
| `HQ-FW` | `10.10.100.1` | Perimeter firewall + router pusat | Trafik `Gi0/1` (WAN), CPU, memori, uptime, status antarmuka |
| `CORE-1` | `10.10.100.252` | Switch inti 1 | Trafik trunk, status `Port-channel 1`, CPU, uptime |
| `CORE-2` | `10.10.100.253` | Switch inti 2 | Trafik trunk, status HSRP standby, CPU, uptime |
| `BR1-FW` | `10.20.100.1` | Router cabang Medan (Cisco IOSv) | Trafik `Gi0/1` (ISP lokal), `Tu1` (VPN), CPU |
| `BR2-FW` | `10.30.100.1` | Router cabang Surabaya (MikroTik CHR) | Trafik `ether1`, `gre-ke-hq`, CPU, memori RouterOS |

Konfigurasi sisi perangkat (contoh Cisco):

```cisco
snmp-server community [REDACTED] RO
snmp-server contact "NOC BARNIBAR <noc@barnibar.my.id>"
snmp-server location "Data Center HQ Jakarta - Rak Perimeter"
```

Konfigurasi sisi MikroTik:

```routeros
/snmp community add name=[REDACTED] addresses=192.168.42.0/24 read-access=yes
/snmp set enabled=yes contact="NOC BARNIBAR" location="Cabang Surabaya"
```

### 3.2 Grup `Bank Barnibar Server Farm` — ICMP + service check

| Host Zabbix | Alamat | Peran | Yang Dicek |
|---|---|---|---|
| `SRV-APP` | `10.10.40.50` | Aplikasi Core Banking & intranet | Ping, latensi, port `8080`/`80` |
| `SRV-DB` | `10.10.41.10` | Database terisolasi (VLAN 41) | Ping, latensi |
| `SRV-AAA` | `10.10.40.40` | TACACS+ server | Ping, port `49` |
| `SRV-DNS-INT` | `10.10.40.20` | DNS internal `barnibar.local` | Ping, resolusi nama |
| `SRV-DHCP` | `10.10.40.10` | DHCP terpusat | Ping |
| `SRV-NTP-LOG` | `10.10.40.30` | Syslog & NTP | Ping |
| `WEB-DMZ` | `172.16.50.10` | Portal publik (DMZ) | Ping, port `80` |
| `WAN-INTERNET-DNS` | `8.8.8.8` | Tolok ukur kualitas internet | Ping, latensi, *packet loss* |

> **Mengapa ICMP untuk server, bukan SNMP/Agent?** Server di lab berjalan sebagai container ringan tanpa agen. ICMP + *service check* sudah cukup untuk menjawab pertanyaan utama NOC: *"layanan ini hidup atau mati, dan seberapa cepat responsnya?"* Jika nanti server dipakai produksi, Zabbix Agent 2 bisa ditambahkan tanpa mengubah struktur host.

---

## 4. APA YANG SEHARUSNYA DIMONITOR DI NOC (CHECKLIST)

Bagian ini adalah kerangka berpikir yang dipakai saat merancang dashboard — berguna sebagai *checklist* ketika menambahkan perangkat baru.

| Lapisan | Pertanyaan NOC | Metrik | Ambang Waspada |
|---|---|---|---|
| **Ketersediaan** | Perangkatnya hidup? | ICMP/SNMP `Available` | Mati > 2 menit → P1 |
| **Sumber daya** | Perangkat kepanasan / kehabisan memori? | CPU, memori, suhu | CPU > 80% selama 5 menit |
| **Antarmuka** | Ada port yang mati padahal harusnya hidup? | `ifOperStatus`, *error/discard counter* | Status `down` pada uplink |
| **Throughput** | Link WAN jenuh? | `Bits received/sent` per antarmuka | > 70% kapasitas link |
| **Kualitas** | Paket banyak yang hilang? | Latensi & *packet loss* ICMP | Loss > 1%, latensi > 100 ms |
| **Layanan** | Aplikasi bank bisa diakses? | *Service check* `TCP 8080`, `TCP 5432` | Gagal 3x berturut-turut |
| **Keamanan** | Ada percobaan akses yang ditolak? | Log `%FW-6-DROP_PKT` (via syslog) | Lonjakan drop > baseline |
| **Konfigurasi** | Ada perubahan tak terotorisasi? | Log `%SYS-5-CONFIG_I` + TACACS+ accounting | Setiap perubahan di luar jam kerja |

---

## 5. DASHBOARD NOC YANG DIBANGUN

**Nama:** `NOC Bank Barnibar — Enterprise Command Center` (Dashboard ID 401)
**Halaman 1 — Executive & Observability Wall:**

| Baris | Widget | Fungsi |
|---|---|---|
| 1 | `itemcard` Uptime HQ-FW | Berapa lama firewall pusat menyala tanpa reboot |
| 1 | `gauge` Beban CPU HQ-FW | Ambang hijau < 70%, kuning 70–90%, merah > 90% |
| 1 | `gauge` Latensi Core Banking | Ambang hijau < 15 ms, kuning < 30 ms, merah ≥ 30 ms |
| 1 | `gauge` Latensi Internet `8.8.8.8` | Deteksi degradasi ISP |
| 1 | `hostavail` Perangkat Jaringan | Ringkasan 5 host SNMP: *Available / Not available* |
| 2 | `honeycomb` Kluster Node Bank | 13 node sebagai sel heksagonal — merah jika satu mati |
| 2 | `tophosts` Server Farm | Tabel status ping + latensi seluruh server |
| 3 | `svggraph` Dual-ISP HQ | Telkomsel `Gi0/1` vs Biznet `Gi0/2` |
| 3 | `svggraph` VPN Cabang | `Tu1` Medan (Cisco) vs `gre-ke-hq` Surabaya (MikroTik) |
| 4 | `problems` Active Incidents | Papan gangguan aktif kedua grup |

**Halaman 2 — Peta Topologi Interaktif (`Map`, Sysmap ID 2):** 10 node (cloud WAN, HQ-FW, WEB-DMZ, CORE-1/2, 3 server, 2 cabang) dengan garis penghubung berwarna: hijau (link fisik), biru (terowongan IPsec), oranye (jalur DMZ), ungu (LACP antar-core). Jika perangkat mati, ikon berubah merah dan *trigger* muncul di papan insiden.

---

## 6. OTOMASI VIA API JSON-RPC

Registrasi 13 host dan pembuatan dashboard dilakukan lewat **API JSON-RPC** Zabbix, bukan klik manual — sehingga konfigurasi bisa diulang (*repeatable*) dan terdokumentasi.

**Autentikasi:**

```python
import urllib.request, json

URL = "http://192.168.18.50/zabbix/api_jsonrpc.php"

def call(method, params, auth=None):
    body = {"jsonrpc": "2.0", "method": method, "params": params, "auth": auth, "id": 1}
    req = urllib.request.Request(URL, data=json.dumps(body).encode(),
                                 headers={"Content-Type": "application/json-rpc"})
    return json.loads(urllib.request.urlopen(req, timeout=5).read())

token = call("user.login", {"username": "Admin", "password": "[REDACTED]"})["result"]
```

**Membuat grup host:**

```python
call("hostgroup.create", {"name": "Bank Barnibar Network"}, token)
call("hostgroup.create", {"name": "Bank Barnibar Server Farm"}, token)
```

**Mendaftarkan perangkat jaringan via SNMPv2c:**

```python
call("host.create", {
    "host": "HQ-FW",
    "groups": [{"groupid": "22"}],
    "interfaces": [{
        "type": 2,                 # 2 = SNMP
        "main": 1,
        "useip": 1,
        "ip": "10.10.100.1",
        "dns": "",
        "port": "161",
        "details": {"version": 2, "community": "[REDACTED]"}
    }],
    "templates": [{"templateid": "<ID template Cisco IOS by SNMP>"}]
}, token)
```

**Mendaftarkan server via ICMP:**

```python
call("host.create", {
    "host": "SRV-APP",
    "groups": [{"groupid": "23"}],
    "interfaces": [{"type": 1, "main": 1, "useip": 1,
                    "ip": "10.10.40.50", "dns": "", "port": "10050"}],
    "templates": [{"templateid": "<ID template ICMP Ping>"}]
}, token)
```

**Membuat widget dashboard:**

```python
call("dashboard.update", {
    "dashboardid": "401",
    "pages": [{
        "name": "NOC Executive & Observability Wall",
        "widgets": [{
            "type": "gauge",
            "name": "Beban CPU Router HQ-FW",
            "x": 12, "y": 0, "width": 12, "height": 5,
            "fields": [
                {"type": 4, "name": "itemid", "value": "50825"},
                {"type": 1, "name": "min", "value": "0"},
                {"type": 1, "name": "max", "value": "100"},
                {"type": 1, "name": "units", "value": "%"},
                {"type": 1, "name": "th.0.color", "value": "00D68F"},
                {"type": 1, "name": "th.0.value", "value": "0"},
                {"type": 1, "name": "th.1.color", "value": "F7B500"},
                {"type": 1, "name": "th.1.value", "value": "70"},
                {"type": 1, "name": "th.2.color", "value": "FF465C"},
                {"type": 1, "name": "th.2.value", "value": "90"}
            ]
        }]
    }]
}, token)
```

---

## 7. PROSEDUR *DRILL*: MENGUJI APAKAH MONITORING BENAR-BENAR BEKERJA

Monitoring yang tidak pernah diuji hanyalah asumsi. Prosedur ini dilakukan minimal sekali setelah perubahan besar.

| Langkah | Tindakan | Hasil yang Diharapkan |
|---|---|---|
| 1 | Buka `Dashboards → NOC Bank Barnibar` di browser | Semua indikator hijau, papan insiden kosong |
| 2 | Di GNS3, matikan antarmuka uplink `HQ-FW Gi0/1` | Dalam ≤ 2 menit muncul *problem* "Interface down" |
| 3 | Lihat halaman **Maps** | Ikon `HQ-FW` berubah merah, garis link terputus |
| 4 | Nyalakan kembali antarmuka | Problem otomatis *resolved*, ikon kembali hijau |
| 5 | Matikan container `SRV-APP` | Baris `SRV-APP` di tabel *Top Hosts* berubah merah |
| 6 | Cek `Reports → Action log` | Terbukti notifikasi/eskalasi terkirim |

> Catatan: pada lab ini notifikasi email/Telegram belum dikonfigurasi — tindakan lanjutan yang disarankan adalah menambahkan *media type* dan *action* agar insiden bisa dikirim ke ponsel petugas jaga.

---

## 8. CATATAN TROUBLESHOOTING SELAMA PEMBANGUNAN NMS

| # | Gejala | Akar Masalah | Solusi |
|---|---|---|---|
| 1 | Host SNMP berstatus merah walau perangkat hidup | `HQ-FW` ZBF memblokir subnet monitoring `192.168.42.0/24` menuju zona server/DMZ | Tambahkan `permit` subnet monitoring pada ACL `ACL-INTERNAL-KE-DMZ` dan `ACL-SENIOR-IT-KE-DB` |
| 2 | Proxy tidak bisa ping subnet `10.x` | GNS3 VM tidak punya rute balik ke subnet lab | Buat `bank-route.service` (rute statis via `virbr0`) |
| 3 | Widget *gauge* kosong | Nama item tidak cocok — Cisco menambahkan deskripsi port ke nama item: `Interface Gi0/1(WAN-PRIMARY-ISP-TELKOMSEL): Bits received` | Sesuaikan nama item di widget agar sama persis dengan nama hasil *discovery* |
| 4 | Widget *Server Farm* menampilkan `Total: 0` | Widget `hostavail` hanya menghitung host ber-*interface* Zabbix Agent/SNMP, sedangkan server dipantau via ICMP | Ganti dengan widget `tophosts` (tabel status + latensi) |
| 5 | Widget `tophosts` error `Undefined array key "base_color"` | Parameter warna kolom wajib diisi pada Zabbix 7.0 | Tambahkan `columns.N.base_color` untuk setiap kolom |
| 6 | OSPF MikroTik v7 tidak membentuk *neighbor* | Struktur perintah RouterOS v7 berubah | Gunakan `routing/ospf/instance`, `routing/ospf/area`, `routing/ospf/interface-template` |

---

## 9. RENCANA PENGEMBANGAN BERIKUTNYA

| Prioritas | Item | Manfaat |
|---|---|---|
| Tinggi | Notifikasi Telegram/Email + eskalasi berjenjang | Insiden diketahui tanpa harus membuka dashboard |
| Tinggi | Zabbix Agent 2 di server farm | Metrik CPU/RAM/disk per server, bukan hanya ping |
| Sedang | *SNMP Trap* + syslog ingestion | Deteksi peristiwa keamanan mendekati *real-time* |
| Sedang | SLA report bulanan per site | Bukti ketersediaan untuk manajemen |
| Rendah | Integrasi Grafana untuk visualisasi lanjutan | Dashboard eksekutif yang lebih fleksibel |

---

<div align="center">
<sub>Dokumen ini adalah bagian dari dokumentasi proyek <b>PT Bank Barnibar v4.0</b>. Lihat <a href="./README.md">README</a> untuk ringkasan proyek.</sub>
</div>
