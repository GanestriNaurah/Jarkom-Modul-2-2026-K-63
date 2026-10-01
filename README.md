# Jarkom-Modul-2-2026-K-63

| Nama | NRP |
| :---: | :---: |
| Nadya Putri Agustin | 5027251013 |
| Ganestri Naurah Sawestri | 5027251014 |

## Topologi dan Rancangan Alamat IP

Rootkit memiliki satu interface WAN (`eth0` ke NAT, DHCP, jaringan `192.168.122.0/24`) dan lima interface LAN, masing-masing sebagai gateway (`.1`) segmen yang dilayani.

| Segmen (Switch) | Subnet | Gateway (rootkit) | Node dan IP |
|---|---|---|---|
| Switch6 (operator) | `10.95.10.0/24` | `10.95.10.1` | alpha `.2`, beta `.3`, gamma `.4` |
| Switch4 (penyaring) | `10.95.20.0/24` | `10.95.20.1` | abbey `.2` |
| Switch1, 2, 3 (directory + repository) | `10.95.30.0/24` | `10.95.30.1` | prab `.2`, tedd `.3`, obladi `.4`, desmond `.5`, oblada `.6`, molly `.7` |
| Switch5 (penyaring) | `10.95.40.0/24` | `10.95.40.1` | penny `.2` |
| Switch7 | `10.95.50.0/24` | `10.95.50.1` | epsilon `.2`, delta `.3` |

Switch2 dan Switch3 tersambung lewat Switch1 (tidak melalui rootkit), sehingga directory (prab, tedd) dan repository (obladi, desmond, oblada, molly) berada dalam satu broadcast domain dan satu subnet.

Peran node:

| Peran | Node | Layanan |
|---|---|---|
| Router pusat | rootkit (Debian) | IP forwarding, NAT |
| Operator / klien | alpha, beta, gamma, delta, epsilon | klien uji |
| Penjaga directory | prab (master), tedd (slave) | BIND9 |
| Gerbang penyaring | abbey (Nginx), penny (Apache) | reverse proxy, redirect |
| Area vault (web statis) | obladi, desmond | Apache |
| Area core (web dinamis) | oblada, molly | Nginx + PHP-FPM |

Perintah ditulis per node sesuai script. Jangan menjalankan seluruh script di satu mesin.

---

## 1. Alamat IP dan Default Gateway

Tetapkan alamat IP dan default gateway untuk seluruh node sesuai pembagian switch, memakai prefix IP kelompok.

Konfigurasi dilakukan lewat **Edit config** tiap node di GNS3 (`/etc/network/interfaces`).

Contoh klien (alpha):
```
auto eth0
iface eth0 inet static
    address 10.95.10.2
    netmask 255.255.255.0
    gateway 10.95.10.1
```

Contoh rootkit (nama interface disesuaikan dengan port yang tersambung ke tiap switch):
```
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 10.95.10.1
    netmask 255.255.255.0
# eth2 .. eth5 dengan pola sama untuk 10.95.20.1, 10.95.30.1, 10.95.40.1, 10.95.50.1
```

Verifikasi di rootkit:
```
ip addr show
ping -c 3 8.8.8.8
ping -c 1 10.95.10.2
ping -c 1 10.95.10.3
ping -c 1 10.95.10.4
ping -c 1 10.95.20.2
ping -c 1 10.95.30.2
ping -c 1 10.95.30.3
ping -c 1 10.95.30.4
ping -c 1 10.95.30.5
ping -c 1 10.95.30.6
ping -c 1 10.95.30.7
ping -c 1 10.95.40.2
ping -c 1 10.95.50.2
ping -c 1 10.95.50.3
```

---

## 2. NAT di rootkit

Pastikan antarmuka WAN rootkit aktif dan konfigurasikan NAT agar seluruh alamat internal dapat keluar ke internet publik menggunakan IP address.

Jalankan ini pada console rootkit (Debian):
```
echo 1 > /proc/sys/net/ipv4/ip_forward
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

mkdir -p /etc/iptables
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/iptables/rules.v4
```

