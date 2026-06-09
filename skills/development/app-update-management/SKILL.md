---
name: app-update-management
description: Guidelines and checklists for releasing and verifying mobile application package updates and announcements to prevent version mismatches, local leaks, and caching bugs.
risk: safe
source: user-personal
date_added: "2026-06-08"
---

# App Update Management Skill

This skill provides standard operational procedures, checklists, and troubleshooting guides for synchronizing mobile application releases (APK/IPA) and database version announcements.

---

## 🚀 Releasing Checklist

Whenever a new application version is ready for release, execute the following steps strictly in order to prevent desynchronization:

### Step 1: Update Local Metadata Configurations
1. Open the mobile app's package config (e.g. `pubspec.yaml` for Flutter).
2. Increment the `version` name and build code (e.g., change `version: 1.0.11+12` to `version: 1.0.12+13`).

### Step 2: Perform Clean Release Build
Always run a clean build to ensure no debug/temporary states are compiled into the binary:
```bash
flutter clean
flutter pub get
flutter build apk --release
```

### Step 3: Server Deployment (Physical File)
Copy the newly built `app-release.apk` directly to the server's backend static file serving folder:
* **Production Path**: `backend/app-release.apk`
* *Verify*: Ensure the file size and checksum of the file on the server match the local build.

### Step 4: Publish Database Announcement
Using the admin JWT credentials, trigger the `/publish` POST endpoint (or use the admin panel) to update the DB config matching the exact release variables:
* `latest_version`: Must match `pubspec.yaml`'s version name (e.g. `"1.0.12"`).
* `version_code`: Must match the build code (e.g. `13`).
* `changelog`: Highlight user-facing enhancements.
* `force_update`: Set to `True` only if breaking changes require blocking older clients.

---

## 🛡️ Anti-Caching Proxy Rules (Nginx)

To prevent browsers, reverse proxies, and mobile package managers from serving cached, stale APK files:

Add the following block to your Nginx virtual host configurations:
```nginx
location ~* \.(apk|ipa)$ {
    # Force direct re-validation on every download request
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header Pragma "no-cache";
    expires 0;
}
```

---

## 🔍 Diagnostics & Troubleshooting

### Q: Why is the installed version still triggering "update required" prompts?
* **A1: Missing Rebuild**: The developer changed `pubspec.yaml` but forgot to run `flutter build apk` before uploading the file.
* **A2: Debug Priority Leak**: The server setting `DEBUG` was set to `True` on the host, causing it to serve old development builds from the local workspace folder instead of the static directory.

### Verification Script
Run this local script to extract and check the manifest version currently served by the cloud:
```python
import urllib.request
import zipfile
import os

url = "http://<server-ip>:<port>/api/v1/system/download/app-release.apk"
output = "downloaded-release.apk"

urllib.request.urlretrieve(url, output)
with zipfile.ZipFile(output, 'r') as z:
    manifest_data = z.read('AndroidManifest.xml')
    print("Version 1.0.12 utf-16 present:", b'1.0.12'.encode('utf-16le') in manifest_data)
os.remove(output)
```
