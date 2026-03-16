# ⌚ iOS + watchOS Architecture Demo

> A production-grade reference architecture for native iOS and Apple Watch apps built with SwiftUI — demonstrating clean MVVM, WatchConnectivity sync, HealthKit integration, and background refresh patterns.

---

## 📌 Purpose

This repository demonstrates how a senior iOS developer structures a real-world **iOS + watchOS** application for long-term maintainability and scalability.

It is not a toy example. Every pattern here reflects decisions made in production apps — from bidirectional Watch sync to HealthKit query handling and background task scheduling on watchOS.

**Live Architecture Dashboard →** [View Demo](https://ios-watchos-demo.netlify.app)
*(Deploy `index.html` to Netlify or GitHub Pages — see [Deployment](#deployment) below)*

---

## 🏗️ Architecture Overview

```
ios-watchos-architecture-demo/
│
├── README.md
├── index.html                        # Interactive architecture dashboard
│
├── iOS-App/
│   ├── WatchConnectivityManager.swift   # Bidirectional iPhone ↔ Watch sync
│   ├── HomeViewModel.swift              # MVVM ViewModel with Combine
│   ├── HealthKitManager.swift           # HealthKit queries & permissions
│   └── APIService.swift                 # Networking layer (protocol-based)
│
└── Watch-App/
    ├── WatchBackgroundRefresh.swift     # Background task scheduling
    ├── WatchViewModel.swift             # Lightweight Watch state
    └── SyncManager.swift                # Receives iPhone data
```

---

## 📱 iOS App — Layer Breakdown

| Layer | Responsibility |
|-------|---------------|
| **SwiftUI Views** | Declarative UI, zero business logic |
| **ViewModel (MVVM)** | `ObservableObject` + `@Published` state |
| **APIService** | Protocol-based networking, async/await |
| **HealthKitManager** | Permissions, queries, `HKStatistics` |
| **WatchConnectivityManager** | Message transport to/from Apple Watch |
| **App State** | Global state via `EnvironmentObject` |

### Key Patterns Used
- `async/await` with structured concurrency throughout
- `Combine` for reactive state bindings
- Protocol-oriented `APIServiceProtocol` for testability
- `weak self` in all closures to prevent retain cycles
- `@MainActor` on ViewModels for safe UI updates

---

## ⌚ watchOS App — Layer Breakdown

| Layer | Responsibility |
|-------|---------------|
| **SwiftUI Watch Views** | Compact, glanceable interface |
| **WatchViewModel** | Lightweight `ObservableObject` |
| **SyncManager** | Receives and processes iPhone data |
| **Background Refresh** | Periodic data sync via `WKApplication` |
| **Complication Provider** | Timeline data for Watch face complications |

---

## 🔗 WatchConnectivity — Communication Patterns

Three sync strategies, each used for different scenarios:

### 1. `sendMessage()` — Real-time foreground sync
```swift
// Use when: Watch is reachable and in foreground
// Latency: ~100ms
session.sendMessage(payload, replyHandler: { reply in
    print("Watch replied: \(reply)")
}, errorHandler: { error in
    transferUserInfo(payload) // automatic fallback
})
```

### 2. `transferUserInfo()` — Background guaranteed delivery
```swift
// Use when: Watch may be asleep or unreachable
// Delivery: guaranteed, queued until Watch wakes
session.transferUserInfo(["steps": stepCount, "updated": Date()])
```

### 3. `updateApplicationContext()` — Latest-value sync
```swift
// Use when: Only the most recent value matters
// Behavior: overwrites previous context, no queue buildup
try session.updateApplicationContext(["heartRate": latestHR])
```

**Decision flow:**
```
Is Watch reachable?
├── YES → sendMessage() with transferUserInfo fallback in errorHandler
└── NO  → transferUserInfo() for guaranteed eventual delivery
```

---

## 🏃 HealthKit Integration

```swift
// Request once at app launch — always handle denied state
try await store.requestAuthorization(toShare: [], read: readTypes)

// Query today's steps — async/await pattern
let steps = try await healthKitManager.queryTodaySteps()

// Send to Watch after fetching
watchSync.sendToWatch(["steps": steps])
```

**Required `Info.plist` keys:**
```xml
<key>NSHealthShareUsageDescription</key>
<string>Used to display your daily activity on Apple Watch.</string>
```

---

## 🔄 watchOS Background Refresh

```swift
// In @main App struct — handles background task
.backgroundTask(.appRefresh("com.app.background-refresh")) {
    await performBackgroundSync()
    scheduleNextRefresh()          // reschedule for next interval
}

// Schedule 15-minute refresh cycle
WKApplication.shared().scheduleBackgroundRefresh(
    withPreferredDate: Date().addingTimeInterval(15 * 60),
    userInfo: nil
) { _ in }
```

> ⚠️ watchOS enforces strict time budgets on background tasks. All network calls must complete within the allotted time or the system terminates the task.

---

## ✅ Production Readiness Checklist

### SwiftUI & Architecture
- [ ] MVVM separation — zero business logic in views
- [ ] No memory leaks in Combine subscriptions (`weak self`, `store(in:)`)
- [ ] All async operations use structured concurrency
- [ ] `EnvironmentObject` injected at root, not passed manually

### WatchConnectivity
- [ ] `WCSession` activated on both iPhone and Watch targets
- [ ] Reachability checked before every `sendMessage` call
- [ ] `transferUserInfo` fallback implemented
- [ ] Message payload stays under 65KB limit

### HealthKit
- [ ] `NSHealthShareUsageDescription` in `Info.plist`
- [ ] Authorization requested before first query
- [ ] Denied/restricted authorization handled gracefully
- [ ] Background delivery enabled for Watch target

### watchOS Background
- [ ] Background App Refresh entitlement enabled
- [ ] `scheduleBackgroundRefresh` called at end of each task
- [ ] Task completes within watchOS time budget
- [ ] Complication timeline refreshed on new data

### Build & Deployment
- [ ] Bundle IDs match in Xcode + App Store Connect
- [ ] Signing certificates valid (check expiry)
- [ ] iPhone app and Watch extension targets linked correctly
- [ ] Archive builds clean with zero warnings
- [ ] TestFlight build tested on **physical** devices (not just Simulator)
- [ ] Privacy manifest included (`PrivacyInfo.xcprivacy`)

### App Store
- [ ] App icons at all required sizes (iPhone + Watch)
- [ ] Screenshots prepared for iPhone and Apple Watch
- [ ] Privacy nutrition labels completed in App Store Connect
- [ ] Export compliance questions answered

---

## 🚀 Deployment

### Deploy the Dashboard to Netlify (2 minutes)
1. Go to [netlify.com/drop](https://netlify.com/drop)
2. Drag and drop `index.html`
3. Your dashboard is live at a public URL

### Deploy to GitHub Pages
1. Push this repo to GitHub
2. Go to **Settings → Pages**
3. Set source to `main` branch, `/ (root)`
4. GitHub Pages serves `index.html` automatically

---

## 🛠️ Tech Stack

| Technology | Usage |
|-----------|-------|
| **Swift 5.9+** | Primary language |
| **SwiftUI** | UI layer — iOS and watchOS |
| **Combine** | Reactive bindings and state management |
| **WatchConnectivity** | iPhone ↔ Apple Watch communication |
| **HealthKit** | Activity data queries and permissions |
| **async/await** | Structured concurrency throughout |
| **MVVM** | Architecture pattern |
| **TestFlight** | Beta distribution and testing |

---

## 📄 License

MIT License — free to use as a reference or starting point.

---

> Built to demonstrate senior-level Apple ecosystem architecture. Every pattern here is production-tested, not theoretical.
