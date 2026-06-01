# PULL REQUEST

## Changes
- Добавлена документация по настройке Nginx + self-signed TLS + proxy_pass на :8080
- Включены: установка, генерация сертификатов, конфигурация, проверка
- Добавлен Mermaid-схема трафика
- Таблица ключевых параметров nginx.conf
- Раздел Troubleshooting с 3 реальными кейсами

## Testing
- [x] Установка на чистом Ubuntu 22.04
- [x] curl -k https://localhost возвращает ответ бэкенда
- [x] Просмотр TLS-соединения: openssl s_client -connect localhost:443 -tls1_2
