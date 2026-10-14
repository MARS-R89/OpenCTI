# Kaspersky TI Connector -> OpenCTI (Docker)

Setup: OpenCTI 7.260326.0 on host `plutox` (192.168.101.71:8080), Docker network `xtm_default`.
Connector: `connectors-7.260326.0/external-import/kaspersky` (login: Kaspersky TIP user + password + certificate).

```bash
   wget https://github.com/OpenCTI-Platform/connectors/archive/7.260326.0.zip
   unzip 7.260326.0.zip
```

## Prerequisites
- OpenCTI running and healthy (`docker ps` -> `xtm-opencti-1`, `xtm-rabbitmq-1` Up)
- Valid OpenCTI API token (verify: `{ me { name } }` GraphQL query returns a name, not AUTH_REQUIRED)
- Kaspersky TIP user, password and `certificate.pem` (saved at `/root/opencti/certificate.pem`)
- Outbound access to `tip.kaspersky.com:443`

## Steps
1. **Go to the connector folder**
   ```bash
   cd ~/opencti/connectors-7.260326.0/external-import/kaspersky
   ```
2. **Build the image** (the trailing `.` is required)
   run in the dir where **DockerFile** is present
   ```bash
   docker build -t opencti/connector-kaspersky .
   ```
3. **Write `docker-compose.yml`** with:
   - `OPENCTI_URL=http://192.168.101.71:8080`, `OPENCTI_TOKEN`, `CONNECTOR_ID` (new UUID)
   - `KASPERSKY_USER`, `KASPERSKY_PASSWORD`, `KASPERSKY_BASE_URL=https://tip.kaspersky.com`
   - `KASPERSKY_CERTIFICATE_PATH=/run/secrets/kaspersky_certificate`
   - top-level `secrets:` -> `file: /root/opencti/certificate.pem`
   - service `networks: [opencti_net]` and top-level `networks: opencti_net: {external: true, name: xtm_default}`
4. **Validate and start**
   ```bash
   docker compose config --quiet && echo OK
   docker compose up -d --force-recreate
   ```
5. **Check**
   ```bash
   docker inspect kaspersky-ti-connector --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'   # xtm_default
   docker compose logs --tail 40      # "sending bundle to queue", no pika/gaierror errors
   ```
6. **Verify in OpenCTI UI**
   - Data -> Ingestion -> Connectors -> Kaspersky: works progressing
   - Analyses -> Reports: Kaspersky threat reports
   - Open a report -> **Knowledge** -> Indicators / Observables = IOCs

## Problems we hit and fixes
| Error | Cause | Fix |
|---|---|---|
| `docker build requires 1 argument` | missing `.` | add `.` at the end |
| `invalid reference format` | en-dash (`–`) from copy-paste | type `--` normally |
| `bind source path does not exist: /path/to/certificate.pem` | sample path in compose | set real cert path |
| `Could not find the TLS certificate file` | host path used inside container | `KASPERSKY_CERTIFICATE_PATH=/run/secrets/kaspersky_certificate` |
| `gaierror(-3)`, AMQP/pika failed, works stuck "In Progress" | connector not on OpenCTI network, `rabbitmq` not resolvable | attach to `xtm_default` |
| `mapping values are not allowed` | broken indentation / CRLF (`^M`) from paste | rewrite with heredoc, `sed -i 's/\r$//'` |
| `401 AUTH_REQUIRED` | invalid or truncated OpenCTI token | create a fresh token, verify with `me` query |

## Notes
- `docker restart` does not pick up env/config changes; use `docker compose up -d --force-recreate`.
- `KASPERSKY_PUBLICATION_START_TIMESTAMP=0` imports full history (very large). Use a recent date for a PoC.
- Master IOC runs Monday, Master YARA runs Tuesday (`*_FETCH_WEEKDAY`).
- `Unable to parse last_modified value` on old publications is harmless.
- Keep tokens and passwords out of chats and screenshots; rotate anything already shared.
