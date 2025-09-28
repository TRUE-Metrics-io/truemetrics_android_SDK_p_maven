# TrueMetrics Android SDK

[![Build Status](https://github.com/TRUE-Metrics-io/truemetrics_android_SDK_p/actions/workflows/build.yml/badge.svg?branch=main)](https://github.com/TRUE-Metrics-io/truemetrics_android_SDK_p/actions/workflows/build.yml)
[![Maven Central](https://img.shields.io/maven-central/v/io.truemetrics/truemetricssdk.svg?label=Maven%20Central)](https://search.maven.org/search?q=g:io.truemetrics%20AND%20a:truemetricssdk)
[![API](https://img.shields.io/badge/API-21%2B-brightgreen.svg?style=flat)](https://android-arsenal.com/api?level=21)
[![Kotlin](https://img.shields.io/badge/kotlin-2.2.20-blue.svg?logo=kotlin)](http://kotlinlang.org)


## Version Information

- **Production Version**: `1.3.0`
- **Snapshot Version**: `1.3.1-SNAPSHOT`


## Installation

### Gradle (Kotlin DSL)

Add the following to your app-level `build.gradle.kts` file:

```kotlin
repositories {
    maven {
        url = uri("https://github.com/TRUE-Metrics-io/truemetrics_android_SDK_p_maven/raw/")
    }
}


dependencies {
    implementation("io.truemetrics:truemetricssdk:1.3.0")
}
```

### Gradle (Groovy)

Add the following to your app-level `build.gradle` file:

```groovy
dependencies {
    implementation 'io.truemetrics:truemetricssdk:1.3.0'
}
```

### Using Snapshot Versions

To use the latest snapshot version, add the snapshot repository to your project-level
`build.gradle.kts`:

```kotlin
repositories {
    maven {
        url = uri("https://github.com/TRUE-Metrics-io/truemetrics_android_SDK_p_maven/raw/snapshots")
    }
}

dependencies {
    implementation("io.truemetrics:truemetricssdk:1.3.1-SNAPSHOT")
}
```

### Maven

Add the following to your `pom.xml`:

```xml
<dependency>
    <groupId>io.truemetrics</groupId>
    <artifactId>truemetricssdk</artifactId>
    <version>1.3.0</version>
    <type>aar</type>
</dependency>
```