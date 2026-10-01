# 1. Product Overview

**Product Name:** Parental Control
**Platform:** Android & iOS
**Product Type:** Family / Parental Control & Child Device Safety Platform

## 1.1 Product Vision

Parental Control adalah aplikasi yang memungkinkan orang tua mengelola, memantau, dan melindungi perangkat anak dari satu aplikasi Parent.

Sistem terdiri dari:

* Parent 1
* Parent 2
* Child 1
* Child 2
* dan seterusnya

Satu keluarga dapat memiliki beberapa Parent dan beberapa Child.

Arsitektur utama:

```text
                    ┌─────────────────┐
                    │   Parent 1 App  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │                 │
                    │  Central Cloud  │
                    │                 │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Child Device  │
                    │                 │
                    │ Android / iOS   │
                    └─────────────────┘

                    Parent 2
                       │
                       ▼
                 Same Family
```

---

# 2. User Roles

## 2.1 Parent

Parent dapat:

* melihat device anak
* melihat lokasi
* melihat aktivitas device yang tersedia
* menerima alert
* mengatur parental-control policy
* melihat histori
* mengelola beberapa child

## 2.2 Child

Child adalah perangkat yang berada di bawah parental-control policy.

Child device harus memberikan authorization/permission yang diperlukan untuk capability yang tersedia pada platform tersebut.

---

# 3. Device Management

Setiap device mempunyai:

```text
family_id
device_id
user_id
role
platform
os_version
app_version
device_model
battery_level
network_status
last_seen
location_status
permission_status
```

Parent dapat melihat:

```text
Child 1
├── Online
├── Battery: 78%
├── Network: WiFi
├── Location: Available
├── Screen Time: 2h 14m
└── Last Sync: Now
```

---

# 4. Permission Management

Sistem harus memiliki centralized permission dashboard.

Contoh:

```text
CHILD 1

Location
[✓] Always

Notifications
[✓] Granted

Screen Capture
[✓] Granted

Microphone
[✓] Granted

Camera
[✓] Granted

App Usage
[✓] Granted

Background Activity
[✓] Available
```

Tetapi status permission harus dibedakan dengan **capability**.

Contoh:

```text
Permission = Granted
Capability = Not Supported
```

Ini penting khususnya pada iOS.

---

# 5. Feature Matrix

| Feature                           | Android      | iOS           | Requirement                    |
| --------------------------------- | ------------ | ------------- | ------------------------------ |
| Live Location                     | YES          | YES           | Location authorization         |
| Location History                  | YES          | YES           | Background location            |
| Geofence                          | YES          | YES           | Location                       |
| Screen Time                       | YES          | YES           | Platform API                   |
| App Usage                         | YES          | LIMITED       | Platform API                   |
| Screen Mirroring                  | YES*         | YES*          | Explicit system authorization  |
| Screen Recording                  | YES*         | YES*          | Explicit system authorization  |
| Live Camera                       | LIMITED      | LIMITED       | Foreground/system restrictions |
| Camera Recording                  | YES*         | YES*          | Camera authorization           |
| Live Microphone                   | LIMITED      | LIMITED       | Mic + background restrictions  |
| Audio Recording                   | YES*         | YES*          | Mic authorization              |
| Notification Log                  | YES          | NO equivalent | Android Notification Listener  |
| Notification Content              | YES          | NO equivalent | Android API                    |
| Hidden Launcher Icon              | POSSIBLE     | LIMITED       | OS dependent                   |
| Completely Invisible App          | NO guarantee | NO            | OS security model              |
| Remote Open Other App             | LIMITED      | NO            | OS restrictions                |
| Remote Background WhatsApp Access | NO           | NO            | Not a normal supported API     |
| Remote Screen Control             | LIMITED      | NO            | OS restrictions                |

`*` berarti capability tersedia dengan restrictions dan system-controlled authorization; bukan berarti bisa berjalan diam-diam tanpa batas.

---

# 6. Feature: Live Screen Mirroring

## Objective

Parent dapat melihat layar Child secara realtime.

Flow:

```text
Parent
   │
   │ Start Live Screen
   ▼
Backend
   │
   │ Signal / WebRTC
   ▼
Child
   │
   │ System Screen Capture Permission
   ▼
Screen Stream
   │
   ▼
Parent
```

## Android

Android menyediakan `MediaProjection`.

User harus memberikan persetujuan sistem untuk screen capture. Android juga mengekspos status active MediaProjection kepada user dan user dapat mencabut akses tersebut.

Android modern juga mensyaratkan foreground-service type `mediaProjection` untuk penggunaan tersebut.

### Requirement

```text
Child:
[Allow Screen Capture]
       ↓
[Start Monitoring]
       ↓
[WebRTC Stream]
```

## iOS

iOS menyediakan ScreenCaptureKit untuk screen streaming/mirroring, tetapi menggunakan mekanisme system content-sharing picker. Apple secara eksplisit mensyaratkan user meminta izin screen recording sebelum capture.

### Important

Tidak boleh diasumsikan:

