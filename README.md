# USB Cordova Plugin — Stabilization Demo

> **Demo** for Android Cordova USB plugin stabilization work.  
> Demonstrates the exact architecture, debounce logic, and auto-reconnect sequence that will be implemented in Milestone 1 & 2.

---

## Live Demo

Open `usb-cordova-demo.html` in any modern browser — no build step, no dependencies.

---

## What This Demo Shows

This interactive demo simulates the stabilization architecture that resolves two critical issues in existing Cordova USB plugins:

### Milestone 1 — USB Stabilization
- **Centralized state machine** (`USBStateManager`) with 4 states: `CONNECTED → DISCONNECTED → RECONNECTING → PERMISSION_PENDING`
- **Debounce logic** — USB attach/detach events within a 1000ms window are suppressed to prevent reconnect loops caused by power fluctuations and bus resets
- **Auto-reconnect sequence** — 6-step recovery: detect loss → close SDK → release `UsbDeviceConnection` → wait 500ms → rescan devices → reinitialize SDK
- **Persistent background service** (`UsbDeviceService` via `IBinder`) so connections survive Cordova activity lifecycle changes without clearing app state

### Milestone 2 — Android 13+ Compatibility
- `getParcelableExtra(UsbDevice.class)` — typed API 33+ signature (replaces deprecated call)
- `android:exported="false"` — explicit flag on all `BroadcastReceiver` entries
- Modern `UsbManager.requestPermission()` flow with persistent access
- Compliant `IntentFilter` handling for USB attach/detach events

---

## Demo Buttons

| Button | What it simulates |
|---|---|
| **Plug USB device** | Full attach flow: BroadcastReceiver → permission dialog → SDK init |
| **Unplug (bus reset)** | Detach event → triggers 6-step auto-reconnect sequence |
| **Rapid attach/detach ×5** | 5 events in 800ms — shows debounce suppressing 4 of 5 |
| **Power surge** | Inrush current disruption on a connected device → auto-recovery |
| **Clear log** | Clears the log output |

---

## Planned Plugin Architecture

```
Cordova JS Bridge
    cordova.exec(success, error, "UsbPlugin", "connect", [])
        ↓
UsbPlugin.kt
    extends CordovaPlugin
        ↓
USBStateManager.kt
    Connection state machine + debounce logic
        ↓
UsbDeviceService.kt
    Persistent background service (ServiceConnection / IBinder)
        ↓
Manufacturer SDK (.aar)
    USB printer / RFID reader
```

---

## Plugin JS Interface (planned)

```javascript
// Connect to USB device
cordova.exec(successCallback, errorCallback, "UsbPlugin", "connectDevice", []);

// Disconnect
cordova.exec(successCallback, errorCallback, "UsbPlugin", "disconnectDevice", []);

// Get current connection state
cordova.exec(successCallback, errorCallback, "UsbPlugin", "getConnectionState", []);

// Force reconnect
cordova.exec(successCallback, errorCallback, "UsbPlugin", "restartConnection", []);
```

---

## Android 13+ Changes Addressed

### `getParcelableExtra` — deprecated in API 33

```java
// Old (deprecated)
UsbDevice device = intent.getParcelableExtra(UsbManager.EXTRA_DEVICE);

// New (API 33+)
UsbDevice device = intent.getParcelableExtra(UsbManager.EXTRA_DEVICE, UsbDevice.class);
```

### BroadcastReceiver — explicit exported flag required (API 31+)

```xml
<!-- plugin.xml -->
<receiver
    android:name=".UsbReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.hardware.usb.action.USB_DEVICE_ATTACHED"/>
        <action android:name="android.hardware.usb.action.USB_DEVICE_DETACHED"/>
    </intent-filter>
</receiver>
```

### USB Permission — modern flow

```java
// Request permission
PendingIntent permissionIntent = PendingIntent.getBroadcast(
    context, 0,
    new Intent(ACTION_USB_PERMISSION),
    PendingIntent.FLAG_IMMUTABLE  // required API 31+
);
usbManager.requestPermission(device, permissionIntent);

// Handle in BroadcastReceiver
if (UsbManager.EXTRA_PERMISSION_GRANTED.equals(intent.getAction())) {
    boolean granted = intent.getBooleanExtra(UsbManager.EXTRA_PERMISSION_GRANTED, false);
    if (granted) openDeviceConnection(device);
}
```

---

## Debounce Logic (pseudo-code)

```kotlin
private var lastUsbEventTime: Long = 0
private val DEBOUNCE_MS = 1000L

fun onUsbEvent(event: UsbEvent) {
    val now = System.currentTimeMillis()
    if (now - lastUsbEventTime < DEBOUNCE_MS) {
        Log.d(TAG, "Debounce: USB event ignored within ${DEBOUNCE_MS}ms window")
        return
    }
    lastUsbEventTime = now
    processUsbEvent(event)
}
```

---

## Auto-Reconnect Logic (pseudo-code)

```kotlin
fun onConnectionLost() {
    setState(USBState.RECONNECTING)
    
    // Step 1: Close SDK session
    manufacturerSdk.close()
    
    // Step 2: Release USB connection
    usbDeviceConnection?.close()
    usbDeviceConnection = null
    
    // Step 3: Wait before rescan (avoids immediate re-failure)
    Handler(Looper.getMainLooper()).postDelayed({
        
        // Step 4: Rescan devices
        val device = usbManager.deviceList.values
            .firstOrNull { it.vendorId == TARGET_VENDOR_ID }
        
        if (device != null) {
            // Step 5: Reinitialize
            initializeDevice(device)
            setState(USBState.CONNECTED)
        } else {
            setState(USBState.DISCONNECTED)
        }
        
    }, 500L)
}
```

---

## Project Scope & Timeline

| Phase | Task | Est. hours |
|---|---|---|
| 1 | Technical investigation & USB lifecycle analysis | 3 hrs |
| 2 | Stabilization — StateManager + debounce + reconnect | 8 hrs |
| 3 | Android 13+ compatibility fixes | 4 hrs |
| 4 | Testing & verification | 5 hrs |
| **Total** | | **20 hrs @ $20/hr = $400** |

---

## Tech Stack

- **Cordova** — hybrid mobile framework
- **Kotlin / Java** — native Android plugin layer
- **Android USB Host API** — `UsbManager`, `UsbDeviceConnection`, `UsbInterface`
- **Android Services** — `ServiceConnection`, `IBinder` for persistent background connection
- **TypeScript + React** — SPA frontend layer
- **Third-party `.aar` libraries** — manufacturer device SDKs

---

## Files

```
├── usb-cordova-demo.html   # Interactive stabilization demo (open in browser)
└── README.md               # This file
```
