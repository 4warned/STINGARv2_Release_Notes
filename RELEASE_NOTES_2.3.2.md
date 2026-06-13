# STINGAR v2.3.2 Release Notes

**Release Date:** June 2026

STINGAR v2.3.2 is a stability and security hotfix for the v2.3 release train. It keeps Conpot sensors healthy under heavy internet scanning (previously one CPU could max out and a sensor would stop reporting data), carries forward the Cowrie and Fluent Bit fixes from the v2.3.1 hotfix series, and closes a security gap in the management UI.

Most customers only need to pull the new images and refresh their sensor configuration. See Upgrade Steps below.

**Published release notes (canonical):** [STINGARv2_Release_Notes on GitHub](https://github.com/4warned/STINGARv2_Release_Notes) - this file: `RELEASE_NOTES_2.3.2.md`.

---

## Upgrade Steps

1. **[ACTION REQUIRED] Update the management UI.** Set the `stingarui` image tag to `4warned/stingarui:v2.3.2` in your management `docker-compose.yml`, then run `docker compose pull stingarui && docker compose up -d stingarui`. This applies the security fix below.
2. **[ACTION REQUIRED] Update every sensor.** On each honeypot host, run `docker compose pull && docker compose up -d`.
3. **[ACTION REQUIRED] Refresh each sensor's configuration.** Re-deploy the sensor (langstroth, or your normal provisioning workflow) so the new Conpot resource limits and health settings reach the host. A plain `docker compose up -d` picks up the new image but not the new limits.

No database migrations. No new environment variables. The other management images (`stingar-api`, `elasticsearch`, `kibana`, `fluentd`) are unchanged from v2.3 and are re-tagged as v2.3.2 only for consistency.

---

## Breaking Changes

None. This release is backward compatible with v2.3 and v2.3.1.

---

## Security Fixes

**Management UI: closed unauthenticated access to internal API routes.** The STINGAR UI's internal API endpoints could be reached without a logged-in session. They now require authentication. The fix is contained to the UI image; logged-in users are unaffected and no configuration changes are needed. Pull the v2.3.2 `stingarui` image to apply it.

---

## Bug Fixes

### Conpot (new in v2.3.2)

- **Fixed: a sensor could max out a CPU and stop reporting under heavy scanning.** Internet background scanners send large volumes of malformed or incomplete traffic to the honeypot's emulated services. Conpot was logging a full error trace for each of these probes, and at scanner volumes that logging work could saturate one CPU core and starve the process that ships events to the management server, so the sensor went quiet while the container kept running. STINGAR now recognizes these routine scanner errors across all of Conpot's services (HTTP, Modbus, S7comm, BACnet, EtherNet/IP, and others) and handles them quietly instead of logging an expensive trace for every probe. Genuine, unexpected errors are still logged normally.
- **Fixed: noisy error logs from normal scanner behavior.** Some scanner requests are simply invalid by protocol design (for example, broadcasting a Modbus command that the spec does not allow to be broadcast). These were being logged as errors; they are now logged at an informational level so they no longer clutter error reporting or trigger false alerts.
- **Added: startup confirmation in the logs.** Each sensor now logs a short line at startup confirming the protective filters loaded, so operators can verify the fix is active.

### Cowrie (carry-over from the v2.3.1 hotfix series)

- **Fixed: emulated shell broke immediately after attacker login.** A change in the upstream Cowrie data-file layout caused a startup error on every successful login. The Cowrie configuration was updated to match, restoring the interactive shell.
- **Fixed: telemetry could stop after a public-key login attempt.** A single SSH public-key attempt could trigger an error that silently disabled Cowrie's STINGAR reporting for the life of the process. Public-key attempts are now handled correctly and reported alongside password attempts.

### Fluent Bit (carry-over from the v2.3.1 hotfix series)

- **Fixed: events dropped between sensors and the management server.** A change in the default forwarding format of a newer Fluent Bit release caused the central collector to reject every event. The forwarder configuration was corrected to restore delivery.
- **Fixed: unintended forwarder upgrades.** The sensor-side Fluent Bit image is now pinned to a known-good version so future upstream changes cannot reach production unintentionally.
- **Fixed:** consolidated two Docker Hub image names that had drifted apart so they always point to the same build.

---

## Improvements

### Stronger guardrails on Conpot sensors

The Conpot sensor configuration now includes resource and lifecycle safeguards, sized for the default 2-CPU / 4 GB sensor VM:

- Automatic restart if the honeypot or its forwarder crashes.
- Memory, CPU, and process limits so that even in a worst case the container is restarted promptly instead of overwhelming the host.
- A longer shutdown grace period so queued events are delivered on restart.
- Log rotation so honeypot logs cannot fill the host disk.

These take effect after you refresh the sensor's configuration (Upgrade Step 3).

---

## Configuration

No new environment variables. Existing v2.3 / v2.3.1 settings still apply.

The new Conpot resource limits are tuned for the default sensor VM size. If you run Conpot on noticeably smaller or larger hardware, you can adjust these values in the sensor's deployed `docker-compose.yml`. A future release is expected to make them selectable from the UI.

---

## Migration Notes

- **Management UI.** After pulling `4warned/stingarui:v2.3.2` and recreating the `stingarui` service, the security fix is active. Existing logins remain valid; no client-side action is needed.
- **Sensor images.** Pull `4warned/conpot:v2.3.2`, `4warned/cowrie:v2.3.2`, and `4warned/fluentbit:v2.3.2` on every sensor before `docker compose up -d`.
- **Refresh sensor configuration.** The new Conpot safeguards only apply after the sensor's configuration is re-fetched from the management server (Upgrade Step 3).
- **Login event volume.** Because Cowrie now reports SSH public-key attempts that earlier versions dropped, sensors that had been quiet may show a one-time increase in failed-login event volume.

---

## Known Limitations

- The Conpot resource limits are sized for the default 2-CPU / 4 GB sensor VM. Increase them if you run on larger hardware; running Conpot on smaller hardware is not recommended.
