# Architecture — Anti-Stalking Device Detector

Status: **Draft for review.** No code has been written yet.

## 1. Purpose and principles

A defensive privacy tool that warns people when:
- an unknown tracker is following them, planted on their car, or newly present at a place they visit regularly;
- a hidden camera of a suspicious class appears around them;
- someone has entered, or is waiting inside, their parked car.

Non-negotiable principles:
1. **Air-gapped by design.** No servers we operate, no cloud relay, no telemetry, no analytics. All data stays on the user's phone and car unit.
2. **Passive only.** The tool never pairs with, connects to, jams, spoofs, deauthenticates or transmits to devices the user doesn't own. The car unit's link to the car's CAN bus is listen-only, enforced in hardware.
3. **No owner identification.** The tool never tries to identify or locate whoever placed a device.
4. **Assume a hostile observer.** The user's phone may be monitored by an abuser: disguise mode, quick-wipe, neutral wording option.
5. **Honest results.** The app never implies "you're safe". The app shows what each scan can and can't detect.

## 2. Decision log

| # | Decision | Rationale |
|---|---|---|
| D1 | Android and iPhone both supported | Reach everyone at risk |
| D2 | Kotlin Multiplatform + Compose Multiplatform | One-person team; one language, one UI |
| D3 | libsodium for app-layer crypto; OS keystores for keys | Audited primitives; no custom crypto |
| D4 | Build our own in-car unit as a discreet OBD-II dongle | Car makers' firmware is closed and their APIs are cloud-only |
| D5 | Car firmware in C on ESP-IDF | Most mature ESP32 toolchain |
| D6 | **No cloud relay.** Away alerts only by SMS straight from the car unit's SIM | Removes a server attack surface and an account an abuser could compromise |
| D7 | Phase order: phone app → car unit → ESP32 companion | Ship protection early; companion reuses car firmware |
| D8 | Connection sniffing out of scope | Adds no detection value; advertisements are sufficient |
| D9 | Camera risk is by brand/category tier plus context, not "anything new" | Business CCTV and doorbells are expected; action cams and spy cams aren't |

Note on D6: SMS travels over the cellular network, so the system isn't strictly air-gapped end to end. No infrastructure we run is involved, and SMS can be switched off, leaving a **Bluetooth-only mode** (approach warning only) that is fully local.

## 3. Phases

| Phase | Deliverable |
|---|---|
| **v1** | Phone app: tracker detection, following check, car mode, saved places, Bluetooth action-cam detection, alerts, detail screen, locate mode, evidence export, guidance pages |
| **v1.1** | Wi-Fi camera scan (Android), local-network camera discovery, guided room and car sweeps, infrared finder, disguise, quick-wipe |
| **v2** | Car unit prototype: presence radar, motion, Bluetooth approach warning, 24/7 tracker scan on the car |
| **v2.1** | Car unit SMS away alerts, CAN door/trunk events for selected models, Wi-Fi camera capture around the parked car |
| **v3** | ESP32 pocket companion (car firmware subset), production hardware, certification |

Each milestone delivers: working code, tests, a changelog, and open questions/risks. Work stops for review after each one.

## 4. Device tiers

### Cameras

| Tier | Base risk | Examples |
|---|---|---|
| 0 Business infrastructure | Ignore | Axis, Hanwha, Avigilon, Verkada, Bosch, Hikvision/Dahua pro lines, Flock/Vigilant plate readers |
| 1 Home consumer | Low (raised by context) | Ring, Nest, Arlo, Blink, Wyze, Eufy, Tapo, Reolink |
| 2 Portable | Medium (high if stationary and persistent) | GoPro, DJI Osmo, Insta360, Akaso; dashcams (BlackVue, Viofo, Nextbase, 70mai) |
| 3 Covert | Very high | Generic mini/spy cams (V380, Lookcam, HDWiFiCam, CamHi, YsxLite, LittleStars app families); disguised chargers, clocks, smoke detectors |
| 4 Cellular / no radio | Not detectable by phone | 4G trail cams (Spypoint, Tactacam, Moultrie), 4G solar cams, SD-only, pen/button cams |

