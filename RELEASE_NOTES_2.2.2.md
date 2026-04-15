> Description
STINGARv2.2.2 Release
Improves web UI accessibility (WCAG 2.5 Level AA), mobile usability, and reliability of login and server-side configuration when credentials are supplied via `stingar.env`.

### Upgrade Steps

* [ACTION REQUIRED]
* Update `docker-compose.yml` to use versioned tags (`v2.2.2`)
* Ensure all services use versioned image tags (e.g. `stingar-api:v2.2.2`, `stingar-ui:v2.2.2`)
* After upgrade, restart the stack so containers pick up new images
* Configure update preferences in UI Settings if you use the automatic update service (unchanged from v2.2.1)

### Breaking Changes

* None. This release is backward compatible with v2.2.1

### New Features

#### Accessibility (WCAG 2.5 Level AA)

* **Broader WCAG 2.5 compliance** across dashboard and public flows (navigation, forms, tables, and interactive components) with improved focus, labels, and semantics where applicable

#### Mobile device support

* **Responsive behavior** – Layout and interaction adjustments for smaller viewports so core workflows remain usable on mobile devices alongside the WCAG work

### Improvements

* **Configurable env file path** – Optional `STINGAR_ENV_PATH` points to the env file to read (default `/app/stingar.env` when used)
* **Session creation** – `createSession` supports flows that complete navigation on the client when server-side redirects are undesirable

### Bug Fixes

* **Fixed: Login redirect after successful authentication** – Successful login could fail to land users on the dashboard reliably due to Server Action redirect handling; the UI now completes navigation after success (`router.refresh()` and `router.push("/dashboard")`) and avoids showing an error state on success
* **Fixed: Missing `PASSPHRASE` / `API_KEY` in some Docker setups** – Session encryption and authenticated API calls now tolerate env values supplied via `stingar.env` with clearer failure modes when `PASSPHRASE` is absent
* **Fixed: Initial admin and login bootstrap checks** – Admin presence checks aligned with the current API response shape
* **Initial admin flow** – Updates to the initial admin experience consistent with the authentication changes above

### Other Changes

N/A

### Configuration

#### Environment Variables

N/A

### Migration Notes

* **Image tags** – Move all STINGAR images to `v2.2.2` tags in compose or your internal deployment manifests; pull fresh images before restart
* **Env files** – If you already mount or generate `stingar.env`, no structural change is required; v2.2.2 improves how the UI container consumes those values at runtime
* **Version state / updater** – If you deploy without the built-in updater, ensure your operational source of truth for “installed version” matches the running images (see product documentation on version state if you rely on update notifications)

### Known Limitations

* Automatic update behavior, update windows, and store connectivity requirements are unchanged from v2.2.1; see `RELEASE_NOTES_2.2.1.md` for that feature set
* Accessibility work is continuous; report specific gaps through your normal support channel
