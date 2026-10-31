# district-tls — HTTPS для района с сертификатом городского УЦ

Добавка к шаблону района (`district-template` недели 2). Скопируйте содержимое этой папки в корень
репозитория района: появятся `tls/`, `scripts/make-csr.sh`, `scripts/check-tls.sh`, `compose.tls.yaml`.

```
житель ──HTTPS──▶ tls (Caddy, :8443) ──HTTP, внутренняя сеть──▶ api (:8080) ──▶ db
                   сертификат bank.city.lab от городского УЦ
```

## Шаг 1. Ключ и запрос (5 минут)
```bash
./scripts/make-csr.sh bank                     # свой код района (Р1)
./scripts/make-csr.sh bank api.bank.city.lab   # если нужно ещё имя
```
- `tls/private/district.key` — **закрытый ключ**. Никуда не отправлять, не коммитить (он в `.gitignore`).
- `tls/district.csr` — **запрос**. Его отправляете Паспортному столу.
- Скрипт печатает **отпечаток** — 16 символов. Назовите его тимлиду Паспортного стола голосом.

## Шаг 2. Получить сертификат
Паспортный стол присылает `district.crt` (ваш сертификат + сертификат выпускающего УЦ) и `city-root.crt`.
Положите оба в `tls/`. Сверьте отпечаток корня с тем, что Паспортный стол объявил на паре:
```bash
openssl x509 -in tls/city-root.crt -noout -fingerprint -sha256
```

## Шаг 3. Запустить район с HTTPS
В `.env` добавьте:
```
DISTRICT_HOST=bank.city.lab
LOCAL_UID=1000      # вывод команды: id -u
LOCAL_GID=1000      # вывод команды: id -g
```
```bash
docker compose -f compose.yaml -f compose.tls.yaml up --build -d
```
Сервис `api` больше не публикует порт: в район можно попасть только по HTTPS через `tls`.
Если Docker Compose старый и ругается на `!reset` — обновите Docker Desktop или удалите блок `ports`
у сервиса `api` в `compose.yaml` вручную.

## Шаг 4. Проверить
```bash
./scripts/check-tls.sh bank
```
Семь проверок: сертификат проверяется по корню города; без корня — отказ; чужое имя — отказ;
TLS 1.1 выключен; сервер отдаёт цепочку; есть HSTS; HTTP перенаправляет на HTTPS.
Вывод приложите к отчёту ЛР2.

В браузере: импортируйте `city-root.crt` как доверенный корень (Firefox: Настройки → Приватность →
Сертификаты → Импорт; **только** «для веб-сайтов») и добавьте в hosts строку `127.0.0.1 bank.city.lab`.
После курса удалите корень из браузера.

## Частые проблемы
| Симптом | Причина |
|---|---|
| `permission denied` на `district.key` в журнале `tls` | `LOCAL_UID`/`LOCAL_GID` не совпадают с владельцем файла (`ls -ln tls/private`) |
| `curl: (60) SSL certificate problem: unable to get local issuer certificate` | в `district.crt` нет второго сертификата (выпускающего УЦ) или не тот `city-root.crt` |
| `no alternative certificate subject name matches` | сертификат выпущен на другое имя; проверьте `DISTRICT_HOST` |
| `address already in use` для 8443 | порт занят другим районом на этом же компьютере — поменяйте `127.0.0.1:8443` на `8444` |