### Trackers

| Tier | Base risk | Examples |
|---|---|---|
| A Finder-network tags | High when separated and following | AirTag, third-party Find My tags, Tile, SmartTag 1/2, Google Find My Device network tags, DULT tags |
| B Modified / DIY | Very high | Speakerless AirTags, OpenHaystack-style clones |
| C Cellular GPS | Not detectable by phone | LandAirSea, Spytec, Tracki, Invoxia, OBD trackers (Bouncie, Vyncs) |

Tier membership lives in versioned signature files (§9), so it can change without code changes.

## 5. Coverage

| Threat | Android | iPhone | + Car unit | + ESP32 | Not covered → fallback |
|---|---|---|---|---|---|
| AirTag / Find My tags | ✅ | OS alerts only | ✅ 24/7 | ✅ | |
| Tile, SmartTag, Google network, DULT | ✅ | ✅ (to verify) | ✅ | ✅ | |
| Speakerless / clone AirTags | ✅ | ❌ | ✅ | ✅ | |
| Cellular GPS trackers | ❌ | ❌ | ⚠️ research | ❌ | Guided car sweep |
| Wi-Fi cams (own hotspot) | ⚠️ manual scan | ❌ | ✅ | ✅ | |
| Wi-Fi cams (joined to a network) | ⚠️ same network only | ⚠️ mDNS only | ✅ | ✅ | |
| Action cams (Bluetooth) | ✅ | ✅ | ✅ | ✅ | |
| Cellular / SD-only cams | ❌ | ❌ | ❌ | ❌ | Infrared + lens sweep |
| Wired cams | ❌ | ❌ | ❌ | ❌ | Physical sweep |
| Someone inside parked car | ❌ | ❌ | ✅ | ❌ | |
| Car door/trunk opened while away | ❌ | ❌ | ✅ (radar/motion all cars; CAN on supported models) | ❌ | |
| Cellular audio bugs | ❌ | ❌ | ⚠️ research | ❌ | Professional sweep |

### iPhone limitations
- CoreBluetooth hides Apple's own advertisements (AirTags, other Find My devices); iOS's built-in alerts cover AirTags instead.
- Device addresses are replaced with random per-app IDs.
- Background scanning must name service UUIDs and is infrequent.
- No Wi-Fi scanning API. SSDP and WS-Discovery need Apple's multicast entitlement (to request).
- Disguise can change the icon only, not the app name.
- Apps can't read SMS; car-unit SMS alerts are read by the user in Messages.

## 6. System overview

```
┌──────────────── Phone (Android / iPhone) ─────────────────┐
│  Sources: BLE scanner · Wi-Fi scanner · LAN discovery ·   │
│           Car-unit link · Replay (simulation)             │
│        │ Flow<RawObservation>                             │
│        ▼                                                  │
│  Classifier ─▶ Fingerprinter ─▶ Sighting store (+location)│
│                                      │                    │
│                                      ▼                    │
│  Engines: Following · Car mode · Saved places · Camera    │
│        │ Score + reasons                                  │
│        ▼                                                  │
│  Alert manager (thresholds, safe list, transit) ─▶ UI     │
└──────────────────────────▲────────────────────────────────┘
                           │ BLE (encrypted, ~30–50 m)
┌──────────── Car unit (OBD-II dongle) ─────────────────────┐
│  Radar · IMU · CAN (listen-only) · Mic classifier ·       │
│  BLE tracker scan · Wi-Fi capture                         │
│        ▼                                                  │
│  Event engine ─▶ Event log (encrypted flash)              │
│        ├─▶ BLE: approach warning + log sync               │
│        └─▶ LTE-M: SMS to user's number (optional)         │
└───────────────────────────────────────────────────────────┘
```

## 7. Repository layout

