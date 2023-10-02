# squidsslbump
langkah instalasi SSL Bump pada Squid di Ubuntu

Awalnya adalah melakukan instalasi squid pada Ubuntu

#install squid
apt-get install squid
systemctl enable squid
service squid start
service squid status
service squid stop
 
#siapkan lingkungan pengembangan
apt-get install build-essential fakeroot devscripts gawk gcc-multilib dpatch
apt-get build-dep squid
apt-get build-dep openssl
apt-get source squid
apt-get install libssl-dev

#tambahkan konfigurasi pada saat kompilasi untuk mendukung --enable-ssl-crtd dan --with-openssl
nano squid-5.7/debian/rules

tambahkan
--with-default-user=proxy \
--enable-ssl-crtd \
--with-openssl

#compile packet
debuild -us -uc -b

#jika mendapatkan error, maka perlu tambahkan deb-src
E: You must put some 'deb-src' URIs in your sources.list
software-properties-gtk

#install
make install

#pastikan hasil compile ada --enable-ssl-crtd dan --with-openssl
squid -v

#persiapkan self sign certificate
cd /etc/squid
mkdir key
cd key

#buat file .key dan .crt
#key untuk 10tahun
openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 -keyout bump.key -out bump.crt
openssl dhparam -outform PEM -out bump_dhparam.pem 2048

#setting security untuk file certificate
chown proxy:proxy bump*
chmod 400 bump*

#buat file .der yang nantinya diimport ke browser yang ada diklien
openssl x509 -in bump.crt -outform DER -out bump.der

#buat directory untuk database sertifikat
mkdir -p /var/lib/squid
rm -rf /var/lib/squid/ssl_db
/usr/lib/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 20MB
chown -R proxy:proxy /var/lib/squid

#Untuk bypass domain
pico ssl_exclude_domains.conf
.example.com
.example.org
.example.net

#Untuk bypass src ip address
pico ssl_exclude_ips.conf
1.1.1.1
2.2.2.2
3.3.3.3

pico squid.conf
#tambahkan baris berikut pada baris sebelum http_access ...
acl intermediate_fetching transaction_initiator certificate-fetching
http_access allow intermediate_fetching

#tambahkan baris berikut pada akhir file
sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/lib/squid/ssl_db -M 20MB
sslproxy_cert_error allow all

acl ssl_exclude_domains ssl::server_name "/etc/squid/ssl_exclude_domains.conf"
acl ssl_exclude_ips     src              "/etc/squid/ssl_exclude_ips.conf"
acl ssl_include_ips     src              "/etc/squid/ssl_include_ips.conf"

#abaikan local host
ssl_bump splice localhost
#pastikan daftar ip pada file ssl_include_domain.conf
ssl_bump stare ssl_include_ips
#abaikan daftar domain pada file ssl_exclude_domains.conf
ssl_bump splice ssl_exclude_domains
#abaikan daftar ip pada file ssl_exclude_domains.conf
ssl_bump splice ssl_exclude_ips
ssl_bump stare all

#ganti baris ini untuk http_port
http_port 0.0.0.0:3128 tcpkeepalive=60,30,3 ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=20MB tls-cert=/etc/squid/key/bump.crt tls-key=/etc/squid/key/bump.key cipher=HIGH:MEDIUM:!LOW:!RC4:!SEED:!IDEA:!3DES:!MD5:!EXP:!PSK:!DSS options=NO_TLSv1,NO_SSLv3,SINGLE_DH_USE,SINGLE_ECDH_USE tls-dh=prime256v1:/etc/squid/key/bump_dhparam.pem

sudo service squid start
