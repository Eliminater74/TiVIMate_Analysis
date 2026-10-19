# 📦 TiViMate v5.3.3 — Complete Module & Version Catalog

> **Extracted from:** Decompiled APK source tree (JADX output)
> **APK Version:** `5.3.3` (versionCode `1000005332`)
> **Package:** `ar.tvplayer.tv`
> **Compile SDK:** `37` (Android 17 / SDK codename `17`)
> **Min SDK:** `23` (Android 6.0 Marshmallow)
> **Target SDK:** `34` (Android 14)
> **Git Revision:** `5f51170e55b4563532816e2823156934c2429204`

---

## 📊 Codebase Statistics

| Metric | v5.2.0 | v5.3.3 | Delta |
| :--- | :---: | :---: | :---: |
| ☕ Decompiled Java Source Files | 6,416 | **6,548** | +132 |
| 📦 Source Packages (directories) | 783 | **303** | restructured |
| 🎨 Resource Files | 1,587 | **1,177** | -410 |
| 📁 Resource Directories | 143 | **127** | -16 |
| 📝 Analysis Documents | 52 | **54** | +2 |
| 📐 Total Analysis Size | ~594 KB | **~638 KB** | +44 KB |
| 🏗️ Native ABI Targets | 4 | **4** | — |

---

## 🔵 Core Application

| Module | Package | Notes |
| :--- | :--- | :--- |
| **TiViMate IPTV Player** | `ar.tvplayer.tv` | Main app — `ProtectedTvPlayerApplication` entry point |
| Application Class | `ar.tvplayer.tv.ProtectedTvPlayerApplication` | DexProtector-wrapped |
| Core Library | `ar.tvplayer.core` | Shared utilities, data repos, domain logic |
| UI Layer | `ar.tvplayer.tv.ui` | `MainActivity` (landscape, PiP-enabled) |
| Settings | `ar.tvplayer.tv.settings` | Playlists, TV Guide, Appearance, Backup/Restore |
| Reminders | `ar.tvplayer.tv.reminders` | `ReminderPopupActivity` with separate task affinity |
| Unlock/Premium | `ar.tvplayer.tv.unlock` | In-app purchase flow via `UnlockPremiumActivity` |
| TV Guide | `ar.tvplayer.tv.tvguide` | EPG channel groups, copy channels |
| Base UI | `ar.tvplayer.tv.base` | File picker, SMB browser, WebView, Update, Boot receiver |
| Phoenix Process | `ar.tvplayer.core.util.RestartProcessActivity` | Runs in `:ar.tvplayer.phoenix` separate process |

---

## 🟢 Google Play & Billing

| Library | Version | Source |
| :--- | :--- | :--- |
| **Google Play Billing** | **9.1.0** | `billing.properties` / `AndroidManifest.xml` meta-data |
| Play Services Base | 18.5.0 | `play-services-base.properties` |
| Play Services Basement | 18.9.0 | `play-services-basement.properties` |
| Play Services Tasks | 18.4.0 | `play-services-tasks.properties` |
| Play Services Stats | 17.0.2 | `play-services-stats.properties` |
| Play Services Location | 19.0.0 | `play-services-location.properties` |
| Play Services Ads Identifier | 18.0.0 | `play-services-ads-identifier.properties` |
| Play Services Places (PlaceReport) | 17.0.0 | `play-services-places-placereport.properties` |

---

## 🔥 Firebase

| Library | Version | Source |
| :--- | :--- | :--- |
| **Firebase Analytics** | **23.2.0** | `firebase-analytics.properties` |
| Firebase Measurement Connector | 20.0.1 | `firebase-measurement-connector.properties` |
| Firebase Encoders | 17.0.0 | `firebase-encoders.properties` |
| Firebase Encoders Proto | 16.0.0 | `firebase-encoders-proto.properties` |
| Firebase Crashlytics | — | Registered via `CrashlyticsRegistrar` |
| Firebase Crashlytics KTX | — | Registered via `FirebaseCrashlyticsKtxRegistrar` |
| Firebase Remote Config | — | Registered via `RemoteConfigRegistrar` |
| Firebase Remote Config KTX | — | Registered via `FirebaseRemoteConfigKtxRegistrar` |
| Firebase Sessions | — | Registered via `FirebaseSessionsRegistrar` |
| Firebase Installations | — | Registered via `FirebaseInstallationsRegistrar` |
| Firebase ABT | — | Registered via `AbtRegistrar` |
| Firebase Data Transport | — | Registered via `TransportRegistrar` |
| Firebase Common KTX | — | Registered via `FirebaseCommonKtxRegistrar` |
| Play Services Measurement (all) | 23.2.0 | `play-services-measurement*.properties` |

