# Changelog

## 1.6.1

### Behaviour Changes

- The SDK now records when it comes up and when it is shut down. An orderly shutdown writes a closing marker — its absence means the app was killed rather than closed
- Recording start and stop rows now carry why they happened: a start says whether it came from the host, a resume or a configured delay, and a stop says whether the host, a shutdown, a traffic limit or a full disk ended it
- Status events (recording start and stop, wake lock, Doze, app foreground and background) are now delivered on every account, not only on accounts that upload around metadata events

### Bug Fixes

- Fixed a recording resumed after a restart reporting the wrong start origin when a start delay is configured
- Fixed status events being deleted before they could upload — by the retention horizon and by the storage-full cleanup
- Fixed a shutdown interrupted mid-write dropping sensor readings it had already taken out of the queue
- Fixed the shutdown status event being lost when the app is closed: it is now stored during shutdown and sent with the next launch
- Fixed `stopRecording()` writing an end-of-recording event when there was no recording to end
- Fixed readings recorded on the way into a traffic limit surviving the wipe the limit requires
- Fixed customer metadata queued at shutdown losing its upload window
- Fixed the SDK reporting a permissions request with no missing permissions, which overwrote the `Initialized` status — an app waiting for `Initialized` could wait forever

## 1.6.0

### Behaviour Changes

- Initializing the SDK no longer starts recording by itself — a new recording begins when the host calls `startRecording()`. This restores the behaviour of 1.2.x and earlier; the delayed-start option added in 1.3.0 defaulted to starting on init, so an app that called `init()` without ever calling `startRecording()` has been recording since then. To keep the previous behaviour, pass `delayAutoStartRecording(SdkConfiguration.AUTO_START_ON_INIT)`, or a positive delay to start after it elapses. A recording the host had started is resumed the next time the app initializes the SDK, so restarting the app does not lose it — an app that is killed or updated mid-recording picks up where it left off. That also means a device that happened to be recording when it was updated keeps recording: stopping it once, with `stopRecording()` or `deinitialize()`, is enough, and every launch after that follows the new default. Any flow that already ends a shift or logs the user out does this if it calls one of those two. Calling `init()` again does not — on an initialized SDK it returns immediately, and in a fresh process it is what resumes the recording

### Bug Fixes

- Fixed a crash when any standard metadata field was `null` — from Java, C# or Dart, where the compiler cannot prevent it. Only `extra` had been made optional in 1.5.4; the other nine fields still threw inside the constructor, before the SDK saw the event at all. Every field is now optional, and an omitted or `null` field is uploaded as an empty string, which is what passing `""` has always produced, so the uploaded event is unchanged
- Fixed the same crash when the metadata call itself was given `null`. A null that says *what* to act on — an event, a payload, a template name, a tag — leaves nothing to act on, so the call is ignored with a log line instead of taking the app down, and logging by a `null` tag reports that nothing was found. A null value inside a payload is missing data rather than a missing target, so it is recorded as an empty string
- Fixed a crash when the additional key-value pairs on an event contained a `null` value; those are now uploaded as empty strings like any other missing value
- Fixed `stopRecording()` being ignored when it arrived during a configured start delay: recording had not begun yet, so the call was discarded and recording then started anyway when the delay elapsed
- Fixed `stopRecording()` being lost when it was called before the SDK finished binding its service, while `startRecording()` in the same moment was kept — turning recording off and on again at app launch could leave it recording. Both calls are now held and the last one wins
- Fixed a crash when a start delay was given in microseconds or nanoseconds
- A negative start delay no longer means "start immediately" in one form of the call and an error in the other; any negative value now means the SDK waits for `startRecording()`
- Fixed a sensor reading with a corrupted timestamp (for example a location fix reporting no time) being able to permanently block local storage and uploads for the rest of the installation; such readings are now detected and dropped at every stage, and devices already stuck on one recover automatically after upgrading
- Fixed uploads stalling permanently when the backend repeatedly rejects the same packet as invalid (HTTP 400/422): after at least three rejections spanning at least 30 minutes the SDK removes that packet and continues uploading the rest of the data. Short-lived backend-side rejection windows (for example during a bad deploy) never delete anything, and size-related rejections (payload too large) are reported but never remove data
- In metadata-triggered mode, fixed readings being duplicated in uploads when a storage write was retried after a transient database error
- Fixed a burst of sensor readings being silently dropped when the storage writer fell behind (present since 1.3.0, most likely while the device was recovering from full storage); the backlog is now merged instead, bounded by shedding the oldest readings first
- Fixed sensor statistics from a stopping recording session leaking into the next session when recording was restarted quickly
- Fixed an internal background watcher not being stopped when the SDK shut down after an authentication error
- `startRecording()` called without an initialized SDK no longer fails silently: it now reports `NOT_INITIALIZED` through the status instead of holding the request that nothing was going to act on. A call made while the SDK is still starting up is still held and applied, as before
- Fixed the reported SDK status staying on "recording" after `deinitialize()`, or after the background service was lost. An integrator that decides whether to call `startRecording()` from the observed status would never start recording again for the rest of the process; the status now returns to uninitialized when the session ends
- Fixed the end-of-recording status event being dropped when nothing followed it: it was written after the session's last flush, so it sat in memory waiting for sensor data that was never going to arrive. Recording start/stop and the other status events now go to storage as soon as they happen