Pastikan nilainya 1 dan aturan MASQUERADE sudah ada:
```
cat /proc/sys/net/ipv4/ip_forward
iptables -t nat -L -v
```

Lalu dari alpha atau node lain:
```
ping -c 3 8.8.8.8
```

---

## 3. Routing Internal dan Resolver Awal

Seluruh node harus saling terhubung lintas jalur lewat rootkit. Setiap host non-router juga menambahkan resolver `192.168.122.1` agar dapat mengunduh paket dari internet sejak awal.

Jalankan di alpha, beta, gamma, delta, epsilon, abbey, prab, tedd, obladi, desmond, oblada, molly, penny:
```
echo "nameserver 192.168.122.1" > /etc/resolv.conf
```

Verifikasi di alpha atau node lain:
```
cat /etc/resolv.conf
ping -c 3 8.8.8.8          # internet
ping -c 1 google.com       # DNS
ping -c 1 10.95.10.1       # gateway
ping -c 1 10.95.30.2       # lintas segmen ke prab
ping -c 1 10.95.40.2       # lintas segmen ke penny
```

---

## 4. DNS Master (prab) dan Slave (tedd)

Pada prab, bangun zona `k63.com` authoritative dengan SOA ke `prab.k63.com`, NS untuk prab dan tedd, A record prab dan tedd, serta A record apex ke penny. Aktifkan notify dan allow-transfer ke tedd, forwarders `192.168.122.1`. Di tedd, tarik zona dari master dan jawab secara authoritative. Setelah itu ubah urutan resolver seluruh node non-router menjadi: IP prab, IP tedd, `192.168.122.1`.

#### Konfigurasi prab (master)

Instalasi dan direktori:
```
apk add bind bind-tools iproute2 nano openrc
mkdir -p /run/openrc
touch /run/openrc/softlevel
mkdir -p /var/bind/pri /var/bind/slave
chown named:named /var/bind
```

Forward zone `/var/bind/pri/db.k63.com`:
```
$TTL 604800
@ IN SOA prab.k63.com. admin.k63.com. (
        2026093001 ; Serial
        86400      ; Refresh
        7200       ; Retry
        3600000    ; Expire
        86400 )    ; Minimum TTL

@       IN NS    prab.k63.com.
@       IN NS    tedd.k63.com.
@       IN A     10.95.40.2          ; apex -> penny

prab    IN A     10.95.30.2
tedd    IN A     10.95.30.3
alpha   IN A     10.95.10.2
beta    IN A     10.95.10.3
gamma   IN A     10.95.10.4
delta   IN A     10.95.50.3
epsilon IN A     10.95.50.2
abbey   IN A     10.95.20.2
penny   IN A     10.95.40.2
obladi  IN A     10.95.30.4
desmond IN A     10.95.30.5
oblada  IN A     10.95.30.6
molly   IN A     10.95.30.7
; (record vault, core, CNAME, TXT, outbound: lihat bagian 7, 17, 19)
```

`/etc/bind/named.conf`:
```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```

`/etc/bind/named.conf.options`:
```
options {
    directory "/var/bind";
    listen-on { any; };
    listen-on-v6 { none; };
    allow-query { any; };
    recursion yes;
    allow-recursion { any; };
};
```

`/etc/bind/named.conf.local` (zona forward; zona reverse ada di bagian 8):
```
zone "k63.com" {
    type master;
    file "/var/bind/pri/db.k63.com";
    allow-transfer { 10.95.30.3; };
    also-notify { 10.95.30.3; };
    forwarders { 192.168.122.1; };
};
```

Atur resolver, hak akses, lalu jalankan named:
```
cat > /etc/resolv.conf << 'EOF'
nameserver 10.95.30.2
nameserver 10.95.30.3
nameserver 192.168.122.1
EOF

chown -R named:named /var/bind
chmod 755 /var/bind
named -u named
```

Verifikasi di prab:
```
dig @127.0.0.1 k63.com
dig @127.0.0.1 www.k63.com
dig @127.0.0.1 prab.k63.com
dig @127.0.0.1 -x 10.95.20.2
dig @127.0.0.1 -x 10.95.30.4
```