```text
Parent clicks "Mirror"
        ↓
iPhone Child diam-diam mulai streaming
```

Model tersebut bukan capability umum iOS.

---

# 7. Feature: Screen Recording

Parent dapat meminta Child melakukan recording terhadap screen stream.

Architecture:

```text
Child Screen
     │
     ▼
Capture Engine
     │
     ├── Live Stream
     │
     └── Recorder
             │
             ▼
          Storage
```

Recording metadata:

```text
recording_id
device_id
started_at
ended_at
duration
file_size
storage_url
```

Retention policy:

```text
Default: 7 days
Configurable: 30 / 90 days
```

---

# 8. Feature: Live Camera

Parent dapat meminta live camera session.

UI:

```text
CHILD 1

Camera
┌──────────────────────┐
│                      │
│      LIVE CAMERA     │
│                      │
└──────────────────────┘

[ Front ] [ Back ]
[ Record ]
```

## Important Platform Constraint

Camera access tidak berarti aplikasi dapat membuka kamera secara diam-diam kapan saja.

Android memiliki pembatasan terhadap camera/microphone foreground services dan background execution. Android 14+ secara khusus membatasi service yang membutuhkan while-in-use permissions ketika dimulai dari background.

iOS juga mengharuskan permission untuk camera/microphone dan penggunaan protected resources dikontrol oleh OS.

Therefore:

**Live Camera = conditional capability, bukan unrestricted remote camera.**

---

# 9. Feature: Audio Monitoring

Parent dapat memulai audio session apabila platform mengizinkan kondisi tersebut.

```text
Parent
   │
   ▼
Start Audio
   │
   ▼
Child
   │
   ▼
Microphone
   │
   ▼
Audio Stream
   │
   ▼
Parent
```

Android mensyaratkan `RECORD_AUDIO` dan foreground-service requirements untuk microphone service tertentu.

iOS juga membutuhkan microphone authorization dan purpose description.

---

# 10. Feature: App Usage Monitoring

Objective:

Mengetahui aplikasi apa yang digunakan Child dan berapa lama.

Example:

```text
CHILD 1 — TODAY

WhatsApp       1h 32m
YouTube        1h 05m
Instagram        42m
Chrome           31m
Games            25m
```

## iOS

Apple menyediakan Family Controls / Device Activity untuk parental-control dan app/website usage. Family Controls membutuhkan entitlement khusus dan authorization.

Apple juga menggunakan privacy-preserving mechanisms untuk family activity data.

---

# 11. Feature: Notification Log

## Android

Target:

```text
WhatsApp

12:30
John:
"Besok jadi?"

13:15
Mom:
"Jangan lupa makan."
```

Android memiliki mekanisme Notification Listener yang dapat digunakan untuk menangkap notification events berdasarkan authorization yang diberikan user.

Data:

```text
notification_id
package_name
application_name
title
body
timestamp
```

## iOS

Requirement ini **tidak dapat diasumsikan tersedia** seperti Android.

iOS tidak menyediakan equivalent public API yang memungkinkan aplikasi parental-control membaca seluruh notification content dari aplikasi lain.

Therefore:

```text
Android → Supported
iOS     → Not Supported
```

---

# 12. Feature: Live Location

Ini merupakan salah satu capability paling feasible untuk kedua platform.

Data:

```text
latitude
longitude
accuracy
altitude
speed
heading
timestamp
```

Parent UI:

```text
CHILD 1

📍 Current Location

Last update:
10 seconds ago

Accuracy:
8 meters

Battery:
72%
```

## iOS

Core Location menyediakan `Always` authorization untuk menerima location events ketika aplikasi tidak sedang digunakan.

Background location tetap tunduk pada lifecycle dan system behavior iOS.

---

# 13. Feature: Find My-like Location

Product dapat menyediakan:

### Current Location

```text
Child 1
     ↓
📍 Current position
```

### Location History

```text
08:00  Home
08:32  School
12:10  Restaurant
14:20  School
17:15  Home
```

### Geofence

```text
HOME
Radius: 100m

When Child enters:
→ Parent notification

When Child exits:
→ Parent notification
```

---

# 14. Feature: Background Application Access

Original requirement:

> Parent 1 membuka WhatsApp Child 1 dari Parent device, sementara Child 1 tidak membuka WhatsApp.

Requirement tersebut harus diubah.

## NOT supported requirement

```text
Parent
   ↓
Open WhatsApp on Child
   ↓
Child doesn't know
   ↓
Parent sees WhatsApp screen
```

Ini bukan capability normal yang dapat kita jadikan cross-platform feature.

## Supported alternative

### App Usage Monitoring

Parent dapat melihat:

```text
WhatsApp
Last used: 21:32
Usage today: 1h 42m
```

### Screen Monitoring

Jika screen capture session aktif dan platform mengizinkan:

```text
Parent
   ↓
Live Screen
   ↓
Child's current screen
```

Tetapi ini bukan remote launching/remote control WhatsApp.

---

# 15. Feature: Hidden Application

