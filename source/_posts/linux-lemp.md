---
title: Debian安裝LEMP(Linux, Nginx, MariaDB, PHP)
date: 2022-03-04 10:51:55
cover: https://justher.tw/filesys/image/Justher_images/Ahex-LEMP-1.png
categories: Linux
tags:
- Linux
- Debian
- LEMP
- GCP
---

LEMP指的是在L - Linux OS下，安裝E - Nginx(Engine x)、M - MySQL/MariaDB、P - PHP，是現在很流行的伺服器組合，是將以往的LAMP中的Apache取代為Nginx，藉此提高伺服器的效能。

今天就來紀錄一下在Debian10下安裝LEMP的過程～

> 以下指令皆使用root執行

## 前置作業
安裝前先更新一下`apt-get`
``` sh
$ apt-get update
```

### 安裝
接著快速安裝一下Nginx, MariaDB和PHP
> Debian10 默認 PHP7.3; Debian11 則默認 PHP7.4
``` sh
$ apt-get install nginx
$ apt-get install mariadb-server
$ apt-get install php php-fpm php-cli php-mysql php-zip php-curl php-xml
```

安裝完成後，就可以將Nginx及MariaDB啟用了
``` sh
$ systemctl start nginx
$ systemctl start mariadb
```

若要在開機時就啟動Nginx及MariaDB，則輸入以下指令
``` sh
$ systemctl enable nginx
$ systemctl enable mariadb
```

測試Nginx是否啟用
``` sh
$ systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2022-03-03 09:10:15 UTC; 18h ago
     Docs: man:nginx(8)
  Process: 457 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
  Process: 468 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
 Main PID: 483 (nginx)
    Tasks: 3 (limit: 4651)
   Memory: 9.6M
   CGroup: /system.slice/nginx.service
           ├─483 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           ├─488 nginx: worker process
           └─489 nginx: worker process
```

測試MariaDB是否啟用
``` sh
$ systemctl status mariadb
● mariadb.service - MariaDB 10.3.31 database server
   Loaded: loaded (/lib/systemd/system/mariadb.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2022-03-03 09:10:16 UTC; 18h ago
     Docs: man:mysqld(8)
           https://mariadb.com/kb/en/library/systemd/
  Process: 460 ExecStartPre=/usr/bin/install -m 755 -o mysql -g root -d /var/run/mysqld (code=exited, status=0/SUCCESS)
  Process: 469 ExecStartPre=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
  Process: 479 ExecStartPre=/bin/sh -c [ ! -e /usr/bin/galera_recovery ] && VAR= ||   VAR=`cd /usr/bin/..; /usr/bin/galera_rec
  Process: 701 ExecStartPost=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
  Process: 703 ExecStartPost=/etc/mysql/debian-start (code=exited, status=0/SUCCESS)
 Main PID: 604 (mysqld)
   Status: "Taking your SQL requests now..."
    Tasks: 30 (limit: 4651)
   Memory: 103.8M
   CGroup: /system.slice/mariadb.service
           └─604 /usr/sbin/mysqld
```

若皆啟用後，也可以確認一下運行的port，一般情況下Nginx會運行在80 port; MariaDB則會運行在3306 port
``` sh
$ ss -antpl
LISTEN    0         128                0.0.0.0:80             0.0.0.0:*        users:(("nginx",pid=489,fd=6),("nginx",pid=488,fd=6),("nginx",pid=483,fd=6))   
LISTEN    0         128                   [::]:80                [::]:*        users:(("nginx",pid=489,fd=7),("nginx",pid=488,fd=7),("nginx",pid=483,fd=7))
LISTEN    0          80            127.0.0.1:3306             0.0.0.0:*        users:(("mariadbd",pid=12181,fd=15))
```
亦可以使用瀏覽器輸入http://[IP] 檢查Nginx是否安裝成功，若成功則可以看到以下畫面
![](https://0xzx.com/wp-content/webp-express/webp-images/doc-root/wp-content/uploads/2021/09/20210917-196.png.webp)


## 設置Nginx
打開`/etc/nginx/sites-available/default`
``` sh
$ vi etc/nginx/site-available/default
```

新增PHP的配置，並儲存離開
```
server {
  listen 80 default_server;
  listen [::]:80 default_server;

  root /data; # <<--變更網頁目錄位置

  # Add index.php to the list if you are using PHP
  index index.html index.htm index.nginx-debian.html index.php; # <<--此處新增index.php

  server_name _;

  location / {
          # First attempt to serve request as file, then
          # as directory, then fall back to displaying a 404.
          try_files $uri $uri/ =404;
  }

  # pass PHP scripts to FastCGI server
  #
  location ~ \.php$ { # <<--此處取消註解
          include snippets/fastcgi-php.conf; # <<--此處取消註解
  #
  #       # With php-fpm (or other unix sockets):
          fastcgi_pass unix:/run/php/php7.3-fpm.sock; # <<--此處取消註解
  #       # With php-cgi (or other tcp sockets):
  #       fastcgi_pass 127.0.0.1:9000;
  } # <<--此處取消註解
}
```

接著確認Nginx是否有配置錯誤，若正確則會看到以下訊息
``` sh
$ nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

最後重新啟動Nginx來應用更改的配置
``` sh
$ systemctl restart nginx
```


## 設置MariaDB
運行`mysql_secure_installation`來初始設置MariaDB
``` sh
$ mysql_secure_installation
Enter current password for root (enter for none): 
Change the root password? [Y/n] Y
New password: 
Re-enter new password: 
Remove anonymous users? [Y/n] Y
Disallow root login remotely? [Y/n] Y
Remove test database and access to it? [Y/n] Y
Reload privilege tables now? [Y/n] Y
```

設置完成後即可以登入確認，並檢查MariaDB版本
``` sh
$ mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 36
Server version: 10.3.31-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> SELECT VERSION();
+---------------------------+
| VERSION()                 |
+---------------------------+
| 10.3.31-MariaDB-0+deb10u1 |
+---------------------------+
1 row in set (0.000 sec)
```

## 設置PHP
若要變更PHP配置，則要打開`/etc/php/7.3/fpm/php.ini`
``` sh
$ vi etc/php/7.3/fpm/php.ini
```

編輯完成後儲存並重啟PHP
``` sh
$ systemctl restart php7.3-fpm
```


在伺服器根目錄的地方加上`index.php`並編輯
``` sh
$ vi data/info.php
```

``` php
<?php
  echo phpinfo();
?>
```

完成後在瀏覽器輸入http://[IP]，若成功則可以看到以下畫面
![Imgur](https://i.imgur.com/JD4o00c.png)
