# Wi-Fi Aware between Android (Pixel 9) and iPhone: pairing works, the data path does not

[日本語版はこちら / Japanese version](README_ja.md)

**Status (2026-11-20):** On a Pixel 9 with Android 17, Wi-Fi Aware **discovery and pairing with an iPhone succeed**, but the **NAN data path (NDP) is never established**. With Wi-Fi verbose logging on, the Android framework always sends the NDP command to the Wi-Fi HAL with **`securityConfig=null`** and `frameProtectionEnabled=true`, and the HAL rejects it:
- **`NAN_STATUS_INTERNAL_FAILURE`** when the Pixel is the initiator.
- **`NAN_STATUS_INVALID_PARAM`** when the Pixel is the responder.

As far as we can tell, an app cannot fix this through the public API.

This repository publishes the test method, the results and sanitized logs. Other developers (and AI assistants) working on Android ↔ iPhone Wi-Fi Aware can then skip the dead ends.

> Context: these tests were done while developing **NEZOLINK**, an offline group voice/chat app for nearby Android and iPhone users (solo developer, Japan). The service name `_nezolink-p2p._tcp` comes from that app.

---

## Key facts

| Item | Value |
|---|---|
| Android device | Google Pixel 9 (`tokay`), Android 17, build `CP3A.261005.005` (user, release-keys), SDK 37 |
| Android Aware capabilities | `supportedCipherSuites=1` (NDP: NCS-SK-128 only), `supportedPairingCipherSuites=48` (PASN-128/256), `gtkCipherSuites=64`, `isNanPairingSupported=true` |
| iPhone | iPhone 12 mini (`iPhone13,1`), iOS build 24B156 |
| Pairing method | Android **framework-offloaded pairing**: `PublishConfig/SubscribeConfig.Builder.setFrameworkOffloadedPairingEnabled(true)` (added in version 37.2). PIN bootstrapping through the system UI. |
| Discovery | ✅ Works in both directions |
| Pairing SETUP (PASN, PIN) | ✅ Succeeded in all 4 observed attempts. Pixel as subscriber and as publisher. iPhone stores the paired device. |
| Pair Verify (stored pairing) | ✅ Succeeds |
| NDP, Pixel = initiator | ❌ 9/9 failed: `initiateDataPath … securityConfig=null` → `NAN_STATUS_INTERNAL_FAILURE` (HAL, about 6 ms) |
| NDP, Pixel = responder | ❌ 3/3 failed: `respondToDataPathRequest … securityConfig null` → `NAN_STATUS_INVALID_PARAM` |
| Encrypted NDP, app data, audio | ❌ Never reached |

## Why an app cannot work around it

- iOS Wi-Fi Aware requires pairing, and its NDP uses a key derived from the pairing. The iPhone log shows `using datapathTransientKey`, and the NDP request carries `security: 1`.
- On Android, the only NDP security an app can set is `WifiAwareDataPathSecurityConfig`. It accepts NCS-SK (PMK or passphrase) or NCS-PK (PMK + PMKID). There is no field for PASN pairing keys and no way to link a paired device (alias) to a `WifiAwareNetworkSpecifier`.
- The app therefore leaves the security config empty and relies on framework-offloaded pairing. The Android documentation says the framework will attempt to perform Wi-Fi Aware pairing automatically before starting the NAN datapath. The framework does pair successfully, but it then requests the NDP with `securityConfig=null`.

## Test cases

Preparation for both cases: enable **Developer options → Wi-Fi verbose logging** on the Pixel. To force a fresh pairing, remove the existing pairing on both devices.

### Case K: iPhone publishes, Pixel subscribes and initiates the NDP

| Side | What the app does |
|---|---|
| iPhone | `DevicePairingView(.wifiAware(.connecting(to: service, from: .userSpecifiedDevices)))` for `_nezolink-p2p._tcp` |
| Pixel | `SubscribeConfig.Builder().setServiceName("_nezolink-p2p._tcp").setSubscribeType(SUBSCRIBE_TYPE_ACTIVE).setFrameworkOffloadedPairingEnabled(true)`. On discovery: `WifiAwareNetworkSpecifier.Builder(subscribeSession, peerHandle).build()` (no security), then `requestNetwork(..., 120000)`. The app sends no data. |

