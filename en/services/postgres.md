# PostgreSQL

PostgreSQL stores Quasar data. It is an infrastructure dependency rather than a v3rt3p application repository.

## Environment variables

| Variable | Required | Default / example | Why it is used |
| --- | --- | --- | --- |
| `POSTGRES_USER` | No | `quasar` | Database role created by the image. |
| `POSTGRES_PASSWORD` | Yes for a real deployment | `replace-me` | Database password. Keep it in a secret store or untracked `.env` file. |
| `POSTGRES_DB` | No | `quasar` | Database created on first startup. |

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

Keep the volume on durable storage. Back it up with `pg_dump`; do not copy the live data directory while PostgreSQL is writing.
