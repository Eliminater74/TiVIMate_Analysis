# TiViMate Parental Controls & Security Architecture — Reference Analysis

This document provides a highly detailed architectural analysis of **TiViMate’s parental control security model**. It dissects the distinction between hidden and blocked stream categories, outlines how PIN hashes are stored securely in Android SharedPreferences, and identifies security weaknesses to avoid in your implementation.

---

## 1. PARENTAL LOCK SECURITY & HASH PERSISTENCE

To prevent kids or unauthorized users from accessing adult content or resetting settings drawers, TV players must implement parental lock security.

### PIN Cryptographic Hashing
To prevent reverse engineers or rooted device users from viewing the PIN in plain text by simply reading the private SharedPreferences XML, TiViMate handles PINs cryptographically:
*   **The Plain Text Trap**: Storing a PIN like `"1234"` in preferences is highly insecure.
*   **TiViMate Hashing Routine**:
    The plain text PIN is processed through a **SHA-256** hashing loop. The resulting 64-character hex string is stored in preferences:
    ```
    Plain PIN: "1234" ──► SHA-256 Hash ──► "03ac674216f3e15c761ee1a5e255f067953623c8b388b4459e13f978d7c846f4"
    ```
    When the user prompts the PIN lock card:
    1.  The plain text input is hashed locally.
    2.  The resulting string is compared directly with the stored preference hash.
    3.  If match, access is granted. The plain PIN is never written or stored anywhere.

---

## 2. THE DISTINCTION BETWEEN HIDDEN AND BLOCKED ITEMS

TiViMate implements two completely different levels of restrictions:

```
                  ┌──────────────────────────────────────────────┐
                  │          Parental Restriction Levels         │
                  └──────────────────────────────────────────────┘
                                  │
         ┌────────────────────────┴────────────────────────┐
         ▼                                                 ▼
  [ Hidden Elements ]                               [ Locked Elements ]
  - Flag: is_visible = 0                            - Flag: is_locked = 1
  - Completely erased from UI.                      - Remains visible in EPG/list.
  - Does not appear in EPG grids,                   - Displays a lock padlock icon.
    category menus, or searches.                    - Selecting triggers PIN card.
```

*   **Hidden Groups/Channels**:
    Marked with `is_visible == 0` or a soft-delete timestamp. The database query filters these rows entirely:
    `WHERE is_visible == 1 AND deleted_time IS NULL`
    They are invisible to standard users and do not appear in search results, favorites, or guides.
*   **Locked Groups/Channels**:
    Marked with `is_locked == 1`. They remain visible in the category drawers and EPG grid, but display a **lock icon**. Tapping D-pad Center on them stops playback initialization, overlaying the secure PIN entry dialogue.

---

## 3. SETTINGS & EXPORT SECURITY

*   **Preference Protection**:
    Attempting to open critical settings drawers (Parental Controls, Playlists, App Settings) prompts the PIN barrier overlay, preventing configuration alterations.
*   **Backup Security**:
    The hashed PIN is included in the `.tmb` backup archive inside the SharedPreferences XML backup. When restored on a new device, the parental lock remains active immediately without requiring configuration.

---

## PureFusionIPTV Beta 10 Recommendations

### Low-Risk Actions
*   Store all parental control PINs inside private SharedPreferences strictly as **SHA-256 hashed strings**.
*   Add a lock padlock icon drawable overlay to channels or categories that hold an `is_locked == 1` flag.

### Medium-Risk Actions
*   Implement a **Temporarily Unlocked Session Manager**. When a valid PIN is entered, unlock the channel but keep an active timer. If no key navigation occurs for 15 minutes, automatically clear the session token and relock the interface.
*   Secure EPG search queries. Exclude both `is_visible == 0` and `is_locked == 1` streams from search results if parental lock is active.

### High-Risk Actions
*   Hardcoding a universal override master PIN (e.g. `9999` or `0000`) inside the Kotlin source. If reverse engineered, this master code can be extracted in seconds and bypasses all user security.

### What to Measure First
*   Verify PIN matching times. Cryptographic validation should complete in **under 2ms** to ensure immediate menu feedback.

### What Not to Copy Directly
*   Do not copy plain-text preference keys or internal lock UI fragments.

### Suggested Implementation Phase
*   **Beta 10 Core Focus**: Strong SHA-256 PIN hashing and the logical separation of hidden vs. locked database elements are essential to establish stable parental control.