#### Konfigurasi tedd (slave)

```
apk add bind bind-tools iproute2 nano openrc
mkdir -p /run/openrc
touch /run/openrc/softlevel
mkdir -p /var/bind/slave
chown named:named /var/bind
```

`named.conf` dan `named.conf.options` sama seperti prab. Untuk `named.conf.local`:
```
zone "k63.com" {
    type slave;
    file "/var/bind/slave/db.k63.com";
    masters { 10.95.30.2; };
};
```
Zona reverse ditambahkan pada bagian 8.

```
cat > /etc/resolv.conf << 'EOF'
nameserver 10.95.30.2
nameserver 10.95.30.3
nameserver 192.168.122.1
EOF

chown -R named:named /var/bind
named -u named
```

Verifikasi di tedd:
```
dig @127.0.0.1 k63.com
dig @10.95.30.3 k63.com
ls -l /var/bind/slave/
```
Pada output `dig`, flag `aa` menunjukkan tedd menjawab secara authoritative.

#### Urutan resolver seluruh node non-router

Jalankan di alpha, beta, gamma, delta, epsilon, abbey, obladi, desmond, oblada, molly, penny:
```
cat > /etc/resolv.conf << 'EOF'
nameserver 10.95.30.2
nameserver 10.95.30.3
nameserver 192.168.122.1
EOF
```
Rootkit tidak diubah karena soal hanya mencakup node non-router.

Verifikasi di alpha:
```
cat /etc/resolv.conf
dig k63.com
dig www.k63.com
dig -x 10.95.20.2
```
Baris `SERVER:` pada output `dig` harus menunjuk `10.95.30.2` (prab).

---

## 5. Hostname Seluruh Node

Namai seluruh node sesuai glosarium, pastikan setiap host mengenali hostname secara system-wide, dan buat domain per node (contoh `alpha.k63.com`) beserta IP-nya. Pengecualian untuk node prab dan tedd.

Jalankan di tiap node (contoh alpha):
```
hostname alpha
echo "127.0.1.1 alpha.k63.com alpha" >> /etc/hosts
```
Diulang untuk rootkit, beta, gamma, delta, epsilon, abbey, penny, obladi, desmond, oblada, molly dengan nama masing-masing. Prab dan tedd dikecualikan karena A record keduanya sudah dibuat di bagian 4. Domain per node berada di forward zone (`alpha.k63.com` sampai `molly.k63.com`).

Verifikasi:
```
# di tiap node
hostname
hostname -f

# di alpha
dig alpha.k63.com
dig beta.k63.com
dig gamma.k63.com
dig delta.k63.com
dig epsilon.k63.com
dig abbey.k63.com
dig penny.k63.com
dig obladi.k63.com
dig desmond.k63.com
dig oblada.k63.com
dig molly.k63.com
```

---

## 6. Verifikasi Zone Transfer

Pastikan tedd menerima salinan zona terbaru dari prab, dan nilai serial SOA keduanya sama.

Pada prab, izin transfer diperluas ke alpha (`10.95.10.2`) agar AXFR dapat diuji dari klien. Ubah `named.conf.local`:
```
zone "k63.com" {
    type master;
    file "/var/bind/pri/db.k63.com";
    allow-transfer { 10.95.30.3; 10.95.10.2; };
    also-notify { 10.95.30.3; };
    forwarders { 192.168.122.1; };
};
# zona reverse: allow-transfer { 10.95.30.3; 10.95.10.2; }; also-notify { 10.95.30.3; };
```

Restart named:
```
killall named
named -u named
```

Verifikasi:
```
# di tedd
dig @10.95.30.2 k63.com axfr

# di alpha
dig @10.95.30.2 k63.com SOA      # serial prab
dig @10.95.30.3 k63.com SOA      # serial tedd, harus sama
dig @10.95.30.2 k63.com axfr
dig @10.95.30.2 30.95.10.in-addr.arpa axfr
dig @10.95.30.2 20.95.10.in-addr.arpa axfr
```

