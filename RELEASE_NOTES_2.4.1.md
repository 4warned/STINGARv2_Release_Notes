# STINGAR v2.4.1 Release Notes

**Release Date:** TBD

## Overview

STINGAR v2.4.1 is a patch release that adds supported **syslog over TLS** for SIEM integrations (customer security hardening), and may include concurrent dependency/CVE fixes from the maintenance process.

## Upgrade Steps

- Update docker-compose (or quickstart) image tags to `v2.4.1` for at least `fluentd`, `stingar-api`, and `stingar-ui`
- Pull and recreate Fluentd after upgrading so outbound syslog TLS config is available
- No action required if you do not use syslog, or if you keep `SYSLOG_TLS=false` (default)

## Breaking Changes

- None. Existing UDP/TCP plaintext syslog behavior is unchanged when `SYSLOG_TLS` is unset or `false`.

## New Features

- **Syslog over TLS** – Optional TCP+TLS forwarding of attack logs to a SIEM collector via Fluentd `fluent-plugin-remote_syslog`
  - New env/settings: `SYSLOG_TLS`, `SYSLOG_TLS_CA_FILE`, `SYSLOG_TLS_VERIFY`
  - Compose mounts `infra/certs/syslog` into Fluentd at `/fluentd/etc/syslog-certs`
  - Settings UI exposes the new Syslog Output fields; API rejects TLS with non-TCP protocol
  - User docs: Syslog to SIEM over TLS

## Improvements

- `SYSLOG_PROTOCOL` validated to `udp` or `tcp` only

## Bug Fixes

- (CVE / dependency fixes to be listed if shipped in the same tag)

## Configuration

```sh
SYSLOG_ENABLED=true
SYSLOG_PROTOCOL=tcp
SYSLOG_TLS=true
SYSLOG_HOST=siem.example.com
SYSLOG_PORT=6514
SYSLOG_TLS_VERIFY=peer
SYSLOG_TLS_CA_FILE=/fluentd/etc/syslog-certs/ca.pem
```

Restart Fluentd after changing syslog TLS settings.

## Known Limitations

- Mutual TLS (client certificates) is not supported
- Fluentd must be recreated/restarted to apply syslog env changes
