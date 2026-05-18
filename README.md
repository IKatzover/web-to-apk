# Website-to-APK

A simple tool to convert any website into an Android APK without requiring Android Studio or Java programming knowledge. The app acts as a WebView wrapper around your chosen website.

## Features

* Simple command-line interface with colorful output
* Automatic Java 17 downloading option
* Automated Android SDK tools installation
* APK signing and building process
* Userscripts support
* Rich JavaScript Interfaces (Notifications, Scheduling, Media Session Control, and UnifiedPush)

## Quick Start 

(please note that i have yet to properly test the newest changes)

1. Clone this repository:

```bash
git clone https://github.com/IKatzover/web-to-apk
cd web-to-apk
```

2. Create a configuration file `webapk.conf`:

```ini
id = com.myapp.webtoapk             # Application ID (will be com.myapp.webtoapk)
name = My App Name                  # Display name of the app
mainURL = https://example.com       # Target website URL
icon = example.png                  # Path to your app icon (PNG format)

allowSubdomains = true              # Allow navigation between example.com and sub.example.com
requireDoubleBackToExit = true      # Require double back press to exit app

enableExternalLinks = true          # Allow external links
openExternalLinksInBrowser = true   # If allowed: open external links in browser or WebView
confirmOpenInBrowser = true         # Show confirmation before opening external browser

allowOpenMobileApp = false          # Block external app links/schemes

```

3. Validate setup and configurations to ensure everything works:

```bash
./make.sh check

```

4. Generate signing key (only needed once, keep the generated file safe):

```bash
./make.sh keygen

```

5. Apply configuration and build:

```bash
./make.sh build

```

The final APK will be created in the current directory.

### YouTube Example

Pre-configured configuration files for YouTube are available in the `confs/youtube` directory. To build a YouTube APK, simply execute:

```bash
./make.sh build confs/youtube/webapk.conf

```

## Available Commands

* `./make.sh check` - Verify environment dependencies and system integrity before building
* `./make.sh build [config]` - Apply configuration and build the release package
* `./make.sh keygen` - Generate signing key
* `./make.sh test` - Install and test APK on connected device (streams logcat output)
* `./make.sh clean` - Clean build targets and intermediate cache files
* `./make.sh apk` - Build APK directly without reapplying structural configurations
* `./make.sh apply_config` - Force map settings from configuration file down to raw source assets
* `./make.sh get_java` - Download OpenJDK 17 binaries locally

## App Links / Deep Links

You can make your app handle links to the website by setting the `deeplink` option in your configuration file. When set, clicking links to your website on the device will open them in your app instead of an external browser.

The app intercepts these incoming links via `onNewIntent` and gracefully updates the running WebView instantly without resetting runtime states.

For example, if your website is `https://example.com`, set:

```ini
deeplink = example.com
# or multiple
deeplink = example.com www.example.com

```

## JavaScript API Reference

The app exposes a powerful `WebToApk` bridge object to your frontend context.

### Notifications & Scheduling

Manage both immediate alerting and background time-delayed events. Background notifications use inexact wake-locks to guarantee background delivery without drawing intensive OS power constraints.

```javascript
// Check and request runtime notification permissions (Android 13+)
const isGranted = WebToApk.hasNotificationPermission();
WebToApk.requestNotificationPermission();

// Handle asynchronous approval callbacks in your JS
window.__onNotificationPermissionResult = function(granted) {
    console.log("Notification status changed to: ", granted);
};

// Fire immediate local system notification with deep-linking payload
WebToApk.showNotification("Title", "Message details", "https://example.com/target-path");

// Schedule notification for later execution (ID, delay in milliseconds, title, text, deepLink)
WebToApk.scheduleNotification(101, 5000, "Reminder", "5 seconds have passed!", "https://example.com/alerts");

// Cancel a pending background execution task by ID
WebToApk.cancelScheduledNotification(101);

```

### Media Control API

Map audio/video background states to OS control overlays.

```javascript
// Synchronize foreground item state to system lockscreen/notification card
WebToApk.updateMediaMetadata("Track Title", "Artist Name", "Album Name", "https://example.com/cover.jpg");
WebToApk.updateMediaPlaybackState("playing"); // Or "paused", "stopped"
WebToApk.updateMediaPositionState(240.0, 1.0, 45.5); // (duration, playbackRate, position)

// Register for transport control event callbacks from hardware/OS overlays
WebToApk.setMediaActionHandlers(["play", "pause", "seekto"]);

// Handle incoming hardware signals
window.__runMediaAction = function(action) {
    if (action === "play") audioTag.play();
    if (action === "pause") audioTag.pause();
};

```

### UnifiedPush Service

Native WebPush compliance architecture decoupled from proprietary Google Play Services.

