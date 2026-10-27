# PostgreSQL

PostgreSQL хранит данные Quasar. Это инфраструктурная зависимость, а не репозиторий приложения v3rt3p.

## Переменные среды

| Переменная | Обязательна | Значение по умолчанию / пример | Назначение |
| --- | --- | --- | --- |
| `POSTGRES_USER` | Нет | `quasar` | Роль базы, создаваемая образом. |
| `POSTGRES_PASSWORD` | Да в реальном запуске | `replace-me` | Пароль базы. Храните его в secret manager или неотслеживаемом `.env`. |
| `POSTGRES_DB` | Нет | `quasar` | База, создаваемая при первом запуске. |

## Compose

```yaml
postgres:
  image: postgres:17
  restart: unless-stopped
  environment:
    POSTGRES_USER: quasar
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD}
    POSTGRES_DB: quasar
  volumes:
    - postgres-data:/var/lib/postgresql/data
```

Храните volume на постоянном диске. Делайте резервные копии через `pg_dump`; не копируйте каталог данных во время записи PostgreSQL.
