# Local Server Setup Guide

This document describes the setup and configuration required to run the Happy app with a local development server.

## Overview

The Happy app is configured to connect to a production server by default. To use a local development server, you need to configure the server URL through the app's UI or environment variables.

## Server Configuration Architecture

### Server URL Resolution

The app determines which server to connect to using a three-tier fallback system (defined in `sources/sync/serverConfig.ts:72`):

```typescript
export function getServerUrl(): string {
    return serverConfigStorage.getString(SERVER_KEY) ||
           process.env.EXPO_PUBLIC_HAPPY_SERVER_URL ||
           DEFAULT_SERVER_URL;
}
```

1. **Custom URL in MMKV storage** - Set via the Server Configuration screen
2. **Environment variable** - `EXPO_PUBLIC_HAPPY_SERVER_URL` (only works in dev builds)
3. **Default production server** - `https://api.cluster-fluster.com`

### WebSocket Connection

The app uses Socket.io to connect to the server for real-time updates at `/v1/updates` path (`sources/sync/apiSocket.ts:58-69`).

## Setup Steps

### 1. Local Server

Ensure your happy-server is running locally:
- Server URL: `http://192.168.0.176:3005`
- Health check endpoint: `http://192.168.0.176:3005/health` should return 200 OK
- WebSocket path: `/v1/updates`

### 2. Android App Setup

#### Option A: Using Server Configuration Screen (Recommended)

1. **Added Server Configuration Link to Welcome Screen**
   - Modified `sources/app/(app)/index.tsx:104-111, 172-179`
   - Added a "Server Configuration" text link below the "Link or Restore Account" button
   - Link navigates to `/server` route
   - Works in both portrait and landscape layouts

2. **Using the Configuration**
   - Open the Happy app
   - On the welcome screen, tap "Server Configuration" link
   - Enter: `http://192.168.0.176:3005`
   - Tap "Save"
   - Return to welcome screen and create/restore account

#### Option B: Using Environment Variables (Dev Builds Only)

1. **Start Expo Dev Server**
   ```bash
   cd /Users/vance/happy
   EXPO_PUBLIC_HAPPY_SERVER_URL=http://192.168.0.176:3005 npx expo start --port 8082
   ```

2. **Run in tmux** (for persistent session)
   ```bash
   tmux new-window -n "expo-happy" "EXPO_PUBLIC_HAPPY_SERVER_URL=http://192.168.0.176:3005 npx expo start --port 8082"
   ```

3. **Connect via Expo Go/Dev Client**
   - Install Expo Go or Expo Dev Client from Google Play
   - Scan QR code from dev server
   - App will use the environment variable

### 3. Building Android APK

#### Debug Build (Recommended for Development)

```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@17
export ANDROID_HOME=$HOME/Library/Android/sdk
cd android
./gradlew assembleDebug
```

**Output:** `android/app/build/outputs/apk/debug/app-debug.apk`

#### Release Build

```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@17
export ANDROID_HOME=$HOME/Library/Android/sdk
cd android
./gradlew assembleRelease
```

**Output:** `android/app/build/outputs/apk/release/app-release.apk`

**Note:** Release builds don't include environment variables - you must use the Server Configuration screen.

### 4. Installing APK on Device

```bash
# List connected devices
adb devices

# Install APK (use -r flag to replace existing installation)
adb -s DEVICE_ID install -r android/app/build/outputs/apk/debug/app-debug.apk
```

### 5. Web Client Setup

#### Build Web Client with Local Server

```bash
cd /Users/vance/happy
EXPO_PUBLIC_HAPPY_SERVER_URL=http://192.168.0.176:3005 npx expo export --platform web
```

**Output:** Static web files in `/dist` folder

The web build is static - the server URL is bundled at build time via the environment variable.

## Key Files Modified

### Server Configuration UI
- **`sources/app/(app)/index.tsx`** - Welcome screen with server config link
- **`sources/app/(app)/server.tsx`** - Server configuration screen (already existed)
- **`sources/sync/serverConfig.ts`** - Server URL management and storage

### Server Connection
- **`sources/sync/apiSocket.ts`** - WebSocket connection setup
- **`sources/sync/sync.ts:2072-2073`** - Socket initialization with server URL

## Build Configuration

### Gradle Properties
**`android/gradle.properties`**
```properties
org.gradle.jvmargs=-Xmx8192m -XX:MaxMetaspaceSize=2048m
```
- Increased heap to 8GB for Android builds
- Required for large builds with many dependencies

### Java Version
- Requires Java 17 (`/opt/homebrew/opt/openjdk@17`)
- Set via `JAVA_HOME` environment variable

### Android SDK
- Required: Android SDK (`$HOME/Library/Android/sdk`)
- Set via `ANDROID_HOME` environment variable

## Troubleshooting

### Issue: "Create Account" Button Doesn't Work

**Cause:** App is trying to connect to production server instead of local server

**Solutions:**
1. Use Server Configuration screen to set custom URL
2. Ensure local server is running and accessible
3. Check WebSocket connection in browser DevTools (for web) or app logs (for mobile)

### Issue: Web Page Hangs with "Disconnected" Status

**Cause:** Web build was bundled with wrong server URL

**Solution:** Rebuild web export with correct environment variable:
```bash
EXPO_PUBLIC_HAPPY_SERVER_URL=http://192.168.0.176:3005 npx expo export --platform web
```

### Issue: Debug APK Shows Expo Dev Environment

**Cause:** Installing wrong APK or running in dev mode

**Solution:** Use release build or connect via Expo dev server for development

### Issue: Server Configuration Link Not Appearing

**Cause:** Old APK cached or not properly installed

**Solutions:**
1. Force close the app completely
2. Clear app cache/data in Android settings
3. Uninstall and reinstall the APK
4. Verify the correct APK was built (check build timestamp)

## Development Workflow

### Recommended Setup for Local Development

1. **Terminal 1:** Run happy-server locally
   ```bash
   cd /path/to/happy-server
   # Start server commands here
   ```

2. **Terminal 2:** Run Expo dev server in tmux
   ```bash
   tmux new-window -n "expo-happy" "EXPO_PUBLIC_HAPPY_SERVER_URL=http://192.168.0.176:3005 npx expo start --port 8082"
   ```

3. **Mobile:** Connect via Expo Go or install debug APK and configure server URL

4. **Web:** Build and serve from `/dist` folder

## Important Notes

- **MMKV Storage:** Server configuration is persisted using react-native-mmkv and survives app restarts
- **No Backward Compatibility:** Changes to server configuration don't require backward compatibility (per `CLAUDE.md:441`)
- **Environment Variables:** Only work in dev builds, not release builds
- **WebSocket Path:** Always `/v1/updates` - hardcoded in `apiSocket.ts:59`
- **Server Validation:** Server config screen validates server by checking for "Welcome to Happy Server!" response

## Server Configuration Screen Features

The server configuration screen (`sources/app/(app)/server.tsx`) provides:

- URL input with validation
- Server connectivity test
- Visual feedback during validation
- Reset to default production server
- Current server status indicator

**Validation:**
- Checks URL format
- Verifies server responds with "Welcome to Happy Server!"
- Shows error messages for connection failures

## References

- Server config storage: `sources/sync/serverConfig.ts`
- Socket initialization: `sources/sync/sync.ts:2072`
- WebSocket client: `sources/sync/apiSocket.ts`
- Welcome screen: `sources/app/(app)/index.tsx`
- Server config UI: `sources/app/(app)/server.tsx`
