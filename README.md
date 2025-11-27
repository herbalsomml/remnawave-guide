# Интро
Тут описана настройка сервера ноды + хостов, для того чтобы на ней работал конфиг с тремя инбаундами:
1. VLESS TCP Reality
2. VLESS xHTTP
3. VLESS gRPC

На домене реализована заглушка, чтобы это дерьмо выглядело как человеческий сайт.
## Важно блять
Я ебал ваши иксреи нахуй, однако данный сетап умудряется работать.
Не отрицаю, что gRPC инбаунд может быть реализован вообще не верно, но он работает - так что и похуй.

Если я хуйню тут какую-то написал, то не стесняйтесь кидать в меня говно и контрибутить.

# Настройка всего говнища
## Настройка файрволла
```
ufw allow OpenSSH && ufw allow 2222/tcp && ufw allow 80/tcp && ufw allow 443/tcp && ufw allow 9443/tcp && ufw allow 44443/tcp && ufw allow 44444/tcp && ufw disable && ufw enable
```

## Запрет ICMP запросов

```
nano /etc/ufw/before.rules
```

В блоке `# ok icmp codes for INPUT` меняем на
```
-A ufw-before-input -p icmp --icmp-type destination-unreachable -j DROP
-A ufw-before-input -p icmp --icmp-type time-exceeded -j DROP
-A ufw-before-input -p icmp --icmp-type parameter-problem -j DROP
-A ufw-before-input -p icmp --icmp-type echo-request -j DROP
-A ufw-before-input -p icmp --icmp-type source-quench -j DROP
```

В блоке `# ok icmp codes for FORWARD` меняем на
```
-A ufw-before-forward -p icmp --icmp-type destination-unreachable -j DROP
-A ufw-before-forward -p icmp --icmp-type time-exceeded -j DROP
-A ufw-before-forward -p icmp --icmp-type parameter-problem -j DROP
-A ufw-before-forward -p icmp --icmp-type echo-request -j DROP
```

Перезапуск файрволла

```
ufw disable && ufw enable
```

## Установка nginx
`sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring`
```
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```

```
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```

```
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

```
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx
```

```
sudo apt update
sudo apt install nginx
```

## Создание заглушки
```
mkdir -p /var/www/html/
```

```
nano /var/www/html/index.html
```

Вставляем HTML-код заглушки
```html
<!DOCTYPE html>
<html>
    <head>
        <title>All good!</head>
    </head>
    <body>
        <h1>All good!</h1>
    </body>
</html>
```

## Генерация SSL для домена
```
sudo apt update
sudo apt install certbot python3-certbot-nginx
```
```
sudo certbot --nginx -d <ДОМЕН>
```

## Настройка nginx
```
nano /etc/nginx/conf.d/xray.conf
```

```
server {
    listen 443 ssl;
    http2 on;
    server_name <ДОМЕН>;

    ssl_certificate /etc/letsencrypt/live/<ДОМЕН>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<ДОМЕН>/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /xhttppath/ {
        proxy_pass http://unix:/dev/shm/xrxh.socket;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

server {
    listen 9443 ssl;
    http2 on;
    server_name <ДОМЕН>;

    ssl_certificate /etc/letsencrypt/live/<ДОМЕН>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<ДОМЕН>/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    root /var/www/html;
    index index.html;
}

server {
    listen 80;
    http2 on;
    server_name <ДОМЕН>;

    location /grpc/ {
        grpc_pass grpc://127.0.0.1:8080;
        grpc_set_header Host $host;
        grpc_set_header X-Real-IP $remote_addr;
        grpc_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        grpc_read_timeout 1h;
        grpc_send_timeout 1h;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

```
systemctl restart nginx
```

## Установка ноды
```
mkdir -p /opt/remnanode/
```
```
nano .env
```
```
NODE_PORT=2222
SECRET_KEY=<СЕКРЕТ ИЗ REMNAWAVE>
```
```
nano docker-compose.yml
```
```yml
services:
  remnanode:
    container_name: remnanode
    hostname: remnanode
    image: remnawave/node:latest
    network_mode: host
    restart: always
    volumes:
      - /dev/shm:/dev/shm
    env_file:
      - .env
```
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```
```
cd /opt/remnanode/
docker compose up
```

## Настройки TCP хоста
![Основные настройки](assets/1.png)
![Расширенные настройки](assets/2.png)

## Настройки XHTTP хоста
![Основные настройки](assets/3.png)
![Расширенные настройки](assets/4.png)

Xray Json & Raw (xHTTP)
```json
{
  "xmux": {
    "cMaxReuseTimes": 0,
    "maxConcurrency": "16-32",
    "maxConnections": 0,
    "hKeepAlivePeriod": 0,
    "hMaxRequestTimes": "600-900",
    "hMaxReusableSecs": "1800-3000"
  },
  "noGRPCHeader": false,
  "xPaddingBytes": "100-1000",
  "downloadSettings": {
    "port": 443,
    "address": "<домен>",
    "network": "xhttp",
    "security": "tls",
    "tlsSettings": {
      "alpn": [
        "h2,http/1.1"
      ],
      "show": false,
      "serverName": "<домен>",
      "fingerprint": "chrome",
      "allowInsecure": false
    },
    "xhttpSettings": {
      "path": "/xhttppath/"
    }
  },
  "scMaxEachPostBytes": 1000000,
  "scMinPostsIntervalMs": 30,
  "scStreamUpServerSecs": "20-80"
}
```

## Настройки gRPC хоста
![Основные настройки](assets/5.png)
![Расширенные настройки](assets/6.png)