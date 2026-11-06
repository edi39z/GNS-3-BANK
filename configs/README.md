# 📦 Configuration Baseline — PT Bank Barnibar v4.0

Folder ini menyimpan **konfigurasi *as-built*** tiap perangkat dalam bentuk berkas terpisah, sehingga bisa di-*diff* antar-versi (misalnya membandingkan konfigurasi sebelum dan sesudah *hardening*) tanpa harus membaca seluruh runbook.

## Daftar Berkas

| Berkas | Node GNS3 | Peran | Milestone |
|---|---|---|---|
| [`R-ISP-TELKOMSEL.cfg`](./R-ISP-TELKOMSEL.cfg) | `R-ISP-TELKOMSEL` | Router ISP-1 (Telkomsel), loopback `8.8.8.8`, peering IXP | M1 |
| [`R-ISP-BIZNET.cfg`](./R-ISP-BIZNET.cfg) | `R-ISP-BIZNET` | Router ISP-2 (Biznet), peering IXP, WAN Cabang 2 | M1 |
| [`HQ-FW.cfg`](./HQ-FW.cfg) | `HQ-FW` *(dok. lama: `R-HQ`)* | Perimeter router, NAT, Zone-Based Firewall, NTP master, syslog client | M1 + M3 + M4 |
| [`CORE-1.cfg`](./CORE-1.cfg) | `CORE-1` | Switch inti 1 — HSRP Active VLAN 10/30/40/100, STP Root Primary, LACP | M2 |
| [`CORE-2.cfg`](./CORE-2.cfg) | `CORE-2` | Switch inti 2 — HSRP Active VLAN 20/50, STP Root Primary, LACP | M2 |
| [`SW-LT1.cfg`](./SW-LT1.cfg) | `SW-LT1` *(dok. lama: `ACC-1`)* | Switch akses — trunk aman, Port-Security, BPDU Guard | M2 |
| [`SRV-DHCP.md`](./SRV-DHCP.md) | `SRV-DHCP` | Container Linux — ISC DHCP Server terpusat | M3 |
| [`SRV-DNS-INT.md`](./SRV-DNS-INT.md) | `SRV-DNS-INT` | Container Linux — dnsmasq untuk `barnibar.local` | M3 |

## Konvensi Penulisan

1. **Cisco IOS / IOSvL2** — ditulis dalam format `show running-config` yang disederhanakan (tanpa baris `Building configuration...`).
2. **Container Linux** — ditulis sebagai skrip shell yang dijalankan di dalam container, karena GNS3 menyimpan konfigurasi Docker sebagai *start command* + file konfigurasi, bukan `running-config`.
3. **Data sensitif** — seluruh password, `snmp-server community`, dan *pre-shared key* disamarkan menjadi `[REDACTED]`. Nilai asli hanya disimpan di lab lokal dan **tidak** boleh masuk repositori publik.
4. **Nama node** — sebagian dokumen lama memakai nama sebelum standar penamaan diterapkan (`R-HQ` → `HQ-FW`, `ACC-1` → `SW-LT1`). Pemetaan lengkap ada di [Bagian 3 SRS](../01-SRS-ARSITEKTUR-BARNIBAR.md).

## Cara Memakai Baseline Ini

```bash
# Melihat perbedaan konfigurasi antar dua versi
diff -u configs/HQ-FW.cfg configs/HQ-FW.cfg.baru

# Menyalin baseline ke perangkat lab (via konsol GNS3)
# 1. Buka console node di GNS3
# 2. configure terminal
# 3. tempel isi berkas .cfg
# 4. copy running-config startup-config
```

> ⚠️ **Jangan menempelkan berkas `.cfg` ini ke perangkat produksi** tanpa menyesuaikan alamat IP, kredensial, dan `hostname`. Berkas ini adalah rekaman lab, bukan templat siap pakai.