---

## 7. Record vault, core, dan CNAME

Tambahkan A record `vault.k63.com` (IP obladi dan desmond) dan `core.k63.com` (IP oblada dan molly). Tetapkan CNAME `www` ke `penny` dan `static` ke `abbey`. Verifikasi dari dua klien berbeda.

Tambahkan di `/var/bind/pri/db.k63.com` (naikkan serial, lalu restart named):
```
vault   IN A     10.95.30.4
vault   IN A     10.95.30.5
core    IN A     10.95.30.6
core    IN A     10.95.30.7
www     IN CNAME penny.k63.com.
static  IN CNAME abbey.k63.com.
```

Jalankan di alpha, lalu ulangi di beta:
```
dig vault.k63.com
dig core.k63.com
dig www.k63.com
dig static.k63.com
```
Hasil yang diharapkan: `vault` menjawab `10.95.30.4` dan `10.95.30.5`; `core` menjawab `10.95.30.6` dan `10.95.30.7`; `www` beralias ke penny (`10.95.40.2`); `static` beralias ke abbey (`10.95.20.2`). Hasil di alpha dan beta harus konsisten.

---

## 8. Reverse Zone

Di prab (master) deklarasikan reverse zone untuk segmen abbey, penny, area vault, dan area core. Di tedd (slave) tarik zona tersebut, isi PTR untuk keempat hostname, dan pastikan query reverse dijawab authoritative.

| Zona | Cakupan |
|---|---|
| `30.95.10.in-addr.arpa` | area vault dan core (`10.95.30.x`) |
| `20.95.10.in-addr.arpa` | abbey (`10.95.20.x`) |

#### Konfigurasi prab

`/var/bind/pri/db.30.95.10`:
```
$TTL 604800
@ IN SOA prab.k63.com. admin.k63.com. (
        2026093001 86400 7200 3600000 86400 )
@ IN NS prab.k63.com.
@ IN NS tedd.k63.com.
4 IN PTR obladi.k63.com.
5 IN PTR desmond.k63.com.
6 IN PTR oblada.k63.com.
7 IN PTR molly.k63.com.
```

`/var/bind/pri/db.20.95.10`:
```
$TTL 604800
@ IN SOA prab.k63.com. admin.k63.com. (
        2026093001 86400 7200 3600000 86400 )
@ IN NS prab.k63.com.
@ IN NS tedd.k63.com.
2 IN PTR abbey.k63.com.
```

Deklarasi di `named.conf.local` prab:
```
zone "30.95.10.in-addr.arpa" {
    type master;
    file "/var/bind/pri/db.30.95.10";
    allow-transfer { 10.95.30.3; };
    also-notify { 10.95.30.3; };
};
zone "20.95.10.in-addr.arpa" {
    type master;
    file "/var/bind/pri/db.20.95.10";
    allow-transfer { 10.95.30.3; };
    also-notify { 10.95.30.3; };
};
```

#### Konfigurasi tedd

```
zone "30.95.10.in-addr.arpa" {
    type slave;
    file "/var/bind/slave/db.30.95.10";
    masters { 10.95.30.2; };
};
zone "20.95.10.in-addr.arpa" {
    type slave;
    file "/var/bind/slave/db.20.95.10";
    masters { 10.95.30.2; };
};
```

Verifikasi:
```
# di alpha, lalu ulangi di tedd
dig -x 10.95.20.2
dig -x 10.95.30.4
dig -x 10.95.30.5
dig -x 10.95.30.6
dig -x 10.95.30.7

# zone transfer reverse (alpha)
dig @10.95.30.2 30.95.10.in-addr.arpa axfr
dig @10.95.30.2 20.95.10.in-addr.arpa axfr
```
Flag `aa` harus muncul pada jawaban.

---

## 9. Web Statis (Apache) di Area Vault

Jalankan web statis pada node area vault (Apache). Buka folder `/arsip/` dan aktifkan autoindex (directory listing). Pengujian lewat hostname, bukan IP.

