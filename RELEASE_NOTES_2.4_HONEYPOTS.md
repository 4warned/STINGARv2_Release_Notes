# STINGAR v2.4 Honeypot Release Notes

**Release Date:** June 2026

**Scope:** Honeypot images only. This document covers the v2.4 honeypot
container updates (Conpot, Cowrie, Dionaea). The v2.4 management/server-side
features are tracked separately and will be published with the full v2.4 release notes.

STINGAR v2.4 honeypots add per-sensor de-fingerprinting and TLS client
fingerprint capture. Each sensor now presents a more believable, individualized
identity, so common ICS and SSH scanners have a harder time flagging the
deployment as a honeypot, and scanner activity can be clustered by JA3/JA4
fingerprint.

---

## Upgrade Steps

- [ACTION REQUIRED] Pull the v2.4 honeypot images (e.g. `conpot:v2.4`,
  `cowrie:v2.4`, `dionaea:v2.4`) and update docker-compose to the v2.4 tags
- Re-deploy or restart sensors so per-sensor identities are generated on
  container start
- For Conpot HTTPS / Dionaea HTTPS fingerprint capture, ensure the sensor
  publishes the TLS port (`443`) and that any cloud firewall / security group
  allows it
- No new environment variables are required

---

## Breaking Changes

None. The v2.4 honeypot images are drop-in replacements for v2.3.x.

---

## New Features

### Conpot (ICS)

- **HTTPS / TLS listener** – New HTTPS service with a per-sensor self-signed
  certificate, so Conpot presents an industrial HTTPS endpoint rather than
  HTTP only.
- **TLS client fingerprints** – Captures JA3 and JA4 from the TLS ClientHello,
  including handshake-only scans that never send an HTTP request.
- **JA4H** – HTTP client fingerprint captured on HTTPS requests.
- **Per-sensor S7 model** – Each sensor presents a realistic identity so a fleet
  shows a believable spread of the devices ICS scanners target instead of one
  shared default value.
- **S7 SZL wire realism** – SZL 0x11 (module identification) returns a realistic
  multi-record partial list that `nmap s7-info` and PLCScan parse correctly,
  reporting Module, Basic Hardware, and a per-family firmware Version
  (S7-1200 V4.x, S7-1500 V1.x-V3.x, S7-300 up to ~V3.3). SZL 0x1C "Module Type"
  now reports the CPU designation (e.g. `CPU 1515-2 PN`) instead of a marketing
  family string.
- **Protocol identity realism** – Coherent ENIP/CIP identity override, a BACnet
  I-Am device-identifier fix, and updated Modbus / FTP / ENIP template values.

### Cowrie (SSH/Telnet)

- **Identity de-fingerprinting** – Per-sensor hostname, OS profile, emulated
  filesystem, and credential database (`userdb.txt`), removing default Cowrie
  tells that scanners key on.

### Dionaea

- **TLS JA3/JA4 capture** – Fingerprint capture on `tls.accept` hardened and
  documented for HTTPS sessions; sensors emit `ja3`, `ja4`, and `ja4_r`.

---

## Bug Fixes

- **Conpot scan resilience** – Suppressed scanner-driven traceback storms that
  could saturate CPU and make ports stop responding after Nmap-style scans
  (upstream Conpot issue #564). The listener now stays up through
  malformed-probe floods.
- **Conpot S7 firmware path** – Fixed an upstream crash on the SZL index 6/7
  firmware-version response (it packed a version string into an integer field).
- **Conpot HTTP** – Added the missing `505` status page and quieted benign
  startup warnings and per-connection ENIP/cpppo log spam.

---

## Configuration

No new environment variables for the v2.4 honeypot images. Per-sensor
identities (S7 model and firmware, Cowrie hostname/OS/filesystem/credentials,
and TLS certificates) are derived deterministically per sensor at container
start; no manual configuration is required.

---

## Verification

- **Conpot S7** – `nmap -sV -p102 --script s7-info <sensor>` should report a
  coherent `Siemens S7 PLC` with populated Module, Basic Hardware, and Version,
  and should complete without script errors.
- **Conpot / Dionaea TLS** – A TLS connection to the HTTPS port should produce a
  JA3/JA4 fingerprint in the sensor telemetry; a bare connect-scan (no HTTP
  request) should still record a handshake-only event.
- **Cowrie** – An SSH session should present the per-sensor hostname and OS
  profile rather than the Cowrie defaults.

---

## Known Limitations

- Per-sensor TLS certificates are self-signed; clients that pin or validate a CA
  chain will reject them (expected for a honeypot).
  
---
