`https-proxy` is the public entry point for Yandex traffic. It runs NGINX on ports 80 and 443 and forwards configured domains to Quasar or directly to the internet.

It uses eMudhra intermediate CA certificates for HTTPS endpoints.

Source: [`v3rt3p/https-proxy`](https://github.com/v3rt3p/https-proxy), current `master`.

## Environment variables

| Variable | Required | Default / example | Why it is used |
| --- | --- | --- | --- |
| `YANDEX_UPSTREAM_URL` | Yes | - | Upstream for Yandex traffic in the `quasar-server`. |
| `*_UPSTREAM_URL` | No | Service-specific | The proxy supports upstream overrides named after entries in `domains/`. |

The exact set of upstream names comes from `domains/`.

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

## Files and operation

- `nginx.conf` defines the reverse-proxy behavior.
- `domains/*.txt` lists the domains handled by the image.
- `start.sh` and `generate.sh` prepare runtime configuration and certificates.

Publish ports 80 and 443 only after DNS points at this host. The container generates certificates at runtime; inspect the repository's certificate and CA handling before using it on a public production host.
