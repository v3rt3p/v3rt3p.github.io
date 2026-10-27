## Logs

Start with the service that owns the failing connection:

```sh
docker compose ps
docker compose logs -f quasar-server
docker compose logs -f quasistreaming-gigam
```

For a full startup view:

```sh
docker compose logs -f --tail=100
```

## Model storage

ASR and TTS assets can be large. Keep them outside the Git repository, mount them read-only when possible, and verify free space before rebuilding. CUDA ASR also uses a model cache; do not delete it while the service is running.

## Secrets and network access

Keep LLM keys, PostgreSQL passwords, and TLS material in an untracked `.env` file or a secret manager. Do not put them in Compose snippets copied into issues. Internal services should use Compose hostnames and should not be published to the internet.

If the proxy stops receiving Yandex traffic, check DNS first, then the proxy's `YANDEX_UPSTREAM_URL`, then the Quasar listener. The current domain list lives in the `https-proxy` repository.