```
app/
  androidApp/        Android entry point, foreground service, notifications
  iosApp/            Xcode project, iOS entry point, background modes
shared/
  model/             Domain types
  signatures/        Signature file loader + schema validation
  classifier/        Raw observation → classification
  fingerprint/       Merging rotated addresses into one identity
  engine/            Following, car mode, saved places, camera scoring
  data/              SQLDelight + SQLCipher, repositories, retention, wipe
  carlink/           Car-unit protocol, crypto, pairing (Kable)
  report/            Report model + JSON; PDF via expect/actual
  platform/          expect/actual: BLE scan, Wi-Fi, LAN, location, keystore
  sim/               Replay of recorded sessions
  ui/                Compose Multiplatform screens
firmware/
  car-unit/          ESP-IDF project
  protocol/          Protocol spec + shared test vectors (used by shared/carlink tests too)
signatures/          Versioned JSON signature files (bundled into app and firmware)
fixtures/            Recorded advertisement and discovery captures
docs/
```

## 8. Phone app

### 8.1 Stack

| Layer | Choice |
|---|---|
| Language / UI | Kotlin Multiplatform, Compose Multiplatform |
| DI | Koin |
| Storage | SQLDelight + SQLCipher; settings in DataStore (multiplatform) |
| Async / serialization | Coroutines, Flow, kotlinx.serialization |
| BLE scanning | Native: Android `BluetoothLeScanner`, iOS `CoreBluetooth` |
| Car-unit link | Kable |
| Location | Fused Location Provider / CoreLocation |
| Maps | MapLibre + OpenStreetMap; tile loading can be switched off (offline plot instead) |
| Crypto | libsodium (multiplatform binding); keys wrapped by Android Keystore / iOS Keychain |
| PDF | Android `PdfDocument` / iOS `UIGraphicsPDFRenderer` |
| Tests | kotlin.test, Turbine, fixtures, replay |
| Minimums | Android 10 (SDK 29), iOS 16 |

### 8.2 Scanning strategy

**Android**
- Foreground service (`connectedDevice` + `location` types) with a persistent notification.
- Filtered scan always on: manufacturer ID `0x004C`; service UUIDs `0xFEED`, `0xFEEC`, `0xFD5A`, `0xFEAA`, `0xFCB2`, plus action-cam identifiers. Filtered scans survive screen-off.
- Unfiltered sweep periodically while the screen is on, to catch unknown recurring devices.
- Respect the limit of 5 scan starts per 30 s. Duty cycle to be measured and documented.
- Wi-Fi: user-started scans only (OS limits to ~4 per 2 min in foreground).

**iPhone**
- `CoreBluetooth` with `bluetooth-central` background mode and state restoration; scan by service UUID only (no Apple payloads visible).
- Significant-location-change and region monitoring to wake for saved places and car mode.

### 8.3 Classifier

| Device | Signal | Notes |
|---|---|---|
| Apple Find My | Manufacturer `0x004C`, type `0x12` | Long payload = separated (carries public-key bytes); short payload = near owner. Status byte gives device class (AirTag / accessory / AirPods) per AirGuard; verify with recordings |
| Tile | Service `0xFEED` / `0xFEEC` | |
| Samsung SmartTag | Service `0xFD5A` | Payload carries state (e.g. overmature offline) |
| Google Find My Device network | Service `0xFEAA`, frame `0x40` / `0x41` | `0x41` = unwanted-tracking-protection mode → strong signal |
| DULT | Service `0xFCB2` | Pin a draft version in the signature file |
| Action cams | GoPro service `0xFEA6`; DJI / Insta360 identifiers from recordings | |
| Unknown | Stable payload structure recurring across places | Logged, low base score |

### 8.4 Fingerprinting
- Apple near-owner: address and key rotate ~every 15 min → can't link across rotations. Documented limitation; this mode isn't the main threat.
- Apple separated: key rotates ~daily → link by key fragment within the day; link across days only by co-occurrence and timing (low confidence).
- Others: combine payload structure, stable payload bytes, timing and signal-strength continuity.
- Every link carries a confidence value that feeds the score.

### 8.5 Engines (all thresholds configurable)

