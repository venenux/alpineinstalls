# Alpine3.14 + Apache2 + Mysql + Php7 + Ospos3

**Cuidado** esto es para **php 7 (php7) y sus dependencias de composer en alpine 3.14** 
si tu ejecutas mas nuevo usa [server-alpine-LAMP-ospos-320.md](server-alpine-LAMP-ospos-320.md)

* [install alpine linux](#install-alpine-linux)
* [0 - Environment](#0---setup-environment)
* [1 - Apache2](#1---apache2)
* [2 - Php](#2---php)
* [3 - DBMS Mysql](#3---databases-mysql)
* [4 - ospos](#4---ospos)
* [Como usar esta guia](#como-usar-esta-guia)
* [LICENSE](#LICENSE)

Este documento no le explicará nada que debe obedecer, como debe ser porque solo funciona y funciona muy bien, por favor si no lo sabe revisar sección [Como usar esta guia](#como-usar-esta-guia) antes de comenzar:

## Install alpine linux

```
mkdir -p /home/general/VM/alpine314 && cd /home/general/VM/alpine314

qemu-img create -f raw computerint1alpine-vitualdisk1-file.raw 6G

wget -c -t8 --no-check-certificate http://dl-cdn.alpinelinux.org/alpine/v3.14/releases/x86_64/alpine-extended-3.14.0-x86_64.iso

qemu-system-x86_64  -m 2048 -name "computerint1alpine314" \
 -cpu host -machine q35 \
 -device rtl8139,netdev=nd1 -netdev user,id=nd1,restrict=off,hostfwd=tcp::3222-:22,hostfwd=tcp::9080-:80,hostfwd=tcp::9443-:443 \
 -device virtio-keyboard -device virtio-mouse -device virtio-tablet -device virtio-vga,max_outputs=1 \
 -drive file=computerint1alpine-vitualdisk1-file.raw,format=raw \
 -cdrom alpine-extended-3.14.0-x86_64.iso -boot d
```

Cuando lo inicie, le pedirá que inicie como "root" simplemente escriba "root" e ingrese para iniciar el comando `setup-alpine`

#### the setup-alpine command procedure

* keyboard: ejemplo para latinoamerica y españa es `es` y después `es-winkeys`
* hostname: solo presione enter, es el nombre de la computadora, debe ser solo letras
* Network: seleccione `eth0` que es el cable de red y responda dhcp.
* Network (again): solo pasa si tienes wifi o segunda tarjeta.. Debe ignorarlo
* DNS Options: recomendado usar 8.8.8.8 y colocar `none` en el dominio
* Root: contraseña para la cuenta administrativa, cuídate y no la olvides
* Timezone: use UTC solo para un SO, de lo contrario `América/Panama` o algo similar
* Proxy Options: Use none si estas conectado directo a internet
* NTP Options: Use chrony que es el que esta ya disponible sin descargar
* APK mirror: Si tienes internet lento saltalo o escribe none
* User: versiones modernas permiten la creación del usuario, omitalo con `no`
* SSH Options: Use openssh el paquete que ya esta en el medio de isntalacion
* Root allow: aquí debe escribir `yes` porque aún no configuramos el usuario!
* SSH key: solo escribe aqui `none`
* Disk Options: Usaras `sda` y el disco duro entero sera borrado y usado
* Mode: Seleccionar `sys` para instalar el sistema en el disco

Luego reinicie y si está utilizando una máquina virtual cambie la línea `-boot d` to ` -boot c`

## 0 - Setup environment

```
cat > /etc/apk/repositories << EOF
http://dl-4.alpinelinux.org/alpine/v$(cat /etc/alpine-release | cut -d'.' -f1,2)/main
http://dl-4.alpinelinux.org/alpine/v$(cat /etc/alpine-release | cut -d'.' -f1,2)/community
EOF

apk update

apk add man-pages nano binutils coreutils readline \
 sed attr dialog lsof less groff wget curl terminus-font \
 file lz4 gawk tree pciutils usbutils lshw tzdata \
 zip p7zip xz tar cabextract cpio binutils lha acpi musl-locales musl-locales-lang \
 e2fsprogs btrfs-progs exfat-utils f2fs-tools dosfstools xfsprogs jfsutils \
 arch-install-scripts util-linux docs

rc-update add consolefont boot
```

## 1 - apache2

```
apk add apache2 apache2-utils apache2-error apache2-proxy-html apache2-proxy

mkdir -p /etc/skel/Devel
mkdir -p /var/www/localhost/cgi-bin /var/www/localhost/htdocs /var/log/apache2
sed -i -r 's#^Listen.*#Listen 80#g' /etc/apache2/httpd.conf
sed -i -r 's#^ServerTokens.*#ServerTokens Minimal#g' /etc/apache2/httpd.conf
chown -R apache:www-data /var/www/localhost/
chown -R apache:wheel /var/log/apache2
sed -i -r 's#.*LoadModule.*modules/mod_cgid.so.*#LoadModule cgid_module modules/mod_cgid.so#g' /etc/apache2/httpd.conf
sed -i -r 's#.*LoadModule.*modules/mod_cgi.so.*#LoadModule cgi_module modules/mod_cgi.so#g' /etc/apache2/httpd.conf
sed -i -r 's#.*LoadModule.*modules/mod_alias.so.*#LoadModule alias_module modules/mod_alias.so#g' /etc/apache2/httpd.conf
sed -i -r 's#.*ScriptAlias /cgi-bin/.*#    ScriptAlias /cgi-bin/ "/var/www/localhost/cgi-bin"#g' /etc/apache2/httpd.conf
sed -i -r 's#.*LoadModule.*modules/mod_usertrack.so.*#LoadModule usertrack_module modules/mod_usertrack.so#g' /etc/apache2/httpd.conf
sed -i -r 's#.*LoadModule.*modules/mod_userdir.so.*#LoadModule userdir_module modules/mod_userdir.so#g' /etc/apache2/httpd.conf
sed -i -r 's#public_html#Devel#g' /etc/apache2/conf.d/userdir.conf
sed -i -r 's#AllowOverride.*#AllowOverride All#g' /etc/apache2/conf.d/userdir.conf
sed -i -r 's#/usr/lib/libxml2.so.*#/usr/lib/libxml2.so.2#g' /etc/apache2/conf.d/proxy-html.conf

rc-update add apache2 default

rc-service apache2 restart

echo "it works" > /var/www/localhost/htdocs/index.html
for i in /home/*; do mkdir $i/Devel ; done
```

Para probar, abra un navegador y vaya a `http://<webserveripaddres>` pero para una forma segura o soporte SSL: 
https://venenux.github.io/alpine-wiki/#/tutorials/server-alpine-LAMP-professional-fast-forward

## 2 - PHP

```
apk add php7-opcache php7-openssl php7-json php7-bcmath php7-mbstring php7-bz2 \
 php7-ctype php7-dev php7-dom php7-enchant php7-fileinfo php7-shmop php7-simplexml php7-tidy \
 php7-tokenizer php7-sysvmsg php7-sysvsem php7-sysvshm php7-xml php7-xmlreader \
 php7-xmlwriter php7-xsl php7-zip php7-intl php7-gettext php7-pspell php7-calendar \
 php7-exif php7-gd php7-pcntl php7-gmp php7-imap php7-session php7-curl php7-pear \
 php7-phar php7-doc php7-embed php7-posix php7-fpm php7-cgi php7-dba php7-mysqli \
 php7-mysqlnd php7-odbc php7-pgsql php7-sodium php7-sqlite3 php7-apache2 \
 php7-pdo php7-pdo_dblib php7-pdo_mysql php7-pdo_odbc php7-pdo_pgsql php7-pdo_sqlite

sed -i -r 's|.*cgi.fix_pathinfo=.*|cgi.fix_pathinfo=1|g' /etc/php*/php.ini
sed -i -r 's#.*safe_mode =.*#safe_mode = Off#g' /etc/php*/php.ini
sed -i -r 's#.*expose_php =.*#expose_php = Off#g' /etc/php*/php.ini
sed -i -r 's#memory_limit =.*#memory_limit = 536M#g' /etc/php*/php.ini
sed -i -r 's#upload_max_filesize =.*#upload_max_filesize = 128M#g' /etc/php*/php.ini
sed -i -r 's#post_max_size =.*#post_max_size = 256M#g' /etc/php*/php.ini
sed -i -r 's#^file_uploads =.*#file_uploads = On#g' /etc/php*/php.ini
sed -i -r 's#^max_file_uploads =.*#max_file_uploads = 12#g' /etc/php*/php.ini
sed -i -r 's#^allow_url_fopen = .*#allow_url_fopen = On#g' /etc/php*/php.ini
sed -i -r 's#^.default_charset =.*#default_charset = "UTF-8"#g' /etc/php*/php.ini
sed -i -r 's#^.max_execution_time =.*#max_execution_time = 150#g' /etc/php*/php.ini
sed -i -r 's#^max_input_time =.*#max_input_time = 90#g' /etc/php*/php.ini
mkdir -p /var/run/php-fpm7/
sed -i -r 's|^.*listen.owner = .*|listen.owner = apache|g' /etc/php*/php-fpm.d/www.conf
sed -i -r 's|^.*listen.group = .*|listen.group = www-data|g' /etc/php*/php-fpm.d/www.conf
sed -i -r 's|^.*listen.mode = .*|listen.mode = 0660|g' /etc/php*/php-fpm.d/www.conf
chown apache:www-data /var/run/php-fpm7

sed -i -r 's|^.*listen =.*|listen = /run/php-fpm7/php-fpm.sock|g' /etc/php7/php-fpm.d/www.conf
sed -i -r 's|^pid =.*|pid = /run/php-php7/php-fpm.pid|g' /etc/php7/php-fpm.conf
rc-update add php-fpm7
rc-service php-fpm7 restart

sed -i -r 's|.*LoadModule.*modules/mod_mpm_event.so.*|LoadModule mpm_event_module modules/mod_mpm_event.so|g' /etc/apache2/httpd.conf
sed -i -r 's|.*LoadModule.*modules/mod_mpm_prefork.so.*|#LoadModule mpm_prefork_module modules/mod_mpm_prefork.so|g' /etc/apache2/httpd.conf
rm /etc/apache2/conf.d/php*.conf
cat >> /etc/apache2/conf.d/php7-fpm.conf << EOF
<FilesMatch \\.php\$>
   <If "-f %{REQUEST_FILENAME}">
    SetHandler "proxy:unix:/run/php-fpm7/php-fpm.sock|fcgi://localhost"
   </If>
</FilesMatch>
EOF
rc-update add apache2
rc-service apache2 restart

echo -e "<?php\nphpinfo( );\n?>" > /var/www/localhost/htdocs/index.php
```

## 3 - Databases mysql

```
apk add mysql mysql-client mariadb-doc mariadb-server-utils mariadb-mytop

mysql_install_db --user=mysql --datadir=/var/lib/mysql

sed -i "s|.*max_allowed_packet\s*=.*|max_allowed_packet = 100M|g" /etc/mysql/my.cnf
sed -i "s|.*max_allowed_packet\s*=.*|max_allowed_packet = 100M|g" /etc/my.cnf.d/mariadb-server.cnf
sed -i "s|.*bind-address\s*=.*|bind-address=0.0.0.0|g" /etc/mysql/my.cnf
sed -i "s|.*bind-address\s*=.*|bind-address=0.0.0.0|g" /etc/my.cnf.d/mariadb-server.cnf
sed -i "s|.*skip-networking.*|#skip-networking|g" /etc/mysql/my.cnf
sed -i "s|.*skip-networking.*|#skip-networking|g" /etc/my.cnf.d/mariadb-server.cnf
rc-update add mariadb
rc-service mariadb restart

mysqladmin -u root password root

mkdir -p /usr/share/webapps/adminer && wget https://github.com/vrana/adminer/releases/download/v4.8.1/adminer-4.8.1.php -O /usr/share/webapps/adminer/adminer-4.8.1.php

ln -s adminer-4.8.1.php /usr/share/webapps/adminer/index.php
cat >> /etc/apache2/conf.d/adminer.conf << EOF
Alias /adminer /usr/share/webapps/adminer/
<Directory /usr/share/webapps/adminer/>
    Require all granted
    DirectoryIndex index.php
</Directory>
EOF
rc-service apache2 restart
```

## 4 - ospos

```
apk add doas bash shadow shadow-uidmap doas musl-locales musl-locales-lang

cat > /etc/doas.conf << EOF
permit nopass general as root cmd apk
EOF
useradd -m -U -c "" -s /bin/bash -G wheel,input,disk,floppy,cdrom,dialout,audio,video,lp,netdev,games,users,ping,wheel general
for u in $(ls /home); do for g in disk lp floppy audio cdrom dialout video lp netdev games users ping wheel; do addgroup $u $g; done;done
echo "general:general" | chpasswd
```

¡Ahora todo utilizará el usuario "general", por lo que ejecuta `su -l general` O INICIAR SESIÓN CON EL USUARIO general!

```
doas apk add git git-doc nodejs nodejs-doc npm npm-doc

mkdir -p /home/general/Devel/opensourcepos && cd Devel/opensourcepos
wget https://github.com/opensourcepos/opensourcepos/releases/download/3.3.9/opensourcepos.3.3.9.c00ff2.zip

unzip opensourcepos.3.3.9.c00ff2.zip

mysql -u root -proot -e "CREATE SCHEMA ospos;"
mysql -u root -proot -e "CREATE USER 'admin'@'%' IDENTIFIED BY 'pointofsale';GRANT ALL PRIVILEGES ON ospos . * TO 'admin'@'%' IDENTIFIED BY 'pointofsale' WITH GRANT OPTION;FLUSH PRIVILEGES;"
mysql -u root -proot -e "CREATE USER 'admin'@'localhost' IDENTIFIED BY 'pointofsale';GRANT ALL PRIVILEGES ON ospos . * TO 'admin'@'localhost' IDENTIFIED BY 'pointofsale' WITH GRANT OPTION;FLUSH PRIVILEGES;"
mysql -u admin -ppointofsale -D ospos < database/database.sql

chown -R general:www-data /home/general/Devel/opensourcepos

rm -f /home/general/Devel/opensourcepos/.htaccess
rm -f /home/general/Devel/opensourcepos/public/.htaccess
cat > /home/general/Devel/opensourcepos/index.php << EOF
<?php 
header('Location: public/index.php', true, 301);
die();
?>
EOF
```

Edita `/home/general/Devel/opensourcepos/application/config/config.php` y coloca `$config['index_page'] = '';` a `$config['index_page'] = 'index.php';`

#### Resultados

* OSPOS: `http://<ip>/~general/opensourcepos/`
* MYSQL: `http://<ip>/adminer/`
* FILES: `/home/general/Devel/opensourcepos/`

## Como usar esta guia

Esta guía estructura todos los **comandos en bloques, cada bloque está separado 
por una línea en blanco**, entonces debe escribir **cada línea como está.. 
y presione enter para ejecutarla**  puede copiar y pegar cada bloque separado de comandos, 
pero copiar/escribir solo bloques separados por una línea vacía, ya todas las líneas 
nuevas(siguientes) se ejecutaran al persionar enter, el terminal detectará si debe ejecutarse o no.

Advertencia Algunos terminales Linux o/y Mac tienen bloqueos de corte/pegado de seguridad, por lo que si pega, la primera línea será precedida por basura, compruebe siempre el primer char de su pasta.

## Vea tambien

- 🗯 IRC
  - 💬 `##alpine_telegram_english`
  - 💬 `#alpine_linux_english`
- 📱 Telegram https://t.me/alpine_linux
  - 🇬🇧 https://t.me/alpine_linux_english
  - 🇷🇺 https://t.me/alpine_linux_pycckuu (dual english russian, low activity)
  - 🇨🇴 https://t.me/alpine_linux_espanol
  - 🇧🇬 https://t.me/alpine_linux_bulgarian (dual english bulgarian, low activity)
  - 🇨🇳 https://t.me/alpine_linux_chinese (dual english chinese, low activity)
  - 📡 https://t.me/opentechnologies (open languajes but english as main)
- Matrix
  - 👥 https://matrix.to/#/#alpine-linux-english:matrix.org

# LICENSE

**CC BY-NC-SA**: the project allows reusers to distribute, remix, adapt, and build upon the material 
in any medium or format for noncommercial purposes only, and only so long as attribution is given 
to the creators involved. If you remix, adapt, or build upon the material, you must license the modified 
material under identical terms,  includes the following elements:

* **BY**  – Credit must be given to the creator of each content respectivelly, starting at the first contributor.
* **NC**  – Only noncommercial uses of the work are permitted, with exceptions if you fill an issue here!
* **SA**  – Adaptations must be shared under the same terms, you must obey this terms and do not change it.

For more information check the [[alpine/copyright.md](../../alpine/copyright.md)](https://venenux.github.io/alpine-wiki/#/alpine/copyright)