Jalankan di obladi (desmond dikonfigurasi dengan pola yang sama pada bagian 11):
```
apk add apache2 apache2-ctl iproute2 nano openrc
mkdir -p /run/openrc
touch /run/openrc/softlevel

mkdir -p /arsip
echo "<h1>Static Archive - obladi</h1>" > /arsip/index.html
echo "file1" > /arsip/file1.txt
echo "file2" > /arsip/file2.txt

cat > /etc/apache2/conf.d/arsip.conf << 'EOF'
<VirtualHost *:80>
    ServerName obladi.k63.com
    DocumentRoot /arsip
    <Directory /arsip>
        Options +Indexes
        AllowOverride None
        Require all granted
    </Directory>
    LogFormat "%{X-Real-IP}i %l %u %t \"%r\" %>s %b" realip
    CustomLog /var/log/apache2/access.log realip
</VirtualHost>
EOF

killall httpd 2>/dev/null
/usr/sbin/apachectl start
```

Verifikasi:
```
curl http://localhost
curl http://obladi.k63.com
```

---

## 10. Web Dinamis (Nginx + PHP-FPM) di Area Core

Jalankan web dinamis (PHP-FPM) pada node core (Nginx). Buat aplikasi sederhana dengan halaman beranda dan profil. Terapkan rewrite sehingga `/profil` berfungsi tanpa akhiran `.php`. Pengujian lewat hostname.

Jalankan di oblada dan molly (identik):
```
apk add nginx php-fpm php-cli iproute2 nano openrc
mkdir -p /run/openrc
touch /run/openrc/softlevel

mkdir -p /var/www/core /etc/nginx/conf.d
cat > /var/www/core/index.php << 'EOF'
<?php
echo "<h1>Beranda Core</h1>";
?>
EOF
cat > /var/www/core/profil.php << 'EOF'
<?php
echo "<h1>Profil Core</h1>";
?>
EOF
```

`/etc/nginx/nginx.conf`:
```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
events { worker_connections 1024; }
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    log_format realip '$http_x_real_ip - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent"';
    include /etc/nginx/conf.d/*.conf;
}
```

`/etc/nginx/conf.d/core.conf`:
```
server {
    listen 80;
    server_name oblada.k63.com molly.k63.com;
    root /var/www/core;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }
    location /profil {
        rewrite ^/profil$ /profil.php last;
    }
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
    access_log /var/log/nginx/access.log realip;
}
```

Jalankan service:
```
rm -f /etc/nginx/http.d/default.conf
nginx -t
killall nginx php-fpm84 2>/dev/null
php-fpm84
sleep 2
nginx
```

Verifikasi:
```
# di oblada dan molly
curl http://localhost
curl http://localhost/profil

# di alpha
curl http://oblada.k63.com
curl http://oblada.k63.com/profil
curl http://molly.k63.com
curl http://molly.k63.com/profil
curl http://core.k63.com
```

---

## 11. Reverse Proxy (penny dan abbey)

Penny (Apache) menjadi reverse proxy ke area vault (obladi dan desmond). Abbey (Nginx) menjadi reverse proxy ke area core (oblada dan molly). Kedua gerbang meneruskan header `Host` dan `X-Real-IP`.

#### Abbey (Nginx)
```
apk add nginx iproute2 nano openrc
mkdir -p /etc/nginx/conf.d

cat > /etc/nginx/conf.d/abbey.conf << 'EOF'
server {
    listen 80;
    server_name abbey.k63.com static.k63.com;

    location / {
        proxy_pass http://10.95.30.6;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF
killall nginx 2>/dev/null
nginx
```

#### Penny (Apache)
Konfigurasi proxy penny diselesaikan pada bagian 14 dan 15 (`/etc/apache2/conf.d/penny.conf`), dengan `ProxyPass / http://10.95.30.4/` ke obladi.

#### Penyesuaian format log pada backend
Agar backend mencatat IP klien dari header forwarding.

