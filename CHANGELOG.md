# Changelog

## 1.5.0

### New Features

- **Sensor Statistics**: New real-time statistical aggregation pipeline for IMU sensors (accelerometer, gyroscope, magnetometer). Computes mean, std, percentiles (p20/p50/p80), and covariance matrix eigenvalues over configurable time windows aligned to UTC clock boundaries. Three modes controlled by backend config:
  - `ON` — both raw IMU readings and aggregated stats are stored and uploaded
  - `STATS_ONLY` — only aggregated stats are stored; raw IMU readings are filtered out, reducing DB and upload volume
  - `OFF` — no stats processing, raw readings only (default)

  Stats configuration (`mode_stats`, `duration_stats`) supports hot-reload — changes apply without restarting recording.

### Bug Fixes

- **Fix duplicate sensor readings from polling catch-up ticks**: When `fixedRateTimer` fired multiple catch-up ticks in quick succession (e.g. after GC pause), Battery and Location sensors saved the same reading on every tick. Battery now uses reference-based dedup (`===`) and Location uses timestamp-based dedup to skip already-polled readings.
- **Fix race condition on deinitialize**: `service.deinitialize()` was called after `unbindService()`, which could cause the service to be destroyed before cleanup completed. Now calls `deinitialize()` before unbinding.
- **Fix spurious reconnects from lifecycle bind/unbind cycle**: Removed `onResume`/`onPause` service unbind/rebind in `TruemetricsLifecycleObserver`. The foreground service stays alive via `startForegroundService()` — the previous unbind/rebind cycle caused race conditions requiring `pendingStartRecording`, reconnect logic, and `pendingConfig` guards.
- **Fix stale dedup state after sensor restart**: Battery and Location now reset their dedup state (`latestReading`, `lastPolledReading`/`lastPolledLocationTime`) in `start()`, preventing the first post-restart reading from being suppressed on singleton sensor instances.
- **Fix UploadWorker not cancelling in-progress uploads on WorkManager stop**: Added `onStopped()` override that cancels the active `runBlocking` coroutine job, preventing stalled or partial uploads when WorkManager cancels the worker. `uploadJob` is marked `@Volatile` for cross-thread visibility.
- **Fix `distinctUntilChanged` predicate inversion for frequency config**: The comparison used `!=` instead of `==`, causing every config emission to trigger sensor re-preparation even when frequencies hadn't changed.
- **Fix incomplete `allFrequencies` map**: Added all missing sensor frequencies (wifi, battery, step counter, motion, mobile data signal, fused orientation) to the frequency change observation map, and removed duplicate IMU entries that were present in main. Hot-reload of frequency changes from backend now works for all sensors.
- **Fix expired metadata not being cleaned up**: Restored `deleteExpiredMetadata()` call after metadata insert, lost during 1.4.x refactor. Without this, metadata entries accumulated indefinitely.
- **Fix HardwareSensor dedup using unreliable reference comparison**: Replaced `SensorEvent.values`/`timestamp` reference comparison with timestamp-based dedup. Clears dedup state on `stop()` to prevent stale replays.
- **Fix DynamicQueryBatchSizer crash on early teardown**: Added guard for uninitialized `memoryCheckHandler` when `stop()` is called before `start()`.
- **Fix UploadWorker reporting network errors to Sentry**: `IOException` now returns `Result.failure()` silently instead of falling through to the generic Sentry-reporting handler.
- **Fix Sentry scope not applied to captured events**: `captureException` and `captureNonFatal` now execute inside `withScope` block, ensuring tags and context are actually attached to the event.

### Improvements

- **Singleton sensor instances**: All sensor Koin registrations changed from `factory` to `single`, stabilizing lifecycle management and preventing redundant sensor re-creation across recording sessions.
- **Disable Sentry in integration tests**: `SENTRY_ENABLED` build config flag provides a no-op `IssueReporter` in the `integrationTest` flavor, preventing test runs from polluting the production Sentry dashboard.
- **CI: Retry and concurrency for device tests**: Integration tests and device farm tests now use `nick-fields/retry@v3` with automatic retries. Docker compose startup retries with USB device re-enumeration. Concurrency groups cancel in-progress runs on the same branch. Simplified device farm runner management.

