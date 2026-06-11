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

### Step 5: Verify the Release Server-Side
After rebuilding and publishing, run these checks to ensure clients can discover the update:
1. **Query API Version**: Query the version endpoint (e.g. `/api/v1/system/version`) and confirm it returns the new `latest_version` and `version_code` matching your release.
2. **Check Router Config**: Verify that `app/routers/system.py` contains the updated `DEFAULT_VERSION_INFO`.
3. **Rebuild Container**: Ensure the docker container is rebuilt (`docker compose up --build -d web`) to load the updated router config.
4. **Test APK Serving**: Make a GET request (with `stream=True` to save bandwidth) to `/api/v1/system/download/app-release.apk` to ensure the endpoint returns `200` with the correct `Content-Length`.

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

### Q: Why is the real device not detecting the new version?
* **A1: Server Router Unpatched**: The server's `app/routers/system.py` has not been updated. The database auto-upgrades the version info based on the hardcoded `DEFAULT_VERSION_INFO` in `system.py`. If this is not updated, the DB config stays on the old version.
* **A2: Container Rebuild Omitted**: The local files were uploaded but the server containers were not rebuilt. Run `docker compose up --build -d web` to apply the updates.
* **A3: Cache / CDN Cache**: Proxies or CDN are caching the `/version` response. Ensure cache headers are `no-cache`.

### Verification Scripts

#### 1. Quick API Status Check
Ensure the returned version and APK downloads are healthy:
```python
import requests

api_url = "http://<server-ip>:<port>/api/v1/system/version"
apk_url = "http://<server-ip>:<port>/api/v1/system/download/app-release.apk"

# Check version json
version_info = requests.get(api_url).json()
print("API Version Info:", version_info)

# Check APK size and status code
r = requests.get(apk_url, stream=True)
print("APK Status Code:", r.status_code)
print("APK File Size (bytes):", r.headers.get("Content-Length"))
```

#### 2. APK Manifest Version Verification
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
    print("Version 1.0.19 utf-16 present:", b'1.0.19'.encode('utf-16le') in manifest_data)
os.remove(output)
```

