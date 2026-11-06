# Server DHCP Terpusat (ISC DHCP)

**Node GNS3:** `SRV-DHCP` · **IP:** `10.10.40.10` · **Image:** `adosztal/net_toolbox`

Dokumen ini adalah *configuration baseline* untuk container Linux di GNS3.
Berbeda dengan router Cisco, konfigurasi server disimpan sebagai skrip shell
yang dijalankan di dalam container (menu **Edit config** pada node GNS3).

> Sumber: [`02-BUKU-KERJA-IMPLEMENTASI.md`](../02-BUKU-KERJA-IMPLEMENTASI.md) — bagian 3.2

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

## Catatan Operasional

- Skrip ini harus dijalankan saat container dalam keadaan **Stopped** (GNS3 hanya membaca *Edit config* saat transisi Stopped → Started).
- Pastikan port switch yang terhubung sudah berada di **VLAN 40** (`access vlan 40`).
- Password / secret disamarkan menjadi `[REDACTED]`.