---

## 🟣 AndroidX Libraries

### Core & Foundation

| Library | Version |
| :--- | :--- |
| **androidx.core** | **1.19.0** |
| **androidx.core-ktx** | **1.19.0** |
| androidx.core-animation | 1.0.0 |
| androidx.core-viewtree | 1.0.0 |
| androidx.activity | 1.8.1 |
| androidx.activity-ktx | 1.8.1 |
| androidx.annotation-experimental | 1.5.0 |
| androidx.startup-runtime | 1.2.0 |
| androidx.profileinstaller | 1.4.1 |

### UI Components

| Library | Version |
| :--- | :--- |
| **androidx.appcompat** | **1.8.0-alpha01** |
| **androidx.appcompat-resources** | **1.8.0-alpha01** |
| **androidx.constraintlayout** | **2.2.1** |
| **com.google.android.material** | **1.14.0** |
| androidx.recyclerview | 1.4.0 |
| androidx.viewpager2 | 1.1.0-beta02 |
| androidx.viewpager | 1.0.0 |
| androidx.cardview | 1.0.0 |
| androidx.coordinatorlayout | 1.1.0 |
| androidx.customview | 1.2.0 |
| androidx.customview-poolingcontainer | 1.0.0 |
| androidx.drawerlayout | 1.1.1 |
| androidx.slidingpanelayout | 1.2.0 |
| androidx.dynamicanimation | 1.1.0 |
| androidx.graphics-shapes | 1.0.1 |
| androidx.interpolator | 1.0.0 |
| androidx.vectordrawable | 1.1.0 |
| androidx.vectordrawable-animated | 1.1.0 |
| androidx.exifinterface | 1.3.6 |
| androidx.asynclayoutinflater | 1.1.0 |
| androidx.cursoradapter | 1.0.0 |

### Leanback (Android TV)

| Library | Version |
| :--- | :--- |
| **androidx.leanback** | **1.3.0-alpha02** |
| **androidx.leanback-grid** | **1.1.0-alpha02** |
| androidx.leanback-preference | 1.2.0 |

### Lifecycle & Architecture

| Library | Version |
| :--- | :--- |
| **androidx.lifecycle** (all modules) | **2.11.0** |
| lifecycle-livedata | 2.11.0 |
| lifecycle-livedata-core | 2.11.0 |
| lifecycle-livedata-core-ktx | 2.11.0 |
| lifecycle-process | 2.11.0 |
| lifecycle-runtime | 2.11.0 |
| lifecycle-runtime-ktx | 2.11.0 |
| lifecycle-service | 2.11.0 |
| lifecycle-viewmodel | 2.11.0 |
| lifecycle-viewmodel-ktx | 2.11.0 |
| lifecycle-viewmodel-savedstate | 2.11.0 |
| androidx.savedstate | 1.4.0 |
| androidx.savedstate-ktx | 1.4.0 |

### Fragment & Navigation

| Library | Version |
| :--- | :--- |
| **androidx.fragment** | **1.9.0-alpha01** |
| **androidx.fragment-ktx** | **1.9.0-alpha01** |

### Data Persistence

