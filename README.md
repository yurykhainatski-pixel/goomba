
# Настройка Nginx с self‑signed TLS и проксированием на backend :8080

## Целевая аудитория

Системный администратор Linux, уверенно работающий в командной строке, но не имевший опыта настройки Nginx с TLS.

## Prerequisites

* Сервер под управлением Ubuntu 22.04 / 24.04 или Debian 11/12
* Пользователь с sudo правами
* Backend-сервис, слушающий на `127.0.0.1:8080`
* Установленный Git
* Открытый порт 443 (и временно 80) в firewall 

## Mermaid: схема трафика

```mermaid
flowchart LR
    Client[Клиент] -->|HTTPS :443| Nginx[Nginx]
    Nginx -->|HTTP :8080| Backend[Backend localhost:8080]
```

## Пошаговая инструкция

### 1. Установка Nginx

```
bash

sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
```

Проверка:
```
bash

curl -I http://localhost
```
![Screenshot: вывод systemctl status nginx и успешный ответ curl]

### 2. Инициализация Git-репозитория и ветки

```
bash

mkdir ~/nginx-tls-lab && cd ~/nginx-tls-lab
git init
git checkout -b docs/nginx-tls-setup
echo "# Nginx TLS setup" > README.md
git add README.md
git commit -m "docs: init documentation skeleton"
```

### 3. Генерация self‑signed сертификата (OpenSSL)

Создаём директорию и сертификат на 10 лет:
```
bash

sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/selfsigned.key \
  -out /etc/nginx/ssl/selfsigned.crt \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=DevLab/CN=localhost"
```

Права доступа:
```
bash

sudo chmod 600 /etc/nginx/ssl/selfsigned.key
sudo chmod 644 /etc/nginx/ssl/selfsigned.crt
```

!Screenshot: содержимое /etc/nginx/ssl/ с файлами .crt и .key]

Коммит:
```
bash

git add README.md
git commit -m "feat: add self-signed TLS cert generation steps"
```

### 4. Настройка server block с SSL и proxy_pass

Создадим конфигурационный файл:
```
bash

sudo nano /etc/nginx/sites-available/backend-proxy
```

Содержимое:
```
nginx

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Активация сайта и перезагрузка:
```
bash

sudo ln -s /etc/nginx/sites-available/backend-proxy /etc/nginx/sites-enabled/
sudo nginx -t          # проверка синтаксиса
sudo systemctl reload nginx
```

![Screenshot: успешный вывод nginx -t]

Коммит:
```
bash

git add /etc/nginx/sites-available/backend-proxy   # символическая ссылка не коммитится, документируем в README
git commit -m "feat: add nginx ssl proxy config to backend :8080"
```

### 5. Проверка работы

**Без игнорирования ошибок сертификата (должна быть ошибка):**
```
bash

curl https://localhost
# curl: (60) SSL: certificate subject name 'localhost' does not match target host
```

**С игнорированием проверки сертификата:**
```
bash

curl -k https://localhost
# должен вернуться ответ от backend-сервиса на :8080
```

**Проверка через браузер:**

Открыть `https://localhost` → принять риск (самоподписанный сертификат) → увидеть ответ backend.

![Screenshot: успешный ответ curl -k https://localhost + страница браузера с предупреждением]

Коммит:
```
bash

git commit -m "test: add verification instructions and screenshots"
```

## Таблица параметров nginx.conf (акцент на SSL + proxy)

| Параметр  | Значение / пример | Назначение |
| ------------- |:-------------:|-------------|
| listen 443 ssl |	443 ssl |	Слушаем HTTPS-порт с включённым SSL/TLS|
|ssl_certificate |	/etc/nginx/ssl/selfsigned.crt |	Публичный сертификат (X.509)|
|ssl_certificate_key |	/etc/nginx/ssl/selfsigned.key |	Приватный ключ (держать в секрете, права 600)|
|ssl_protocols |	TLSv1.2 TLSv1.3 |	Отключить устаревшие SSLv3, TLSv1.0/1.1|
|ssl_ciphers |	HIGH:!aNULL:!MD5 |	Только надёжные шифры, без анонимных и MD5|
|proxy_pass |	http://127.0.0.1:8080	| Куда перенаправлять запросы (бэкенд)|
|proxy_set_header |	Host $host	| Передавать оригинальный Host для бэкенда|
|proxy_set_header | X-Forwarded-Proto	$scheme	| Сообщать бэкенду, что оригинал был HTTPS|

## Troubleshooting (3 типовые проблемы + решения)

### 1. nginx: [emerg] bind() to 0.0.0.0:443 failed (98: Address already in use)

**Причина:** Другой процесс уже слушает порт 443.

**Решение:**
```
bash

sudo lsof -i :443
sudo systemctl stop apache2   # или kill <PID>
sudo systemctl restart nginx
```

### 2. SSL: error:0B080074:x509 certificate routines:X509_check_private_key:key values mismatch

**Причина:** Пара `selfsigned.crt` и `selfsigned.key` не соответствуют друг другу (например, подставлен другой ключ).

**Решение:** Заново сгенерировать оба файла за один вызов `openssl req -x509 -newkey ...` (как в шаге 3) и перезагрузить Nginx.

### 3. Бэкенд на :8080 отвечает, но Nginx возвращает 502 Bad Gateway

**Причина:** Nginx не может соединиться с `127.0.0.1:8080` – бэкенд не слушает, либо firewall блокирует.

**Решение:**
```
bash

# Проверить сам бэкенд
curl http://127.0.0.1:8080

# Проверить, что nginx имеет доступ (selinux/apparmor)
sudo setsebool -P httpd_can_network_connect 1   # для CentOS/RHEL
# Для Ubuntu проверить, что бэкенд не слушает только на IPv6 ::1
```