obladi dan desmond (Apache):
```
LogFormat "%{X-Forwarded-For}i %l %u %t \"%r\" %>s %b" realip
CustomLog /var/log/apache2/access.log realip
```
oblada dan molly (Nginx):
```
log_format realip '$http_x_forwarded_for - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" "$http_user_agent"';
```
Lalu `apachectl restart` (Apache) atau `nginx -s reload` (Nginx).

Verifikasi di alpha:
```
curl http://www.k63.com
curl http://penny.k63.com
curl http://static.k63.com
curl http://abbey.k63.com
curl -H "X-Real-IP: 10.95.10.2" http://www.k63.com
curl -H "X-Real-IP: 10.95.10.2" http://static.k63.com
```

Cek log di backend:
```
tail -5 /var/log/apache2/access.log     # obladi, desmond
tail -5 /var/log/nginx/access.log       # oblada, molly
```

---

## 12. Basic Authentication `/admin`

Terapkan basic authentication untuk path `/admin` di penny. Pengunjung tanpa kredensial harus ditolak.

| username | password |
|---|---|
| `prabs` | `pakar_pinter_jadi_gob***` |

Jalankan di penny:
```
apk add apache2-utils
htpasswd -cb /etc/apache2/.htpasswd prabs 'pakar_pinter_jadi_gob***'

cat >> /etc/apache2/conf.d/penny.conf << 'EOF'
<Location /admin>
    AuthType Basic
    AuthName "Restricted Area"
    AuthUserFile /etc/apache2/.htpasswd
    Require valid-user
</Location>
EOF
/usr/sbin/apachectl restart
```

Verifikasi di alpha:
```
curl http://www.k63.com/admin                                       # 401 Unauthorized
curl -u 'prabs:pakar_pinter_jadi_gob***' http://www.k63.com/admin   # berhasil
```

Di penny:
```
cat /etc/apache2/.htpasswd     # prabs:$apr1$...
```

---

## 13. Redirect 301 dan 302

Akses ke IP penny dan `penny.k63.com` diarahkan permanen (301) ke `www.k63.com`. Akses ke IP abbey dan `abbey.k63.com` diarahkan sementara (302) ke `static.k63.com`.

Di penny (Apache):
```
cat > /etc/apache2/conf.d/penny-redirect.conf << 'EOF'
<VirtualHost *:80>
    ServerName penny.k63.com
    Redirect 301 / http://www.k63.com/
</VirtualHost>
EOF
/usr/sbin/apachectl restart
```

Di abbey (Nginx):
```
cat > /etc/nginx/conf.d/abbey-redirect.conf << 'EOF'
server {
    listen 80;
    server_name abbey.k63.com;
    return 302 http://static.k63.com$request_uri;
}
EOF
nginx -s reload
```

Verifikasi di alpha:
```
curl -I http://penny.k63.com     # 301, Location: http://www.k63.com/
curl -I http://abbey.k63.com     # 302, Location: http://static.k63.com/
curl -I http://10.95.40.2        # 301
curl -I http://10.95.20.2        # 302
```

---

## 14. Forwarding Header dan Real IP

Access log setiap server web di area vault dan area core harus mencatat IP asli klien, bukan IP penny atau abbey.

penny (`/etc/apache2/conf.d/penny.conf`):
```
<VirtualHost *:80>
    ServerName penny.k63.com
    ServerAlias www.k63.com

    ProxyPreserveHost On
    ProxyPass / http://10.95.30.4/
    ProxyPassReverse / http://10.95.30.4/

    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
    RequestHeader set Host "obladi.k63.com"

    ErrorLog /var/log/apache2/penny_error.log
    CustomLog /var/log/apache2/penny_access.log combined
</VirtualHost>
```

abbey memakai `proxy_set_header` untuk `Host`, `X-Real-IP`, dan `X-Forwarded-For` seperti bagian 11, lalu `nginx -s reload`.

Di backend, obladi dan desmond memakai `LogFormat "%{X-Forwarded-For}i ..."`, sedangkan oblada dan molly memakai `log_format realip '$http_x_forwarded_for ...'` (lihat bagian 11).