| Library | Version |
| :--- | :--- |
| **androidx.room (v2)** | **2.7.0** |
| **androidx.room3** | **3.0.0-rc01** |
| **androidx.sqlite** | **2.7.0-rc01** |
| **androidx.sqlite-async** | **2.7.0-rc01** |
| **androidx.sqlite-bundled** | **2.7.0-rc01** |
| **androidx.sqlite-framework** | **2.7.0-rc01** |
| **androidx.datastore** | **1.1.7** |
| androidx.datastore-core | 1.1.7 |
| androidx.datastore-preferences | 1.1.7 |
| androidx.datastore-preferences-core | 1.1.7 |

### Background Processing

| Library | Version |
| :--- | :--- |
| **androidx.work-runtime** | **2.11.2** |

### Other AndroidX

| Library | Version |
| :--- | :--- |
| androidx.databinding-viewbinding | 9.2.1 |
| androidx.emoji2 | 1.3.0 |
| androidx.emoji2-views-helper | 1.3.0 |
| androidx.loader | 1.0.0 |
| androidx.localbroadcastmanager | 1.0.0 |
| androidx.media | 1.0.0 |
| androidx.preference | 1.2.1 |
| androidx.preference-ktx | 1.2.1 |
| androidx.print | 1.0.0 |
| androidx.privacysandbox.ads-adservices | 1.1.0-beta11 |
| androidx.privacysandbox.ads-adservices-java | 1.1.0-beta11 |
| androidx.tracing | 1.2.0 |
| androidx.tracing-ktx | 1.2.0 |
| androidx.transition | 1.7.0 |
| androidx.transition-ktx | 1.7.0 |
| androidx.versionedparcelable | 1.1.1 |
| androidx.window | 1.0.0 |
| androidx.documentfile | 1.1.0 |
| androidx.legacy-support-core-utils | 1.0.0 |

---

## 🎵 Media Playback (ExoPlayer / Media3)

| Library | Package | Notes |
| :--- | :--- | :--- |
| **androidx.media3.common** | `androidx.media3.common` | Core media types, formats, exceptions |
| **androidx.media3.exoplayer** | `androidx.media3.exoplayer` | Player engine, LoadControl, renderers |
| **androidx.media3.datasource** | `androidx.media3.datasource` | HTTP, File, Content, UDP data sources |
| **androidx.media3.decoder** | `androidx.media3.decoder` | Video/audio decoder interfaces |
| **androidx.media3.extractor** | `androidx.media3.extractor` | TS, FMP4, MP4, MKV, HLS extractors |
| **androidx.media3.database** | `androidx.media3.database` | Media cache database |
| **androidx.media3.ui** | `androidx.media3.ui` | Player view, subtitle view |

> **Note:** Media3 is the successor to ExoPlayer. The exact version is embedded in the decompiled source; the module structure maps to the `1.x` Media3 line (estimated **1.5.x** based on APIs present and compileSdk=37).

---

## 🌐 Networking

| Library | Package/Module | Notes |
| :--- | :--- | :--- |
| **OkHttp 3** | `okhttp3` | HTTP client with connection pooling; `PlatformInitializer` via AndroidX Startup |
| **Retrofit 2** | `retrofit2` | Type-safe HTTP client wrapper (only `HttpException` in decompiled tree — rest ProGuarded) |
| **Google Gson** | `com.google.gson` | JSON serialization/deserialization |
| **Square Moshi** | `com.squareup.moshi` | Additional JSON adapter (modern alternative to Gson) |

---

## 🖼️ Image Loading

| Library | Package | Notes |
| :--- | :--- | :--- |
| **Glide** | `com.bumptech.glide` | Image loading + 250MB disk cache |
| **Glide OkHttp3 Integration** | `com.bumptech.glide.integration.okhttp3` | Uses OkHttp as Glide's HTTP transport (registered in Manifest) |

---

## 🔐 SMB / Network Shares

| Library | Package | Notes |
| :--- | :--- | :--- |
| **SMBJ** | `com.hierynomus.smbj` | SMBv2/v3 client for network recording storage |
| SMBJ ASN.1 | `com.hierynomus.asn1` | ASN.1 parsing |
| SMBJ NTLM | `com.hierynomus.ntlm` | NTLM authentication |
| SMBJ SPNEGO | `com.hierynomus.spnego` | SPNEGO/Kerberos negotiation |
| SMBJ MS-DFSC | `com.hierynomus.msdfsc` | DFS referral client |
| SMBJ MS-SMB | `com.hierynomus.mssmb` | SMBv1 messages |
| SMBJ MS-SMB2 | `com.hierynomus.mssmb2` | SMBv2 messages |
| **Rapid7 client** | `com.rapid7.client` | SSH/helper client |

