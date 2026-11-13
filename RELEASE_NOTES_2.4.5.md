# STINGAR v2.4.5 Release Notes

**Release Date:** November 2026

**Release Type:** Patch (security hotfix)

STINGAR v2.4.5 is a security hotfix for current v2.4.x deployments. It closes an unauthenticated attack chain identified in an external AI driven pentest, hardens settings and edge configuration, and is recommended for all customers still on v2.4.4 or earlier.

---

## Upgrade Steps

- [ACTION REQUIRED] Update docker-compose image tags to `v2.4.5` for `stingar-api` (`4warned/stingar-api:v2.4.5`) and `stingar-ui` (`4warned/stingar-ui:v2.4.5`)
- Pull and recreate those services after updating tags
- Apply the reference compose / nginx (See below) / env updates from this release (Fluentd port publish, nginx security headers, env example cleanup) if you maintain a customized stack
- [ACTION REQUIRED] Rotate secrets on any install that was internet-reachable before this upgrade: `API_KEY`, `FLUENTD_KEY`, `CIF_TOKEN`, `PASSPHRASE`, `SALT`, and user API tokens
- Honeypot images (`cowrie`, `conpot`, etc.) are unchanged in this patch; no honeypot recreate is required solely for v2.4.5

---

## Breaking Changes

None for normal operator workflows. Behavioral security tightenings:

---

## Security

This release remediates the critical unauthenticated chain and related high findings from the external assessment.

### Additional hardening

- **Nginx edge headers** – Reference nginx config enables TLSv1.3 alongside TLSv1.2.
- **Env example cleanup** – Removed the unused / confusing `ENABLE_DEV_LOGIN`

### Deferred to next release : (not in this hotfix)

- Full user role model (admin vs viewer) and broader authz enforcement (F-03)
- Removal of the raw Docker socket from the API in favor of a filtered broker

---

## New Features

None. This is a security patch only.

---

## Improvements

- Regression coverage for the security hardening paths in apiarist (`test_security_hardening.py` and related user/model tests)

---

## Bug Fixes

N/A for this release (security remediations only).

---

### Reference `nginx.conf`

Apply these changes if you maintain a customized nginx config instead of the shipped reference file.

**TLS protocols**

```nginx
# Before (v2.4.4 and earlier)
ssl_protocols TLSv1.2;
# After (v2.4.5)
ssl_protocols TLSv1.2 TLSv1.3;
```

**Security response headers** (server block, with `always` so they apply on error responses too):

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

Reload the nginx/`web` container after updating the config (`docker compose restart web`)

## Configuration

### Changed behavior / compose

| Item | Change |
|------|--------|
| nginx | Security headers + TLSv1.3 in the reference config |

### New Environment Variables

No new required environment variables for this release.

### Existing Settings

CIF, syslog, LDAP/SAML, and honeypot settings are unchanged by this patch.

---

## Migration Notes

1. Upgrade `stingar-api` and `stingar-ui` to `v2.4.5`, then recreate those containers.
2. Merge reference compose/nginx/env changes if your deployment diverges from the shipped templates.
3. **Rotate all secrets** if the instance was reachable before the upgrade (the prior chain could disclose the master API key and other settings).
4. Confirm `/initial-admin` is no longer usable after setup (expect closed / redirected behavior).
5. Confirm Settings no longer shows recoverable plaintext for `API_KEY` / `FLUENTD_KEY` / `CIF_TOKEN` / related secrets.
6. If you scrape Fluentd forward on a public host port, move collectors to the private network or an explicit allowlist before relying on the new compose defaults.

---

## Known Limitations

- Docker socket remains mounted for self-update / version detection on the API container until the v2.5 brokered-access work; keep admin credentials tightly controlled.
- Role-based access control (viewer vs admin) is not introduced in v2.4.5.
- Historical Elasticsearch / feed data is unaffected; this patch does not reindex events.

---

## Images Changed

- `4warned/stingar-api:v2.4.5` (linux/amd64, linux/arm64)
- `4warned/stingar-ui:v2.4.5` (linux/amd64, linux/arm64)

Reference config (not separate image tags): `infra/docker/docker-compose.yml`, `infra/docker/nginx.conf`, `infra/env/stingar.env.example`.

---

## Future Enhancements

- v2.5 continues security remediation: role model, Docker socket broker, and remaining defense-in-depth items
