---
title: Debian環境下Nginx使用SSL憑證
date: 2022-03-04 10:51:55
cover: https://whc.ca/wp-content/uploads/blog_15418_large.jpg
categories: Linux
tags:
- Linux
- Debian
- Nginx
- GCP
- SSL
- Let's Encrypt
---
為了在GCP打造安全的虛擬機環境，打算將所有的資料傳輸改為HTTPS安全傳輸協定，確保使用者能夠擁有安全連線，今天就來紀錄一下如何在GCP內的Debian環境使用[Let's Encrypt](https://letsencrypt.org)簽發有效期為90天的SSL憑證，並能夠在快到期時自動更新憑證，省得每次都忘記更新憑證導致使用者無法連線。

## GCP端
首先進入**VPC Network(虛擬私有雲網路)** > **防火牆**，確認欲使用SSL的port已經開啟。
![Imgur](https://i.imgur.com/T9RtxOR.png)
> 預設的SSL port為443

## Linux端
:::tip
以下指令皆使用root執行
:::

### 安裝`Certbot`套件
此套件為Let's Encrypt官方推薦的憑證工具。他可以在不停止伺服器的狀態下，執行憑證的頒發、安裝及自動更新
``` sh
apt-get install certbot python3-certbot-nginx -y
```

### 申請SSL憑證
```sh
certbot certonly --nginx --email [your-email] --agree-tos -d [your-domain]
```

:::tip
certonly: 僅申請SSL憑證，不自動設定Nginx的config檔
--email: 用於接收憑證異常或即將到期之通知
[your-email]: example@example.com
[your-domain]: example.com

過程中可能會詢問是否要收到Let's Encrypt的相關郵件，可以看個人需求，我這邊選**N**
:::

### Nginx設定SSL憑證
打開Nginx的設定檔`/etc/nginx/sites-available/default`
``` sh
$ vi etc/nginx/sites-available/default
```

將以下內容填入設定檔並儲存
``` conf
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;
    ssl_certificate /etc/letsencrypt/live/[your-domain]/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/[your-domain]/privkey.pem;
    ssl_ecdh_curve X25519:secp384r1;
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1440m;
    ssl_session_tickets off;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH+ECDSA+3DES:EECDH+aRSA+3DES:RSA+3DES:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/[your-domain]/chain.pem;
    add_header Strict-Transport-Security "max-age=31536000; preload";
}
```

接著確認是否有配置錯誤，若正確則會看到以下訊息
``` sh
$ nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

最後重新啟動Nginx來應用更改的配置
``` sh
$ systemctl restart nginx
```

### SSL憑證自動更新
確認此伺服器的所有憑證
```sh
$ certbot certificates
Saving debug log to /var/log/letsencrypt/letsencrypt.log
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Found the following certs:
  Certificate Name: [your-domain]
    Domains: [your-domain]
    Expiry Date: 2022-06-06 02:30:12+00:00 (VALID: 87 days)
    Certificate Path: /etc/letsencrypt/live/[your-domain]/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/[your-domain]/privkey.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

確認憑證自動更新運行狀態
``` sh
$ systemctl status certbot.timer
● certbot.timer - Run certbot twice daily
   Loaded: loaded (/lib/systemd/system/certbot.timer; enabled; vendor preset: enabled)
   Active: active (waiting) since Tue 2022-03-08 03:29:46 UTC; 2 days ago
  Trigger: Thu 2022-03-10 11:00:51 UTC; 7h left

Mar 08 03:29:46 linux systemd[1]: Started Run certbot twice daily.
```
:::tip
此結果顯示憑證每日會檢查**2次**，若發現憑證有效期**小於30天**，則會自動更新憑證
:::

測試憑證自動更新
``` sh
$ certbot renew --dry-run
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/[your-domain].conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Cert not due for renewal, but simulating renewal for dry run
Plugins selected: Authenticator nginx, Installer nginx
Renewing an existing certificate

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
new certificate deployed with reload of nginx server; fullchain is
/etc/letsencrypt/live/[your-domain]/fullchain.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
** DRY RUN: simulating 'certbot renew' close to cert expiry
**          (The test certificates below have not been saved.)

Congratulations, all renewals succeeded. The following certs have been renewed:
  /etc/letsencrypt/live/[your-domain]/fullchain.pem (success)
** DRY RUN: simulating 'certbot renew' close to cert expiry
**          (The test certificates above have not been saved.)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```
:::tip
此結果顯示**Cert not due for renewal**，表示憑證有效期超過一個月，無需更新
:::

### 測試
緊接著在瀏覽器輸入**https://[your-domain]**，若網址旁有鎖頭則代表設定成功！
![Imgur](https://i.imgur.com/JTG4CRY.png)