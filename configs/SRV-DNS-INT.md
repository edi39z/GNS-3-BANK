# Server DNS Internal (dnsmasq)

**Node GNS3:** `SRV-DNS-INT` · **IP:** `10.10.40.20` · **Image:** `adosztal/dns`

Dokumen ini adalah *configuration baseline* untuk container Linux di GNS3.
Berbeda dengan router Cisco, konfigurasi server disimpan sebagai skrip shell
yang dijalankan di dalam container (menu **Edit config** pada node GNS3).

> Sumber: [`02-BUKU-KERJA-IMPLEMENTASI.md`](../02-BUKU-KERJA-IMPLEMENTASI.md) — bagian 3.3

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

## Catatan Operasional

- Skrip ini harus dijalankan saat container dalam keadaan **Stopped** (GNS3 hanya membaca *Edit config* saat transisi Stopped → Started).
- Pastikan port switch yang terhubung sudah berada di **VLAN 40** (`access vlan 40`).
- Password / secret disamarkan menjadi `[REDACTED]`.