**Following** (default): ≥3 distinct places (200 m clusters), spanning ≥30 min, with ≥1 km travelled between them.
Modifiers: + separated mode, + strong signal share, + seen while moving; suppressed for safe-listed devices and transit sessions.

**Car mode**: session starts on car Bluetooth connection or driving detection. Flags a tracker present on most drives (default ≥3 drives), regardless of distance.

**Saved places**: user saves home, work, regular parking. The engine learns a baseline over several visits; it alerts on new devices that persist across visits.

**Camera score** = tier base risk
- \+ new at a saved place
- \+ persists over time / repeat visits
- \+ strong and stationary signal
- \+ streaming activity (car unit / ESP32)
- \+ concurrent tracker on the user's car
- − seen once or moving past

All engines output a 0–100 score plus a list of reasons, which become the plain-language explanation.

### 8.6 Storage schema (SQLCipher)

| Table | Columns |
|---|---|
| `device` | id, kind, tier, fingerprint_key, link_confidence, first_seen, last_seen, separated, user_label (NONE/SAFE/SUSPICIOUS), last_score |
| `sighting` | id, device_id, session_id, ts, rssi, address, raw_payload, lat, lon, accuracy_m, speed_mps |
| `scan_session` | id, start, end, source (LIVE/SIM/CAR_UNIT/COMPANION), kind (GENERAL/CAR/PLACE/SWEEP), transit_tag |
| `place` | id, name, lat, lon, radius_m, baseline_ready |
| `place_baseline` | place_id, device_id, visits_seen, first_seen, last_seen |
| `wifi_observation` | id, session_id, ts, bssid, ssid, rssi, channel, signature_id, source |
| `lan_host` | id, session_id, ts, ip, hostname, discovery_info, open_ports, signature_id |
| `car_event` | id, unit_id, ts, type (OCCUPIED/DOOR/MOTION/GLASS/TRACKER/CAMERA), detail, synced_at |
| `alert` | id, subject_type, subject_id, ts, score, reasons_json, state |
| `signature_meta` | file, version, sha256 |

Retention: configurable; secure delete via SQLCipher re-key + vacuum.

### 8.7 Security features
- **Database key**: random, wrapped by Keystore/Keychain, never leaves the device.
- **Quick-wipe**: destroys the wrapped key, then deletes the database (crypto-shred). Reachable from a hidden gesture and from settings.
- **Disguise**: Android swaps launcher icon and name via activity aliases; iPhone swaps icon only. Optional PIN to open.
- **Neutral wording**: notification text option that doesn't mention trackers or cameras.
- **Permissions**: each one explained in plain language before the system prompt; the app degrades gracefully if refused.

## 9. Signature files

Versioned JSON under `signatures/`, bundled with the app and firmware, validated against a schema at load time. Updates ship with app releases (no network fetch, consistent with air-gapping).

```json
{
  "schema": 1,
  "version": "2026.10.0",
  "entries": [
    {
      "id": "gopro-ble",
      "category": "camera",
      "tier": 2,
      "brand": "GoPro",
      "match": { "ble_service_uuid": "FEA6" },
      "base_risk": 50
    }
  ]
}
```

Files: `trackers.json`, `cameras.json` (BLE identifiers, SSID patterns, OUIs, mDNS/SSDP names, HTTP banners).
Caveat: OUIs often identify the Wi-Fi chip vendor (Espressif, Realtek, Beken), not the camera brand, so the matcher combines several clues per entry.

## 10. Car unit

### 10.1 Hardware

| Part | Choice |
|---|---|
| MCU | ESP32-S3 |
| CAN | ESP32 TWAI + transceiver in listen-only (silent) mode |
| Cellular | LTE-M module with SMS (Quectel BG95 or SIMCom SIM7080G), pre-certified |
| Presence | TI IWRL6432 60 GHz radar (prototype: HLK-LD2410) |
| Motion | Low-power accelerometer with wake interrupt |
| Audio | MEMS mic; on-chip glass-break classifier; audio never stored or transmitted |
| Power | OBD pin 16 (12 V), buck regulator, low-voltage cutoff, deep sleep woken by IMU/CAN |
| Form factor | Discreet OBD-II dongle; optional low-profile extension cable; optional rear-seat radar satellite |
| Design tools | KiCad; 3D-printed enclosure → injection-moulded |