### Improvements

- Improved data collection while the device runs on battery with the screen off: a recording session now keeps the processor awake, so sensors keep sampling at the configured rate throughout
- App foreground/background status events are now included in the recorded data
- The recorded data now also reports when the SDK held the CPU wake lock and when the device entered or left Doze — the two things that decide how much the sensors actually capture while the screen is off. Neither was visible in the data before, so a thinned-out stretch of a recording could not be told apart from a device that was simply idle
- Remote configuration is now validated when read: an upload period below 3 seconds is raised to 3 seconds, so a misconfigured value can no longer make devices upload at extreme rates

## 1.5.8

### Bug Fixes

- Fixed data uploads that could silently stop for the rest of a session when the SDK was re-initialized right after a previous shutdown (for example after an authentication error with a corrected API key)
- On Android 15, if the system ends extended background recording (possible when location permission was never granted), the SDK now reports the recording as stopped through the status listener instead of appearing active after it had silently ended
- Fixed the background recording service sometimes failing to start on Android 14+ when location permission was not granted
- The SDK no longer fails to initialize on devices without WiFi hardware — the WiFi sensor is skipped and everything else records
- In metadata-triggered mode, fixed readings being duplicated across upload packets, and a small window's remaining data not being uploaded

### Improvements

- In metadata-triggered mode, unsent data is now kept for up to 3 days instead of being dropped once it falls outside the metadata window — so a metadata event arriving late (for example after the device was offline) can still trigger the upload of the data it covers
- If device storage fills up, the SDK now drops the oldest unsent readings to keep recording — preserving the newest data and recent history — instead of losing new data, and recovers automatically once space frees up

## 1.5.7

### Bug Fixes

- Fixed sensor recording that could silently stop on some devices while metadata kept being sent; if a sensor can't be initialized (fused orientation), it's now skipped instead of bringing the whole recording down — the SDK keeps recording with the remaining sensors
- Fixed a crash on startup when the SDK was initialized without location permission (for example before the permission was granted, or after it was revoked); the background service now starts reliably, and automatically uses location once the permission is granted
- Fixed rare crashes that could occur when the SDK was stopped and re-initialized in quick succession, or when it was stopped while a configuration update or upload was still in flight
- Fixed sensor data being uploaded repeatedly without being cleared from local storage when `payload_limit_kb` was set high enough to produce packets with tens of thousands of readings; uploads now reliably free local storage on success
- Fixed background data uploads stopping permanently on some devices after the system interrupted an upload (most common with aggressive battery/background management on certain manufacturers); uploads now resume automatically instead of staying stopped until the app was restarted. In metadata-triggered mode this previously risked losing buffered data

