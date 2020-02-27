# 注意事项

基于 `centos-7-x86_64` 系统构建。

## 安装 sqlite3

先确定 sqlite3 的版本是否大于 3.7，需要安装版本大于 3.7 的 sqlite3 及 sqlite-devel。

`sqlite3 --version`

### 先将旧版本从 `rpm` 中删掉
[https://blog.csdn.net/THMAIL/article/details/104081254]

```
# 查看包名
rpm -q sqlite 
# 删除
rpm -e --nodeps sqlite3xxx(full name)
```


```
wget https://www.sqlite.org/2019/sqlite-autoconf-3270200.tar.gz
tar zxvf sqlite-autoconf-3260000
cd sqlite-autoconf-3260000
yum install gcc
./configure --prefix=/usr/local
make
make install
mv /usr/bin/sqlite3 /usr/bin/sqlite3_old
cp sqlite3 /usr/local/bin/sqlite3 # If not exists here.
ln -s /usr/local/bin/sqlite /usr/bin/sqlite3
echo "/usr/local/lib" > /etc/ld.so.conf.d/sqlite3.conf
ldconfig
sqlite3 --version

vi ~/.bashrc
# Add next line into it
# export LD_LIBRARY_PATH="/usr/local/lib"
source ~/.bashrc

# test
python3
import sqlite3
sqlite3.sqlite_version
# 3.27.2
```

## 安装 python3 及 python3x-devel

## 安装 gcc, virtualenv

`sudo yum install python3-virtualenv`
or `sudo yum install python-virtualenv`

## 安装 uwsgi

不能在 `virtualenv` 环境下安装 `uwsgi`，也不能在 `yum` 下安装，需要在 `pip3` 下安装。

```
pip3 install uwsgi
```

```
# terminal commands
cd project_dir
mkdir config
vi uwsgi.ini
```

```
# uwsgi.ini contents
[uwsgi]

# variables
projectname = easythings
base = /documents/python/easythings

# configurations
master = true
virtualenv = %(base)/venv
pythonpath = %(base)
chdir = %(base)
env = DJANGO_SETTINGS_MODULE=%(projectname).settings_pro
module = easythings.wsgi:application

# Note: use `run` don't use `tmp`
socket = /run/%(projectname).sock
```

## 安装 certbot（用于 ssl 验证）

Enable EPEL  
[https://fedoraproject.org/wiki/EPEL#How_can_I_use_these_extra_packages.3F]
```
# CentOS 6
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm

# CentOS 7
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

# CentOS 8
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```

```
# For centos 7
yum -y install yum-utils
yum-config-manager --enable rhui-REGION-rhel-server-extras rhui-REGION-rhel-server-optional
```

`sudo yum install certbot`

> 生成的证书用 `scp` 下载下来，同一个域名可以复用。

~~`sudo certbot certonly --standalone --standalone-supported-challenges http-01 -d easythings.smartcodecloud.com`~~

`sudo certbot certonly --standalone --preferred-challenges http-01 -d easythings.smartcodecloud.com`


生成的文件位于 `/etc/letsencrypt/live/xxx`（centos），
`/usr/local/etc/letsencrypt/live/xxx`（macOS）。

`sudo EDITOR=vi crontab -e`

```
# contents
15 3 * * * certbot renew --noninteractive --post-hook "systemctl restart mosquitto"
```

## 安装 nginx

`yum install nginx`

```
# terminal commands
cd project_dir 
cd config
vi nginx.conf
```

```
# nginx.conf contents
upstream easythings {
    server unix:///run/easythings.sock;
}

server {
    listen 80;
    listen 443 ssl;
    server_name easythings.smartcodecloud.com;

    ssl_certificate /etc/letsencrypt/live/easythings.smartcodecloud.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/easythings.smartcodecloud.com/privkey.pem;

    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass easythings;
    }

    location /static/ {
        alias /documents/python/easythings/static/;
    }

    location /media/ {
        alias /documents/python/easythings/media/;
    }

}
```

`ln -s /documents/python/easythings/config/nginx.conf /etc/nginx/sites-enabled/easythings.conf`

在 `/etc/nginx/nginx.conf` 中将 `user` 值改成 `root`，删掉默认的 `server`，并在 `http` 中添加 `include /etc/nginx/sites-enabled/*.conf`。

在 `settings_pro` 中将 `ALLOWED_HOSTS` 改成 `['easythings.smartcodecloud.com']`。

在 `settings` 中添加 `STATIC_ROOT = os.path.join(BASE_DIR, 'static/')`。

```
# terminal commands
source venv/bin/activate
python manage.py collectstatic

# start nginx
service nginx start
# stop nginx
service nginx stop
# reload nginx config
service nginx reload
# restart nginx?
service nginx restart
```

## 安装 mosquitto 并打开 ssl 验证

### centos 准备

#### Configure firewall

__NOTE:__ 如果不能安装，在 `root` 下去掉 `sudo`

`sudo yum install firewalld`

`sudo systemctl start firewalld`

`sudo firewall-cmd --permanent --add-service=ssh`

```
# If you have changed the SSH port for your server, ...
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --permanent --add-port=4444/tcp
```

`sudo firewall-cmd --permanent --add-service=http`

`sudo firewall-cmd --permanent --add-service=https`

`sudo firewall-cmd --permanent --add-service=smtp`

`sudo firewall-cmd --get-services`

`sudo firewall-cmd --permanent --list-all`

`sudo firewall-cmd --reload`

```
# Make sure the firewall will be stared at boot:
sudo systemctl enable firewalld
```

#### Configure Timezones

`sudo timedatectl list-timezones`

`sudo timedatectl set-timezone Asia/Shanghai`

`sudo timedatectl`

如果上面命令执行完成的 log 中有 `NTP enabled: yes` 字样，则建议不执行 `NTP` 中的命令。

#### NTP

`cat /etc/system-release`

`rpm -qa|grep chrony`

`yum install chrony`

`systemctl start chronyd.service`

`systemctl enable chronyd.service`

`systemctl status chronyd.service`

`firewall-cmd --add-service=ntp --permanent`

`firewall-cmd --reload`

服务端配置及客户端配置（略）

### 安装 mosquitto

> CentOS 7 用 `yum` 安装，CentOS 8 用源码安装。

`sudo yum -y install epel-release`

```
# Failed to set locale, defaulting to C
# 没有设置正确的语言环境
echo "export LC_ALL=en_US.UTF-8"  >>  /etc/profile 
source /etc/profile
```

```
# 用源码安装 (centos 8)
git clone https://github.com/eclipse/mosquitto.git
cd mosquitto
yum install make
yum install openssl-devel
yum install gcc-c++
yum install docbook-style-xsl

find / -name docbook.xsl
#/usr/share/sgml/docbook/xsl-stylesheets-1.79.2/manpages/docbook.xsl
#/usr/share/sgml/docbook/xsl-stylesheets-1.79.2/epub/docbook.xsl
#/usr/share/sgml/docbook/xsl-stylesheets-1.79.2/html/docbook.xsl
#/usr/share/sgml/docbook/xsl-stylesheets-1.79.2/fo/docbook.xsl
#/usr/share/sgml/docbook/xsl-stylesheets-1.79.2/xhtml/docbook.xsl
#/usr/share/sgml/docbook/xsl-stylesheets-1.79.2/xhtml-1_1/docbook.xsl

WITH_SYSTEMD:=yes
WITH_WEBSOCKETS:=yes

sudo make
sudo make install

adduser mosquitto

cp service/systemd/mosquitto.service.simple /usr/lib/systemd/system/mosquitto.service

sudo ln -s /usr/local/lib/libmosquitto.so.1 /usr/lib/libmosquitto.so.1
ldconfig

# 用命令安装（centos 7）
sudo yum install mosquitto

sudo systemctl start mosquitto
sudo systemctl enable mosquitto

sudo mosquitto_passwd -c /etc/mosquitto/passwd easythings
# s8 mqtt

sudo rm /etc/mosquitto/mosquitto.conf
sudo vi /etc/mosquitto/mosquitto.conf
sudo systemctl restart mosquitto

# Remove `User=mosquitto` if it't exists
sudo vi /etc/systemd/system/multi-user.target.wants/mosquitto.service
sudo systemctl daemon-reload
sudo systemctl restart mosquitto

sudo firewall-cmd --permanent --add-port=8883/tcp
sudo firewall-cmd --permanent --add-port=8083/tcp
sudo firewall-cmd --reload
```

__NOTE: 复制内容到 `vi` 中时，有可能出现丢失的情况，仔细检查复制后的内容无误，再进行 `wq`，否则浪费大量调试时间__

```
# mosquitto.conf contents
allow_anonymous false
password_file /etc/mosquitto/passwd

listener 1883 localhost

listener 8883
certfile /etc/letsencrypt/live/easythings.smartcodecloud.com/cert.pem
cafile /etc/letsencrypt/live/easythings.smartcodecloud.com/fullchain.pem
keyfile /etc/letsencrypt/live/easythings.smartcodecloud.com/privkey.pem

listener 8083
protocol websockets
certfile /etc/letsencrypt/live/easythings.smartcodecloud.com/cert.pem
cafile /etc/letsencrypt/live/easythings.smartcodecloud.com/fullchain.pem
keyfile /etc/letsencrypt/live/easythings.smartcodecloud.com/privkey.pem

```