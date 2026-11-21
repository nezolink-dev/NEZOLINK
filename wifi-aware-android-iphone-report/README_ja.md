# AndroidとiPhoneのWi-Fi Aware相互接続：ペアリングは成功、データ通信路（NDP）は確立しない

[English version](README.md)

**状況（2026-11-20時点）**

Pixel 9（Android 17）とiPhoneの組み合わせでは、Wi-Fi Awareの**発見とペアリングは成功します**。しかし、**データ通信路（NDP）は一度も確立しませんでした**。

Wi-Fi詳細ログをONにして調べると、Android OS（フレームワーク）はNDPの命令を、Wi-Fiチップ側（HAL）へ**常に`securityConfig=null`（暗号設定なし）**、かつ`frameProtectionEnabled=true`の状態で送っていました。HALはこれを拒否します。
- Pixelが開始側のとき：**`NAN_STATUS_INTERNAL_FAILURE`**
- Pixelが応答側のとき：**`NAN_STATUS_INVALID_PARAM`**

調べた範囲では、アプリ側の公開APIでこれを回避する手段はありません。

AndroidとiPhoneのWi-Fi Awareに取り組む開発者やAIが同じ行き止まりで時間を使わずに済むよう、試験方法・結果・伏せ字処理済みのログを公開します。

> 背景：この試験は、近くにいるAndroidとiPhoneのユーザーがオフラインでグループ通話・チャットするアプリ **NEZOLINK**（個人開発・日本）の開発中に行いました。サービス名`_nezolink-p2p._tcp`はこのアプリのものです。

---

## 要点

| 項目 | 内容 |
|---|---|
| Android端末 | Google Pixel 9（`tokay`）、Android 17、ビルド`CP3A.261005.005`（user、release-keys）、SDK 37 |
| AndroidのAware能力 | `supportedCipherSuites=1`（NDPはNCS-SK-128のみ）、`supportedPairingCipherSuites=48`（PASN-128/256）、`gtkCipherSuites=64`、`isNanPairingSupported=true` |
| iPhone | iPhone 12 mini（`iPhone13,1`）、iOSビルド24B156 |
| ペアリング方式 | Androidの**OS任せのペアリング（framework-offloaded pairing）**：`PublishConfig/SubscribeConfig.Builder.setFrameworkOffloadedPairingEnabled(true)`（37.2で追加）。PINはシステム画面で入力・表示 |
| 発見 | ✅ 両方向で成功 |
| ペアリングSETUP（PASN、PIN方式） | ✅ 観測した4回すべて成功。Pixelが検索側でも公開側でも成功し、iPhone側にもペアリング済み機器として保存された |
| 保存済みペアリングの再確認（Pair Verify） | ✅ 成功 |
| NDP（Pixelが開始側） | ❌ 9/9回失敗：`initiateDataPath … securityConfig=null` → `NAN_STATUS_INTERNAL_FAILURE`（HALが約6ミリ秒で拒否） |
| NDP（Pixelが応答側） | ❌ 3/3回失敗：`respondToDataPathRequest … securityConfig null` → `NAN_STATUS_INVALID_PARAM` |
| 暗号化NDP・アプリのデータ・音声 | ❌ 一度も到達せず |

## アプリでは回避できない理由

- iOSのWi-Fi Awareはペアリングが必須で、NDPにはペアリングから作った鍵を使います。iPhoneのログには`using datapathTransientKey`と出ており、NDP要求には`security: 1`が付いています。
- Androidのアプリが指定できるNDPの暗号設定は`WifiAwareDataPathSecurityConfig`だけです。これはNCS-SK（PMKまたはパスフレーズ）かNCS-PK（PMK＋PMKID）しか受け付けません。PASNペアリングの鍵を入れる項目も、ペアリング済みの相手（alias）を`WifiAwareNetworkSpecifier`に結び付ける方法もありません。
- そこでアプリは暗号設定を空にして、OS任せのペアリングに頼りました。Androidの公式ドキュメントには、この機能を有効にするとNDPを始める前にOSが自動でペアリングを試みる、と書かれています。実際にペアリングは成功します。しかしその後のNDP要求は`securityConfig=null`で出されています。

## 試験ケース

共通の準備：Pixelで「開発者向けオプション → Wi-Fi詳細ログ」をONにします。新規ペアリングから試すときは、両端末のペアリングを削除します。

