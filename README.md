# TrueNAS Tailscale WebUI Certificate Automation

This repository provides a **generic, safe, and cron-friendly script** to automatically:

- Obtain or renew a **Tailscale HTTPS certificate**
- Export it from a **Tailscale Docker container**
- Import it into **TrueNAS SCALE**
- Apply it to the **TrueNAS Web UI**
- Restart the UI safely

The script is designed to work reliably with **TrueNAS SCALE cron jobs**, which run in a restricted environment.

---

## ✨ Features

- Works with **Tailscale running in Docker**
- No Kubernetes required
- Cron-safe (handles limited PATH and swallowed output)
- Does **not rely on fragile return values** from `midclt`
- Optional internal logging
- Uses a **stable certificate name** to avoid clutter
- No hardcoded paths, hostnames, or secrets in the template

---

## 📋 Requirements

- TrueNAS SCALE
- Docker
- A running Tailscale container (`tailscaled`)
- A bind mount from the host into the container at `/certs`
- `jq` installed on the TrueNAS host

---

## 🔧 How it works

1. Runs `tailscale cert` inside a Docker container
2. Writes `ts.crt` and `ts.key` to a host-mounted directory
3. Imports the certificate into the TrueNAS certificate store
4. Locates the certificate by **name**
5. Sets it as the Web UI certificate
6. Restarts the Web UI

---

## 📁 Files

- `import_tailscale_cert.sh`  
  Main automation script (generic template, no personal data)

---

## ⚙️ Configuration

The script uses **placeholders only**.  
You must edit the following variables before use:

```bash
CONTAINER_NAME="__TAILSCALE_CONTAINER_NAME__"
TS_HOSTNAME="__TAILSCALE_DNS_NAME__"
HOST_CERT_DIR="__HOST_CERT_DIR__"
LOG_FILE="__LOG_FILE__"
TRUENAS_CERT_NAME="__TRUENAS_CERT_NAME__"
