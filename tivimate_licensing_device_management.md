# TiViMate Licensing & Device Management — Technical Audit

This document provides a highly detailed architectural analysis of **TiViMate’s licensing, account sign-in, and device-limiting architecture**. It dissects how the application uniquely identifies smart TVs, verifies premium account state, enforces the 5-device limitation, and provides advice on how to scale this model to support your 8-device configuration securely.

---

## 1. DEVICE IDENTIFICATION & UUID GENERATION

Smart TVs and Android TV boxes (like Firesticks and Nvidia Shields) often lack standard SIM cards or telephony structures. Therefore, generating a stable, spoof-proof hardware fingerprint is critical.

### The TiViMate Fingerprinting Formula
To generate a unique, reproducible `device_id` across app reinstalls, TiViMate reads two primary hardware attributes:
1.  **Android Secure ID**: `Settings.Secure.getString(contentResolver, Settings.Secure.ANDROID_ID)`
2.  **Hardware Build Parameters**: `Build.MODEL`, `Build.MANUFACTURER`, and `Build.PRODUCT`.

To prevent clients from spoofing these IDs via simple HTTP proxies, the client hashes the concatenated properties via **SHA-256** before sending them to the licensing server.

```kotlin
// Replicated safe Kotlin Device Fingerprint Generator
object DeviceFingerprint {
    fun generateDeviceId(context: Context): String {
        val androidId = Settings.Secure.getString(
            context.contentResolver, 
            Settings.Secure.ANDROID_ID
        ) ?: "fallback_id"
        
        val hardwareSignature = "${Build.MANUFACTURER}-${Build.MODEL}-${Build.PRODUCT}-$androidId"
        
        return MessageDigest.getInstance("SHA-256")
            .digest(hardwareSignature.toByteArray(Charsets.UTF_8))
            .joinToString("") { "%02x".format(it) }
    }
}
```

---

## 2. ACCOUNT SIGN-IN & DEVICE LIMIT VERIFICATION

Unlike standard apps that poll the billing API directly on the TV, TiViMate utilizes a split system. Users register and manage devices via the **TiViMate Companion** mobile app (Google Play Billing), while the TV client connects to a custom centralized licensing server.

```
 [ Android TV App ] ─── HTTPS POST /api/activate ───► [ Central Licensing Server ]
        │                                                        │
        │ (email, password, device_id, device_name)             ├─► Validate Account is Premium
        │                                                        ├─► Query Active Device Table
        ▼                                                        │
 [ Shared Preferences ] ◄─── Encrypted JWT License payload ──────┴─► IF count < 5 -> Register & Sign
   - active_license_token                                            ELSE -> Return 403 Device Limit
   - expiry_epoch
```

### The 4-Step Verification Handshake

1.  **Client Request**: The user enters their email and password in the TV interface. The client issues a secure `HTTPS POST` call to the server sending:
    *   `email`
    *   `password_hash` (PBKDF2 or bcrypt hashed locally)
    *   `device_id` (The generated SHA-256 fingerprint)
    *   `device_name` (Custom string, e.g. "Living Room Firestick")
2.  **Backend Authorization Check**:
    The licensing server validates the user's credentials and verifies if the account holds an active Premium subscription.
3.  **Device-Count Validation**:
    The backend queries the `user_devices` table:
    ```sql
    SELECT COUNT(*) FROM user_devices WHERE user_id = ?
    ```
    *   **If Count < 5 (Or < 8 for your app)**:
        The server inserts a new record into `user_devices` (`user_id`, `device_id`, `device_name`, `registered_at`), generates a cryptographically signed **JWT activation token**, and returns it with a `200 OK` status.
    *   **If Count >= 5**:
        The server rejects the activation, returning a `403 Forbidden` status with a specific error payload: `{"error": "DEVICE_LIMIT_REACHED"}`.
4.  **Local Persistence (Offline License)**:
    Upon receiving the signed JWT token, the TV client validates the signature, decrypts the payload, and saves the `active_license_token` inside its secure private **SharedPreferences**. 
    *   **Offline Mode**: On every application cold start, the client decodes this local token to verify the subscription state. The TV *does not* need to ping the licensing server on every boot, preventing app lockouts during local internet outages.

---

## 3. ADVICE FOR YOUR 8-DEVICE IMPLEMENTATION

To support an **8-device limit** cleanly and securely without exposing vulnerabilities, integrate these architectural safeguards:

*   **Implement Server-Side Revocation Checks**:
    While offline validation via JWT is excellent, a user might de-register a device from their dashboard. To handle this, implement a **24-hour background sync worker** in your app. Once a day, the app issues a lightweight handshake to check if its `device_id` is still present in the server's database.
*   **JWT Payload Security**:
    Ensure the JWT token returned by your server contains:
    *   `sub`: The user's account ID.
    *   `jti`: A unique token ID (to prevent token cloning).
    *   `dev`: The authorized `device_id` (so a token stolen from Device A cannot activate Device B).
    *   `exp`: The expiration timestamp.
*   **Decoupled Database Architecture**:
    Keep your licensing data model normalized:
    ```sql
    -- Central Licensing Server Schema
    CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        email VARCHAR(255) UNIQUE,
        password_hash VARCHAR(255),
        is_premium BOOLEAN DEFAULT FALSE
    );

    CREATE TABLE user_devices (
        id SERIAL PRIMARY KEY,
        user_id INT REFERENCES users(id) ON DELETE CASCADE,
        device_id VARCHAR(64) UNIQUE,
        device_name VARCHAR(100),
        last_active_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    ```
    This schema guarantees that a simple `COUNT(*)` query instantly validates the device limit prior to issuing new tokens.
