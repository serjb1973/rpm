## Домашнее задание: Создать свой RPM пакет.
### Цель:
#### Научиться самостоятельно создавать RPM пакеты
#### Создать свой репо и разместить в нём RPM пакеты [http://158.160.10.137]

### Для выполнения работы используются следующие инструменты:
- VirtualBox - среда виртуализации, позволяет создавать и выполнять виртуальные машины;
- Vagrant - ПО для создания и конфигурирования виртуальной среды. В данном случае в качестве среды виртуализации используется VirtualBox;
- Git - система контроля версий
- Yandex Cloud - облако yandex для размещения репозитария

### Аккаунты:
- GitHub - https://github.com/

## Создание пакета postgres 15 с блоком 32KB (по дефолту 8KB)
### Создаём виртуалку для сбора пакета
```
vagrant up
```
### Подготовка системы
```
yum install gcc wget -y
yum install rpm-build -y
yum install bison flex perl-ExtUtils-Embed perl python-devel tcl-devel readline-devel openssl-devel krb5-devel e2fsprogs-devel libxml2-devel libxslt-devel pam-devel libuuid-devel openldap-devel openjade opensp docbook-dtds -y
yum install libicu-devel gettext systemd-devel -y
yum install git -y
useradd postgres
```
### Готовим каталоги юзера postgres
```
sudo -iu postgres
git clone git://git.postgresql.org/git/pgrpms.git
mkdir -p -v ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
cp ~/pgrpms/rpm/redhat/15/postgresql-15/EL-7/* ~/rpmbuild/SOURCES
cd ~/rpmbuild/SOURCES
cp ~/pgrpms/rpm/redhat/main/non-common/postgresql-17/main/postgresql-17-A4.pdf ./postgresql-15-A4.pdf
wget --no-check-certificate https://ftp.postgresql.org/pub/source/v15.5/postgresql-15.5.tar.bz2
```
### Правим postgresql-15.spec - строка pgmajorversion и указание на 32KB блок
1)
```
# These are macros to be used with find_lang and other stuff
%global packageversion 150
%global pgpackageversion 15
%global pgmajorversion 15
%global prevmajorversion 14
%global sname postgresql
%global pgbaseinstdir   /usr/pgsql-%{pgmajorversion}
```
2)
```
./configure --enable-rpath \
        --prefix=%{pgbaseinstdir} \
        --includedir=%{pgbaseinstdir}/include \
        --mandir=%{pgbaseinstdir}/share/man \
        --datadir=%{pgbaseinstdir}/share \
        --libdir=%{pgbaseinstdir}/lib \
        --with-blocksize=32 \
        --with-wal-blocksize=32 \
        --with-lz4 \
```
### Сборка пакетов
```
rpmbuild -bb postgresql-15.spec
```
## Создание репозитария
### Хост в облаке yandex 158.160.10.137
### Добавляем репозитарии
```
yum install -y epel-release
yum install -y nginx
yum install -y firewalld
```
### Поднимаем и настраиваем repo+nginx+firewall
```
systemctl start firewalld
systemctl status firewalld
firewall-cmd --zone=public --permanent --add-service=http
firewall-cmd --zone=public --permanent --add-service=https
firewall-cmd --reload
firewall-cmd --state
systemctl status nginx
systemctl enable nginx
systemctl start nginx
mkdir /usr/share/nginx/html/repo
cp /home/serjb/rpm/* /usr/share/nginx/html/repo
cd /usr/share/nginx/html/repo
yum install -y createrepo
createrepo /usr/share/nginx/html/repo/
systemctl restart nginx
...
vi /etc/nginx/nginx.conf
    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html/repo;
        location / {
                index  index.php index.html index.htm;
                autoindex on;   #enable listing of directory index
        }
...
nginx -t
nginx -s reload
```
### Локальная проверка содержимого
```
lynx http://localhost/
```
## Инсталляция на хост vagrant
### Создаём виртуалку для сбора пакета
```
vagrant up
```
### Добавляем репозитарии
```
cat >> /etc/yum.repos.d/otus.repo << EOF
[otus]
name=otus-linux
baseurl=http://158.160.10.137
gpgcheck=0
enabled=1
EOF
yum repolist enabled | grep otus
--yum makecache
yum list | grep otus
yum repo-pkgs otus list
yum install -y epel-release
yum install -y centos-release-scl-rh
```
### Ставим пакеты с нашего репо
```
yum repo-pkgs otus install
```
### Инициируем кластер postgres  и проверяем что блок database и wal стал размером 32KB
```
/usr/pgsql-15/bin/postgresql-15-setup initdb
sudo -iu postgres /usr/pgsql-15/bin/pg_controldata |grep -i block
Database block size:                  32768
Blocks per segment of large relation: 32768
WAL block size:                       32768
```