### 10.2 Firmware (ESP-IDF, C, FreeRTOS)
Tasks: sensor sampling (radar duty-cycled), event engine, BLE peripheral (phone link) + BLE observer (tracker scan), Wi-Fi promiscuous capture (v2.1), modem/SMS, power manager.
Security: secure boot v2, flash encryption, signed OTA delivered from the phone over BLE, no debug ports enabled in production.

### 10.3 Phone ↔ car unit protocol
- Pairing in person: QR code on the unit + BLE LE Secure Connections; then libsodium `crypto_kx` key exchange; long-term keys in Keystore/Keychain and the unit's encrypted flash.
- App-layer messages encrypted with `crypto_secretbox`, with counters to stop replay.
- Messages: status, event-log sync, approach warning, config, OTA chunks.
- Written spec and shared test vectors in `firmware/protocol/`, run by both firmware and phone tests.

### 10.4 Alerts
- **Approach warning (Bluetooth)**: when the paired phone comes in range, the unit sends status before the user reaches the car. Occupancy or an intrusion while away triggers a full-screen warning ("Someone may be inside your car. Don't approach. Call emergency services.").
- **SMS (optional)**: sent to the user's number on intrusion. Neutral-wording option. Each SMS includes a short code derived from the shared key, so the user can verify it's genuine in the app.

## 11. Testing
- Unit tests for classifier, fingerprinter and engines against recorded fixtures (real AirTag, Tile and SmartTag captures recorded by the developer, plus synthetic cases marked as such).
- Replay mode feeds recorded sessions through the full pipeline ("followed", "not followed", "car", "saved place" scenarios).
- Protocol test vectors shared between firmware and phone.
- Battery: measured duty cycles and drain per platform, documented in `docs/BATTERY.md`.
- CI: GitHub Actions; Linux for Android/shared, macOS runner for iOS, ESP-IDF container for firmware.

## 12. Known limitations
- Cellular trackers, cellular/SD-only/wired cameras and audio bugs can't be detected by the phone.
- Apple near-owner devices can't be linked across 15-minute rotations.
- iPhone can't see Apple Find My advertisements or scan Wi-Fi.
- Camera brand classification can be defeated by renaming or hiding SSIDs.
- CAN door events vary by make and model; radar and motion are the universal signals.
- SMS crosses the cellular network and can be read by anyone with access to the user's messages (neutral-wording option mitigates).
- "No suspicious devices found" never means "safe"; the UI states this.

## 13. Risks and open questions
1. Rear-seat radar coverage from an under-dash dongle.
2. OBD port power behaviour and parasitic drain across vehicles.
3. App Store / Play review of disguise mode, background location and Bluetooth background use.
4. Apple multicast entitlement approval for SSDP/WS-Discovery on iPhone.
5. DULT draft churn.
6. Legal review per jurisdiction: radar and microphone in vehicles, Wi-Fi capture, LAN port checks.
7. Certification costs and timeline for the car unit (v3).
8. Action-cam BLE identifiers beyond GoPro need confirming from recordings.

## 14. Milestones (v1 / v1.1)

| # | Scope |
|---|---|
| M1 | Project scaffold (KMP, Android, iOS), signature loader, BLE scanning on both platforms, raw logger, classifier (Find My, Tile, SmartTag, Google network, DULT, GoPro), fixtures from real devices |
| M2 | Fingerprinting, encrypted storage, location tagging, following / car mode / saved places engines, alerts, replay mode |
| M3 | Detail screen, locate mode, safe/suspicious labels, PDF + JSON export, guidance pages |
| M4 | Wi-Fi scan (Android), LAN discovery, camera signatures and scoring |
| M5 | Guided room and car sweeps, infrared finder, disguise, quick-wipe, retention, README |

Car unit milestones will be planned at the end of v1.1.