### ケースK：iPhoneが公開、Pixelが検索してNDPを開始

| 端末 | アプリの処理 |
|---|---|
| iPhone | `_nezolink-p2p._tcp`を`DevicePairingView(.wifiAware(.connecting(to: service, from: .userSpecifiedDevices)))`で公開 |
| Pixel | `SubscribeConfig.Builder().setServiceName("_nezolink-p2p._tcp").setSubscribeType(SUBSCRIBE_TYPE_ACTIVE).setFrameworkOffloadedPairingEnabled(true)`で検索。発見したら`WifiAwareNetworkSpecifier.Builder(subscribeSession, peerHandle).build()`（暗号設定なし）で`requestNetwork(..., 120000)`。アプリのデータは送らない |

結果（[`evidence/android_caseK_pixel_initiator_wifi_verbose.log`](evidence/android_caseK_pixel_initiator_wifi_verbose.log)より）：
```
21:19:42.364 WifiAwareStateManager: initiateDataPathSetupLocal: starting auto bootstrapping and pairing
21:19:42.364 WifiAwareNativeApi: initiateBootstrapping: ... method=32            （PixelでPIN入力）
21:19:48.419 WifiHAL : Pairing request type: 0                                    （SETUP）
21:19:48.421 WifiAwareAutoPairingManager: Ending session AutoPairingSession{isPublisher=false, state=4, ... sessionEndReason=Pairing successful}
21:19:48.450 WifiAwareNativeApi: initiateDataPath: ... channel=2437, ... securityConfig=null, isOutOfBand=false, appInfo.length=0, ... pubSubId=1, frameProtectionEnabled=true
21:19:48.456 WifiHAL : Received nan_error string NAN_STATUS_INTERNAL_FAILURE
21:19:48.456 WifiNanIfaceCallbackAidlImpl: notifyInitiateDataPathResponse: status=1 (NAN_STATUS_INTERNAL_FAILURE), ndpInstanceId=0
```
ペアリング成功の29ミリ秒後にNDP命令が出て、その6ミリ秒後にHALが拒否しています。2回目は保存済みペアリングの再確認（`Pairing request type: 1`）を経て、同じ結果になりました。

補足：ケースKでは、iPhoneのアプリは`NetworkListener`を開いていませんでした。ただしHALは命令をその場で拒否している（`ndpInstanceId=0`）ので、iPhone側の待ち受けの有無は結果に関係していないと考えられます（推定）。

### ケースJ：Pixelが公開・応答、iPhoneが検索してNDPを開始

| 端末 | アプリの処理 |
|---|---|
| Pixel | `PublishConfig.Builder().setServiceName("_nezolink-p2p._tcp").setFrameworkOffloadedPairingEnabled(true)`で公開。`WifiAwareNetworkSpecifier.Builder(publishSession).build()`（相手を限定しない形、暗号設定なし、90秒）で`requestNetwork` |
| iPhone | `DevicePicker(.wifiAware(.connecting(to: .userSpecifiedDevices, from: service)))`でPixelを選択（PINはPixelに表示）。その後`NetworkBrowser(for: .wifiAware(.connecting(to: .allPairedDevices, from: service)))` → `NetworkConnection`（TCP） |

結果（[`evidence/android_caseJ_pixel_responder_wifi_verbose.log`](evidence/android_caseJ_pixel_responder_wifi_verbose.log)、[`evidence/ios_caseJ_iphone_initiator.log`](evidence/ios_caseJ_iphone_initiator.log)より）：
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
（`securityConfignull`は、フレームワークのログ自体が`=`なしで出力しているものです。また、iPhoneの時計はPixelより約0.5〜1秒遅れていたため、両端末の時刻を混ぜた並びは厳密な順番ではありません。）

## ログから分かったその他のOSの動き

- OS任せのペアリングを有効にすると、OSがアプリのペアリング設定とSSI（サービス固有情報）を上書きします。ログ：`maybeInjectAutoPairingConfiguration: Injected default AwarePairingConfig for PIN/Passphrase Keypad|Display`、`SSI is not valid for auto-pairing. Injecting default SSI for auto-pairing.`
- OSのペアリングセッション（`AutoPairingSession`）はSETUPの完了で終わります。iPhoneはNDP要求の前に必ずPair Verifyを行いますが、そのときAndroidは`Failed to get AutoPairingSession`と記録します。
- 以前の試験版では、ほかの組み合わせも試しました（アプリからのブートストラップ・ペアリング、暗号なしの指定、受信側の各種設定など）。どれもiPhoneとのNDPは確立しませんでした。ここでは、OSからHALへの引数をWi-Fi詳細ログで確認できた2ケースだけをまとめています。

