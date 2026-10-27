`https-proxy` — публичная точка входа для трафика Яндекса. Он запускает NGINX на портах 80 и 443 и пересылает настроенные домены на Quasar или напрямую в интернет.

Сервис использует промежуточные сертификаты CA eMudhra для HTTPS endpoint-ов.

Исходный код: [`v3rt3p/https-proxy`](https://github.com/v3rt3p/https-proxy), текущая ветка `master`.

## Переменные среды

| Переменная | Обязательна | Значение по умолчанию / пример | Назначение |
| --- | --- | --- | --- |
| `YANDEX_UPSTREAM_URL` | Да | - | Upstream для трафика Яндекса в `quasar-server`. |
| `*_UPSTREAM_URL` | Нет | Зависит от сервиса | Proxy поддерживает override upstream-ов из `domains/`. |

Точный набор имён upstream-ов определяется файлами в `domains/`.

## Compose

```yaml
https-proxy:
  build: https://github.com/v3rt3p/https-proxy.git
  restart: unless-stopped
  environment:
    YANDEX_UPSTREAM_URL: http://quasar-server:31115
  ports:
    - "80:80"
    - "443:443"
  depends_on:
    - quasar-server
```

## Файлы и эксплуатация

- `nginx.conf` описывает reverse proxy;
- `domains/*.txt` содержит список доменов, которые обрабатывает образ;
- `start.sh` и `generate.sh` готовят runtime-конфигурацию и сертификаты.

Публикуйте порты 80 и 443 только после настройки DNS. Контейнер генерирует сертификаты во время запуска; перед использованием в production изучите обработку сертификатов и CA в репозитории.
