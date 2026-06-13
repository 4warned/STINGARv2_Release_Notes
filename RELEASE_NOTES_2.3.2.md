# STINGAR v2.3.2 Release Notes

**Release Date:** June 2026

STINGAR v2.3.2 is an operational-stability hotfix for the v2.3 release train. It addresses three customer-reported failure modes in the honeypot tier (Cowrie filesystem path, Cowrie pubkey-auth KeyError, Conpot vCPU pegging with telemetry stopping), pins the honeypot-side Fluent Bit forwarder to a wire-compatible version, and adds defensive resource caps and exception handling so that scanner-driven greenlet leaks can no longer take a sensor offline.
This release also includes a management-plane security fix in `stingar-ui` that closes unauthenticated access to the UI's internal API route handler. Operators should pull the new `stingar-ui` image on the management server as part of the upgrade.

**Published release notes (canonical):** [STINGARv2_Release_Notes on GitHub](https://github.com/4warned/STINGARv2_Release_Notes) - this file: `RELEASE_NOTES_2.3.2.md`.

---

## Upgrade Steps

- [ACTION REQUIRED] Pull the v2.3.2 `stingar-ui` image on the management server (edit the docker-compose.yml file to update image tag for 4warned/stingarui:v2.3.2 then) `docker compose pull stingarui && docker compose up -d stingarui`) to pick up the authentication fix described in Security Fixes
- Remaining management-plane images (`stingar-api`, `elasticsearch`, `kibana`, `fluentd`) are unchanged from v2.3 in behaviour; tags re-issued as v2.3.2 for stack consistency only no action required
- No database migrations
- No new environment variables
- [ACTION REQUIRED] Pull v2.3.2 honeypot images on every sensor: `docker compose pull && docker compose up -d`
- [ACTION REQUIRED] Re-fetch each sensor's `docker-compose.yml` from the management server (langstroth re-deploy or the equivalent in your provisioning workflow) so that the new resource-limit / healthcheck stanzas land on the sensor host

---

## Breaking Changes

None. This release is backward compatible with v2.3 and v2.3.1.

---

## Security Fixes

### stingar-ui: authentication gate on the UI's internal API route handlers

- **Fixed: unauthenticated access to the management UI's `/api/*` route handlers.** The Next.js middleware matcher previously excluded `/api`, so the UI's own route handlers under `stingar-ui/app/api/**` ran with no session check.
- **Scope.** The fix is confined to `stingar-ui`. The internal stingar-ui -> apiarist call path (over the Docker network) is unchanged, as is all authenticated UI behaviour; logged-in clients send the session cookie with same-origin requests and are unaffected. No database migrations and no new environment variables.

---

## Bug Fixes

### Cowrie (carry-over from v2.3.1 hotfix series)

- **Fixed: `FileNotFoundError: 'share/cowrie/fs.pickle'` at attacker login.** Upstream Cowrie 2.9.x relocated its static data files from `share/cowrie/` to `src/cowrie/data/` and renamed the corresponding config key from `share_path` to `data_path`. STINGAR's Cowrie config template now references `data_path = src/cowrie/data` (with `share_path` kept as a backward-compatible alias), restoring the medium-interaction shell that was breaking immediately after every successful auth.
- **Fixed: STINGAR output plugin permanently disabled by `KeyError: 'password'`.** Cowrie 2.x emits `cowrie.login.success` and `cowrie.login.failed` for both password and public-key authentication, but pubkey events have no `password` field. The STINGAR output plugin's direct dictionary access raised a `KeyError`, which Twisted treated as a fatal error in a log observer and *permanently disabled the STINGAR output plugin for the lifetime of the process*. The plugin now uses `entry.get('password', '')` for both events, so a single pubkey attempt no longer takes the entire telemetry stream offline.
- **Fixed: `DeprecationWarning: datetime.datetime.utcnow() is deprecated`** in the Cowrie sensor `checkin.py`. Replaced with the timezone-aware `datetime.datetime.now(datetime.UTC)` form while preserving the wire format (`...Z` suffix) that downstream consumers match against.

### Fluent Bit (carry-over from v2.3.1 hotfix series)

- **Fixed: central Fluentd dropping every event with `skip invalid event: time=[..., {}]`.** Fluent Bit 5.0 changed its default forward-output protocol to inline per-event option tuples (`[time, {options}]`), which central Fluentd 1.19.0's `in_forward` plugin rejects as malformed. The STINGAR Fluent Bit config now sets `Send_options Off` and `Time_as_Integer On`, restoring the scalar timestamp shape Fluentd's Message-mode parser expects.
- **Fixed: silent base-image upgrades.** The honeypot-side `Dockerfile` previously used `FROM fluent/fluent-bit:latest`, which is how the v5.0 protocol change reached production unintentionally. Pinned to `FROM fluent/fluent-bit:5.0`. Re-test the cowrie -> fluent-bit -> fluentd -> elasticsearch wire format before any future major bump.
- **Fixed: twin-repo drift between `4warned/fluentbit` and `4warned/fluent-bit`.** The Docker Hub repositories had diverged because they were being tagged through different ad-hoc workflows. `scripts/build_all_images.sh` now builds the canonical `4warned/fluentbit:vX.Y.Z` and mirrors the same digest to `4warned/fluent-bit:vX.Y.Z` via `docker buildx imagetools create`. Both names always resolve to the same image.

