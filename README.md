# 自架NAS 網頁架上公網     創作者:謝秉棋

## 主要學習:
1. 基礎Docker與Portainer介紹
2. CloudFlare 介紹與教學
3. Nginx反向代理介紹與教學

## 基礎Docker與Portainer介紹

### Docker 是一種開源平台，允許開發者自動化應用程序的部署、擴展和管理，其核心概念是容器，容器是一種輕量級的虛擬化技術，可以在單一操作系統上運行多個應用。
#### 主要特點：
1. 容器化：將應用及其依賴項打包在一個可移植的容器中。
2. 高效性：容器共享主機的內核，啟動速度快，佔用資源少。
3. 可移植性：可以在任何支持 Docker 的環境中運行容器，包括本地開發環境、測試伺服器和生產環境。
4. 版本控制：可以輕鬆管理和回滾容器的版本。

### Portainer 是一個用於 Docker 的輕量級管理工具，提供用戶友好的 Web 界面，方便用戶管理 Docker 環境中的容器、映像、網絡和卷等資源。

#### 主要特點：
1. 簡易管理：用戶可以通過直觀的界面管理 Docker 環境，無需深入命令行。
2. 多 Docker 環境支持：可以管理本地 Docker 主機、Docker Swarm 集群或遠端 Docker 主機。
3. 可視化監控：提供容器狀態、性能指標等視覺化監控功能，幫助用戶輕鬆掌握系統狀態。
4. 用戶管理：支持多用戶和權限管理，適合團隊合作環境。

## CloudFlare 可將內網變外網使用
### CloudFlare 介紹

#### Cloudflare 是一家提供網站安全和性能優化服務的公司。其主要功能包括：
1. 內容分發網絡（CDN）：加速網站加載速度。
2. DDoS 保護：防止分散式拒絕服務攻擊。
3. 網站防火牆：保護網站免受常見安全威脅。
4. SSL 加密：保障數據傳輸安全。
5. DNS 解析：快速、安全的域名解析服務。
6. 邊緣計算：允許在邊緣節點運行代碼，提高應用效能。

### Cloudflare 教學

1. 先要有自己的網域(建議直接購買Cloudflare,一年200~300多台幣,對比其他很便宜，也可以找免費，但免費網域要不定期去啟動或有廣告，不一定能找到自己想要的名字，我之前有用過)
2. 辦好 CloudFlare 並新增網域就可以新增隧道

![contents](/images/Cloudflare/cloudflare.png)

![contents](/images/Cloudflare/cloudflare1.png)

3. 產生隧道 docker 令牌貼在終端機下載 記得sudo

![contents](/images/Cloudflare/cloudflare2.png)

4. 在Portainer可以看到docker已有cloudflare

![contents](/images/Cloudflare/portainer.png)

5. 剩下就新增自己的網域名與內網資料

![contents](/images/Cloudflare/cloudflare3.png)

![contents](/images/Cloudflare/cloudflare4.png)

5. 記得要先到cloudflare 的自己網域概覽 生成DNS API (Nginx需要生成SLL)(基本上可以不需要反向代理,除非真的很注重安全,不然cloudflare就很安全了)

![contents](/images/Cloudflare/API.png)

![contents](/images/Cloudflare/API2.png)

![contents](/images/Cloudflare/API3.png)

6. 要修改所有區域

![contents](/images/Cloudflare/API4.png)

7. 複製好自己的API(建議留存)

![contents](/images/Cloudflare/API5.png)


## Nginx反向代理介紹
### NGINX 是一個高性能的網頁伺服器和反向代理伺服器，主要特點包括：

1. 高性能：能夠處理大量並發連接。
2. 反向代理：分發請求到後端伺服器。
3. 負載均衡：支持多種負載均衡算法。
4. 靜態文件服務：高效提供靜態內容。
5. SSL/TLS 支持：保障安全的 HTTPS 連接。
6. 緩存功能：提高響應速度，減少伺服器負擔。

## 基本上用Cloudflare就夠了,反向代理會許多限制,安全度Cloudflare就很夠了,需多大企業也都有用Cloudflare
## 必須要有docker,建議需要有portainer圖形化介面比較好操作docker

## Nginx反向代理安裝
1. mkdir nginx
2. cd nginx
3. nano config.json
```
{ 
  "database": { 
     "engine": "mysql", 
     "host": "db", 
     "name": "npm", 
     "user": "npm", 
     "password": "npm", 
     "port": 3306
   }
 }

```
4. nano docker-compose.yml
```
--- 
version: '3' 
services: 
  app: 
    image: 'jc21/nginx-proxy-manager:latest' 
    ports: 
      - '80:80' #HTTP Traffic 
      - '81:81' #Dashboard Port
      - '443:443' #HTTPS Traffic 
    volumes: 
      - ./config.json:/app/config/production.json 
      - ./data:/data 
      - ./letsencrypt:/etc/letsencrypt
  db: 
    image: 'jc21/mariadb-aria:10.4.15-innodb'
    environment: 
      MYSQL_ROOT_PASSWORD: 'npm' 
      MYSQL_DATABASE: 'npm' 
      MYSQL_USER: 'npm' 
      MYSQL_PASSWORD: 'npm'
    volumes:
      - ./data/mysql:/var/lib/mysql

```
### 進去 sudo nano /etc/nginx/nginx.conf
#### 在http內增加以下

```
 include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
        client_max_body_size 50M;
        client_header_timeout 3600s;
        client_body_timeout 3600s;
        fastcgi_connect_timeout 3600s;
        fastcgi_send_timeout 3600s;
        fastcgi_read_timeout 3600s;
```

### 完整nginx.conf

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on;

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
        client_max_body_size 50M;
        client_header_timeout 3600s;
        client_body_timeout 3600s;
        fastcgi_connect_timeout 3600s;
        fastcgi_send_timeout 3600s;
        fastcgi_read_timeout 3600s;
}

#mail {
#       # See sample authentication script at:
#       # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#
#       # auth_http localhost/auth.php;
#       # pop3_capabilities "TOP" "USER";
#       # imap_capabilities "IMAP4rev1" "UIDPLUS";
#
#       server {
#               listen     localhost:110;
#               protocol   pop3;
#               proxy      on;
#       }
#
#       server {
#               listen     localhost:143;
#               protocol   imap;
#               proxy      on;
#       }
#}
```

5. 進入網頁http://localhost:81 初始帳號:admin@example.com 密碼:changeme
6. 點選SSL

![contents](/images/nginx/SSL.png)

7.  將cloudflare 複製的API 用與SSL

![contents](/images/nginx/SSL1.png)

8. 加入Host

![contents](/images/nginx/nginx.png)

9. 輸入自己的網域以及內網地址

![contents](/images/nginx/nginx1.png)

10. 選擇SSL 添加剛剛申請的證書

![contents](/images/nginx/nginx2.png)

11. 按下確認就完成了


## 提醒 nginx 與 cloudflare 一起用會更加安全但基本上用Cloudflare就可以達到安全度
## Cloudflare 輸入內網地址部份也可以輸入Zerotier 的外網地址並加上內網端口就可以更安全

![contents](/images/Cloudflare/zero.png)




