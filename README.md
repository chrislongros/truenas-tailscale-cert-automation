# TrueNAS Tailscale WebUI Certificate Automation

This repository provides a **generic, safe, and cron-friendly script** to automatically:

- Obtain or renew a **Tailscale HTTPS certificate**
- Export it from a **Tailscale Docker container**
- Import it into **TrueNAS SCALE**
- Apply it to the **TrueNAS Web UI**
- Restart the UI safely
- **Clean up old certificates** to avoid clutter

The script is designed to work reliably with **TrueNAS SCALE cron jobs**, which run in a restricted environment.

---

## Features

- Works with **Tailscale running in Docker**
- Cron-safe (handles limited PATH and swallowed output)
- Internal logging to a configurable file
- **Date-based certificate naming** (`tailscale-ui-YYYY-MM-DD`) for easy tracking
- **Automatic cleanup** of old `tailscale-ui-*` certificates after each renewal
- Ensures cert directory exists with secure permissions (`700`)
- Imports the certificate through the **TrueNAS API client**, so multi-KB PEM
  payloads never hit command-line argument limits and the private key never
  appears in `ps` output
- Waits for the import job to finish before switching the UI over
- No hardcoded paths, hostnames, or secrets in the template

---

## Requirements

- TrueNAS SCALE
- Docker
- A running Tailscale container (`tailscaled`)
- A bind mount from the host into the container at `/certs`
- `jq` installed on the TrueNAS host
- `python3` with the TrueNAS API client (`truenas_api_client`), which ships
  preinstalled on TrueNAS and also provides `midclt`

---

## Compatibility

Verified against the documented middleware API for **TrueNAS SCALE 24.10 through 26.x**.

The import uses the preinstalled WebSocket API client rather than the REST API,
which is removed in TrueNAS 26. `certificate.create`
(`CERTIFICATE_CREATE_IMPORTED`), `certificate.query`, `system.general.update`
(`ui_certificate`) and `system.general.ui_restart` are all still present in the
26.x API.

---

## How it works

1. Runs `tailscale cert` inside a Docker container
2. Writes `ts.crt` and `ts.key` to a host-mounted directory
3. Imports the certificate into the TrueNAS certificate store with a date-based name,
   passing the PEM data over the middleware socket via the Python API client and
   waiting for the import job to complete
4. Reuses the existing certificate if one with today's name is already present,
   so re-running the script the same day is safe
5. Sets it as the Web UI certificate
6. Restarts the Web UI
7. Deletes any previous `tailscale-ui-*` certificates to keep the store clean

---

## Files

- `import_tailscale_cert`
  Main automation script (generic template, no personal data)

---

## Configuration

The script uses **placeholders only**.
You must edit the following variables before use:

```bash
CONTAINER_NAME="__TAILSCALE_CONTAINER_NAME__"
TS_HOSTNAME="__TAILSCALE_DNS_NAME__"
HOST_CERT_DIR="__HOST_CERT_DIR__"
LOG_FILE="__LOG_FILE__"
```

| Variable | Description | Example |
|---|---|---|
| `CONTAINER_NAME` | Docker container running `tailscaled` | `ix-tailscale-tailscale-1` |
| `TS_HOSTNAME` | Your Tailscale device's HTTPS name | `mynas.my-tailnet.ts.net` |
| `HOST_CERT_DIR` | Host path bind-mounted to `/certs` in the container | `/mnt/pool/certs` |
| `LOG_FILE` | Path to the log file | `/mnt/pool/logs/cert-import.log` |

---

## Setting up the cron job

1. Copy the script to your TrueNAS host and make it executable:
   ```bash
   chmod +x import_tailscale_cert
   ```

2. Edit the placeholder variables at the top of the script.

3. Add a cron job in the TrueNAS Web UI:
   - **System > Advanced > Cron Jobs > Add**
   - **Command:** `/path/to/import_tailscale_cert`
   - **Run As User:** `root`
   - **Schedule:** Monthly (Tailscale certs are valid for 90 days)

---

## Troubleshooting

### The import log ends in `Killed`

Symptom (see issue #1):

```
import_tailscale_cert: line 81: 53481 Killed
```

Older versions of this script passed the certificate and private key to
`midclt` as a single command-line argument. A PEM chain plus key runs to
several KB and can exceed the argument length limit, killing the import
part-way through. The script now hands that payload to middleware over its
local socket instead, so the longest argument it passes is a few dozen bytes.

`midclt` itself only gained the ability to read a JSON payload from stdin
(`midclt call certificate.create -`) in 2026, so that alternative is not
available on 25.10 and earlier.

### `ERROR: no python3 found that can import the TrueNAS API client`

The script checks `python3` and, failing that, the interpreter from `midclt`'s
shebang. If neither can import `truenas_api_client` or `middlewared.client`,
confirm `midclt call system.info` works on the host.

---

## License

MIT
