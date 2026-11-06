# BUKU KERJA IMPLEMENTASI JARINGAN (*NETWORK IMPLEMENTATION RUNBOOK*)
## PT BANK BARNIBAR — PROYEK GNS3 v4.0

**Kode Dokumen:** `NIP-BARNIBAR-v4.0`  
**Status:** *As-Built* untuk Milestone 1–6, *Ready-to-Execute* untuk Milestone 7, Kerangka untuk Milestone 8.

> Seluruh blok konfigurasi pada Milestone 1 s.d. 3 di bawah ini adalah versi bersih (*As-Built*) yang sudah mencakup seluruh perbaikan dan penguatan keamanan Layer-2 yang telah lolos uji di GNS3.

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

# KERANGKA MILESTONE 7 S.D. MILESTONE 8 *(AKAN DIISI SAAT EKSEKUSI)*

* **Milestone 7:** Pembangunan Cabang 1 (`BR1-FW` Cisco IOSv ZBF) & Cabang 2 (`BR2-FW` MikroTik CHR RouterOS) + *Local Breakout* Wi-Fi Tamu.
* **Milestone 8:** Tulang Punggung MPLS L3VPN (`AS 65000`, VRF `BANK-BARNIBAR`, MP-BGP) sebagai jalur utama + IPsec VTI OSPF sebagai jalur cadangan otomatis.