### Conpot (new in v2.3.2)

- **Fixed: vCPU pegging + telemetry stopping under sustained scanner traffic.** Customer report 5/27-5/28: one of two vCPUs maxed out, the management server stopped receiving events, but the container kept logging. Root cause: three classes of scanner-driven exception were leaking out of conpot's per-protocol greenlet bodies and saturating gevent's hub `handle_error`, starving the `log_worker` greenlet that ships events to fluent-bit. New overlay `gevent_exception_filter.py` wraps the hub's `handle_error` and:
  - swallows `ConnectionResetError`, `ConnectionAbortedError`, `BrokenPipeError`, `socket.timeout` at DEBUG (benign peer-close patterns, dominant cause of the screenshot 1 HTTP traceback storm);
  - swallows `modbus_tk.modbus_tcp.ModbusInvalidMbapError` and other `modbus_tk.exceptions.ModbusError` subclasses at INFO (Modbus parser correctly rejecting malformed MBAP headers like the 17,476-byte / 19-bytes-received probe in screenshot 2);
  - swallows `conpot.protocols.s7comm.exceptions.ParseException` at INFO (S7 magic-byte mismatch on non-S7 probes, screenshot 3);
  - falls through to gevent's original handler for everything else, so genuine bugs still surface.
- **Fixed: noisy ERROR-severity logs for expected s7comm protocol behaviour.** Conpot's `s7_server` logs every "bad magic number" rejection at ERROR severity even though the protocol is doing its job rejecting non-S7 traffic on the S7 port. A new logging filter demotes these specific records to INFO so operators' alerting on ERRORs is no longer drowned in scanner noise. Unrelated ERRORs from the same logger are unaffected.
- **Fixed: `-t cowrie` healthcheck typo in the Conpot deployment template.** The Conpot `docker-compose.yml` template in apiarist was carrying a copy-paste from the Cowrie template that mis-reported every Conpot sensor's type to the apiarist check-in endpoint as "cowrie". Corrected to `-t conpot`.

---

## Improvements

### Operational guardrails for the Conpot sensor compose template

The customer-facing Conpot `docker-compose.yml` template (served by apiarist `/api/v2/deployments/{id}/compose`) now ships with resource caps and lifecycle hardening:

- `restart: always` on both `conpot` and `fluentbit` services so a crash auto-recovers
- `mem_limit: 2g`, `mem_reservation: 1g`, `cpus: 1.5`, `pids_limit: 1024` on the `conpot` service - sized for the 2-vCPU / 4 GB Azure VM profile langstroth provisions by default. The cap exists so that even if a future greenlet-leak regression slips past the in-process filter, the container is OOM-killed and restarted within seconds instead of pegging the host.
- `stop_grace_period: 30s` so gevent has time to drain its `log_queue` on `SIGTERM` (the docker default of 10 s is not enough under load)
- `logging.driver: json-file` with `max-size: 50m, max-file: 3` so a runaway log stream cannot fill the host disk

These keys round-trip cleanly through apiarist's `BaseFoundation.build_docker_compose` YAML serializer; the only fields apiarist substitutes at deploy time remain `env_file`, `image` (when `docker_repository` is set), and `ports`.

---

## Configuration

No new environment variables. Existing v2.3 / v2.3.1 settings apply.

The new Conpot resource caps (`mem_limit`, `cpus`, etc.) are hard-coded in the compose template for the default sensor VM profile. Customers who run conpot on smaller or larger VMs and need different caps should override them in their copy of the deployed `docker-compose.yml` after the langstroth pull; future versions may expose these as `hp_options` for selection from the UI (tracked in the v2.4 roadmap).

---

## Migration Notes

- **Management UI pull.** Pull `4warned/stingar-ui:v2.3.2` on the management server and recreate the `stingarui` service so the API authentication fix takes effect. No client-side action is required; existing logged-in sessions remain valid and same-origin requests continue to work. After upgrading, an unauthenticated `GET /api/settings/env` returns `401` instead of configuration data.
- **Honeypot Image tags.** Pull `4warned/conpot:v2.3.2`, `4warned/cowrie:v2.3.2`, `4warned/fluentbit:v2.3.2` (and the mirrored `4warned/fluent-bit:v2.3.2`) on every sensor before `docker compose up -d`.
- **Compose re-pull on sensor.** The Conpot operational guardrails only take effect after the sensor's `docker-compose.yml` has been re-fetched from apiarist. A simple `docker compose up -d` against the old compose will only pick up the new image, not the new resource limits / healthcheck / logging caps. Re-running the langstroth deploy is the supported way to refresh the sensor's compose.
- **Pubkey-auth callers.** Cowrie's STINGAR output plugin now ingests pubkey-auth attempts where prior versions silently dropped them after the first pubkey event of each process. Expect a one-time uptick in `cowrie.login.failed` event volume for sensors that had been quiet since the Cowrie 2.x upgrade.

---

## Known Limitations

- The new resource caps (`mem_limit: 2g`, `cpus: 1.5`) are sized for the default 2-vCPU / 4 GB VM. Operators running on larger hardware should raise them; running on smaller hardware is not recommended for Conpot.
