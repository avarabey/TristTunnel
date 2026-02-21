# TristTunnel

Документ для фиксации изменений по проекту.

Версия: `1`

## Инструкция развертывания на VDS

Цель: развернуть TrustTunnel endpoint на VDS и получить рабочие параметры подключения клиента.

Исходные данные:
- сервер: `<SERVER_IP>`
- домен: `<VPN_DOMAIN>`
- ОС: Ubuntu 22.04/24.04

### 1. DNS
1. Создать `A`-запись: `@ -> <SERVER_IP>`.
2. Если используется Cloudflare, временно включить `DNS only` (без прокси).
3. Проверить:

```bash
dig +short <VPN_DOMAIN>
```

Ожидаемый ответ: `<SERVER_IP>`.

### 2. Подготовка сервера

```bash
ssh root@<SERVER_IP>
apt update && apt upgrade -y
apt install -y curl ca-certificates ufw openssl
```

### 3. Firewall

```bash
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
ufw status verbose
```

### 4. Установка endpoint

```bash
curl -fsSL https://raw.githubusercontent.com/TrustTunnel/TrustTunnel/refs/heads/master/scripts/install.sh | sh -s -
cd /opt/trusttunnel
ls -la
```

### 5. Настройка через мастер
1. Сгенерировать пароль:

```bash
openssl rand -base64 24
```

2. Запустить мастер:

```bash
cd /opt/trusttunnel
sudo ./setup_wizard
```

3. Рекомендуемые значения в мастере:
- `listen address`: `0.0.0.0:443`
- `credentials file`: `credentials.toml`
- `username`: `client1`
- `password`: сгенерированный пароль
- `add one more user`: `no`
- `rules file`: `rules.toml`
- фильтрация: `n` (разрешить все)
- `main settings file`: `vpn.toml`
- сертификат: Let's Encrypt
- домен: `<VPN_DOMAIN>`
- email для Let's Encrypt: `<ADMIN_EMAIL>`
- challenge method: `HTTP-01` (если открыт и свободен порт `80/tcp`)
- staging environment:
  - для тестового первого прогона можно `yes`
  - для рабочего сертификата обязательно `no`
- alternative SNIs: `no` (если используется только `<VPN_DOMAIN>`)
- TLS hosts file: `hosts.toml`

### 6. Важно для HTTP-01: порт 80 должен быть свободен

Перед выпуском сертификата проверить, кто слушает `80`:

```bash
sudo ss -ltnp 'sport = :80'
```

Если порт занят `nginx`, временно остановить:

```bash
sudo systemctl stop nginx
```

После успешного выпуска сертификата можно вернуть `nginx`:

```bash
sudo systemctl start nginx
```

Примечание: если `nginx` слушает еще и `443`, он будет конфликтовать с TrustTunnel (`0.0.0.0:443`).

### 7. Запуск сервиса

```bash
cd /opt/trusttunnel
sudo cp trusttunnel.service.template /etc/systemd/system/trusttunnel.service
sudo systemctl daemon-reload
sudo systemctl enable --now trusttunnel
sudo systemctl status trusttunnel --no-pager -l
```

### 8. Генерация клиентского конфига (TOML)

В используемой версии `trusttunnel_endpoint`:
- параметр `--format` не поддерживается;
- в `--address` нужно указывать только `IP` или `IP:port` (домен не подходит).

Команда:

```bash
cd /opt/trusttunnel
sudo ./trusttunnel_endpoint -c client1 -a <SERVER_IP>:443 vpn.toml hosts.toml
```

Команда выведет TOML-конфиг c полями `hostname`, `addresses`, `username`, `password` и т.д.

При необходимости сохранить его в файл:

```bash
sudo ./trusttunnel_endpoint -c client1 -a <SERVER_IP>:443 vpn.toml hosts.toml | sudo tee /opt/trusttunnel/client1.toml >/dev/null
```

### 9. Настройка клиента через экран Add server

Если в приложении нет импорта файла и есть только форма `Add server`, заполнять так:
- `Server name`: любое имя (например, `vpn-main`)
- `IP address of the running VPN server`: `<SERVER_IP>`
- `Domain name from server certificate`: `<VPN_DOMAIN>`
- `Username`: `client1`
- `Password`: значение из вывода `trusttunnel_endpoint`
- `Protocol`: `HTTP/2`
- `Routing profile`: `Default profile`
- `DNS server addresses`: обязательно заполнить, по одной строке:
  - `1.1.1.1`
  - `8.8.8.8`

Примечание: в этом приложении DNS-адреса работают и в виде обычных IP (без `tls://`).

### 10. Диагностика

```bash
sudo journalctl -u trusttunnel -n 100 --no-pager
sudo ss -ltnp '( sport = :80 or sport = :443 )'
```

## Изменения

### 2026-02-21
- Создан `README.MD` для ведения истории изменений.
- Добавлена пошаговая инструкция развертывания TrustTunnel до получения параметров подключения.
- Уточнены шаги выпуска Let's Encrypt: выбор `HTTP-01`, поведение `staging`, `alternative SNI`, и устранение ошибки `Port 80 is already in use` через временную остановку `nginx`.
- Инструкция обновлена под фактическую версию клиента/endpoint: ручной `Add server`, обязательное поле DNS, и актуальный синтаксис `trusttunnel_endpoint` без `--format` и с `IP[:port]` в `--address`.
- Удалены персональные данные из инструкции для безопасной публикации репозитория.
