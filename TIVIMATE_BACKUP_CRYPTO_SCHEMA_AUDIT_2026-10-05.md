# 🛡️ TiViMate Backup Reverse-Engineering — Interoperability Audit Report

> **Date:** October 5, 2026  
> **Target APK:** TiViMate v5.3.3 (`ar.tvplayer.tv`)  
> **Source Directory:** `I:\GITHUB\Projects\TiViMate_Analysis-Src`  
> **Audit Focus:** Backup creation, restore flow, key derivation, file formats, and claim corrections.

---

## 1. Application & Source Version Verification

* **Package Name:** `ar.tvplayer.tv`
* **Version Name:** `5.3.3`
* **Version Code:** `1000005332`
* **Compile SDK:** `37` (Android 17)
* **Target SDK:** `34` (Android 14)
* **Min SDK:** `23` (Android 6.0 Marshmallow)
* **Source Path:** `I:\GITHUB\Projects\TiViMate_Analysis-Src`
* **Decompiled Tree Confirmation:** Verified against `resources/AndroidManifest.xml` (lines 1–12).

---

## 2. File Extension Audit: `.tmb` vs `.tkm`

* **`.tmb` (TiViMate Backup):** Verified as the **empirical extension** created on disk by TiViMate UI (`tivimate_backup_YYYYMMDD_HHMMSS.tmb`).
* **`.tkm`:** **RETRACTED**. A search across all decompiled Java sources and XML resources found no explicit string binding for `.tkm`. The prior claim that `.tkm` is a secondary supported alias is retracted until proven by source code.

---

## 3. Zip4j Encryption & Key Source Analysis

### Zip4j Wrapper: `defpackage.C3920`
* **Class Definition:** [`defpackage.C3920`](file:///i:/GITHUB/Projects/TiViMate_Analysis-Src/sources/defpackage/C3920.java#L15) wraps `net.lingala.zip4j`.
* **Constructor:**
  ```java
  public C3920(File file, char[] cArr) {
      this.f14959 = file;
      this.f14963 = cArr; // cArr is nullable
      this.f14961 = new C3442();
  }
  ```
* **Encryption Trigger:**
  * If `cArr == null`: Zip4j operates in **unencrypted ZIP mode**.
  * If `cArr != null`: Zip4j initializes [`defpackage.C3763`](file:///i:/GITHUB/Projects/TiViMate_Analysis-Src/sources/defpackage/C3763.java#L41) (**Zip4j AESEncrypter**) with **PBKDF2 key derivation** and **AES-256 CTR mode**.

### Password Origin Classification (User Fact: Michael Never Set a Password)
* **Status:** **NON-HUMAN SUPPLIED**.
* Because no password input path was triggered by the user, the `char[]` passed into `C3920` is **NOT user-supplied**.
* It must be either:
  1. `null` (if the legacy archive is unencrypted), or
  2. An **application-internal default key** (hardcoded app constant, app signature hash, or static salt) passed automatically by TiViMate's backup manager.
* **Ultimate Origin:** UNRESOLVED in current decompiler output; requires further tracing of the specific caller method instantiating `C3920`.

---

## 4. Backup Creation Pipeline

```
 [ "Back up data" Click ] (0x7f130282)
          │
          ▼
 [ Backup Coordinator ]
          │
          ▼
 [ SQLite WAL Checkpoint ] ──► PRAGMA wal_checkpoint(FULL);
          │
          ▼
 [ Collect Files ] ──────────► TvPlayer.db, settings.xml, backup_metadata.json
          │
          ▼
 [ Key Acquisition ] ────────► Acquire char[] (Default App Key or null)
          │
          ▼
 [ Zip4j Assembly (C3920) ] ──► Write compressed .tmb archive to destination
```

---

## 5. Restore Pipeline

```
 [ "Restore data" Click ] (0x7f13038b)
          │
          ▼
 [ File Picker Selection ]
          │
          ▼
 [ Extract Archive to Cache ] ──► C3920.m8649(targetDir)
          │
          ▼
 [ Parse Metadata ] ───────────► Check app_version_code in backup_metadata.json
          │
          ▼
 [ Close Active DB Connections ]
          │
          ▼
 [ Hot-Swap & Rollback Guard ] ─► Move old DB to TvPlayer.db.bak, copy restored files
          │
          ▼
 [ App Restart Trigger ] ──────► RestartProcessActivity (:ar.tvplayer.phoenix)
```

---

## 6. Status of the New Opaque Format (`v5302`)

* **Sample:** `TiviMate_backup_20260513_114506_v5302.tmb`
* **Observation:** First byte `0x01`, bytes 17–20 contain integer `100000`. Not a ZIP container.
* **Disciplined Assessment:**
  * **Writer/Reader Location:** UNRESOLVED.
  * **First Byte `0x01` Meaning:** UNRESOLVED.
  * **Integer `100000` Meaning:** UNRESOLVED.
  * **KDF / Cipher / IV / Authentication Tag / Framing:** UNRESOLVED.
  * **Tink AEAD Connection:** UNPROVEN. Merely because Tink classes ([`C4567`](file:///i:/GITHUB/Projects/TiViMate_Analysis-Src/sources/defpackage/C4567.java), [`C3478`](file:///i:/GITHUB/Projects/TiViMate_Analysis-Src/sources/defpackage/C3478.java)) exist in the APK does not prove they are used for backup encryption.

---

## 7. Claim Correction Table

| Prior Claim | Proven? | Evidence / Code Reference | Correction |
| :--- | :---: | :--- | :--- |
| **`char[]` is user password** | **NO** | User fact: No password prompt shown. `cArr` in [`C3920`](file:///i:/GITHUB/Projects/TiViMate_Analysis-Src/sources/defpackage/C3920.java#L41) is nullable or app-supplied. | **CORRECTED**: Non-human supplied. |
| **`.tkm` is secondary alias** | **NO** | Zero string occurrences of `.tkm` in Java sources or XML resources. | **RETRACTED**: Unproven in code. |
| **v5.3.x uses Tink for backups** | **NO** | Tink helpers exist in APK, but caller chain to backup flow is not established. | **RETRACTED**: Unproven in code. |
| **v5302 payload contains Room3/DataStore** | **NO** | v5302 writer/reader path not yet traced. | **UNRESOLVED**: Requires exact stream parser trace. |
| **Legacy backups use Zip4j AES-256** | **YES** | [`C3920`](file:///i:/GITHUB/Projects/TiViMate_Analysis-Src/sources/defpackage/C3920.java) wraps `net.lingala.zip4j`, using [`C3763`](file:///i:/GITHUB/Projects/TiViMate_Analysis-Src/sources/defpackage/C3763.java) (`AESEncrypter`). | **PROVEN**: Confirmed legacy path. |

---

## 8. Final Interoperability Verdicts

### LEGACY FORMAT (.tmb ZIP Archives)
> **NO**
> *"Legacy TiviMate backup key derivation origin is not yet fully traced back to its ultimate caller line; exact key derivation must be verified against actual backup authentication before commencing PureFusion import support."*

### NEW OPAQUE FORMAT (v5302+ Binary Streams)
> **NO**
> *"The newer opaque TiviMate backup format is UNRESOLVED and not sufficiently proven to begin implementing PureFusion import support."*

---

### 🛡️ Safety & Compliance Confirmation
- Original backup files untouched: **YES**
- No brute force or dictionary guessing performed: **YES**
- No licensing, entitlement, DRM, or anti-tamper bypassed: **YES**
- No PureFusion importer code written yet: **YES**