Original requirement:

> Aplikasi benar-benar hilang dari app list dan desktop.

PRD mengubah requirement ini menjadi:

### Android

Application dapat dirancang tanpa launcher shortcut/icon dalam kondisi tertentu, tetapi **tidak boleh dianggap completely undetectable**.

### iOS

Tidak menjadikan "completely invisible application" sebagai product requirement.

Sebaliknya:

```text
Parental Control Authorization
        ↓
OS-managed protection
        ↓
Child cannot easily remove parental-control app
```

Apple Family Controls bahkan menyediakan mekanisme sehingga setelah parental-control app diotorisasi oleh parent/guardian, child tidak dapat menghapus app tersebut melalui mekanisme normal.

Jadi untuk iOS, pendekatan yang benar adalah:

**Protected, bukan invisible.**

---

# 16. Security Architecture

Karena sistem menangani data yang sangat sensitif, semua communication harus encrypted.

```text
Child Device
     │
     │ TLS
     ▼
API Gateway
     │
     ├── Authentication
     ├── Authorization
     ├── Device Service
     ├── Location Service
     ├── Notification Service
     ├── Realtime Service
     └── Media Service
```

Untuk realtime media:

```text
Child
  │
  │ WebRTC
  ▼
Signaling Server
  │
  ▼
Parent
```

Backend tidak perlu menjadi media relay untuk semua traffic jika direct WebRTC connection memungkinkan.

---

# 17. Family Model

```text
Family
│
├── Parent 1
├── Parent 2
│
├── Child 1
│   └── iPhone
│
├── Child 2
│   └── Android
│
└── Child 3
    └── Android
```

Permissions:

```text
Family Owner
    │
    ├── Parent Admin
    │
    ├── Parent
    │
    └── Child
```

---

# 18. Permission Matrix

Setiap capability memiliki status:

```text
NOT_REQUESTED
REQUESTING
GRANTED
DENIED
RESTRICTED
NOT_SUPPORTED
REVOKED
```

Contoh:

```text
Child 1 — iOS

Location          GRANTED
Background Loc    GRANTED
Screen Capture    GRANTED
Camera             GRANTED
Microphone         GRANTED
Notification Read  NOT_SUPPORTED
App Usage          GRANTED
```

Parent UI harus menampilkan:

> **Not Supported**

bukan:

> Permission Denied

karena kedua kondisi tersebut berbeda.

---

# 19. Privacy & Consent

Setiap monitoring capability harus memiliki:

* explicit authorization
* audit log
* data retention
* encryption
* access control
* revoke mechanism

Contoh:

```text
Parent 1 started screen session
Device: Child 1
Time: 21:42
Duration: 15 minutes
```

Semua sensitive actions harus masuk audit log.

---

# 20. MVP

## MVP Phase 1

Fokus pada capability yang paling feasible:

### Parent

* Login
* Family management
* Add Child
* Device pairing
* Device status
* Battery
* Online/offline
* Live location
* Location history
* Geofence
* Screen time
* App usage

### Child

* Device registration
* Permission management
* Location service
* Device heartbeat
* App usage collection
* Secure communication

Target:

**Android + iOS**

---

# 21. Phase 2

### Android

* Notification log
* Notification content
* Screen capture
* Live screen
* Screen recording

### iOS

* Family Controls
* Device Activity
* Managed Settings
* Screen Time features
* Screen capture capability where system authorization permits

---

# 22. Phase 3

Experimental / platform-dependent:

* Live camera
* Camera recording
* Live audio
* Audio recording
* Advanced device controls

Feature availability harus ditentukan berdasarkan OS/API/device state.

---

# 23. Out of Scope

Untuk initial product, sistem tidak menjanjikan:

* completely invisible application
* hidden surveillance
* bypassing OS permission
* bypassing App Store restrictions
* bypassing Android security
* remotely opening arbitrary third-party applications
* reading arbitrary private data from third-party applications
* defeating user/system security controls

---

# 24. Success Criteria

Product dianggap berhasil apabila:

### Family Management

* Parent dapat membuat family.
* Parent dapat menambahkan beberapa child.
* Child device dapat dipair.
* Multiple parent dapat mengakses child sesuai authorization.

### Monitoring

* Location dapat diperoleh secara reliable.
* Device status tersedia.
* App usage tersedia sesuai platform.
* Screen monitoring tersedia apabila OS authorization/capability mengizinkan.

### Security

* Child device authentication aman.
* Parent authorization tidak dapat digunakan oleh user lain.
* Semua sensitive data encrypted.
* Semua monitoring session tercatat dalam audit log.

---

# 25. Core Product Principle

Produk tidak menganggap:

```text
Permission Granted
        =
Everything Allowed
```

Tetapi:

```text
Permission
    +
OS Capability
    +
Entitlement
    +
App Lifecycle
    +
System Restrictions
    =
Actual Capability
```

Dengan demikian aplikasi dapat memberikan UX yang konsisten walaupun kemampuan Android dan iOS berbeda.
