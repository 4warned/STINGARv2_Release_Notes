# STINGAR v2.4 Release Notes

**Release Date:** July 2026

STINGAR v2.4 extends IDS/IPS Rules with a Zeek notice-policy output format, adds JA4 and JA4H fingerprint rules alongside existing HASSH and JA3 support, and defaults Dionaea deployments to publish HTTPS on port 443 for TLS fingerprint capture. The v2.4 honeypot images also add per-sensor de-fingerprinting (Conpot and Cowrie) and TLS client fingerprint capture, making sensors harder for ICS and SSH scanners to identify and clustering scanner activity by JA3/JA4. Honeypot deployment supports both Docker and Podman on remote sensor hosts (automatic runtime selection via Langstroth). The STINGAR **admin server** can be installed on RedHat-family Linux (RHEL 8/9, Rocky, AlmaLinux) using Podman and native `podman compose`, with an OS-aware quickstart installer and updated installation documentation. **Also in this release:** optional **LDAP / Active Directory login** for the UI (LDAPS bind; users must still exist in STINGAR User Management), and a **Delete** control for honeypot deployments in the UI.

---

## Upgrade Steps

- [ACTION REQUIRED] Update docker-compose to use v2.4 image tags
- Ensure management services use versioned image tags (e.g. `stingar-api:v2.4`, `stingar-ui:v2.4`, `langstroth:v2.4`)
- Pull `langstroth:v2.4` to enable Docker/Podman host auto-detection, Python interpreter bootstrap on modern Ubuntu and RHEL-family honeypot hosts, RHEL 8 `podman-compose` fix, and other honeypot deployment fixes (see New Features and Bug Fixes)
- **RHEL / Rocky / AlmaLinux admin server:** clone [stingar-quickstart](https://github.com/4warned/stingar-quickstart), run `sudo ./scripts/install_prerequisites.sh`, then `python3 configure_stingar.py` and `./scripts/compose.sh up -d` (see Admin server — Podman install below)
- On Podman admin installs, plan for **manual upgrades** (`./scripts/compose.sh pull && ./scripts/compose.sh up -d`); in-app auto-update remains disabled when `CONTAINER_RUNTIME=podman`
- Pull the v2.4 honeypot images (e.g. `conpot:v2.4`, `cowrie:v2.4`, `dionaea:v2.4`) to pick up de-fingerprinting and TLS capture; per-sensor identities are generated on container start
- [ACTION REQUIRED] The `conpot:v2.4` image was updated with the critical Modbus stability fix (see Bug Fixes). If you pulled an earlier `conpot:v2.4`, force a fresh pull (`docker compose pull conpot`) and recreate (`docker compose up -d --force-recreate conpot`); `docker compose up -d` alone reuses the cached tag
- Re-deploy or refresh sensor configuration so Dionaea sensors receive the updated `443:443` port mapping
- For Zeek integration: install required Zeek packages on sensors before loading the generated script (see New Features)
- **Optional — LDAP login:** set `LDAP_ENABLED=true` and the LDAP host/base (and optionally `LDAP_UID_ATTRIBUTE`) in `stingar.env` or **Settings → Environment Variables → LDAP Configuration**, then restart STINGAR so the **LDAP Login** button appears (see Configuration)
- No new environment variables required for core features if you are not enabling LDAP

---

## Breaking Changes

None. This release is backward compatible with v2.3.x.

---

## New Features

### LDAP / institutional login

- **LDAP Login button** – When `LDAP_ENABLED=true`, the login page shows an **LDAP Login** option alongside local Login
- **LDAPS bind** – Apiarist authenticates against your directory over TLS (default port 636)
- **Allowlisted STINGAR users** – Each person must already exist under User Management; LDAP supplies the password check against the directory (no local STINGAR password required for LDAP users)
- **UID attribute** – `LDAP_UID_ATTRIBUTE` defaults to `uid`; set `sAMAccountName` for Active Directory
- **Settings UI** – LDAP variables are editable under **Settings → Environment Variables → LDAP Configuration**
- **Documentation** – User docs cover setup under Login / LDAP Login (`/docs/login_out/`)

### Honeypot deployments — Delete control

- **Delete deployment** – Operators can remove a honeypot deployment from the Deployments UI (with confirmation), in addition to existing start/stop controls

### IDS/IPS Rules — Zeek output format

- **Zeek tab and export** – Preview and download a self-contained `stingar-honeypot.zeek` notice-policy script alongside Suricata and Snort
- **Feed endpoint** – `GET /api/v2/ids-rules/feed?format=zeek` for Zeek sensors to pull the cached script
- **Background job** – Writes Suricata, Snort, and Zeek cache files on the same schedule
- **Notice types** – `STINGAR::Honeypot_JA3`, `Honeypot_JA4`, `Honeypot_JA4H`, `Honeypot_HASSH`, and related types raised via the Zeek Notice framework
- **Deploy header** – Generated script includes `zkg install` and `@load` instructions for FoxIO JA4, ja3, and hassh packages
- **Event timing** – TLS fingerprints checked on `connection_state_remove` (after FoxIO JA4 population); JA4H checked on `http_message_done`
- **Documentation** – User docs, API docs, and OpenAPI updated for `format=zeek`

### IDS/IPS Rules — JA4 

- **JA4 rules** – Suricata 7+ (`ja4.hash`) and Snort 3 (`ja4_hash`) rules generated from Dionaea TLS sessions
- **JA4H rules** – HTTP client fingerprint extraction from Conpot when present in Elasticsearch
- **Conpot filter** – Honeypot Type selector and API `app=conpot` support
- **Default rule types** – API and feed default to `hassh,ja3,ja4`; status endpoint reports `total_ja4` and `total_ja4h`

### Dionaea HTTPS defaults

- **Port 443** – Apiarist Dionaea template defaults `https_enabled: true` and publishes `443:443`
- **TLS fingerprints** – Works with Dionaea v2.3.3+ images that emit `ja3` and `ja4` on HTTPS connections

### Honeypot images — de-fingerprinting and TLS capture

**Conpot (ICS)**

- **TLS / HTTPS** – New HTTPS listener with a per-sensor self-signed certificate, plus TLS client fingerprint capture (JA3/JA4) and JA4H (HTTP client fingerprint). Handshake-only scans that never send an HTTP request are still captured.
- **Per-sensor S7 model** – Each sensor presents a realistic Siemens SIMATIC S7-300 / S7-1200 / S7-1500 identity (MLFB order number, CPU module type, family) so a fleet shows a believable spread of the devices ICS scanners target instead of one shared default.
- **S7 SZL wire realism** – SZL 0x11 (module identification) now returns a realistic multi-record partial list that `nmap s7-info` and PLCScan parse correctly (Module, Basic Hardware, and a per-family firmware Version). Stock Conpot returned a single short record under the scanner's minimum length (all fields read blank) and crashed on the firmware-version path. SZL 0x1C "Module Type" now reports the CPU designation (e.g. `CPU 1515-2 PN`) instead of a truncated family string.
- **Protocol identity realism** – Coherent ENIP/CIP identity override, BACnet I-Am device-identifier fix, and Modbus/FTP/ENIP template values.

**Cowrie (SSH/Telnet)**

- **Identity de-fingerprinting** – Per-sensor hostname, OS profile, filesystem, and credential database (`userdb.txt`) so default Cowrie tells are removed. See `honeypots/cowrie/HARDENING.md`.

**Dionaea**

- **TLS JA3/JA4 capture** – Fingerprint capture on `tls.accept` hardened and documented for HTTPS sessions.

### Honeypot deployment — Docker and Podman support (remote sensor hosts)

- **Automatic runtime detection** – The v2.4 Langstroth deployment engine inspects the honeypot host's OS family and selects the container runtime automatically: Docker on Debian/Ubuntu hosts and Podman on RedHat-family hosts (RHEL, Rocky, AlmaLinux, Fedora, CentOS Stream, Oracle Linux). No manual configuration or UI change is required.
- **Python interpreter bootstrap** – Before Ansible gathers facts, Langstroth detects the honeypot host OS and ensures a working Python ≥ 3.9 is available. On Debian/Ubuntu (including **Ubuntu 26.04**, where the default may be Python 3.14), it installs `python3.12-minimal` when needed and prefers 3.12 for module compatibility, falling back through 3.14, 3.13, and `python3`. On RHEL-family hosts (including **Oracle Linux**), it installs via `dnf`/`yum` (`python3.12`, `python3.11`, or `python39`) when no suitable interpreter is present.
- **Podman provisioning** – On RedHat hosts, deployment installs `podman` and `podman-docker`, installs `podman-compose` into an isolated virtualenv at `/opt/podman-compose-venv` (Python 3.9 on RHEL 8; system Python 3 on RHEL 9+), and enables the Podman socket for Docker API compatibility. Honeypots start and stop via `podman-compose --env-file stingar-hp.env`.
- **Unchanged Docker path** – Debian/Ubuntu hosts continue to use Docker and `docker compose` exactly as before, so existing deployments are unaffected.
- **Image** – `4warned/langstroth:v2.4` (also tagged `:latest`).

### Admin server — Podman install on RHEL (quickstart)

- **OS-aware installer** – `scripts/install_prerequisites.sh` in [stingar-quickstart](https://github.com/4warned/stingar-quickstart) selects Docker on Debian/Ubuntu and Podman on RedHat-family hosts; override with `CONTAINER_RUNTIME`.
- **Native `podman compose`** – Admin installs use the Compose V2 plugin (`podman compose`), not the Python `podman-compose` package. Validated on Rocky Linux and RHEL UBI 8/9; avoids RHEL 8 Python 3.6 limitations that break pip `podman-compose`.
- **Compose wrapper** – `./scripts/compose.sh` runs `docker compose` or `podman compose` from `.stingar-runtime`.
- **Quickstart templates** – v2.4 image tags, SELinux `:Z` volume labels on Podman, `CONTAINER_RUNTIME` in `stingar.env`.
- **Auto-update** – Podman admin installs set `UPDATE_SERVICE_ENABLED=false`; apiarist disables the update service when `CONTAINER_RUNTIME=podman`. Upgrade manually with `./scripts/compose.sh pull` and `./scripts/compose.sh up -d`.
- **Documentation** – Readthedocs installation guide: new RHEL setup page, container runtime matrix, Compose V2 commands; user-docs updated for Podman admin upgrades and honeypot-host RHEL 8/9 notes.

---

## Bug Fixes

- **Conpot Modbus stability (critical)** – Fixed a 100%-CPU stall where a single malformed Modbus/TCP connection (a frame that declares more bytes than it sends and then closes) pegged a CPU core and made the sensor stop responding on every port until restarted — one crafted packet could take a sensor offline. Verified on a live sensor: ~2,000 such connections keep CPU under 20% and recover to idle immediately.
- **Conpot FTP stability** – Fixed a related CPU-spin on the FTP command channel triggered by rapid connect/disconnect; connect/close scan floods no longer drive sustained CPU.
- **Conpot scan resilience** – Suppressed scanner-driven traceback storms that could saturate CPU and make ports stop responding after Nmap-style scans (upstream Conpot issue #564); the listener now stays up through malformed-probe floods. Benign per-request Modbus rejection errors are demoted from ERROR to INFO.
- **Conpot HTTP** – Added the missing `505` status page and quieted benign startup warnings and per-connection ENIP/cpppo log spam.
- **Honeypot deployment — Compose v1 removal (critical)** – Fixed honeypot deploy and teardown failing with `ERROR! [DEPRECATED]: community.docker.docker_compose has been removed` on hosts running current Ansible (community.docker 4.x, where the docker-compose v1 module was tombstoned). The deploy and stop playbooks now use `community.docker.docker_compose_v2`. The stop playbook filename and an undefined-variable error in the stop path were also corrected.
- **Honeypot deployment — RHEL 8 Podman compose venv** – Langstroth now creates the honeypot `podman-compose` virtualenv with **Python 3.9** on RHEL 8 (system Python 3.6 cannot run current `podman-compose` releases). RHEL 9+ continues to use system Python 3.
- **Honeypot deployment — Docker GPG key on modern Ubuntu** – Replaced the deprecated `apt_key` step (which fails on Ubuntu 24.10/25.04+ and Debian trixie, where the `apt-key` binary was removed) with the keyring-based `signed-by` method, and switched the repo codename lookup to `ansible_distribution_release` so it no longer depends on `lsb_release` being installed.
- **Honeypot deployment — Python on Ubuntu 26.04 and RHEL-family hosts** – Fixed deploy failures on **Ubuntu 26.04** honeypot hosts where Ansible could not gather facts because the default Python 3.14 (or no interpreter) was unsuitable or absent. Langstroth now bootstraps `python3.12` via apt when needed and selects the first working interpreter ≥ 3.9. On **Oracle Linux** and other RHEL-family minimal images with no Python ≥ 3.9, Langstroth installs one via `dnf`/`yum` before continuing with Podman provisioning.

---

## Other Changes

- **UDP port mappings** – Foundation layer publishes UDP honeypot services with `/udp` Docker Compose port syntax where applicable
- **HP App Store templates** – Compose and env templates from the store render correctly when deploying store honeypots
- **Dependency / image hardening** – Apiarist image and lockfile updates for known CVEs in bundled tooling (including Go crypto/net-related fixes and Alpine OpenSSL pin bumps)

---

## Configuration

**LDAP (optional)** — add or set in `stingar.env` (restart API and UI after changes):

```bash
LDAP_ENABLED=true
LDAP_HOST=ldap.example.edu
LDAP_PORT=636
LDAP_BASE=ou=People,dc=example,dc=edu
LDAP_UID_ATTRIBUTE=uid   # use sAMAccountName for Active Directory
```

Existing IDS/IPS feed settings continue to apply.

The Zeek cache file is written alongside the Suricata and Snort caches (default: `stingar-honeypot.zeek` in the same directory as `stingar-honeypot.rules`).

**Zeek sensor prerequisites** (on each Zeek 5+ host):

```bash
zkg install zeek/foxio/ja4
zkg install ja3
zkg install hassh
```

Add to `share/zeek/site/local.zeek`:

```zeek
@load ja4
@load ja3
@load hassh
@load ./stingar-honeypot.zeek
```

Validate with `zeek -b stingar-honeypot.zeek` after installing packages.

---

## Migration Notes

- Existing deployments: update docker-compose to v2.4 tags after pull
- IDS/IPS Rules: configure `format=zeek` feed URL in your Zeek deployment if using the auto-generated script
- Dionaea sensors deployed before v2.4 may need a configuration refresh to pick up the default `443:443` mapping
- JA4 rules require Suricata 7+ or Snort 3 with JA4 keyword support
- **LDAP:** leave `LDAP_ENABLED` unset or `false` to keep local-only login; enabling LDAP does not remove local Login
- **Podman admin server:** in-app automated updates are not available in v2.4; use `./scripts/compose.sh pull` and `./scripts/compose.sh up -d` from the quickstart directory
- **Admin vs honeypot compose tooling:** the admin server uses native `podman compose`; remote honeypot hosts deployed by Langstroth use `podman-compose` in `/opt/podman-compose-venv` (by design)
- **Honeypot host Python:** Langstroth auto-installs Python on supported distros; if deploy still fails with "No Python >= 3.9 interpreter found", ensure the honeypot host can reach its package repos (`apt` or `dnf`) and that `opc`/`ubuntu`/your deploy user can `sudo` without a password prompt

---

## Known Limitations

- Zeek script syntax validation requires the ja3, hassh, and FoxIO ja4 packages to be installed; bare `zeek -b` without those packages will report missing SSL/HTTP/SSH fields
- JA3S and JA4H fingerprint sets may be empty when honeypots in the feed window did not emit those signature types
- LDAP login requires a reachable LDAPS endpoint from the STINGAR API host; directory bind failures appear as authentication failures on the login form

---

**Canonical public URL (after publish):** `https://github.com/4warned/STINGARv2_Release_Notes/blob/main/RELEASE_NOTES_2.4.md`

---
