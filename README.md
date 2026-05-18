# 📱 App Store

A curated collection of Android apps available for direct download.

---

## Apps

### Pay4Me

<img src="https://raw.githubusercontent.com/kirubanandem/pay4me/2b0c4cf3340a9d5d1d87a4c18c3bc7be8ddf305b/logo.png" width="80" alt="Pay4Me logo" />

> Simplified shared payments and expense tracking.

| Detail | Info |
|---|---|
| Package | `com.kirubas.pay4me` |
| Download | [Pay4Me-V-1_3_8.apk](https://github.com/kirubanandem/pay4me/raw/refs/heads/main/Pay4Me-V-1_3_8.apk) |

---

### eXmanager

<img src="https://raw.githubusercontent.com/kirubanandem/eXmanager/c6dd4bb71f83bac848532e4d72102ca7352b8b89/logo.png" width="80" alt="eXmanager logo" />

> Your personal expense manager.

| Detail | Info |
|---|---|
| Package | `com.firebaseapp` |
| Download | [eXmanager_v22.apk](https://github.com/kirubanandem/eXmanager/raw/refs/heads/main/eXmanager_v23.apk) |

---

## Installation

1. Download the `.apk` file for the app you want.
2. On your Android device, go to **Settings → Security** and enable **Install unknown apps** for your browser or file manager.
3. Open the downloaded `.apk` file and tap **Install**.
4. Once installed, you can disable the unknown sources setting again.

> **Note:** These APKs are distributed outside the Google Play Store. Only install apps from sources you trust.

---

## App Index (JSON)

The app listing is maintained in [`apps.json`](./apps.json) with the following structure:

```json
[
  {
    "title": "App Name",
    "iconUrl": "https://...",
    "downloadUrl": "https://...",
    "description": "Short description.",
    "packageName": "com.example.app"
  }
]
```

---

## Contributing

To add or update an app entry, edit `apps.json` and open a pull request. Please ensure:

- The `downloadUrl` points to a stable release asset.
- The `iconUrl` is a publicly accessible image (PNG recommended).
- The `packageName` matches the app's actual Android package identifier.

---

## License

Distributed as-is. Refer to individual app repositories for their respective licenses.