```javascript
// Initialize distributor lookup and subscription request
WebToApk.unifiedPushSubscribe("VAPID_PUBLIC_KEY_BASE64...");

// Listen for system generated Push Endpoints
window.__shim_onNewEndpoint = function(subscriptionJson) {
    const sub = JSON.parse(subscriptionJson);
    // Send endpoint sub object structure back to your app backend server
};

// Terminate registration bindings
WebToApk.unifiedPushUnregister();

// Synchronously query local active configuration dump
const activeSub = WebToApk.getUnifiedPushSubscriptionJson();

```

## Userscripts Support

The app supports userscripts (similar to Tampermonkey/Violentmonkey scripts) through the `scripts` configuration option:

```ini
scripts = scripts/*.js             # Load all .js files from scripts directory
# OR
scripts = site-*.js                # Load all files matching pattern
# OR
scripts = script1*.js script20.js  # Load specific script files

```

### How Userscripts Work

* Scripts can use Tampermonkey/Violentmonkey/etc [`@match`](https://www.google.com/search?q=%5Bhttps://violentmonkey.github.io/api/metadata-block/%23match--exclude-match%5D(https://violentmonkey.github.io/api/metadata-block/%23match--exclude-match)) and [`@run-at`](https://www.google.com/search?q=%5Bhttps://violentmonkey.github.io/api/metadata-block/%23run-at%5D(https://violentmonkey.github.io/api/metadata-block/%23run-at)) directives; others are ignored.
* If no `@match` is specified, the script will run on all pages.
* Only `GM_addStyle` is supported from the Greasemonkey API.
* There is a native `toast("short message")` helper injected globally.
* Script console output (`console.log`/`alert`/`warn`) can be monitored safely using:

```bash
./make.sh test

```

## Additional WebView Options

The following advanced options can also be configured inside `webapk.conf`:

```ini
cookies = "key1=value1; key2=value2"  # Cookies for mainURL host
basicAuth = login:password            # HTTP Basic Auth credentials for mainURL host
userAgent = "MyCustomUserAgent/1.0"   # Custom UserAgent header
JSEnabled = true                      # Enable JavaScript execution
JSCanOpenWindowsAutomatically = true  # Allow JS to open new windows/popups

DomStorageEnabled = true              # Enable HTML5 DOM storage
DatabaseEnabled = true                # Enable HTML5 Web SQL Database
SavePassword = true                   # Allow saving passwords in WebView
AllowFileAccess = true
AllowFileAccessFromFileURLs = true
forceLandscapeMode = false            # Lock screen orientation to landscape

showDetailsOnErrorScreen = false      # Show connection error details for user
confirmOpenExternalApp = true         # Show confirmation before opening external app
blockLocalhostRequests = true         # Block requests to 127.0.0.1
trustUserCA            = false        # Allow app to trust user-installed SSL certs
geolocationEnabled     = false        # Allow access to device location (GPS)
cameraEnabled          = false        # Allow access to the camera for WebRTC or scanning
microphoneEnabled      = false        # Allow access to the microphone for audio recording or calls
allowMixedContent      = false        # Allow loading HTTP content on HTTPS sites
cacheMode              = default      # Or: "no_cache" (always network), "aggressive" (offline-first)
fadeInDuration         = 400          # Duration(in ms) of WebView fadeIn animation after spinner gone

```

## Edge-to-Edge Display

You can enable an immersive `edge-to-edge` mode where your web content draws behind the system bars (the status bar at the top and the navigation bar at the bottom). This is ideal for modern designs that mimic a native app feel.

```ini
edgeToEdge = true

```

**Important:** When this option is enabled, you **must** update your website's CSS to prevent important content from being obscured by the system bars.

The app injects the necessary safe area insets as CSS custom properties on the `<html>` element. You can use these to add padding to your content.

```css
body {
  padding-top: var(--safe-area-inset-top);
  padding-bottom: var(--safe-area-inset-bottom);
  padding-left: var(--safe-area-inset-left);
  padding-right: var(--safe-area-inset-right);
}

```

Additionally, once the inset variables are applied, the app dispatches a custom event `WebToApkInsetsApplied` on the `document` object.

## Technical Details

* Target Android API: 33 (Android 13)
* Minimum Android API: 24 (Android 7.0)
* Build tools version: 33.0.2
* Gradle version: 7.4
* Required Java version: 17

## Notes

* All app data is stored in the app's private directory.
* The keystore `app/my-release-key.jks` password is set to `"123456"` by default.
* Internet permission is required and automatically included.
* If you need to support [different Android versions](https://apilevels.com/), edit `app/build.gradle` accordingly.
* Vibe coded fork of [Jipok's work](https://github.com/Jipok/website-to-apk)