Verifikasi:
```
# di alpha
curl http://www.k63.com
curl http://static.k63.com

# lalu cek log: IP pada baris terakhir harus 10.95.10.2 (alpha)
tail -5 /var/log/apache2/access.log     # obladi, desmond
tail -5 /var/log/nginx/access.log       # oblada, molly
```

---

## 15. Path `/eternal` (penny) dan `/orion` (abbey)

Di penny, buat reverse proxy path `/eternal` yang menyajikan `/var/www/eternal` dan dapat merender PHP. Di abbey, buat jalur `/orion` yang menyajikan `/var/www/orion` secara statis tanpa PHP.

#### Penny
```
apk add php-fpm
mkdir -p /var/www/eternal
echo "<?php echo 'Eternal Page'; ?>" > /var/www/eternal/index.php
/usr/sbin/php-fpm84
```

Tambahan pada `penny.conf`:
```
ProxyPass /eternal !
AliasMatch ^/eternal/?$ /var/www/eternal/index.php
<Directory /var/www/eternal>
    Options +Indexes
    AllowOverride None
    Require all granted
    DirectoryIndex index.php
    <FilesMatch \.php$>
        SetHandler "proxy:fcgi://127.0.0.1:9000"
    </FilesMatch>
</Directory>
```
Lalu `apachectl restart`.

#### Abbey
```
mkdir -p /var/www/orion
echo "<h1>Orion Static</h1>" > /var/www/orion/index.html
```

Tambahan pada `abbey.conf`:
```
location = /orion {
    return 301 /orion/;
}
location /orion/ {
    root /var/www;
    index index.html;
    try_files $uri $uri/ =404;
}
```
Lalu `nginx -s reload`.

Verifikasi di alpha:
```
curl http://www.k63.com/eternal        # Eternal Page
curl http://static.k63.com/orion
curl http://static.k63.com/orion/      # Orion Static
```

---

## 16. Stress Test (ApacheBench)

Dari satu klien (alpha), lakukan 250 request dengan konkurensi 10 untuk `www.k63.com` dan `static.k63.com`, lalu tampilkan rangkuman hasilnya.

Jalankan di alpha:
```
apk add apache2-utils
ab -n 250 -c 10 http://www.k63.com/
ab -n 250 -c 10 http://static.k63.com/
```

Hasil yang dicatat:

| Metrik | www.k63.com | static.k63.com |
|---|---|---|
| Complete requests | `<isi>` | `<isi>` |
| Failed requests | `<isi>` | `<isi>` |
| Requests per second | `<isi>` | `<isi>` |
| Time per request (mean) | `<isi>` | `<isi>` |
| Transfer rate | `<isi>` | `<isi>` |

---

## 17. TXT Record

Tambahkan TXT record untuk alpha, beta, gamma, delta, epsilon. Query TXT ke nama domain mereka harus mengembalikan nama hostname masing-masing.

Tambahkan ke zone file di prab, naikkan serial, lalu restart:
```
alpha   IN TXT "alpha"
beta    IN TXT "beta"
gamma   IN TXT "gamma"
delta   IN TXT "delta"
epsilon IN TXT "epsilon"
```

Verifikasi di alpha:
```
for host in alpha beta gamma delta epsilon; do
  echo "=== $host.k63.com ==="
  dig @10.95.30.2 $host.k63.com TXT +short
done
```
Hasil yang diharapkan: `"alpha"`, `"beta"`, `"gamma"`, `"delta"`, `"epsilon"`.

---

## 18. Pengujian TTL

Ubah A record `abbey.k63.com` ke IP fiktif yang valid, naikkan serial SOA di prab dan pastikan tedd tersinkron, tetapkan TTL 15 detik pada record yang relevan, lalu verifikasi tiga fase: sebelum perubahan (IP lama), saat perubahan baru terjadi dalam 15 detik (masih IP lama karena cache), dan setelah TTL habis (IP baru).

Jalankan di prab:
```
sed -i 's/abbey IN A 10.95.20.2/abbey IN A 10.95.99.99/' /var/bind/pri/db.k63.com
sed -i 's/2026093001/2026093002/' /var/bind/pri/db.k63.com
killall named
named -u named
```

