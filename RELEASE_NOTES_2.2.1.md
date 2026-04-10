> Description
STINGARv2.2.1 Release
Introduces automatic container version checking and update capabilities, enabling seamless updates with user control and safety features.

### Upgrade Steps

* [ACTION REQUIRED]
* Update docker-compose.yml to use versioned tags (v2.2.1)
* Ensure all services use versioned image tags (e.g. `stingar-api:v2.2.1`, `stingar-ui:v2.2.1`))
* Background update service will automatically check for new versions
* Configure update preferences in UI Settings after upgrade

### Breaking Changes

* None - This release is backward compatible with v2.2

### New Features

#### Automatic Container Update System

* **Update Notification on Login**:
  * Users are notified of available updates via popup when logging in
  * Popup displays: "New version available - install (y/n)"
  * Shows version information and changelog link
  * Users can choose: Install Now, Later, or Dismiss

* **Update Modes**:
  * **Manual Mode (Default)**: Shows popup on login when update is available
  * **Auto Mode**: Automatically applies updates when available (during update window)
  * Configurable in UI Settings > System Settings

* **Update Window Configuration**:
  * Default update window: 00:00 - 06:00 (midnight to 6am)
  * Configurable start and end times via settings
  * Image pulls and container restarts restricted to update window
  * Minimizes impact on production usage

* **Rollback Capability**:
  * Automatic rollback on update failure
  * Manual rollback available for 24 hours after update
  * Previous images cached for 24-hour rollback window
  * Rollback via UI or API endpoint

* **Version Management**:
  * Centralized version checking via store.4warned.io
  * Current version displayed in UI Settings
  * Update history tracking
  * Version comparison and changelog access

* **System Settings Integration**:
  * New System Settings page in UI
  * Update mode configuration (Manual/Auto)
  * Update window configuration
  * Current and latest version display
  * Update history view
  * Manual update trigger button

* **API Endpoints**:
  * `GET /api/v2/system/version` - Get current installed version
  * `GET /api/v2/system/update-check` - Check for available updates
  * `GET /api/v2/system/update-status` - Get update availability status
  * `POST /api/v2/system/update/apply` - Apply available update
  * `GET /api/v2/system/update/status/{job_id}` - Check update job status
  * `POST /api/v2/system/update/rollback` - Rollback to previous version
  * `PUT /api/v2/settings/update` - Configure update settings

### Bug Fixes

* N/A - Initial release of auto-update feature

### Other Changes

* N/A

### Configuration

#### Environment Variables

* `UPDATE_CHECK_INTERVAL`: Seconds between version checks (default: 3600)
* `UPDATE_WINDOW_START`: Start of update window in HH:MM format (default: "00:00")
* `UPDATE_WINDOW_END`: End of update window in HH:MM format (default: "06:00")
* `CLEANUP_OLD_IMAGES`: Enable old image cleanup (default: false)
* `CLEANUP_AFTER_HOURS`: Hours to keep old images (default: 24)

#### Settings API

* `update_mode`: "manual" | "auto" (default: "manual")
* `update_window_start`: HH:MM format (default: "00:00")
* `update_window_end`: HH:MM format (default: "06:00")
* `cleanup_old_images`: boolean (default: false)
* `cleanup_after_hours`: number (default: 24)

### Migration Notes

* Existing deployments will need to ensure docker-compose.yml uses versioned tags
* Update service will automatically detect current version on first run
* Default behavior: Manual mode with update window (midnight to 6am)
* Users can configure update preferences in UI after upgrade
* Old images will be kept for 24 hours to allow rollback

### Known Limitations

* Update window must be configured to avoid conflicts with production usage
* Image cleanup disabled by default until migration period confirmed
* Requires network connectivity to store.4warned.io for version checks