### Tests

- Added unit tests for sensor statistics pipeline: `SensorStatisticsBufferTest`, `SensorStatisticsProcessorTest`, `StatisticsCalculatorTest`, `StatisticsSensorStoreTest`
- Added unit tests for config parsing: `ConfigResponseExtStatsTest`
- Added unit tests for metadata cleanup and repository: `MetadataCleanupTest`, `MetadataSensorReadingsRepositoryTest`
- Added unit tests for polling dedup: `StartPollingFlowDeduplicationTest`
- Added integration tests: `StatsSensorIntegrationTest` (stats modes ON/STATS_ONLY/OFF), `MetadataDaoTest`, `SensorReadingDaoTest`
- Removed obsolete `DefaultConfigRepositoryTest`

## 1.4.6

### Bug Fixes

- **Fix CoroutineWorker not executing in Xamarin.Android binding**: `SensorInsertWorker` and `UploadWorker` now extend `Worker` instead of `CoroutineWorker`. The coroutine dispatch mechanism fails in the Xamarin Mono VM runtime, causing `doWork()` to never execute and workers to be immediately cancelled in a loop. Using `Worker` with `runBlocking` bypasses the issue while keeping all suspend logic intact.
- **Disable sensor watchdog provider by default**: `SENSOR_WATCHDOG_ENABLED` is now `false` in all build configurations. The watchdog module is not bundled in the SDK AAR, so `contentResolver.insert()` calls to `sensorwatchdog.provider` were producing ~25 errors/sec in logcat.

## 1.4.5

### Bug Fixes

- **Fix ForegroundServiceDidNotStartInTimeException crash**: `startForeground()` was only called after async service bind completed (`onServiceConnected` → `initializeEngine` → `displayForegroundNotification`). If bind took longer than Android's ~5 second deadline (e.g. via Xamarin JNI overhead), the OS killed the process. Now calls `startForeground()` immediately in `onCreate()` with a default notification, then updates it with the user-provided notification once initialization completes.
- **Fix NullPointerException race condition on deinitialize()**: `mainScope` was never cancelled during `deinitialize()`, so async coroutines could access Koin after it was closed, causing NPE in `IsolatedKoinComponent.getKoin()`
- **Fix startRecording() silently failing after lifecycle unbind**: If the service was temporarily unbound (e.g. during permission dialogs), `startRecording()` was silently dropped. Now always queues the request and processes it when the service reconnects.

## 1.4.4

### Bug Fixes

- **Fix swapped callbacks in startRecording**: `onEngineInitialized()` had `action` and `onError` lambdas passed in wrong order, causing recording to never start when engine was still initializing
- **Fix payloadLimitKb crash**: `require(payloadLimitKb in 1..1024)` threw an exception when backend sent `null` or out-of-range values. Now defaults to 700 KB gracefully
- **Fix recording starting without config**: `startRecording()` set status to `RecordingInProgress` before checking if configuration was loaded. Now checks config first and emits `Error` status if missing
- **Fix status flow lost on service reconnect**: `dispatchSdkEvents()` was only called when `pendingConfig != null`. On service reconnect (e.g., after backgrounding), status flow subscriptions were never re-established
- **Fix getDeviceId() returning null after stopRecording**: Now reads device ID directly from the engine instead of extracting it from the current status object
- **Fix stale status on deinit after error**: `deinitialize()` now preserves `Error` status instead of resetting to `Uninitialized`, preventing the error from being silently swallowed
- **Fix startRecording() ignored before service bind**: Calling `startRecording()` immediately after `init()` was silently dropped. Now queues the request and starts recording once the service is connected and config is loaded

### Improvements

- **API compatibility check robustness**: `apiCheck` now compares unobfuscated (debug) builds to avoid false positives from R8 obfuscation reshuffling internal field names
- **Integration tests**: Added 9 integration tests covering all startup/recording bug fixes, cached config fallback, and deinit cache clearing