### Improvements

- `payload_limit_kb` now supports values up to 9,000 KB (previously capped silently at 1,024 KB); larger packets mean fewer, larger uploads (fewer network round-trips per recording)
- `period_upload` now acts as the maximum age before a partial flush rather than a fixed interval between uploads. Each upload fires as soon as the local buffer reaches 80% of `payload_limit_kb` OR when the oldest pending reading exceeds `period_upload`, whichever comes first. Low values behave the same as before; higher values now produce proportionally larger packets instead of more frequent small ones.

### Notes

- The SDK's foreground services no longer use the "special use" foreground-service type. If your Google Play foreground-service declaration listed "special use" for any Truemetrics service (the main recording service or the optional sensor-watchdog service), update the declaration to "data sync" — the SDK now declares the data-sync type
- On Android 15, if location permission is never granted, the system may pause background recording after extended background runtime; granting location avoids this limit

## 1.5.6

### Bug Fixes

- Fixed excessive mobile data usage and battery drain that could occur when the SDK was unable to reach the server (for example on a restricted or offline network); uploads now back off between retries instead of resending continuously, and recorded data is preserved for delivery once the connection is restored

## 1.5.5

### Bug Fixes

- Fixed background sensor batching not being cancelled on `deinitialize()`, which could leave residual work scheduled after the SDK was stopped
- Fixed a rare crash when background sensor batching ran before the host app re-initialized the SDK; pending sensor data now resumes on the next SDK initialization
- Improved cancellation of in-flight uploads when the SDK is stopped mid-operation

## 1.5.4

### Bug Fixes

