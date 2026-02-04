# Changelog

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