---

## 🔒 Cryptography & Security

| Library | Package | Notes |
| :--- | :--- | :--- |
| **Bouncy Castle** | `org.bouncycastle` | Cryptographic provider |
| **Google Tink / Crypto** | `com.google.crypto` | Modern crypto APIs |
| **DexProtector** | `libdexprotector.so` | APK protection/anti-tampering (native) |

---

## 📦 Kotlin Ecosystem

| Library | Version |
| :--- | :--- |
| **Kotlin Stdlib** | Present in `kotlin/`, `j$/` (desugared) |
| **Kotlinx Coroutines Core** | **1.11.0** |
| **Kotlinx Coroutines Android** | **1.11.0** |
| **Kotlinx Coroutines Play Services** | **1.11.0** |
| **Kotlinx Serialization** | Present in `kotlinx.serialization` |

---

## 🗜️ Compression & Utilities

| Library | Package | Notes |
| :--- | :--- | :--- |
| **Zip4j** | `net.lingala.zip4j` | Password-protected ZIP for encrypted backups |
| **XZ for Java** | `org.tukaani.xz` | LZMA/XZ compression |
| **Apache Commons** | `org.apache.commons` | General-purpose utility classes |
| **SLF4J** | `org.slf4j` | Logging facade (used by SMBJ) |

---

## 📨 Event Bus

| Library | Package | Notes |
| :--- | :--- | :--- |
| **MBassador** | `net.engio.mbassy` | High-performance event bus for loosely-coupled component messaging |

---

## 🔗 Google Data Transport

| Library | Package | Notes |
| :--- | :--- | :--- |
| Google Data Transport Runtime | `com.google.android.datatransport.runtime` | Background data upload scheduling |
| Google Data Transport CCT | `com.google.android.datatransport.cct` | Cloud transport backend |

---

## 📱 Native Libraries (per ABI)

Supported ABIs: **arm64-v8a**, **armeabi-v7a**, **x86**, **x86_64**

| Native Library | Size (arm64) | Purpose |
| :--- | :---: | :--- |
| `libffmpegJNI.so` | 1,385 KB | FFmpeg JNI bridge — software video/audio decoding |
| `libsqliteJni.so` | 1,040 KB | Bundled SQLite native (used by Room3/sqlite-bundled) |
| `libdexprotector.so` | 433 KB | DexProtector anti-tampering/obfuscation runtime |
| `libavutil.so` | 397 KB | FFmpeg utility library |
| `libdatastore_shared_counter.so` | 7 KB | DataStore shared counter (multi-process support) |

---

## 🔑 Key Version Changes: v5.2.0 → v5.3.3

| Area | v5.2.0 | v5.3.3 | Change |
| :--- | :--- | :--- | :--- |
| **App Version** | 5.2.0 (build 5208) | **5.3.3** (build 1000005332) | Major update |
| **Compile SDK** | 34 | **37** | +3 (Android 17) |
| **Play Billing** | 7.x | **9.1.0** | Major upgrade |
| **AndroidX Core** | 1.12.x | **1.19.0** | Major upgrade |
| **Lifecycle** | 2.7.x | **2.11.0** | Major upgrade |
| **Room** | 2.6.x | **2.7.0 + Room3 3.0.0-rc01** | ⚡ Dual Room! |
| **SQLite** | Framework | **2.7.0-rc01 (bundled)** | Now uses bundled SQLite |
| **DataStore** | — | **1.1.7** | ✨ New addition |
| **Coroutines** | 1.7.x | **1.11.0** | Major upgrade |
| **Material Design** | 1.11.x | **1.14.0** | Upgrade |
| **WorkManager** | 2.9.x | **2.11.2** | Upgrade |
| **ConstraintLayout** | 2.1.x | **2.2.1** | Upgrade |
| **Fragment** | 1.6.x | **1.9.0-alpha01** | Alpha upgrade |
| **Appcompat** | 1.6.x | **1.8.0-alpha01** | Alpha upgrade |
| **Firebase Analytics** | 21.x | **23.2.0** | Major upgrade |
| **Source Files** | 6,416 | **6,548** | +132 files |