Result, from [`evidence/android_caseK_pixel_initiator_wifi_verbose.log`](evidence/android_caseK_pixel_initiator_wifi_verbose.log):
```
21:19:42.364 WifiAwareStateManager: initiateDataPathSetupLocal: starting auto bootstrapping and pairing
21:19:42.364 WifiAwareNativeApi: initiateBootstrapping: ... method=32            (PIN keypad on Pixel)
21:19:48.419 WifiHAL : Pairing request type: 0                                    (SETUP)
21:19:48.421 WifiAwareAutoPairingManager: Ending session AutoPairingSession{isPublisher=false, state=4, ... sessionEndReason=Pairing successful}
21:19:48.450 WifiAwareNativeApi: initiateDataPath: ... channel=2437, ... securityConfig=null, isOutOfBand=false, appInfo.length=0, ... pubSubId=1, frameProtectionEnabled=true
21:19:48.456 WifiHAL : Received nan_error string NAN_STATUS_INTERNAL_FAILURE
21:19:48.456 WifiNanIfaceCallbackAidlImpl: notifyInitiateDataPathResponse: status=1 (NAN_STATUS_INTERNAL_FAILURE), ndpInstanceId=0
```
The NDP command is issued 29 ms after pairing succeeds and is rejected by the HAL 6 ms later. The second attempt used the stored pairing (`Pairing request type: 1`) and failed the same way.

Note: in Case K the iPhone app did not open a `NetworkListener`. The HAL rejected the command synchronously (`ndpInstanceId=0`), so the iPhone listener state most likely played no role. This is an inference.

### Case J: Pixel publishes and responds, iPhone subscribes and initiates the NDP

| Side | What the app does |
|---|---|
| Pixel | `PublishConfig.Builder().setServiceName("_nezolink-p2p._tcp").setFrameworkOffloadedPairingEnabled(true)`, then a wildcard `requestNetwork` with `WifiAwareNetworkSpecifier.Builder(publishSession).build()` (no security, 90 s) |
| iPhone | `DevicePicker(.wifiAware(.connecting(to: .userSpecifiedDevices, from: service)))`, PIN shown on the Pixel. Then `NetworkBrowser(for: .wifiAware(.connecting(to: .allPairedDevices, from: service)))` → `NetworkConnection` (TCP). |

Result, from [`evidence/android_caseJ_pixel_responder_wifi_verbose.log`](evidence/android_caseJ_pixel_responder_wifi_verbose.log) and [`evidence/ios_caseJ_iphone_initiator.log`](evidence/ios_caseJ_iphone_initiator.log):
```
[Pixel]  21:45:11.633 AutoPairingSession{isPublisher=true, state=4, ... sessionEndReason=Pairing successful}
[iPhone] 21:45:11.31  Successfully added new paired Device / WiFi Aware Pairing succeeded
[iPhone] 21:45:23.85  [Pair Verify] ... Not authenticated. datapathTransientKey is nil
[iPhone] 21:45:24.89  state: authenticated → Pairing is in authenticated state, using datapathTransientKey
[iPhone] 21:45:24.89  APPLE80211_IOC_NAN_DP_REQ datapathID=1 responderPublishID=1
[Pixel]  21:45:25.248 WifiAwareAutoPairingManager: Failed to get AutoPairingSession for peerId: 265
[Pixel]  21:45:25.775 WifiHAL : Received NAN_EVENT_DATA_REQUEST_INDICATION / security: 1 / peer csia_capabilities: 4
[Pixel]  21:45:25.783 WifiAwareNativeApi: respondToDataPathRequest: ... accept=true, ... securityConfignull, ... frameProtectionEnabled=true
[Pixel]  21:45:25.788 WifiHAL : Received nan_error string NAN_STATUS_INVALID_PARAM
[Pixel]  21:45:26.297 WifiAwareDataPathStMgr: onDataPathConfirm: ... accept=false, reason=0
[iPhone] 21:45:25.93  APPLE80211_M_NAN_DP_END → #### Connection failed
```
(`securityConfignull` is printed by the framework without `=`. The iPhone clock was about 0.5–1 s behind the Pixel clock, so mixed timestamps are not perfectly ordered.)