Agar TTL 15 detik berlaku pada record abbey, tuliskan TTL di record:
```
abbey 15 IN A 10.95.99.99
```

Verifikasi tiga fase di alpha:
```
echo "=== Fase 1: sebelum perubahan ==="
dig abbey.k63.com
echo "=== Fase 2: dalam 15 detik (cache) ==="
sleep 5
dig abbey.k63.com
echo "=== Fase 3: setelah TTL habis ==="
sleep 15
dig abbey.k63.com
```

Verifikasi sinkronisasi di tedd:
```
dig @10.95.30.2 k63.com SOA
dig @10.95.30.3 k63.com SOA     # serial harus sama
```

| Fase | Hasil yang diharapkan |
|---|---|
| 1. Sebelum perubahan | `10.95.20.2` |
| 2. Dalam jeda 15 detik | `10.95.20.2` (dari cache) |
| 3. Setelah TTL habis | `10.95.99.99` |

Sesuai soal 20, konfigurasi nomor 18 diabaikan pada kondisi akhir: setelah pengujian, A record abbey dikembalikan ke `10.95.20.2` dan serial dinaikkan lagi.

---

## 19. CNAME Outbound

Buat CNAME `outbound.k63.com` ke domain eksternal `http.badssl.com`, lalu `curl` ke `http://outbound.k63.com` dan pastikan outputnya sesuai dengan isi halaman `http.badssl.com`.

Tambahkan di zone file prab:
```
outbound IN CNAME http.badssl.com.
```
Naikkan serial dan restart named.

Verifikasi di alpha:
```
dig outbound.k63.com
curl http://outbound.k63.com
```

---

## 20. Autostart Service

Pastikan seluruh service dan konfigurasi tetap berjalan normal dan autostart saat node di-restart (konfigurasi nomor 18 diabaikan, koordinat kembali normal).

| Node | Perintah |
|---|---|
| prab, tedd | `rc-update add named` |
| obladi, desmond | `rc-update add apache2` |
| oblada, molly | `rc-update add nginx` dan `rc-update add php-fpm84` |
| penny | `rc-update add apache2` dan `rc-update add php-fpm84` |
| abbey | `rc-update add nginx` |
| rootkit (Debian) | `apt update && apt install -y iptables-persistent` lalu `update-rc.d netfilter-persistent enable` |

Verifikasi:
```
# node Alpine
rc-status | grep -E "named|apache2|nginx|php-fpm"

# rootkit (Debian)
ls -la /etc/rc*.d/ | grep netfilter
```
Setelah itu restart node, lalu uji ulang `dig`, `curl`, dan `getent hosts` untuk memastikan layanan kembali aktif tanpa intervensi manual.

---

# Kendala

Beberapa masalah yang muncul selama pengerjaan:

**1. Paste multi-baris di console GNS3 menghilangkan baris pertama**

Akibatnya `options {` di `named.conf` dan `$TTL` di zone file hilang, sehingga muncul error `unknown option 'directory'` dan `no TTL specified`. Solusinya, cek isi file dengan `cat -n`, lalu tulis ulang dengan `echo` per baris.

**2. Salah ketik path**

Contohnya `/var/bind/pri.db,mesh.com` dan `named-checkxone`. Selain itu, `named-checkzone` tanpa argumen file membuat console menggantung karena menunggu stdin.

**3. Record tanpa `IN A`**

Baris `rootkit 192.168.122.1` memunculkan error `unknown RR type`.

**4. Isi node hilang setelah restart**

Pada container, paket, `named.conf`, zone file, `resolv.conf`, dan `/etc/hosts` dapat kembali ke kondisi awal. Cadangan disimpan di `/etc/network/` dan dipulihkan lewat skrip.

**5. `rc-service` dan `rc-update` tidak ditemukan**

Image AlpiNet tidak membawa OpenRC aktif secara bawaan, jadi named dijalankan manual dengan `named -u named`. Untuk bagian 20, paket `openrc` diinstal dan folder `/run/openrc` disiapkan agar `rc-update` tersedia.