- Fixed metadata values being corrupted: characters like `:`, `/`, `\`, `*`, `?`, and `;` are now preserved as sent, so ISO 8601 timestamps, addresses, URLs, paths, and values with semicolons roundtrip correctly
- Fixed crash when `StandardMetadata.extra` is `null` (e.g. from Java callers); the field is now optional
- Fixed a rare crash when the background upload worker was restarted by the system before the host app re-initialized the SDK; pending uploads now resume on the next SDK initialization

## 1.5.3

### Bug Fixes

- Improved crash report accuracy — stack traces now include correct line numbers and method attribution

## 1.5.2

### Bug Fixes

- Fixed spurious error logs when optional stats module is not bundled
- Fixed crash when metadata contains null values (e.g. from Java/Xamarin callers)
- Removed debug recording notification that could appear on some builds

## 1.5.1

### New Features

- Added `logMetadata(StandardMetadata)` for logging standardized delivery/pickup event metadata with structured fields

### Deprecations

- `logMetadata(Map)` is deprecated in favor of `logMetadata(StandardMetadata)`

### Bug Fixes

- Fixed device ID rotation attaching a new device ID to sensor data collected before the rotation
- Increased default device ID rotation period from 14 to 30 days

## 1.5.0

### New Features

- **Sensor Statistics**: Real-time statistical aggregation for IMU sensors (accelerometer, gyroscope, magnetometer). Three modes controlled by backend config:
  - `ON` — both raw IMU readings and aggregated stats are stored and uploaded
  - `STATS_ONLY` — only aggregated stats are stored, reducing storage and upload volume
  - `OFF` — no stats processing, raw readings only (default)

  Stats configuration supports hot-reload — changes apply without restarting recording.

- **Device ID Rotation**: Automatic device ID rotation based on server-configured TTL. When the device ID expires, a new one is generated transparently on next initialization.

- **Status.Initializing**: New SDK status emitted during config fetch on startup. On recoverable errors (network failures, server 5xx), the SDK retries automatically instead of failing immediately.

### Bug Fixes

- Fixed duplicate sensor readings that could occur after system pauses or sensor restarts
- Fixed race condition when calling `deinitialize()` that could cause incomplete cleanup
- Fixed spurious service reconnects caused by lifecycle transitions
- Fixed stale sensor readings leaking across recording sessions
- Fixed in-progress uploads not being cancelled when stopping the SDK
- Fixed sensor frequency hot-reload not applying for some sensor types
- Fixed expired metadata entries not being cleaned up
- Fixed crash when stopping SDK during early initialization
- Fixed partial server config overwriting all local settings with defaults
- Fixed potential crash when Location sensor receives null readings

### Improvements

- Improved sensor lifecycle stability across recording sessions

## 1.4.6

### Bug Fixes

- Fixed workers not executing correctly in Xamarin.Android binding
- Disabled sensor watchdog provider by default (not bundled in SDK)

## 1.4.5

### Bug Fixes

- Fixed crash on startup when service binding takes longer than expected (e.g. via Xamarin)
- Fixed potential crash during `deinitialize()` when called while async operations are in progress
- Fixed `startRecording()` being silently dropped after temporary service unbind (e.g. during permission dialogs)

## 1.4.4

### Bug Fixes

- Fixed `startRecording()` not working due to internal callback ordering issue
- Fixed crash when backend sends invalid payload size limit
- Fixed recording starting before configuration was loaded
- Fixed SDK status updates being lost after service reconnect
- Fixed `getDeviceId()` returning null after `stopRecording()`
- Fixed error status being silently cleared on `deinitialize()`
- Fixed `startRecording()` being ignored when called immediately after `init()`

## 1.4.3

### Improvements

- SDK crash reports in Sentry are now automatically deobfuscated for better diagnostics

## 1.4.2

### Bug Fixes

- Fixed obfuscation issues with Statistics API classes

## 1.4.1

### Bug Fixes

- Fixed Sentry version conflict with host apps using Sentry 9.x or other incompatible versions

## 1.4.0

### New Features

- **Statistics API**: New methods for monitoring SDK health
  - `getUploadStatistics()`: Returns successful uploads count and last upload timestamp
  - `getSensorStatistics()`: Returns per-sensor statistics including configured vs actual frequency and data quality assessment
  - `getDeviceId()`: Convenience method to get the device identifier

  ```kotlin
  val sdk = TruemetricsSdk.getInstance()

  val uploadStats = sdk.getUploadStatistics()
  println("Successful uploads: ${uploadStats?.successfulUploadsCount}")
  println("Last upload: ${uploadStats?.lastSuccessfulUploadTimestamp}")

  val sensorStats = sdk.getSensorStatistics()
  sensorStats?.forEach { stat ->
      println("${stat.sensorName}: ${stat.actualFrequencyHz}Hz (configured: ${stat.configuredFrequencyHz}Hz) - ${stat.quality}")
  }

  val deviceId = sdk.getDeviceId()
  ```

- **Metadata Templates API**: New API for working with reusable metadata templates
  - Create, get, list, and remove templates
  - Tag-based metadata management: append, create from template, get by tag, log by tag

  ```kotlin
  val sdk = TruemetricsSdk.getInstance()

  sdk.createMetadataTemplate("delivery", mapOf(
      "type" to "delivery",
      "appVersion" to "1.0.0"
  ))

  sdk.createMetadataFromTemplate("current_delivery", "delivery")
  sdk.appendToMetadataTag("current_delivery", "orderId", "ORDER123")
  sdk.appendToMetadataTag("current_delivery", "address", "123 Main St")

  sdk.logMetadataByTag("current_delivery")
  ```

- **Device ID in Status**: `Initialized`, `RecordingInProgress`, and `DelayedStart` states now include `deviceId` property

- **New Status `ReadingsDatabaseFull`**: Indicates when the readings database is full due to insufficient phone storage

- **Config Hot Reload**: Configuration can now be updated on the fly without restarting the SDK

- **Payload Chunking**: Large uploads are automatically split into smaller chunks for reliability

### Removed

- `SensorWatchdogService`, `LowMemoryListener`, `DatabaseFileSizeObserver`
- Deprecated APIs:
  - `observerSensorStats()`, `observerRecordingCount()` — replaced by `getSensorStatistics()`
  - `getDatabaseSize()`, `observeDatabaseSize()`, `getStorageInfo()` — no longer needed
  - `deviceIdFlow` — replaced by `getDeviceId()` and `deviceId` property in Status

### Bug Fixes

- Fixed buffer race conditions and data loss on errors
- Fixed tight polling loop causing excessive CPU usage

### Improvements

- Improved buffer performance and reliability
- Improved test coverage

## 1.3.11

- Fix recording all metadata entries from batch (previously only first was recorded)
- Fix duplicate sensor readings when metadata ranges overlap

## 1.3.10

- Fix device ID changing after first app restart

## 1.3.9

- Improve sensor data buffering stability

## 1.3.8

- Fix dependency conflict crash: removed Ktorfit code generation library to prevent `NoSuchMethodError` when host app uses different Ktorfit version. Replaced with pure Ktor HTTP client calls.

## 1.3.7

- Fix index out of bounds exception in sensor buffer: improved thread safety with single-thread executor for buffer operations
- Fix race condition when closing sensor buffer: proper channel draining before flush
- Fix Proguard/R8 obfuscation issue: added keep rules for Ktorfit internal methods

## 1.3.6

- Fix crash when app goes to background with active recording: add missing WAKE_LOCK permission

## 1.3.5

- Add gzip compression for upload requests to reduce bandwidth usage

## 1.2.3

- Backport: Add gzip compression for upload requests to reduce bandwidth usage

## 1.3.4

- Fix thread safety for lifecycle observer registration: lifecycle operations now properly dispatch to main thread when SDK initialization is called from background threads

## 1.3.3

- Fix offline startup issue: SDK can now start recording with cached configuration when no internet connection is available

## 1.3.1 / 1.3.2

- Fix crash when `deinitialize()` is called from non-main thread

## 1.3.0

This section covers all major features and improvements that are present in the `develop`:

---

### 🔧 **API & Architecture Changes**

- **Feature: Rework Front-facing API (#123)** - Major API redesign with improved public interfaces,
  removed loggers and status listeners, enhanced KDoc documentation, and added class diagrams and
  usage guides

---

### 🏗️ **Database Architecture Overhaul**

- **Migration from Raw SQLite to Room Database** - Complete migration from raw SQLite implementation
  to Android Room persistence library 
  - **Multiple Database Architecture**: Separate databases for readings, stats, and metadata

---

### ⚡ **Buffering System Implementation**

- **Feature: Advanced Buffering Mechanism (#107)** - Implemented sophisticated multi-buffer system
  between sensors and database that **achieved desired frequency targets**:
  - **Dynamic Buffer Sizing**: `DynamicBufferSizer` automatically adjusts buffer size and count
    based on device performance (and available RAM)
  - **Asynchronous Processing**: Coroutine-based buffering with separate channels for individual
    readings and batch operations
  - **Worker-Based Database Writes**: `SensorInsertWorker` handles database operations in background
  - **Automatic Buffer Restructuring**: Real-time buffer optimization based on system performance
  - **Fast-Track Metadata**: Priority handling for critical `CustomerMetaDataReading` entries

---

### 🚀 **CI/CD Pipeline Restructuring for Parallel Development**

- **Pipeline Architecture Overhaul** - Completely restructured CI/CD pipeline to accommodate *
  *multiple parallel development efforts**:
  - **Parallel Job Architecture**: Separated `lint_check`, `unit_tests`, and `api_check` jobs to run
    simultaneously
  - **Conditional Branch Processing**: Smart branch detection for `main`, `develop`, `development`,
    `rc/**`, `hotfix/**`, `release/**` branches
  - **Separate Artifact Caching**: Independent build artifact caching with SHA-based cache keys for
    optimal parallel builds
  - **Multi-Environment Support**: Dynamic environment selection (`development`/`production`) based
    on branch type
  - **Branch-Specific Deployments**: Targeted APK uploads only on specific branches to optimize
    build times
  - **Isolated Build Stages**: Separate `build_app`, `build_sdk`, and `publish_sdk` jobs for
    parallel execution
  - **Maven Repository Automation**: Automated PR creation with separate jobs for snapshots,
    releases, and hotfixes
  - **Build Failure Safeguards**: Fail builds when no changes detected for maven branches to prevent
    unnecessary operations

---

### ⚙️ **Comprehensive Feature Flags System**

- **Build-Time Feature Flag Architecture** - Implemented sophisticated feature flag system with
  environment-specific configurations:

  #### 🔧 Feature Flag Infrastructure:
  - **Environment-Specific Configuration**: Separate `flags.development.properties` and
    `flags.production.properties` files
  - **Build Integration**: Gradle `prepareFeatureFlagFields()` function dynamically loads flags
    based on `truemetricsBuildEnv` property
  - **BuildConfig Integration**: Feature flags compiled into `BuildConfig` constants for zero
    runtime overhead
  - **Validation System**: Build fails if required feature flags are missing from configuration
    files

  #### 📋 Available Feature Flags:

  | Feature Flag | Development | Production | Purpose |
    |--------------|-------------|------------|---------|
  | `truemetrics.sensor.stats.enabled` | ✅ `true` | ✅ `true` | Controls sensor statistics collection and upload functionality |
  | `truemetrics.sensor.traffic.counter.enabled` | ✅ `true` | ✅ `true` | Enables network traffic monitoring and bandwidth usage tracking |
  | `truemetrics.sensor.observing.connectivity.enabled` | ❌ `false` | ❌ `false` | Controls WiFi and connectivity sensor data collection |
  | `truemetrics.sensor.watchdog.enabled` | ✅ `true` | ❌ `false` | Enables/disables sensor watchdog service for monitoring sensor health |
  | `truemetrics.sensor.show.notification.when.upload.fails.enabled` | ✅ `true` | ❌ `false` | Shows user notifications when data upload operations fail |
  | `truemetrics.sensor.show.recording.debug.notifications.enabled` | ✅ `true` | ❌ `false` | Displays debug notifications showing real-time sensor recording status |

  #### 🎯 Feature Flag Usage Examples:
  - **Stats Collection**: `BuildConfig.STATS_ENABLED` used in `DefaultUploadExecutor` to
    conditionally enable statistics gathering
  - **Traffic Monitoring**: `BuildConfig.TRAFFIC_COUNTER_ENABLED` integrated into `KtorConfig` for
    network usage tracking
  - **Connectivity Sensors**: `BuildConfig.OBSERVING_CONNECTIVITY_ENABLED` controls availability of
    WiFi and connectivity sensors
  - **Watchdog Service**: `BuildConfig.SENSOR_WATCHDOG_ENABLED` manages `SensorWatchdogService` and
    `LowMemoryListener` activation
  - **Debug Notifications**: `BuildConfig.SHOW_RECORDING_DEBUG_NOTIFICATIONS` controls
    `DebugNotificationHelper` behavior
  - **Upload Failure Alerts**: `BuildConfig.SHOW_NOTIFICATION_WHEN_UPLOAD_FAILED` manages
    `UploadWorker` notification display
---

### 🚀 **Performance & Storage**

- **Feature: Payload Size Limit (#118)** - Made payload size limits configurable to optimize network
  usage and upload performance. Breaking the request into two or more smaller if above payload size limit
- **Feature: Improve DB Insert and Connectivity (#115)** - Enhanced database insertion logic and
  connectivity handling for better reliability
- **Feature: Start on Boot (#114)** - Added automatic service startup when device boots 
- **Feature: Delayed Start Recording** - Implemented delayed recording initialization to optimize
  startup performance and resource allocation

---

### 🔧 **Bug Fixes & Improvements**

- **Feature: Show Notification When Upload Fails (#109)** - Added user-friendly notifications for
  upload failure scenarios
- **Feature: Remove Mark for Upload (#110)** - Simplified upload marking mechanism
- **Feature: Fix Config Rendering (#108)** - Fixed configuration rendering issues
- **Feature: Always Deploy Release (Obfuscated) Builds to Firebase (#119)** - Improved CI/CD pipeline for
  consistent release deployment

---

### 🗑️ **Removed Features**

- **Removed WiFi Connectivity Recording** - Eliminated WiFi connectivity sensor data collection to
  reduce resource usage and improve performance
- **Removed WIFI_ONLY Uploading Feature** - Simplified upload mechanism by removing WiFi-only upload
  restriction, allowing more flexible network usage

---

### 🏗️ **Development & Infrastructure**

- **Feature: Signed Release 1.3.0 (#116)** - Added proper APK signing for production releases
- **SDK Obfuscation** - Added code obfuscation with ProGuard rules for production builds
- **Sentry Mapping Upload** - Automated ProGuard mapping file uploads to Sentry for better crash
  reporting

---

### 📦 **Version Increments**

- **Version bumped to 16** (from the baseline after pipeline-update #105)
- Multiple incremental version updates throughout development cycle

---

### 🗂️ **Configuration Management**

- **Enhanced Config System** - Improved configuration management with better validation and error
  handling
- **Environment-Aware Builds** - Dynamic configuration loading based on build environment (
  development vs production)

---

## 1.0.23

- Fix reporting when traffic limit is reached limit
- Add `use_work_manager` option in config to control WorkManager
- Add ignore list to exclude some exceptions from reporting on Sentry
- Fix nullable casting transportInfo to WiFiInfo
- Add Sentry logging (info) for WorkManager jobs
- Exclude clearing traffic stats on deinit

## 1.0.22

- Fix for writing stats to DB on background thread

## 1.0.21

- Fix upload scope creation after SDK deinitialization
- Update Sentry SDK and Sentry scope metadata

## 1.0.20

- Support for metadata triggered uploads
- Simplified SDK initialization (foreground notification is now optional)

## 1.0.19

- Info about non-working sensors
- Support for HTTP traffic limit
- Support for DB Encryption

## 1.0.18

- Add gzip compression for data upload requests

## 1.0.17

- Revert reading location sensors (don't do up-sampling)

## 1.0.14

- Fix lateinit issues for GNSS and raw location sensors

## 1.0.13

- Fix reading gnsslocation at desired frequency from configuration

## 1.0.12

- Fix data loss bug

## 1.0.11

- Fix writing cached readings into DB
- Fix sending default values for readings if values are not received from sensor
- Delete readings from DB on init if API key changes, not on de-init

## 1.0.10

- Fix uninitialized property crash
- Fix concurrent modification crash
- Reduce amount of telemetry logging

## 1.0.9

- Add more telemetry logging
- Use custom scopes for data uploading

## 1.0.8

- Add more telemetry logging for debugging purposes

## 1.0.7

- Fix file logging crash

## 1.0.6

- Refactor config fetching and uploading files

## 1.0.5

- Fix uploads stalling sometimes when switching networks
- Integrate Sentry reporting

## 1.0.4

- Fix initializing callbacks on lower API levels
- Add raw location fields (GNSS-based, not fused location)

## 1.0.3

- Fix Foreground service init crash
- Update Koin to 3.5.0

## 1.0.2

- Asking for permissions based on configuration
- Battery level, status and health reporting
- Connectivity events reporting, e.g. network connected/disconnected
- WiFi signal strength reporting
- Mobile data signal strength reporting
- Step count reporting
- Motion mode reporting
- Remotely controlled periodic configuration fetching

## 1.0.1

- Fixed DI framework context isolation

## 1.0.0

- 1.0.0 SDK release with support for:
  - fused location reporting, GNSS, accelerometer, gyroscope, barometer, magnetometer sensors
  - customer metadata reporting