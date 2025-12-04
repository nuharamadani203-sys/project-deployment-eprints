# Cara Instal Eprints di Linux Mint

## 1. Update Server dan Instal Paket
Konek ke server dan lakukan update:
```bash
apt update
apt upgrade -y
```
Install semua paket yang dibutuhkan oleh EPrints:
```bash
apt install perl libncurses6 libselinux1 apache2 libapache2-mod-perl2 libxml-libxml-perl \
libunicode-string-perl libterm-readkey-perl libmime-lite-perl libmime-types-perl libdigest-sha-perl \
libdbd-mysql-perl libxml-parser-perl libxml2-dev libxml-twig-perl libarchive-any-perl libjson-perl \
liblwp-protocol-https-perl libtext-unidecode-perl lynx wget ghostscript poppler-utils antiword elinks \
texlive-base texlive-base-bin psutils imagemagick adduser tar gzip unzip libsearch-xapian-perl \
libtex-encode-perl libio-string-perl libdbd-mysql-perl git xpdf python3-html2text make -y
```

## 2. Instal Mysql
Install MySQL server dan client:
```bash
apt install mysql-server mysql-client -y
```
Login ke MySQL:
```bash
sudo mysql -u root -p
```

## 3. Membuat User Eprints di Mysql
```bash
CREATE USER 'eprints'@'localhost' IDENTIFIED by 'changeme';
GRANT ALL PRIVILEGES ON *.* TO 'eprints'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit
```

## 4. Membuat User di Sistem
```bash
adduser eprints
```
Membuka file /etc/apache2/envvars:
```bash
nano /etc/apache2/envvars
```
Update konfigurasinya agar menggunakan user dan group eprints:
```bash
export APACHE_RUN_USER=eprints
export APACHE_RUN_GROUP=eprints
```
Restart Apache:
```bash
systemctl restart apache2
```

## 5. Download Source EPrints
Membuat direktori eprints3 untuk penyimpanan source EPrints:
```bash
mkdir /opt/eprints3
chown eprints:eprints /opt/eprints3
chmod 2775 /opt/eprints3
```
Download source EPrints dari GitHub:
```bash
su -l eprints
git clone https://github.com/eprints/eprints3.4.git /opt/eprints3
```
Gunakan EPrints v3.4.7:
```bash
cd /opt/eprints3
git checkout tags/v3.4.5
```
## 6. Membuat Repository
Membuat repository dengan flavour publication:
```bash
bin/epadmin create pub
```
> Masukkan Archive Id
> Masukkan hostname untuk repository
> Masukan hostname untuk HTTPS
> Masukkan email untuk akun administrator
> Masukkan nama repository
> Masukkan nama organisasi
> Masukkan user MySQL yang sudah dibuat sebelumnya
> Membuat akun administrator

## 7. Konfigurasi Apache
Beralih dari user eprints ke user root:
```bash
exit
```
Menambahkan ServerName Public_IP_Address ke dalam file konfigurasi default virtual host:
```bash
ip=$(dig +short myip.opendns.com @resolver1.opendns.com -4)
sed -i "s/#ServerName www.example.com/ServerName ${ip}/g" /etc/apache2/sites-available/000-default.conf
```
Menambahkan konfigurasi eprints ke dalam apache.conf:
```bash
echo "Include /opt/eprints3/cfg/apache.conf" >> /etc/apache2/apache2.conf
```
Restart Apache:
```bash
systemctl restart apache2
```
Memeriksa apakah status UFW firewall sedang aktif:
```bash
ufw status
```
Jika UFW aktif, ijinkan trafik ke port HTTP dan HTTPS:
```bash
ufw allow http
ufw allow https
```
Browse hostname atau subdomain untuk menguji apakah EPrints sudah dapat diakses:
<img width="1080" height="595" alt="image" src="https://github.com/user-attachments/assets/c0866bcb-9b6c-4faa-b0a4-6de9f54a2b2e" />
<img width="1920" height="1080" alt="login eprints" src="https://github.com/user-attachments/assets/ab08d485-f6aa-403e-9bff-ca4ced3cf420" />
a2b2e" />
<img width="1920" height="1080" alt="udah login" src="https://github.com/user-attachments/assets/a1cc48b9-b0b1-4726-a55c-5c3594f19299" />

## 8. Konfigurasi HTTPS
Install certbot:
```bash
apt install certbot python3-certbot-apache -y
```
Request sertifikat SSL untuk subdomain:
```bash
certbot --non-interactive -m admin@aminlabs.my.id --agree-tos --no-eff-email --apache certonly -d repo.aminlabs.my.id
```
Beralih ke user eprints:
```bash
su -l eprints
```
Membuat direktori ssl di dalam direktori archive:
```bash
mkdir /opt/eprints3/archives/repo/ssl
```
Membuat file securevhost.conf di dalam direktori ssl:
```bash
nano /opt/eprints3/archives/repo/ssl/securevhost.conf
```
Masukkan konfigurasi virtual host berikut untuk protokol HTTPS:
```bash
<VirtualHost *:443>

 ServerName repo.aminlabs.my.id:443

 ErrorLog /var/log/apache2/repo.aminlabs.my.id_error.log
 TransferLog /var/log/apache2/repo.aminlabs.my.id_access.log
 LogLevel warn

 SSLEngine on
 SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
 SSLHonorCipherOrder on
 SSLCipherSuite ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256

 SSLCertificateFile /etc/letsencrypt/live/repo.aminlabs.my.id/fullchain.pem
 SSLCertificateKeyFile /etc/letsencrypt/live/repo.aminlabs.my.id/privkey.pem
 SSLCertificateChainFile /etc/letsencrypt/live/repo.aminlabs.my.id/chain.pem

 SetEnvIf User-Agent ".*MSIE.*" \
 nokeepalive ssl-unclean-shutdown \
 downgrade-1.0 force-response-1.0

 CustomLog /var/log/apache2/repo.aminlabs.my.id_access.log \
 "%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \"%r\" %b"


 Include /opt/eprints3/cfg/apache_ssl/repo.conf
 
 PerlTransHandler +EPrints::Apache::Rewrite

</VirtualHost>
```
Generate ulang konfigurasi Apache untuk EPrints:
```bash
/opt/eprints3/bin/generate_apacheconf --system --replace
```
Pesan yang ditampilkan jika generate ulang konfigurasi Apache berhasil:
```bash
Wrote /opt/eprints3/cfg/apache.conf
Wrote /opt/eprints3/cfg/apache_ssl.conf
Wrote /opt/eprints3/cfg/perl_module_isolation.conf
Wrote /opt/eprints3/cfg/perl_module_isolation_vhost.conf
Wrote /opt/eprints3/cfg/apache/repo.conf
Wrote /opt/eprints3/cfg/apache_ssl/repo.conf

You must restart apache for any changes to take effect!
```
Beralih ke user root:
```bash
exit
```
Menambahkan konfigurasi virtual host HTTPS securevhost.conf ke dalam file apache.conf:
```bash
echo "Include /opt/eprints3/archives/repo/ssl/securevhost.conf" >> /etc/apache2/apache2.conf
```
Mengaktifkan modul SSL dan restart Apache:
```bash
a2enmod ssl
systemctl restart apache2
```
Browse subdomain untuk menguji apakah konfigurasi HTTPS sudah berhasil, dapat diakses. Selain itu, lakukan juga pengujian login