---

## 🏗️ Architecture Summary

```
┌──────────────────────────────────────────────────────────┐
│                    TiViMate v5.3.3                       │
│              ar.tvplayer.tv (compileSdk=37)               │
├──────────────────────────────────────────────────────────┤
│  UI Layer                                                │
│  ├── MainActivity (Leanback, PiP, landscape)             │
│  ├── Settings Activities (playlists, guides, appearance) │
│  ├── Reminder / Recording Popups                         │
│  ├── Premium Unlock (Play Billing 9.1.0)                 │
│  └── File Picker / SMB Browser                           │
├──────────────────────────────────────────────────────────┤
│  Core Domain                                             │
│  ├── Room 2.7.0 + Room3 3.0.0-rc01 (Dual!)              │
│  ├── DataStore 1.1.7 (Preferences)                       │
│  ├── Lifecycle 2.11.0 (ViewModel, LiveData, Process)     │
│  ├── WorkManager 2.11.2 (EPG/Playlist sync)              │
│  └── Coroutines 1.11.0 (async everything)                │
├──────────────────────────────────────────────────────────┤
│  Media Playback                                          │
│  ├── Media3 (ExoPlayer successor)                        │
│  ├── FFmpeg JNI (software decode fallback)               │
│  └── Bundled SQLite (media cache DB)                     │
├──────────────────────────────────────────────────────────┤
│  Networking                                              │
│  ├── OkHttp 3 (connection pooling, TLS)                  │
│  ├── Retrofit 2 (API client)                             │
│  ├── Gson + Moshi (JSON parsing)                         │
│  └── SMBJ (SMBv2/v3 network shares)                     │
├──────────────────────────────────────────────────────────┤
│  Security & Protection                                   │
│  ├── DexProtector (native anti-tamper)                   │
│  ├── Bouncy Castle (crypto)                              │
│  ├── Google Tink (modern crypto)                         │
│  └── Zip4j (encrypted backup archives)                   │
├──────────────────────────────────────────────────────────┤
│  Analytics & Monitoring                                  │
│  ├── Firebase Crashlytics                                │
│  ├── Firebase Analytics 23.2.0                           │
│  ├── Firebase Remote Config                              │
│  └── Firebase Sessions                                   │
└──────────────────────────────────────────────────────────┘
```

---

## ⚠️ Notable Architectural Changes in v5.3.3

1. **Dual Room Database**: Both Room 2.7.0 AND Room3 3.0.0-rc01 are present, suggesting an incremental migration to the new Room3 KMP architecture with bundled SQLite. See [`tivimate_room3_bundled_sqlite_architecture.md`](TiViMate-NFO/tivimate_room3_bundled_sqlite_architecture.md) for the deep-dive source audit.
2. **Bundled SQLite**: Ships its own `libsqliteJni.so` via `sqlite-bundled 2.7.0-rc01`, no longer relying on the device's system SQLite — ensures consistent behavior across all Android versions (`AbstractC5031` / `@FastNative`).
3. **DataStore Addition**: `androidx.datastore 1.1.7` is a new addition, likely replacing some SharedPreferences usage for type-safe, coroutine-friendly preferences.
4. **compileSdk 37**: Compiled against the latest Android 17 SDK, indicating preparation for future OS features.
5. **Kotlinx Serialization**: Present alongside Gson and Moshi, suggesting gradual migration to Kotlin-native serialization.
6. **Foreground Service Types**: Now declares `mediaPlayback`, `dataSync`, and `specialUse` foreground service types for Android 14+ compliance.

---

> *Generated from TiViMate v5.3.3 decompiled APK analysis — July 2026*