## 1.4.3

### Improvements

- **Sentry crash deobfuscation**: SDK crash reports in Sentry are now automatically deobfuscated. ProGuard mapping files are uploaded to Sentry during CI build, enabling readable stack traces for SDK-related crashes.

## 1.4.2

### Bug Fixes

- **Fix obfuscation issues with Statistics API**: Added ProGuard keep rules for `UploadStatistics`, `SensorStatistics`, `SensorDataQuality`, and `TrafficStatus` classes. 

## 1.4.1

### Bug Fixes

- **Fix Sentry version conflict**: Isolated SDK's Sentry dependency using package relocation (`io.sentry.*` → `io.truemetrics.internal.sentry.*`). This prevents `ClassNotFoundException: io.sentry.Hub` crashes when host app uses Sentry 9.x or other incompatible versions.

## 1.4.0

### New Features

- **Statistics API**: New methods for monitoring SDK health
  - `getUploadStatistics()`: Returns successful uploads count and last upload timestamp
  - `getSensorStatistics()`: Returns per-sensor statistics including configured vs actual frequency and data quality assessment
  - `getDeviceId()`: Convenience method to get the device identifier

  ```kotlin
  val sdk = TruemetricsSdk.getInstance()

  // Get upload statistics
  val uploadStats = sdk.getUploadStatistics()
  println("Successful uploads: ${uploadStats?.successfulUploadsCount}")
  println("Last upload: ${uploadStats?.lastSuccessfulUploadTimestamp}")

  // Get sensor statistics
  val sensorStats = sdk.getSensorStatistics()
  sensorStats?.forEach { stat ->
      println("${stat.sensorName}: ${stat.actualFrequencyHz}Hz (configured: ${stat.configuredFrequencyHz}Hz) - ${stat.quality}")
  }

  // Get device ID
  val deviceId = sdk.getDeviceId()
  ```

- **Metadata Templates API**: New API for working with reusable metadata templates
  - Create, get, list, and remove templates
  - Tag-based metadata management: append, create from template, get by tag, log by tag
  - Simplifies repetitive metadata logging patterns

  ```kotlin
  val sdk = TruemetricsSdk.getInstance()

  // Create a reusable template
  sdk.createMetadataTemplate("delivery", mapOf(
      "type" to "delivery",
      "appVersion" to "1.0.0"
  ))

  // Create tagged metadata from template and append data
  sdk.createMetadataFromTemplate("current_delivery", "delivery")
  sdk.appendToMetadataTag("current_delivery", "orderId", "ORDER123")
  sdk.appendToMetadataTag("current_delivery", "address", "123 Main St")

  // Log all accumulated metadata at once
  sdk.logMetadataByTag("current_delivery")
  ```

- **Device ID in Status**: `Initialized`, `RecordingInProgress`, and `DelayedStart` states now include `deviceId` property for easier access

- **New Status `ReadingsDatabaseFull`**: Indicates when the readings database is full due to insufficient phone storage

- **Config Hot Reload**: Configuration can now be updated on the fly without restarting the SDK

- **Payload Chunking**: Large uploads are automatically split into smaller chunks based on configured payload size limit

### Removed

- **SensorWatchdogService**: Removed from SDK 
- **LowMemoryListener**: Removed from SDK
- **DatabaseFileSizeObserver**: Removed from SDK
- **Deprecated APIs removed**:
  - `observerSensorStats()`, `observerRecordingCount()` - replaced by `getSensorStatistics()`
  - `getDatabaseSize()`, `observeDatabaseSize()`, `getStorageInfo()` - no longer needed
  - `deviceIdFlow` - replaced by `getDeviceId()` method and `deviceId` property in Status

### Bug Fixes

- Fix buffer race conditions and data loss on errors
- Fix tight polling loop causing excessive CPU usage

### Improvements

- Simplified buffer architecture: single buffer instead of complex multi-buffer system
- Build migrated to Kotlin DSL (build.gradle.kts)
- Improved test coverage with comprehensive unit and integration tests
- Better error handling throughout the SDK

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