# BUKU KERJA IMPLEMENTASI JARINGAN (*NETWORK IMPLEMENTATION RUNBOOK*)
## PT BANK BARNIBAR — PROYEK GNS3 v4.0

**Kode Dokumen:** `NIP-BARNIBAR-v4.0`  
**Status:** *As-Built* — Milestone 1 sampai 9 selesai, terpasang, dan lolos UAT  
**Pembaruan Terakhir:** 2026-11-04

> Seluruh blok konfigurasi di bawah ini adalah versi bersih (*as-built*) yang sudah mencakup seluruh perbaikan dan penguatan keamanan yang telah lolos uji di GNS3. Nilai sensitif (password, *pre-shared key*, SNMP community) diganti `[REDACTED]`.

**Navigasi cepat:**
[M1 WAN & Dual ISP](#milestone-1--pondasi-wan--dual-isp-kantor-pusat-) ·
[M2 Core & L2](#milestone-2--collapsed-core-lan-hq--penguatan-keamanan-layer-2-) ·
[M3 Layanan](#milestone-3--jalur-transit--server-docker-dhcp-dns-ntp-syslog-) ·
[M4 ZBF](#milestone-4--firewall-zbf-pusat--sekat-divisi-) ·
[M5 DMZ & DB](#milestone-5--zona-publik-dmz--isolasi-basis-data-3-tier-) ·
[M6 Dual ISP](#milestone-6--redundansi-dual-isp-otomatis-di-hq-) ·
[M6.5 Lantai & AAA](#milestone-65--ekspansi-hq-gedung-3-lantai--server-aaa-tacacs-) ·
[M7 Cabang](#milestone-7--integrasi-dua-kantor-cabang-multi-vendor-) ·
[M8 WAN Ganda](#milestone-8--backbone-wan-dual-path-mpls-l3vpn--ipsec-vti-) ·
[M9 Monitoring](#milestone-9-bonus--monitoring-noc-dengan-zabbix-)

> **Cara membaca dokumen ini:** setiap milestone berisi tujuan → konfigurasi → cara verifikasi. Bukti *output* konsol asli tidak diduplikasi di sini, melainkan tersimpan di [`03-LAPORAN-UJI-DAN-JURNAL.md`](./03-LAPORAN-UJI-DAN-JURNAL.md) agar dokumen ini tetap ringkas.

---

## DAFTAR SAMBUNGAN KABEL FISIK SAAT INI (AKTUAL DI GNS3)

| Dari Perangkat | Port | Ke Perangkat | Port | Peran Jalur |
|---|---|---|---|---|
| `R-ISP-TELKOMSEL` | `Gi0/2` | `R-ISP-BIZNET` | `Gi0/2` | Peering IXP Antar-ISP (`198.18.0.0/30`) |
| `R-HQ` | `Gi0/1` | `R-ISP-TELKOMSEL` | `Gi0/0` | Link WAN Utama HQ (`203.0.113.0/29`) |
| `R-HQ` | `Gi0/2` | `R-ISP-BIZNET` | `Gi0/0` | Link WAN Cadangan HQ (`198.51.200.0/29`) |
| `R-HQ` | `Gi0/0` | `CORE-1` | `Gi0/3` | Trunk Utama Router Pusat ◄──► Switch Inti (Native 999) |
| `CORE-1` | `Gi0/0 - 1`| `CORE-2` | `Gi0/0 - 1`| Bundel LACP `Port-channel 1` Antar-Core (Native 999) |
| `CORE-1` | `Gi0/2` | `ACC-1` | `Gi0/1` | Uplink Trunk 1 ke Switch Akses (Native 999) |
| `CORE-2` | `Gi0/2` | `ACC-1` | `Gi0/2` | Uplink Trunk 2 ke Switch Akses (Native 999) |
| `CORE-1` | `Gi1/0` | `SRV-DHCP` | `eth0` | Akses Server Pembagi IP (VLAN 40) |
| `CORE-1` | `Gi1/1` | `SRV-DNS-INT` | `eth0` | Akses Server DNS Internal (VLAN 40) |
| `CORE-2` | `Gi1/0` | `SRV-NTP-LOG` | `eth0` | Akses Server Pencatat Pusat Syslog (VLAN 40) |
| `CORE-2` | `Gi1/1` | `SRV-APP` | `eth0` | Akses Server Aplikasi Kasir `:8080` & Intranet `:80` (VLAN 40) |
| `CORE-1` | `Gi1/2` | `SRV-DB` | `eth0` | Akses Brankas Basis Data `:5432` (VLAN 41 Terisolasi) |
| `R-HQ` | `Gi0/4` | `WEB-DMZ` | `eth0` | Jalur Fisik Terpisah Zona DMZ Publik (`172.16.50.0/24`) |
| `ACC-1` | `Gi0/3` | `PC-STAFF` | `e0` | Meja Kerja Karyawan (VLAN 10 + Port-Security) |
| `ACC-1` | `Gi1/0` | `PC-TELLER` | `e0` | Meja Kerja Kasir (VLAN 20 + Port-Security) |

---

# MILESTONE 1 — PONDASI WAN & DUAL ISP KANTOR PUSAT ✅

### 1.1 Konfigurasi `R-ISP-TELKOMSEL` (ISP Utama & Simulasi `8.8.8.8`)
```cisco
hostname R-ISP-TELKOMSEL
no ip domain-lookup

interface Loopback0
 description SIMULASI-DNS-PUBLIK
 ip address 8.8.8.8 255.255.255.255
 no shutdown

interface GigabitEthernet0/0
 description WAN-UTAMA-HQ
 ip address 203.0.113.2 255.255.255.248
 no shutdown

interface GigabitEthernet0/1
 description WAN-CABANG-1
 ip address 198.51.100.1 255.255.255.248
 no shutdown

interface GigabitEthernet0/2
 description PEERING-IXP-KE-BIZNET
 ip address 198.18.0.1 255.255.255.252
 no shutdown

ip route 198.51.200.0 255.255.255.248 198.18.0.2
ip route 192.0.2.0 255.255.255.248 198.18.0.2
```

### 1.2 Konfigurasi `R-ISP-BIZNET` (ISP Cadangan)
```cisco
hostname R-ISP-BIZNET
no ip domain-lookup

interface GigabitEthernet0/0
 description WAN-CADANGAN-HQ
 ip address 198.51.200.2 255.255.255.248
 no shutdown

interface GigabitEthernet0/1
 description WAN-CABANG-2-MIKROTIK
 ip address 192.0.2.1 255.255.255.248
 no shutdown

interface GigabitEthernet0/2
 description PEERING-IXP-KE-TELKOMSEL
 ip address 198.18.0.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 198.18.0.1
```

### 1.3 Konfigurasi Dasar WAN di `R-HQ`
```cisco
hostname R-HQ
no ip domain-lookup

interface GigabitEthernet0/1
 description WAN-UTAMA-ISP-TELKOMSEL
 ip address 203.0.113.1 255.255.255.248
 no shutdown

interface GigabitEthernet0/2
 description WAN-CADANGAN-ISP-BIZNET
 ip address 198.51.200.1 255.255.255.248
 no shutdown

ip route 0.0.0.0 0.0.0.0 203.0.113.2
```

---

# MILESTONE 2 — COLLAPSED CORE LAN HQ & PENGUATAN KEAMANAN LAYER-2 ✅

*(Sudah digabung dengan penguatan Native VLAN 999, penonaktifan DTP, parkir port mati VLAN 888, dan Port-Security).*

### 2.1 Konfigurasi Lengkap `CORE-1` (Termasuk Transit & Relay Milestone 3)
```cisco
hostname CORE-1
no ip domain-lookup
ip routing

vlan 10
 name LAN-STAFF
vlan 20
 name LAN-TELLER
vlan 30
 name LAN-IT-MGMT
vlan 40
 name RUANG-SERVER
vlan 50
 name WLAN-STAFF
vlan 100
 name TRANSIT-FIREWALL
vlan 888
 name BLACKHOLE-UNUSED
vlan 999
 name NATIVE-DEAD
exit

spanning-tree mode rapid-pvst
spanning-tree vlan 10,30,40,100 root primary
spanning-tree vlan 20,50 root secondary

interface range GigabitEthernet0/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport nonegotiate
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel 1
 description TRUNK-INTER-CORE-LACP
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport nonegotiate
 no shutdown
exit

interface range GigabitEthernet0/2 - 3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport nonegotiate
 no shutdown
exit

interface range GigabitEthernet1/0 - 1
 description AKSES-SERVER-DOCKER
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

interface Vlan10
 ip address 10.10.10.252 255.255.255.0
 ip helper-address 10.10.40.10
 standby version 2
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt
 no shutdown

interface Vlan20
 ip address 10.10.20.252 255.255.255.0
 ip helper-address 10.10.40.10
 standby version 2
 standby 20 ip 10.10.20.254
 standby 20 priority 90
 standby 20 preempt
 no shutdown

interface Vlan30
 ip address 10.10.30.252 255.255.255.0
 ip helper-address 10.10.40.10
 standby version 2
 standby 30 ip 10.10.30.254
 standby 30 priority 110
 standby 30 preempt
 no shutdown

interface Vlan50
 ip address 10.10.50.252 255.255.255.0
 ip helper-address 10.10.40.10
 standby version 2
 standby 50 ip 10.10.50.254
 standby 50 priority 90
 standby 50 preempt
 no shutdown

interface Vlan100
 ip address 10.10.100.252 255.255.255.0
 standby version 2
 standby 100 ip 10.10.100.254
 standby 100 priority 110
 standby 100 preempt
 no shutdown
exit

ip route 0.0.0.0 0.0.0.0 10.10.100.1
clock timezone WIB 7 0
service timestamps log datetime msec localtime show-timezone
ntp server 10.10.100.1
logging host 10.10.40.30
logging trap informational
```

### 2.2 Konfigurasi Lengkap `CORE-2` (Termasuk Transit & Relay Milestone 3)
```cisco
hostname CORE-2
no ip domain-lookup
ip routing

vlan 10
 name LAN-STAFF
vlan 20
 name LAN-TELLER
vlan 30
 name LAN-IT-MGMT
vlan 40
 name RUANG-SERVER
vlan 50
 name WLAN-STAFF
vlan 100
 name TRANSIT-FIREWALL
vlan 888
 name BLACKHOLE-UNUSED
vlan 999
 name NATIVE-DEAD
exit

spanning-tree mode rapid-pvst
spanning-tree vlan 20,50 root primary
spanning-tree vlan 10,30,40,100 root secondary

interface range GigabitEthernet0/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport nonegotiate
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel 1
 description TRUNK-INTER-CORE-LACP
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport nonegotiate
 no shutdown
exit

interface GigabitEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport nonegotiate
 no shutdown
exit

interface GigabitEthernet1/0
 description AKSES-SRV-NTP-LOG
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

interface Vlan10
 ip address 10.10.10.253 255.255.255.0
 ip helper-address 10.10.40.10
 standby version 2
 standby 10 ip 10.10.10.254
 standby 10 priority 90
 standby 10 preempt
 no shutdown

interface Vlan20
 ip address 10.10.20.253 255.255.255.0
 ip helper-address 10.10.40.10
 standby version 2
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt
 no shutdown

interface Vlan30
 ip address 10.10.30.253 255.255.255.0
 ip helper-address 10.10.40.10
 standby version 2
 standby 30 ip 10.10.30.254
 standby 30 priority 90
 standby 30 preempt
 no shutdown

interface Vlan50
 ip address 10.10.50.253 255.255.255.0
 ip helper-address 10.10.40.10
 standby version 2
 standby 50 ip 10.10.50.254
 standby 50 priority 110
 standby 50 preempt
 no shutdown

interface Vlan100
 ip address 10.10.100.253 255.255.255.0
 standby version 2
 standby 100 ip 10.10.100.254
 standby 100 priority 90
 standby 100 preempt
 no shutdown
exit

ip route 0.0.0.0 0.0.0.0 10.10.100.1
clock timezone WIB 7 0
service timestamps log datetime msec localtime show-timezone
ntp server 10.10.100.1
logging host 10.10.40.30
logging trap informational
```

### 2.3 Konfigurasi Lengkap `ACC-1` (Trunk Aman, Port-Security, Parkir Port Kosong)
```cisco
hostname ACC-1
no ip domain-lookup

vlan 10
 name LAN-STAFF
vlan 20
 name LAN-TELLER
vlan 30
 name LAN-IT-MGMT
vlan 50
 name WLAN-STAFF
vlan 888
 name BLACKHOLE-UNUSED
vlan 999
 name NATIVE-DEAD
exit

spanning-tree mode rapid-pvst

interface range GigabitEthernet0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport nonegotiate
 no shutdown
exit

interface GigabitEthernet0/3
 description AKSES-PC-STAFF
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

interface GigabitEthernet1/0
 description AKSES-PC-TELLER
 switchport mode access
 switchport access vlan 20
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

interface range GigabitEthernet0/0 , GigabitEthernet1/1 - 3 , GigabitEthernet2/0 - 3 , GigabitEthernet3/0 - 3
 description PORT-TIDAK-TERPAKAI-DIKUNCI
 switchport mode access
 switchport access vlan 888
 switchport nonegotiate
 shutdown
exit
```

---

# MILESTONE 3 — JALUR TRANSIT & SERVER DOCKER (DHCP, DNS, NTP, SYSLOG) ✅

### 3.1 Konfigurasi Sub-interface, Rute LAN, NTP Master & Syslog di `R-HQ`
```cisco
clock timezone WIB 7 0
service timestamps log datetime msec localtime show-timezone
ntp master 2

interface GigabitEthernet0/0
 description TRUNK-UTAMA-KE-CORE-1
 no ip address
 no shutdown

interface GigabitEthernet0/0.999
 description NATIVE-VLAN-AMAN-KE-CORE
 encapsulation dot1Q 999 native

interface GigabitEthernet0/0.100
 description JALUR-TRANSIT-CORE
 encapsulation dot1Q 100
 ip address 10.10.100.1 255.255.255.0

interface GigabitEthernet0/0.40
 description GERBANG-RUANG-SERVER
 encapsulation dot1Q 40
 ip address 10.10.40.1 255.255.255.0
exit

ip route 10.10.10.0 255.255.255.0 10.10.100.254
ip route 10.10.20.0 255.255.255.0 10.10.100.254
ip route 10.10.30.0 255.255.255.0 10.10.100.254
ip route 10.10.50.0 255.255.255.0 10.10.100.254

logging host 10.10.40.30
logging trap informational
```

### 3.2 Konfigurasi Server Docker `SRV-DHCP` (`10.10.40.10`)
Image: `adosztal/net_toolbox:latest`. Pastikan `auto eth0` tanpa `#` di **Edit config**, serta isi **Start command** container dengan `sh -c "/etc/init.d/isc-dhcp-server start; /sbin/starter.sh"` agar otomatis menyala setiap reboot.
```bash
echo 'INTERFACESv4="eth0"' > /etc/default/isc-dhcp-server
touch /var/lib/dhcp/dhcpd.leases

cat << 'EOF' > /etc/dhcp/dhcpd.conf
option domain-name "barnibar.local";
option domain-name-servers 10.10.40.20;
default-lease-time 3600;
max-lease-time 7200;
authoritative;

subnet 10.10.40.0 netmask 255.255.255.0 {
}

subnet 10.10.10.0 netmask 255.255.255.0 {
  range 10.10.10.50 10.10.10.150;
  option routers 10.10.10.254;
}

subnet 10.10.20.0 netmask 255.255.255.0 {
  range 10.10.20.50 10.10.20.150;
  option routers 10.10.20.254;
}

subnet 10.10.30.0 netmask 255.255.255.0 {
  range 10.10.30.50 10.10.30.150;
  option routers 10.10.30.254;
}

subnet 10.10.50.0 netmask 255.255.255.0 {
  range 10.10.50.50 10.10.50.150;
  option routers 10.10.50.254;
}
EOF

/etc/init.d/isc-dhcp-server restart
```

### 3.3 Konfigurasi Server Docker `SRV-DNS-INT` (`10.10.40.20`)
Image: `adosztal/dns:latest`. Pastikan `auto eth0` tanpa `#` di **Edit config**.
```bash
sed -i 's/--local-service//g' /etc/init.d/dnsmasq

cat << 'EOF' > /etc/dnsmasq.conf
user=root
group=root
interface=eth0
domain-needed
bogus-priv
no-resolv
server=8.8.8.8
local=/barnibar.local/
expand-hosts
domain=barnibar.local
EOF

cat << 'EOF' >> /etc/hosts
10.10.40.10 srv-dhcp.barnibar.local srv-dhcp
10.10.40.20 srv-dns.barnibar.local srv-dns
10.10.40.30 srv-log.barnibar.local srv-log
10.10.40.50 app.barnibar.local srv-app
10.10.41.10 db.barnibar.local srv-db
172.16.50.10 www.barnibar.co.id portal.barnibar.local
EOF

/etc/init.d/dnsmasq restart
```

---

# MILESTONE 4 — FIREWALL ZBF PUSAT & SEKAT DIVISI ✅

### 4.1 Deploy `SRV-APP` (`10.10.40.50` di `CORE-2 Gi1/1`)
* Di `CORE-2`: set `Gi1/1` sebagai `access vlan 40` + `portfast` + `bpduguard`.
* Di `SRV-APP` (`adosztal/net_toolbox:latest`): aktifkan Nginx di port `:80` (Intranet) dan `:8080` (Core Banking Teller).

### 4.2 ACL Isolasi Antar-Divisi di `CORE-1` & `CORE-2`
```cisco
ip access-list extended ISOLASI-TELLER
 permit ip 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255
 deny   ip 10.10.20.0 0.0.0.255 10.10.10.0 0.0.0.255 log
 deny   ip 10.10.20.0 0.0.0.255 10.10.50.0 0.0.0.255 log
 permit ip any any
exit

interface Vlan20
 ip access-group ISOLASI-TELLER in
exit

ip access-list extended ISOLASI-STAFF
 deny   ip 10.10.10.0 0.0.0.255 10.10.20.0 0.0.0.255 log
 permit ip any any
exit

interface Vlan10
 ip access-group ISOLASI-STAFF in
exit
```

### 4.3 Konfigurasi NAT & Zone-Based Firewall (ZBF) di `R-HQ`
```cisco
interface GigabitEthernet0/0.100
 ip nat inside
 zone-member security ZONE-INTERNAL
interface GigabitEthernet0/0.40
 ip nat inside
 zone-member security ZONE-SERVER
interface GigabitEthernet0/1
 ip nat outside
 zone-member security ZONE-WAN
interface GigabitEthernet0/2
 zone-member security ZONE-WAN
exit

ip access-list standard NAT-LAN-BARNIBAR
 permit 10.10.0.0 0.0.255.255
exit
ip nat inside source list NAT-LAN-BARNIBAR interface GigabitEthernet0/1 overload

ip access-list extended ACL-LAYANAN-DASAR
 permit udp any host 10.10.40.10 eq bootps
 permit udp any host 10.10.40.10 eq bootpc
 permit udp any host 10.10.40.20 eq domain
 permit tcp any host 10.10.40.20 eq domain
 permit udp any host 10.10.40.30 eq syslog
exit

ip access-list extended ACL-TELLER-KE-APP
 permit tcp 10.10.20.0 0.0.0.255 host 10.10.40.50 eq 8080
exit

ip access-list extended ACL-STAFF-KE-SERVER
 permit tcp 10.10.10.0 0.0.0.255 10.10.40.0 0.0.0.255 eq www
 permit icmp 10.10.10.0 0.0.0.255 10.10.40.0 0.0.0.255
exit

ip access-list extended ACL-IT-PENUH
 permit ip 10.10.30.0 0.0.0.255 any
exit

ip access-list extended ACL-TELLER-INTERNET-BLOKIR
 permit ip 10.10.20.0 0.0.0.255 any
exit

ip access-list extended ACL-IZIN-INTERNET
 permit tcp 10.10.10.0 0.0.0.255 any eq www
 permit tcp 10.10.10.0 0.0.0.255 any eq 443
 permit udp 10.10.10.0 0.0.0.255 any eq domain
 permit icmp 10.10.10.0 0.0.0.255 any
 permit ip 10.10.30.0 0.0.0.255 any
exit

class-map type inspect match-all CM-LAYANAN-DASAR
 match access-group name ACL-LAYANAN-DASAR
class-map type inspect match-all CM-TELLER-APP
 match access-group name ACL-TELLER-KE-APP
class-map type inspect match-all CM-STAFF-SRV
 match access-group name ACL-STAFF-KE-SERVER
class-map type inspect match-all CM-IT-PENUH
 match access-group name ACL-IT-PENUH
class-map type inspect match-all CM-BLOKIR-TELLER-WAN
 match access-group name ACL-TELLER-INTERNET-BLOKIR
class-map type inspect match-all CM-IZIN-WAN
 match access-group name ACL-IZIN-INTERNET
class-map type inspect match-any CM-SERVER-KE-WAN
 match protocol dns
 match protocol icmp
 match protocol http
 match protocol https
exit

policy-map type inspect PM-INTERNAL-KE-SERVER
 class type inspect CM-LAYANAN-DASAR
  inspect
 class type inspect CM-TELLER-APP
  inspect
 class type inspect CM-STAFF-SRV
  inspect
 class type inspect CM-IT-PENUH
  inspect
 class class-default
  drop log
exit

policy-map type inspect PM-INTERNAL-KE-WAN
 class type inspect CM-BLOKIR-TELLER-WAN
  drop log
 class type inspect CM-IZIN-WAN
  inspect
 class class-default
  drop log
exit

policy-map type inspect PM-SERVER-KE-WAN
 class type inspect CM-SERVER-KE-WAN
  inspect
 class class-default
  drop log
exit

zone-pair security ZP-INT-KE-SRV source ZONE-INTERNAL destination ZONE-SERVER
 service-policy type inspect PM-INTERNAL-KE-SERVER
zone-pair security ZP-INT-KE-WAN source ZONE-INTERNAL destination ZONE-WAN
 service-policy type inspect PM-INTERNAL-KE-WAN
zone-pair security ZP-SRV-KE-WAN source ZONE-SERVER destination ZONE-WAN
 service-policy type inspect PM-SERVER-KE-WAN
exit
```

---

# MILESTONE 5 — ZONA PUBLIK (DMZ) & ISOLASI BASIS DATA 3-TIER ✅

* `WEB-DMZ` (`172.16.50.10/24` di `R-HQ Gi0/4`, `ZONE-DMZ`): Portal publik via DNAT `ip nat inside source static tcp 172.16.50.10 80 203.0.113.1 80`.
* `SRV-DB` (`10.10.41.10/24` di `CORE-1 Gi1/2` VLAN 41, `R-HQ Gi0/0.41`, `ZONE-DB`): Listener PostgreSQL `:5432`.
* Zero-Trust DB & Anti-Pivot aktif: hanya `SRV-APP` dan `WEB-DMZ` diizinkan `inspect` ke `10.10.41.10:5432`; akses langsung dari pengguna (`ZP-BLOKIR-USER-KE-DB`) dan loncatan dari DMZ ke LAN/Server (`ZP-ANTI-PIVOT-LAN`, `ZP-ANTI-PIVOT-SRV`) ditolak dengan `drop log`.

---

# MILESTONE 6 — REDUNDANSI DUAL ISP OTOMATIS DI HQ ✅

* Titik pantau eksklusif `Loopback1` (`1.1.1.1/32`) pada `R-ISP-TELKOMSEL` dikunci rutenya lewat `203.0.113.2` (anti-*flapping*).
* Sensor `ip sla 1` (`icmp-echo 1.1.1.1 source-ip 203.0.113.1`, `threshold 1000`, `timeout 1500`, `frequency 5`) dipantau `track 1` (`delay down 6 up 6`).
* Rute utama `ip route 0.0.0.0 0.0.0.0 203.0.113.2 track 1` otomatis berpindah ke rute cadangan `ip route 0.0.0.0 0.0.0.0 198.51.200.2 10` beserta NAT berbasis `route-map` (`RM-NAT-ISP1` & `RM-NAT-ISP2`).

---

# MILESTONE 6.5 — EKSPANSI HQ GEDUNG 3 LANTAI & SERVER AAA TACACS+ ✅

* **Lantai 1 (`SW-LT1` IOSvL2):** `PC-TELLER-1` & `PC-TELLER-2` (VLAN 20), `ATM-LOBBY-01` (VLAN 25 terisolasi), dan `PC-TAMU-1` (VLAN 60). Port-Security sticky + BPDU Guard aktif.
* **Lantai 2 (`SW-LT2` Built-in):** `PC-STAFF-1` & `PC-STAFF-2` (VLAN 10), `PC-MANAGER` (VLAN 15 dengan hak ekstra laporan `:8080` & internet).
* **Lantai 3 (`SW-LT3` Built-in):** `PC-SENIOR-IT` (`10.10.30.10` - izin penuh ke DB `:5432` dan AAA Level 15), `PC-NOC-1` (`10.10.30.20` - monitoring only, diblokir dari DB & `conf t`), `PC-NOC-2` (VPCS DHCP).
* **Server `SRV-AAA` (`10.10.40.40` TACACS+ TCP 49):**
  - Autentikasi login SSH terpusat via `GRUP-TACACS` fallback local.
  - Otorisasi perintah Level 15: akun `senior_it` diizinkan penuh, akun `noc_op` hanya boleh perintah pantau (`show`, `ping`, `traceroute`).
  - Rekam audit (*Accounting*): seluruh perintah Level 1 (`show ...`), Level 15, dan sesi login/logout dicatat di `/var/log/tac_plus.acct` dengan penanda waktu WIB (`TZ=WIB-7`).

---

# MILESTONE 7 — INTEGRASI DUA KANTOR CABANG MULTI-VENDOR ✅

Tujuan: setiap cabang harus **mandiri** — punya firewall sendiri, DHCP sendiri, dan jalur internet lokal (*local breakout*) — sehingga cabang tetap bisa beroperasi meski link ke pusat sedang bermasalah.

## 7.1 Topologi & Alamat Cabang

| Elemen | Cabang 1 — Medan (`BR1-FW`) | Cabang 2 — Surabaya (`BR2-FW`) |
|---|---|---|
| Platform | Cisco IOSv 15.9(3)M6 | **MikroTik CHR RouterOS v7** |
| Link WAN lokal | `Gi0/1` → `198.51.100.2` (ISP Telkomsel) | `ether1` → `192.0.2.2` (ISP Biznet) |
| Link MPLS (M8) | `Gi0/2` → `10.254.1.2` | `ether2` → `10.254.2.2` |
| Gateway LAN | Sub-interface `Gi0/0.<vlan>` | VLAN interface di atas `bridge` |
| VLAN | 10 Staff · 25 ATM · 30 IT · 60 Guest | 10 Staff · 25 ATM · 30 IT · 60 Guest |
| Subnet | `10.20.x.0/24` | `10.30.x.0/24` |

> **Mengapa nomor VLAN 25 dan 30 dipakai untuk ATM/IT?** VLAN 25 khusus mesin ATM memudahkan penulisan ACL yang sangat ketat (hanya mengizinkan port transaksi), sedangkan VLAN 30 memisahkan jalur pemeliharaan IT dari meja pengguna biasa.

## 7.2 Konfigurasi `BR1-FW` (Cisco IOSv)

**A. Sub-interface VLAN + DHCP lokal**

```cisco
hostname BR1-FW
ip dhcp excluded-address 10.20.10.1 10.20.10.49
ip dhcp excluded-address 10.20.25.1 10.20.25.49
ip dhcp excluded-address 10.20.30.1 10.20.30.49
ip dhcp excluded-address 10.20.60.1 10.20.60.49

interface GigabitEthernet0/0
 description TRUNK-KE-BR1-SW
 no ip address
 no shutdown

interface GigabitEthernet0/0.10
 description GW-BR1-STAFF
 encapsulation dot1Q 10
 ip address 10.20.10.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/0.25
 description GW-BR1-ATM
 encapsulation dot1Q 25
 ip address 10.20.25.1 255.255.255.0

interface GigabitEthernet0/0.30
 description GW-BR1-IT
 encapsulation dot1Q 30
 ip address 10.20.30.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/0.60
 description GW-BR1-GUEST
 encapsulation dot1Q 60
 ip address 10.20.60.1 255.255.255.0
 ip nat inside

ip dhcp pool POOL-STAFF
 network 10.20.10.0 255.255.255.0
 default-router 10.20.10.1
 dns-server 10.10.40.20

ip dhcp pool POOL-ATM
 network 10.20.25.0 255.255.255.0
 default-router 10.20.25.1
 dns-server 10.10.40.20

ip dhcp pool POOL-IT
 network 10.20.30.0 255.255.255.0
 default-router 10.20.30.1
 dns-server 10.10.40.20

ip dhcp pool POOL-GUEST
 network 10.20.60.0 255.255.255.0
 default-router 10.20.60.1
 dns-server 8.8.8.8
```

**B. NAT *local breakout* + kebijakan isolasi**

```cisco
ip access-list standard ACL-NAT-BR1
 permit 10.20.10.0 0.0.0.255
 permit 10.20.30.0 0.0.0.255
 permit 10.20.60.0 0.0.0.255
! CATATAN: 10.20.25.0/24 (ATM) sengaja TIDAK didaftarkan → ATM tidak pernah di-NAT,
! sehingga tidak mungkin mencapai internet.

interface GigabitEthernet0/1
 description WAN-KE-ISP-TELKOMSEL
 ip address 198.51.100.2 255.255.255.248
 ip nat outside
 no shutdown

ip nat inside source list ACL-NAT-BR1 interface GigabitEthernet0/1 overload
ip route 0.0.0.0 0.0.0.0 198.51.100.1
```

**C. Pencegahan akses tamu ke jaringan bank**

```cisco
ip access-list extended ACL-BLOKIR-GUEST
 deny   ip 10.20.60.0 0.0.0.255 10.20.0.0 0.0.255.255
 deny   ip 10.20.60.0 0.0.0.255 10.10.0.0 0.0.255.255
 permit ip 10.20.60.0 0.0.0.255 any

interface GigabitEthernet0/0.60
 ip access-group ACL-BLOKIR-GUEST in
```

> **Kesalahan yang pernah terjadi:** nama ACL pada perintah NAT sempat ditulis `ACL-NAT-BRI` (huruf I) sedangkan ACL yang dideklarasikan `ACL-NAT-BR1` (angka 1), sehingga NAT tidak jalan dan paket keluar membawa IP privat. Selain itu tag `encapsulation dot1Q 20` pada sub-interface ATM seharusnya `25`. Keduanya tercatat sebagai Insiden #08.

## 7.3 Konfigurasi `BR2-FW` (MikroTik RouterOS v7)

**A. Bridge, VLAN, dan alamat IP**

```routeros
/interface bridge add name=br-lan
/interface vlan
add name=vlan10-staff interface=br-lan vlan-id=10
add name=vlan25-atm   interface=br-lan vlan-id=25
add name=vlan30-it    interface=br-lan vlan-id=30
add name=vlan60-guest interface=br-lan vlan-id=60

/ip address
add address=10.30.10.1/24  interface=vlan10-staff
add address=10.30.25.1/24  interface=vlan25-atm
add address=10.30.30.1/24  interface=vlan30-it
add address=10.30.60.1/24  interface=vlan60-guest
add address=192.0.2.2/29   interface=ether1 comment="WAN ISP Biznet"
```

**B. DHCP server per VLAN**

```routeros
/ip pool
add name=pool-staff ranges=10.30.10.50-10.30.10.150
add name=pool-atm   ranges=10.30.25.50-10.30.25.150
add name=pool-it    ranges=10.30.30.50-10.30.30.150
add name=pool-guest ranges=10.30.60.50-10.30.60.150

/ip dhcp-server
add name=dhcp-staff interface=vlan10-staff address-pool=pool-staff disabled=no
add name=dhcp-atm   interface=vlan25-atm   address-pool=pool-atm   disabled=no
add name=dhcp-it    interface=vlan30-it    address-pool=pool-it    disabled=no
add name=dhcp-guest interface=vlan60-guest address-pool=pool-guest disabled=no

/ip dhcp-server network
add address=10.30.10.0/24  gateway=10.30.10.1  dns-server=10.10.40.20
add address=10.30.25.0/24  gateway=10.30.25.1  dns-server=10.10.40.20
add address=10.30.30.0/24  gateway=10.30.30.1  dns-server=10.10.40.20
add address=10.30.60.0/24  gateway=10.30.60.1  dns-server=8.8.8.8
```

**C. NAT *masquerade* + firewall filter (ATM & tamu diblokir)**

```routeros
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade \
    src-address-list=LAN-BOLEH-INTERNET comment="Local breakout"

/ip firewall address-list
add address=10.30.10.0/24 list=LAN-BOLEH-INTERNET
add address=10.30.30.0/24 list=LAN-BOLEH-INTERNET
add address=10.30.60.0/24 list=LAN-BOLEH-INTERNET

/ip firewall filter
add chain=forward action=drop src-address=10.30.25.0/24 out-interface=ether1 \
    comment="ATM dilarang ke internet"
add chain=forward action=drop src-address=10.30.60.0/24 \
    dst-address=10.30.0.0/16 comment="Tamu dilarang ke LAN bank"
add chain=forward action=drop src-address=10.30.60.0/24 \
    dst-address=10.10.0.0/16 comment="Tamu dilarang ke HQ"
```

> **Perbedaan penting antar-vendor:** di Cisco, kebijakan NAT ditulis sebagai ACL yang di-*match*; di MikroTik, kebijakan ditulis sebagai daftar alamat (`address-list`) lalu dipanggil di rantai `srcnat`. Keduanya menghasilkan perilaku sama, tetapi cara membacanya berbeda — hal yang harus dipahami saat mengelola jaringan multi-vendor.

---

# MILESTONE 8 — BACKBONE WAN DUAL-PATH (MPLS L3VPN + IPsec VTI) ✅

Tujuan: menyediakan **dua jalur WAN dengan teknologi berbeda** antara kantor pusat dan kedua cabang, dengan perpindahan otomatis tanpa intervensi NOC.

## 8A. Terowongan Cadangan: IPsec VTI / GRE-IPsec + OSPF

### A.1 `HQ-FW` — dua terowongan ke cabang

```cisco
! --- Proposal IKEv2 (disamakan dengan MikroTik: aes-256 / sha256 / group 14) ---
crypto ikev2 proposal PROP-BARNIBAR
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL-BARNIBAR
 proposal PROP-BARNIBAR

crypto ikev2 keyring KR-BARNIBAR
 peer BR1-MEDAN
  address 198.51.100.2
  pre-shared-key [REDACTED]
 peer BR2-SURABAYA
  address 192.0.2.2
  pre-shared-key [REDACTED]

crypto ikev2 profile PROF-BARNIBAR
 match identity remote address 198.51.100.2 255.255.255.255
 match identity remote address 192.0.2.2 255.255.255.255
 identity local address 203.0.113.1
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-BARNIBAR

crypto ipsec transform-set TS-BARNIBAR esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile PROF-IPSEC
 set transform-set TS-BARNIBAR
 set ikev2-profile PROF-BARNIBAR

! --- Terowongan ke Medan ---
interface Tunnel1
 description VPN-KE-BR1-MEDAN
 ip address 10.255.1.1 255.255.255.252
 tunnel source GigabitEthernet0/1
 tunnel destination 198.51.100.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-IPSEC

! --- Terowongan ke Surabaya ---
interface Tunnel2
 description VPN-KE-BR2-SURABAYA
 ip address 10.255.2.1 255.255.255.252
 tunnel source GigabitEthernet0/2
 tunnel destination 192.0.2.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-IPSEC
```

### A.2 OSPF di atas terowongan

```cisco
router ospf 1
 router-id 10.255.255.1
 network 10.255.1.0 0.0.0.3 area 0
 network 10.255.2.0 0.0.0.3 area 0
 network 10.20.0.0 0.0.255.255 area 0
 network 10.30.0.0 0.0.255.255 area 0
```

### A.3 `BR2-FW` (MikroTik) — GRE + IPsec + OSPF v7

```routeros
/interface gre add name=gre-ke-hq remote-address=203.0.113.1 local-address=192.0.2.2 \
    ipsec-secret=[REDACTED] keepalive=10s
/ip address add address=10.255.2.2/30 interface=gre-ke-hq

# RouterOS v7 memisahkan OSPF menjadi instance → area → interface-template
/routing ospf instance add name=ospf-barnibar router-id=10.30.255.1
/routing ospf area add instance=ospf-barnibar name=backbone area-id=0.0.0.0
/routing ospf interface-template add area=backbone interfaces=gre-ke-hq \
    cost=10 type=ptp
/routing ospf interface-template add area=backbone interfaces=vlan10-staff,vlan25-atm,vlan30-it
```

**Hasil tahap 8A (bukti di dokumen UAT):** tetangga OSPF `FULL` di kedua sisi, transaksi ATM cabang ke `10.10.40.50:8080` sukses, dan komunikasi antar-cabang Medan ↔ Surabaya tembus lewat HQ.

## 8B. Jalur Utama: MPLS L3VPN Operator (AS 65000)

### B.1 Sisi operator — `PE-HQ` & `PE-BR`

```cisco
! --- VRF pelanggan bank ---
vrf definition BANK-BARNIBAR
 rd 65000:100
 route-target export 65000:100
 route-target import 65000:100

! --- MPLS LDP di backbone ---
mpls ldp router-id Loopback0 force

! --- MP-BGP antar-PE (distribusi rute VPN) ---
router bgp 65000
 no bgp default ipv4-unicast
 neighbor 10.255.255.22 remote-as 65000
 neighbor 10.255.255.22 update-source Loopback0
 !
 address-family vpnv4
  neighbor 10.255.255.22 activate
  neighbor 10.255.255.22 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf BANK-BARNIBAR
  neighbor 10.254.0.2 remote-as 65100
  neighbor 10.254.0.2 activate
 exit-address-family
```

### B.2 Sisi bank (CE) — eBGP + timer cepat

```cisco
router bgp 65100
 bgp log-neighbor-changes
 timers bgp 3 9
 neighbor 10.254.0.1 remote-as 65000
 neighbor 10.254.0.1 advertisement-interval 0
!
address-family ipv4
 neighbor 10.254.0.1 activate
 network 10.10.0.0 mask 255.255.0.0
exit-address-family
```

AS tiap lokasi: HQ `65100`, Medan `65200`, Surabaya `65300`.

### B.3 Hasil: BGP AD 20 mengalahkan OSPF AD 110

```text
HQ-FW# show ip route bgp
B  10.20.10.0/24 [20/0] via 10.254.0.1, GigabitEthernet0/3
B  10.30.10.0/24 [20/0] via 10.254.0.1, GigabitEthernet0/3

[admin@BR2-FW] > /ip/route/print where dst-address in 10.10.40.0/24
0 DAb 10.10.40.0/24  10.254.2.1%ether2      20    ← BGP, dipakai
1 D o 10.10.40.0/24  10.255.2.1%gre-ke-hq  110    ← OSPF, cadangan
```

## 8C. Uji Bencana & Pemulihan (*Disaster Recovery Test*)

| Skenario | Tindakan | Hasil |
|---|---|---|
| Normal | — | Jalur lewat `10.254.2.1` (MPLS), RTT ≈ 8 ms |
| Bencana | `HQ-FW(config)# interface Gi0/3` → `shutdown` | Dalam ≈ 7 detik jalur berpindah ke `10.255.2.1` (IPsec VTI), **loss 0%** |
| Pemulihan | `no shutdown` | Rute BGP kembali, jalur otomatis memakai MPLS lagi tanpa perintah tambahan |

---

# MILESTONE 9 (BONUS) — MONITORING NOC DENGAN ZABBIX ✅

Setelah jaringan selesai, dibangun lapisan observability agar kondisi jaringan bisa dibuktikan, bukan diasumsikan.

| Komponen | Detail |
|---|---|
| Zabbix Server | 7.0.32 LTS di Proxmox LXC (CT 101) — `192.168.18.50`, PostgreSQL 17 |
| Zabbix Proxy | Container Docker di GNS3 VM (`192.168.65.128`), koneksi keluar TCP 10051 |
| Protokol | SNMPv2c pada 5 perangkat jaringan + ICMP/service check pada 8 host server |
| Otomasi | Registrasi host & pembuatan dashboard via **JSON-RPC API** |
| Dashboard | `NOC Bank Barnibar — Enterprise Command Center` (2 halaman + peta topologi) |

Langkah instalasi lengkap, alasan pemilihan arsitektur proxy, checklist metrik, prosedur *drill*, dan troubleshooting widget ada di **[`04-MONITORING-NOC-ZABBIX.md`](./04-MONITORING-NOC-ZABBIX.md)**.

---

# LAMPIRAN — URUTAN EKSEKUSI YANG DISARANKAN

Bila topologi dibangun ulang dari nol, urutan berikut terbukti paling sedikit menimbulkan masalah:

1. **M1** — Nyalakan ISP & WAN HQ, pastikan ping `8.8.8.8` tembus sebelum lanjut.
2. **M2** — Bangun core & akses; verifikasi `Po1`, HSRP, dan STP **sebelum** memasang server.
3. **M3** — Nyalakan server DHCP/DNS/NTP/syslog; uji DORA dari satu PC.
4. **M4–M5** — Pasang ZBF bertahap: mulai dari `inspect` longgar, verifikasi, lalu perketat.
5. **M6** — Uji failover ISP dengan mencabut link, bukan dengan mengubah konfigurasi.
6. **M6.5** — Tambah lantai & TACACS+; uji otorisasi dengan akun `noc_op`.
7. **M7** — Bangun cabang satu per satu; jangan menyalakan dua cabang bersamaan saat pertama kali (mempermudah isolasi kesalahan).
8. **M8** — Bangun jalur cadangan (IPsec) **lebih dulu**, pastikan OSPF `FULL`, baru pasang jalur utama MPLS.
9. **M9** — Baru pasang monitoring setelah jaringan stabil, agar alarm yang muncul benar-benar berarti.