## 解釈（仮説・未確定）

このPixelのビルドでは、「ペアリングの結果（ND-PMK／NPK）をNDPの暗号設定に入れる」処理が抜けているか、つながっていないように見えます。その結果、HALは「フレーム保護はONなのに鍵がない」命令を受け取って拒否している、と考えています。

もう一つの可能性として、PixelのHALやファームウェアが、ペアリング由来の鍵を使うNDPにそもそも対応していないことも考えられます（NDPの`supportedCipherSuites=1`）。どちらの場合でも、直せるのはアプリではなく、GoogleかWi-Fiチップのメーカーです。

## 代わりにどうしているか（実用上の回避策）

AndroidとiPhoneの間では、NEZOLINKは**Android端末が作るWi-Fiホットスポット**と、端末内のWebSocket通信を使います。Android同士は、以前の版で成功したパスフレーズ方式のWi-Fi Aware NDPを使い続けます。

PixelのOSが更新されるたびに、ケースKとJを再試験する予定です。`securityConfig`が`null`以外になったら、このページを更新します。

## 自分の端末で再現する方法

1. Pixel：開発者向けオプション → **Wi-Fi詳細ログをON**にし、試験中の`adb logcat`（またはバグレポート）を取得する。
2. 両端末のWi-Fi Awareペアリングを削除する（iPhoneは「設定 → プライバシーとセキュリティ → ペアリング済みデバイス」）。
3. 上のケースKまたはJを実行する。
4. Androidのログで`WifiAwareNativeApi: initiateDataPath`、`respondToDataPathRequest`、`NAN_STATUS_`、`AutoPairingSession`を検索する。iPhoneのコンソール（プロセス`wifip2pd`）で`Pair Setup`、`Pair Verify`、`NAN_DP_REQ`、`NAN_DP_END`を検索する。

## 証拠ファイル

どのログも抜粋で、MACアドレス、PIN、鍵、SSI（端末名を含む）、UUIDは削除しています。時刻は日本時間（UTC+9）、日付は2026-11-20です。

| ファイル | 内容 |
|---|---|
| `evidence/android_caseK_pixel_initiator_wifi_verbose.log` | Pixel、ケースK、2回（新規ペアリング → 保存済みペアリング） |
| `evidence/android_caseJ_pixel_responder_wifi_verbose.log` | Pixel、ケースJ、2回 |
| `evidence/ios_caseK_iphone_publisher.log` | iPhone、ケースK（ペアリング側） |
| `evidence/ios_caseJ_iphone_initiator.log` | iPhone、ケースJ（ペアリングとNDP要求） |

## 違う結果が出た方へ

AndroidとiPhoneのWi-Fi Aware **NDP**が成功した方がいれば、端末・OSビルド・役割（公開／検索）と上のログ行を添えてissueを立ててください。特に`securityConfig`が`null`以外になった例を歓迎します。「別の端末でも同じ失敗だった」という報告も役に立ちます。

## 関連リンク

- Androidリファレンス：`SubscribeConfig.Builder.setFrameworkOffloadedPairingEnabled`（37.2で追加）、`WifiAwareDataPathSecurityConfig`
- Apple Developer Forums：スレッド790195「Wi-Fi Aware between iOS 26 and Android device」、スレッド797186（ペアリング後のAndroid⇔AppleのNDP）
- Google Issue Tracker：（報告後にURLを追記）

## キーワード

Wi-Fi Aware、NAN、AndroidとiPhoneの相互接続、iOS 26 Wi-Fi Aware、Wi-Fi Aware 4.0、PASNペアリング、NDP、NAN data path、setFrameworkOffloadedPairingEnabled、AutoPairingSession、initiateDataPath、respondToDataPathRequest、securityConfig=null、NAN_STATUS_INTERNAL_FAILURE、NAN_STATUS_INVALID_PARAM、datapathTransientKey、Pixel 9、Android 17