## Other framework behaviour seen in the logs

- With offload enabled, the framework replaces the app's pairing config and SSI: `maybeInjectAutoPairingConfiguration: Injected default AwarePairingConfig for PIN/Passphrase Keypad|Display` and `SSI is not valid for auto-pairing. Injecting default SSI for auto-pairing.`
- The `AutoPairingSession` ends when SETUP completes. When the iPhone runs Pair Verify before its NDP request (it always does), the framework logs `Failed to get AutoPairingSession`.
- Earlier lab builds tried other combinations: app-initiated bootstrapping and pairing, OPEN specifiers, inbound/responder variants. None produced an NDP with the iPhone. This repository documents only the two clean cases that have verbose framework→HAL arguments.

## Interpretation (hypothesis, not confirmed)

On this Pixel build, the step that turns the pairing result (ND-PMK/NPK) into the NDP security configuration seems to be missing or not wired. The HAL then receives "frame protection on, no key" and rejects it. The alternative is that the Pixel HAL/firmware does not support pairing-derived NDP security, since NDP `supportedCipherSuites=1`. Either way, a fix has to come from Google or the chip vendor, not from apps.

## What we do instead (practical workaround)

For Android ↔ iPhone, NEZOLINK uses a **Wi-Fi hotspot hosted by the Android device** plus a local WebSocket connection. Android ↔ Android keeps using Wi-Fi Aware with a passphrase-based NDP, which worked in earlier builds.

We plan to re-run Cases K and J after each Pixel OS update. We will update this page if `securityConfig` ever becomes non-null.

## How to reproduce on your own devices

1. Pixel: Developer options → **Enable Wi-Fi verbose logging**. Then capture `adb logcat` (or a bug report) during the test.
2. Remove existing Wi-Fi Aware pairings on both devices (iPhone: Settings → Privacy & Security → Paired Devices).
3. Run Case K or Case J as described above.
4. Search the Android log for `WifiAwareNativeApi: initiateDataPath`, `respondToDataPathRequest`, `NAN_STATUS_`, `AutoPairingSession`. Search the iPhone Console (process `wifip2pd`) for `Pair Setup`, `Pair Verify`, `NAN_DP_REQ`, `NAN_DP_END`.

## Evidence files

All logs are excerpts with MAC addresses, PINs, key material, service-specific info (device names) and UUIDs removed. Times are JST (UTC+9), 2026-11-20.

| File | Content |
|---|---|
| `evidence/android_caseK_pixel_initiator_wifi_verbose.log` | Pixel, Case K, 2 attempts (new pairing, then stored pairing) |
| `evidence/android_caseJ_pixel_responder_wifi_verbose.log` | Pixel, Case J, 2 attempts |
| `evidence/ios_caseK_iphone_publisher.log` | iPhone, Case K (pairing side) |
| `evidence/ios_caseJ_iphone_initiator.log` | iPhone, Case J (pairing and NDP request) |

## Have you seen something different?

If you have an Android ↔ iPhone Wi-Fi Aware **NDP** working, please open an issue with the device, OS build, roles and the log lines above. Results where `securityConfig` is not null are especially welcome. Even "same failure on another device" helps.

## Related links

- Android reference: `SubscribeConfig.Builder.setFrameworkOffloadedPairingEnabled` (added in 37.2), `WifiAwareDataPathSecurityConfig`
- Apple Developer Forums: thread 790195 "Wi-Fi Aware between iOS 26 and Android device"; thread 797186 (NDP between Android and Apple after pairing)
- Google Issue Tracker: (add the issue link here after filing)

## Keywords

Wi-Fi Aware, NAN, Neighbor Awareness Networking, Android iPhone interoperability, iOS 26 Wi-Fi Aware, Wi-Fi Aware 4.0, PASN pairing, NDP, NAN data path, setFrameworkOffloadedPairingEnabled, WifiAwareAutoPairingManager, AutoPairingSession, initiateDataPath, respondToDataPathRequest, securityConfig=null, frameProtectionEnabled, NAN_STATUS_INTERNAL_FAILURE, NAN_STATUS_INVALID_PARAM, datapathTransientKey, Pixel 9, Android 17, CP3A.261005.005
