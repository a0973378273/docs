# App 進階知識 — iOS / Android / Flutter 三平台對照

## 一、執行緒與非同步處理

**Android — Coroutines**
- Kotlin Coroutines 是官方推薦的非同步方案，基於**結構化並行（Structured Concurrency）**設計
- 核心概念：`suspend fun`（掛起函式）、`CoroutineScope`（作用域）、`Dispatcher`（指定執行緒）
- `Dispatchers.Main` 主執行緒、`Dispatchers.IO` I/O 操作、`Dispatchers.Default` CPU 密集運算
- 搭配 `viewModelScope`、`lifecycleScope` 自動在元件銷毀時取消，避免記憶體洩漏
- `Flow` / `StateFlow` / `SharedFlow` 用於串流資料，搭配 `repeatOnLifecycle` 確保只在前景收集

**iOS — async/await + GCD**
- Swift 5.5 引入 `async/await`，是現代 iOS 的首選非同步方案
- **GCD（Grand Central Dispatch）**：底層執行緒管理，用 `DispatchQueue` 派發任務（舊寫法，仍常見）
- **Actor**：Swift 的並行安全機制，保證同一時間只有一個任務存取 Actor 的狀態
- `@MainActor` 標記必須在主執行緒執行的程式碼，SwiftUI View 的 `body` 已隱式繼承
- `AsyncSequence` / `AsyncStream` 用於串流資料

**Flutter — Isolate + async/await**
- Dart 是**單執行緒**模型，透過 Event Loop 處理非同步（類似 JavaScript）
- `async/await` + `Future` 處理非同步操作，不會阻塞 UI
- `Stream` 處理持續性資料（類似 Kotlin Flow / Swift AsyncSequence）
- `Isolate` 是獨立執行緒，擁有自己的記憶體空間，適合 CPU 密集運算
- `compute()` 是簡化版 Isolate，適合一次性任務。大多數場景不需要 Isolate（網路/檔案 I/O 已是非同步）

### 三平台非同步對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 主流方案 | Kotlin Coroutines | async/await（Swift Concurrency） | async/await + Future |
| 執行緒模型 | 多執行緒 | 多執行緒 | 單執行緒 + Isolate |
| 切換主執行緒 | `withContext(Dispatchers.Main)` | `@MainActor` / `DispatchQueue.main` | 不需要（預設就在主執行緒） |
| 背景執行 | `Dispatchers.IO` / `Dispatchers.Default` | `Task.detached` / `DispatchQueue.global` | `Isolate.run()` / `compute()` |
| 串流 | `Flow` / `StateFlow` / `SharedFlow` | `AsyncSequence` / `AsyncStream` | `Stream` / `StreamController` |
| 結構化並行 | `coroutineScope { }` | `TaskGroup` / `withTaskGroup` | 無原生支援（需手動管理） |
| 並行安全 | `Mutex`、`@Volatile` | `Actor`、`Sendable` | 天生安全（Isolate 記憶體隔離） |
| 取消機制 | `Job.cancel()`、scope 自動取消 | `Task.cancel()` | `Future` 不可取消；需用 `CancelableOperation` |

---

## 二、本地儲存

**Android — DataStore / Room**
- **DataStore**（官方推薦，取代 SharedPreferences）：鍵值對儲存，支援 Flow 非同步讀取，避免主執行緒阻塞
  - `Preferences DataStore`：簡單 key-value
  - `Proto DataStore`：用 Protocol Buffers 定義型別安全的結構化資料
- **Room**：SQLite 的 ORM 抽象層，用 `@Entity`、`@Dao`、`@Query` 定義，支援 Flow 觀察資料變化與自動遷移
- **MMKV**（騰訊開源）：高效能 key-value，基於 mmap，讀寫速度遠勝 SharedPreferences

**iOS — UserDefaults / CoreData / SwiftData**
- **UserDefaults**：簡單 key-value（等同 SharedPreferences），適合少量設定。不要存大量資料（一次載入全部到記憶體）
- **CoreData**：Apple 的 ORM 框架，功能強大但學習成本極高
- **SwiftData**（iOS 17+）：CoreData 的現代化替代，用 `@Model` macro 宣告，語法大幅簡化
- **Keychain**：安全儲存（加密），適合 token、密碼等敏感資料

**Flutter — SharedPreferences / Hive / sqflite**
- **shared_preferences**：官方套件，簡單 key-value（底層用各平台原生實作，有相同限制）
- **Hive**：輕量 NoSQL，純 Dart 實作，速度快，適合中等複雜度的結構化資料
- **sqflite**：SQLite 封裝，適合需要複雜查詢的關聯式資料
- **flutter_secure_storage**：安全儲存（Android 用 EncryptedSharedPreferences，iOS 用 Keychain）
- **Isar**（Hive 作者新作）：高效能 NoSQL，支援全文搜尋、自動索引

### 三平台本地儲存對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 簡單 key-value | DataStore（Preferences） | UserDefaults | shared_preferences |
| 關聯式資料庫 | Room（SQLite ORM） | CoreData / SwiftData | sqflite / drift |
| NoSQL | 無官方方案（可用 MMKV） | 無官方方案 | Hive / Isar |
| 安全儲存 | EncryptedSharedPreferences / Keystore | Keychain | flutter_secure_storage |
| 型別安全 | Room Entity + DataStore Proto | SwiftData @Model | Hive TypeAdapter / drift |
| 非同步 API | DataStore 用 Flow；Room 用 Flow/suspend | SwiftData 用 @Query + SwiftUI | 全部都是 async（Future） |
| 遷移機制 | Room Migration | CoreData Migration | sqflite onUpgrade |

---

## 三、導航與路由

**Android — Navigation Component / Compose Navigation**
- **Navigation Component**（XML 時代）：用 NavGraph 定義路由圖，`NavController` 控制跳轉
- **Compose Navigation**：Jetpack Compose 的路由方案，2.8+ 支援 type-safe route（用 `@Serializable` data class），避免字串路由
- 支援 Deep Link、動畫轉場、Safe Args 傳參、巢狀 NavGraph

**iOS — NavigationStack / UINavigationController**
- **UINavigationController**（UIKit）：傳統 push/pop 導航堆疊
- **NavigationStack**（SwiftUI，iOS 16+）：宣告式導航，用 `NavigationPath` 管理路由堆疊
- **Coordinator Pattern**：常見架構模式，將導航邏輯從 ViewController 抽離
- NavigationStack 僅支援 iOS 16+，需向下相容時仍需 NavigationView 或 UIKit

**Flutter — Navigator 2.0 / GoRouter**
- **Navigator 1.0**（命令式）：`Navigator.push()` / `pop()`，簡單但不支援 Deep Link
- **Navigator 2.0**（宣告式）：功能強大但 API 極其冗長，不建議直接使用
- **GoRouter**（官方推薦）：Navigator 2.0 的簡化封裝，支援 Deep Link、Redirect、巢狀路由
- `context.go()` 替換整個堆疊（像 web 跳頁），`context.push()` 才是堆疊推入

### 三平台導航對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 官方方案 | Compose Navigation | NavigationStack（SwiftUI） | Navigator 2.0 |
| 主流封裝 | 不需要（官方已足夠） | 不需要（SwiftUI 內建） | GoRouter（官方推薦） |
| 路由定義 | NavGraph（XML/DSL） | navigationDestination | GoRoute tree |
| 傳參方式 | Type-safe route / Safe Args | NavigationLink value | pathParameters / queryParameters |
| Deep Link 支援 | NavGraph 內建 | 需另外設定（SceneDelegate） | GoRouter 內建 |
| 巢狀導航 | nested NavGraph | NavigationSplitView | GoRouter ShellRoute |

---

## 四、登入機制

### 登入方式分類

**1. 帳號密碼登入**
- 最傳統的方式，使用者輸入帳號（Email/手機）與密碼
- 後端驗證後回傳 Token（JWT / Session）
- 三平台流程一致：輸入 → API 呼叫 → 儲存 Token → 進入主頁

**2. 第三方社群登入（OAuth 2.0）**

| 登入方式 | Android | iOS | Flutter |
|----------|---------|-----|---------|
| Google 登入 | Google Sign-In SDK / Credential Manager | Google Sign-In SDK | google_sign_in |
| Apple 登入 | 需自行整合（Web-based） | AuthenticationServices（原生） | sign_in_with_apple |
| Facebook 登入 | Facebook Login SDK | Facebook Login SDK | flutter_facebook_auth |
| LINE 登入 | LINE SDK | LINE SDK | flutter_line_sdk |

**3. Firebase Authentication**
- Google 提供的統一身份驗證服務，封裝多種登入方式（Email、Google、Apple、Facebook、手機 OTP）
- 三平台都有官方 SDK，簡化第三方登入整合
- 提供統一的 `FirebaseUser` 物件，不用各自處理不同平台的回傳格式

### 第三方登入流程（以 Google 登入為例）

```
1. 使用者點擊「Google 登入」按鈕
2. 呼叫 Google Sign-In SDK，跳轉 Google 授權頁面
3. 使用者選擇帳號並授權
4. SDK 回傳 Google ID Token
5. 將 Google ID Token 傳送給自家後端
6. 後端用 Google ID Token 向 Google 驗證身份
7. 後端回傳自家的 Access Token + Refresh Token
8. App 儲存 Token（安全儲存）並進入主頁
```

### Token 管理機制

**Access Token + Refresh Token 雙 Token 機制**
- **Access Token**：短效（通常 15 分鐘～1 小時），用於 API 請求的身份驗證
- **Refresh Token**：長效（通常 7～30 天），用於換取新的 Access Token
- 當 Access Token 過期時，自動用 Refresh Token 換新，使用者無感

**Token 刷新流程**

```
1. API 請求帶上 Access Token
2. 後端回傳 401（Token 過期）
3. 攔截器攔截 401，用 Refresh Token 呼叫刷新 API
4. 取得新的 Access Token，重試原請求
5. 若 Refresh Token 也過期 → 登出，導回登入頁
```

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| Token 刷新位置 | OkHttp Interceptor / Authenticator | URLSession delegate / Alamofire RequestInterceptor | dio Interceptor |
| Token 儲存 | EncryptedSharedPreferences | Keychain | flutter_secure_storage |
| 並行刷新處理 | Mutex / synchronized 確保只刷新一次 | Actor / NSLock | Completer 確保只刷新一次 |

### Apple 登入特殊規定

- **iOS App Store 規定**：若 App 提供任何第三方登入，**必須同時提供 Apple 登入**
- Apple 登入只在首次提供使用者 Email，之後只回傳 user identifier，需在首次登入時儲存
- 刪除帳號時需同時撤銷 Apple 的授權（Apple 審查要求）

### 注意事項與 Best Practice

- **Android**：Credential Manager 是 Google 推薦的新 API，整合密碼、Passkey、Google 登入於一處
- **iOS**：Apple 登入是審查硬性要求。注意 Firebase Auth 本地快取 — 刪帳後需呼叫 `Firebase.auth.signOut()` 清除 session
- **Flutter**：各登入 SDK 的原生設定（Google Services JSON/Plist、Facebook App ID）仍需手動處理
- **通用**：Token 一定要存在安全儲存中（不要用 SharedPreferences / UserDefaults），Refresh Token 刷新時要避免多個請求同時觸發重複刷新

---

## 五、應用內購買（IAP）

### 購買類型

| 類型 | 說明 | 範例 |
|------|------|------|
| **消耗型（Consumable）** | 購買後使用即消失，可重複購買 | 遊戲金幣、虛擬貨幣 |
| **非消耗型（Non-Consumable）** | 購買一次永久擁有 | 去廣告、解鎖進階功能 |
| **自動續訂訂閱（Auto-Renewable）** | 週期性扣款，到期自動續訂 | 月費 / 年費會員 |
| **非自動續訂訂閱** | 到期不自動續訂，需使用者手動續約 | 季節通行證 |

### IAP 購買流程

```
1. App 啟動時，向商店查詢可用的商品清單與價格（SKU / Product ID）
2. 使用者點擊購買按鈕
3. 呼叫平台 IAP SDK 發起購買
4. 平台彈出付款確認（指紋/Face ID/密碼）
5. 付款成功，平台回傳 Purchase Token / Receipt
6. App 將 Token/Receipt 傳送給自家後端
7. 後端向平台 Server API 驗證收據（Server-to-Server 驗證）
8. 驗證成功，後端發放權益（解鎖功能/加值貨幣）
9. App 呼叫「確認消費（Acknowledge / Finish Transaction）」告知平台已處理完畢
```

### 三平台 IAP 對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| IAP SDK | Google Play Billing Library 7+ | StoreKit 2（iOS 15+）/ StoreKit 1 | in_app_purchase（官方）/ purchases_flutter（RevenueCat） |
| 商品設定 | Google Play Console | App Store Connect | 各平台各自設定 |
| 收據驗證 | Google Play Developer API（Server） | App Store Server API / Server Notifications V2 | 後端各自呼叫對應平台 API |
| 訂閱管理 | Play Store 訂閱中心 | 設定 → 訂閱項目 | 各平台各自管理 |
| 退款處理 | Play Developer Notifications（RTDN） | App Store Server Notifications | 後端監聽各平台通知 |
| 抽成比例 | 15%（年營收 < 100 萬美元）/ 30% | 15%（年營收 < 100 萬美元）/ 30% | 依平台 |
| 沙盒測試 | License Testing（測試帳號） | Sandbox Environment | 各平台各自測試 |

### Server-to-Server 驗證（為何不能只在 App 端驗證）

- **安全性**：App 端可以被竄改，偽造購買結果。後端直接跟 Google/Apple Server 確認才可靠
- **防重複兌換**：後端記錄已驗證的 Purchase Token，避免同一筆交易被重複兌換
- **訂閱狀態同步**：訂閱的續訂、取消、退款都是平台 Server 主動通知後端，App 無法即時感知

### 訂閱生命週期

```
使用者訂閱 → 付款成功 → 權益啟用
    ↓
到期前 → 平台自動扣款 → 續訂成功 → 權益延續
    ↓
使用者取消 → 當期結束前仍可使用 → 到期後權益關閉
    ↓
使用者退款 → 平台通知後端 → 後端撤銷權益
```

### 注意事項與 Best Practice

- **Android**：購買後 3 天內未 Acknowledge 會被自動退款。Billing Library 7+ 大幅簡化 API，建議升級
- **iOS**：StoreKit 2 比 StoreKit 1 簡潔很多（async/await），但需要 iOS 15+。App Store Server Notifications V2 提供即時的訂閱狀態變更通知
- **Flutter**：`in_app_purchase` 是官方套件但偏底層，RevenueCat（`purchases_flutter`）封裝更完整（含後端驗證、分析、跨平台訂閱狀態同步）
- **通用**：永遠在後端驗證收據，不要信任 App 端的購買結果。訂閱類商品要處理取消、退款、升降級、寬限期等多種狀態

---

## 六、推播通知

**Android — FCM（Firebase Cloud Messaging）**
- 必須整合 Firebase，透過 `google-services.json` 設定
- **FCM Token**：每台裝置對每個 App 的唯一識別碼，需上傳至後端
- 通知類型：**Notification Message**（系統自動顯示）vs **Data Message**（App 自行處理）
- Android 8+ 必須建立 NotificationChannel，否則通知不顯示
- Android 13+ 需要 `POST_NOTIFICATIONS` 執行時權限

**iOS — APNs（Apple Push Notification service）**
- Apple 的推播基礎建設，所有 iOS 推播都經過 APNs
- 可搭配 FCM 使用（FCM 作為中間層，底層仍走 APNs）
- iOS 10+ 支援 **Rich Notification**（圖片、影片、自訂 UI）
- **Notification Service Extension**：在通知顯示前攔截修改內容
- 首次權限請求被拒絕後，只能引導使用者到設定中開啟

**Flutter — firebase_messaging**
- 統一封裝 FCM（Android）和 APNs（iOS）
- 背景處理需要 `@pragma('vm:entry-point')` 標記的**頂層函式**（不能是類別方法）
- 前景通知顯示需搭配 `flutter_local_notifications`

### 推播流程

```
1. App 啟動時請求推播權限
2. 取得 FCM Token / APNs Device Token
3. 將 Token 上傳至自家後端，與使用者帳號綁定
4. 後端需要推播時，將訊息傳送給 FCM / APNs Server
5. 平台 Server 將訊息推送到對應裝置
6. 裝置收到推播 → 前景：App 自行處理 / 背景：系統顯示通知
7. 使用者點擊通知 → App 解析 payload → 導航到對應頁面
```

### 三平台推播對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 推播服務 | FCM | APNs（可搭配 FCM） | FCM（firebase_messaging） |
| Token 類型 | FCM Token | APNs Device Token / FCM Token | FCM Token |
| 權限請求 | Android 13+ 需 runtime 權限 | 必須主動請求 | `requestPermission()` 統一處理 |
| 前景通知 | 需自行建立 Notification | `willPresent` 回傳展示選項 | 需搭配 flutter_local_notifications |
| 背景處理 | `FirebaseMessagingService` | Notification Service Extension | 頂層函式 + `@pragma` |
| 通知頻道 | Android 8+ 必須建立 Channel | 無此概念（用 Category 分類） | Android 端仍需設定 Channel |
| Rich Notification | 支援（大圖、自訂 layout） | Notification Content Extension | 需各平台原生實作 |
| 靜默推播 | Data Message | `content-available: 1` | Data-only message |

---

## 七、深度連結（Deep Link）

**Android — App Links**
- **Deep Link**：自訂 scheme（`myapp://path`），任何 App 都能註冊，可能衝突
- **App Links**（Android 6+）：使用 `https://` scheme + Digital Asset Links 驗證，確保只有你的 App 能處理
- 需在網站放置 `assetlinks.json`，在 `AndroidManifest.xml` 宣告 `<intent-filter>`

**iOS — Universal Links**
- **URL Scheme**：`myapp://path`，任何 App 都能註冊，舊做法
- **Universal Links**（iOS 9+）：使用 `https://` + AASA 檔案驗證
- 需在網站放置 `apple-app-site-association`，Xcode 加入 Associated Domains
- AASA 會被 Apple CDN 快取，更新後可能需等待刷新

**Flutter — Deep Linking**
- 底層依賴各平台原生設定（Android intent-filter / iOS Associated Domains）
- GoRouter 內建 Deep Link 支援，自動匹配路由
- 原生設定仍需手動處理，Flutter 只負責接收和路由

### Deep Link 啟動流程

```
1. 使用者點擊連結（瀏覽器/訊息/Email）
2. 系統判斷是否有 App 能處理此連結
3. 若已安裝且驗證通過 → 直接開啟 App
4. 若未安裝 → 開啟瀏覽器（Deferred Deep Link 需另外實作）
5. App 接收 URL，解析路徑與參數
6. 導航到對應頁面
```

### 三平台 Deep Link 對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 驗證型連結 | App Links（https + assetlinks.json） | Universal Links（https + AASA） | 依各平台原生設定 |
| 自訂 scheme | `myapp://`（intent-filter） | `myapp://`（URL Scheme） | 兩邊都要設定 |
| 驗證檔案位置 | `https://domain/.well-known/assetlinks.json` | `https://domain/.well-known/apple-app-site-association` | 兩邊都要放 |
| 處理入口 | `Activity.intent.data` | `SceneDelegate` / `.onOpenURL` | GoRouter 自動 / `app_links` |
| 未安裝時 | 開啟瀏覽器 | 開啟瀏覽器 | 開啟瀏覽器 |

---

## 八、WebRTC（即時音視訊通訊）

### 什麼是 WebRTC

WebRTC（Web Real-Time Communication）是一種**點對點（P2P）即時通訊協定**，支援音訊、視訊、資料傳輸，無需安裝插件。最初由 Google 開發，現為 W3C 標準。

### 核心概念

| 概念 | 說明 |
|------|------|
| **PeerConnection** | 兩端之間的連線物件，管理媒體串流與資料通道 |
| **SDP（Session Description Protocol）** | 描述媒體能力（編碼格式、解析度等），用於協商連線參數 |
| **ICE（Interactive Connectivity Establishment）** | 穿越 NAT/防火牆的機制，找到最佳連線路徑 |
| **STUN Server** | 幫助裝置發現自己的公網 IP（輕量，免費） |
| **TURN Server** | 當 P2P 無法直連時，作為中繼伺服器轉發資料（耗流量，需自建或付費） |
| **Signaling Server** | 用於交換 SDP 和 ICE Candidate 的信令伺服器（WebRTC 本身不定義信令方式，通常用 WebSocket） |
| **MediaStream** | 音訊/視訊軌道的容器，從麥克風/攝影機取得 |

### WebRTC 連線建立流程

```
1. 發起端（Caller）建立 PeerConnection
2. Caller 取得本地媒體串流（攝影機/麥克風）並加入 PeerConnection
3. Caller 建立 Offer（SDP），設為 Local Description
4. Caller 透過 Signaling Server（通常是 WebSocket）將 Offer 傳給接收端
5. 接收端（Callee）收到 Offer，設為 Remote Description
6. Callee 建立 Answer（SDP），設為 Local Description
7. Callee 透過 Signaling Server 將 Answer 傳回 Caller
8. Caller 收到 Answer，設為 Remote Description
9. 雙方持續透過 Signaling Server 交換 ICE Candidate（網路路徑資訊）
10. ICE 協商完成，P2P 連線建立，開始傳輸音視訊資料
```

### 三平台 WebRTC 對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 官方 SDK | Google WebRTC（libwebrtc） | Google WebRTC（libwebrtc） | flutter_webrtc |
| 封裝層 | 直接使用 libwebrtc 或 Location:Stream SDK | 直接使用 libwebrtc 或 Twilio/Vonage SDK | flutter_webrtc + 自訂 Signaling |
| 權限 | CAMERA + RECORD_AUDIO（runtime） | NSCameraUsageDescription + NSMicrophoneUsageDescription | 各平台原生權限 |
| 硬體加速 | 支援（MediaCodec） | 支援（VideoToolbox） | 底層依賴各平台 |
| 螢幕分享 | MediaProjection API | ReplayKit | screen_capturer plugin |
| 常用 SFU 伺服器 | Janus / mediasoup / LiveKit | Janus / mediasoup / LiveKit | LiveKit（有 Flutter SDK） |

### P2P vs SFU vs MCU

| 架構 | 說明 | 適用場景 |
|------|------|----------|
| **P2P** | 端對端直連，無伺服器中繼 | 1 對 1 通話 |
| **SFU（Selective Forwarding Unit）** | 伺服器轉發串流，不做編解碼 | 多人視訊（主流方案，如 LiveKit、mediasoup） |
| **MCU（Multipoint Control Unit）** | 伺服器混合所有串流為一路 | 大規模會議（伺服器負擔重，較少使用） |

### 注意事項與 Best Practice

- **Signaling 方式不限**：WebRTC 不規定信令協定，最常見搭配 WebSocket，也可用 HTTP Polling 或 Firebase Realtime Database
- **TURN Server 是必要的**：約 10-20% 的連線無法 P2P 直連（對稱 NAT、企業防火牆），沒有 TURN 會連線失敗
- **頻寬管理**：多人視訊時需動態調整解析度（Simulcast），避免頻寬不足
- **電量與發熱**：視訊通話非常耗電，需注意電池優化與散熱提示

---

## 九、WebSocket（即時雙向通訊）

### 什麼是 WebSocket

WebSocket 是一種在**單一 TCP 連線上進行全雙工通訊**的協定。相比 HTTP 的請求-回應模式，WebSocket 建立連線後雙方可隨時主動發送訊息，延遲更低。

### WebSocket vs HTTP 比較

| 面向 | HTTP | WebSocket |
|------|------|-----------|
| 通訊模式 | 請求-回應（單向發起） | 全雙工（雙向隨時發送） |
| 連線生命週期 | 每次請求建立新連線（HTTP/1.1 可 keep-alive） | 建立一次，持續保持 |
| 適用場景 | CRUD API、檔案下載 | 聊天室、即時通知、股票報價、遊戲 |
| 資料格式 | Header 較大（每次都帶） | 建立後 overhead 極小（2-14 bytes frame header） |
| 協定 | `http://` / `https://` | `ws://` / `wss://`（加密） |

### WebSocket 連線流程

```
1. Client 發送 HTTP Upgrade 請求（帶 Upgrade: websocket header）
2. Server 回應 101 Switching Protocols，連線升級為 WebSocket
3. 雙向通道建立，Client 和 Server 可隨時互發訊息
4. 任一方發送 Close Frame → 對方回應 Close Frame → 連線關閉
```

### 常見應用場景

| 場景 | 說明 | 搭配機制 |
|------|------|----------|
| **聊天室** | 即時收發訊息 | 心跳偵測 + 斷線重連 + 訊息佇列 |
| **即時通知** | 伺服器主動推送（比推播更即時） | 搭配推播作為離線備援 |
| **直播彈幕/互動** | 高頻小訊息推送 | 訊息壓縮 + 限流 |
| **股票/幣價報價** | 高頻資料串流 | 只傳差異資料（delta update） |
| **多人協作** | 文件同步編輯 | OT / CRDT 衝突解決演算法 |
| **遊戲** | 即時同步狀態 | UDP-like 優化（不可靠但快速） |

### 三平台 WebSocket 對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 原生 API | OkHttp WebSocket | URLSessionWebSocketTask（iOS 13+） | dart:io WebSocket |
| 主流套件 | OkHttp（最常用）/ Scarlet（自動重連） | Starscream / URLSession 原生 | web_socket_channel（官方）/ socket_io_client |
| 連線管理 | 手動管理（或 Scarlet 自動） | 手動管理 | 手動管理 |
| 心跳機制 | OkHttp 內建 ping interval | URLSession 內建 sendPing | 需手動實作 Timer |
| 資料格式 | 文字（JSON）/ 二進制（Protobuf） | 文字 / 二進制 | 文字 / 二進制 |
| 生命週期綁定 | 需綁定 Activity/ViewModel 生命週期 | 需綁定 ViewController 生命週期 | 需綁定 Widget 生命週期 |

### Socket.IO vs 原生 WebSocket

| 面向 | 原生 WebSocket | Socket.IO |
|------|---------------|-----------|
| 協定 | 標準 WebSocket 協定 | 自訂協定（底層可用 WebSocket 或 HTTP Long Polling） |
| 自動重連 | 不支援（需手動實作） | 內建自動重連 |
| 房間/命名空間 | 不支援（需自行實作） | 內建 Room / Namespace |
| 事件機制 | 無（只有 onMessage） | 內建事件名稱（`emit('chat', data)`） |
| 相容性 | 需要 Server 支援 WebSocket | 自動降級（WebSocket → HTTP Polling） |
| 適用場景 | 高效能、低延遲、自訂協定 | 快速開發、需要房間功能、相容性優先 |

### 斷線重連策略

```
1. 偵測連線斷開（onClose / onError）
2. 立即嘗試重連（第 1 次）
3. 失敗 → 等待 1 秒 → 重連（第 2 次）
4. 失敗 → 等待 2 秒 → 重連（第 3 次）
5. 失敗 → 等待 4 秒 → 重連（第 4 次）
6. ...指數退避（Exponential Backoff），最大間隔 30 秒
7. 重連成功 → 同步離線期間遺漏的訊息（用 lastMessageId 向 Server 查詢）
8. 超過最大重試次數 → 提示使用者網路異常
```

### 心跳偵測（Keep-Alive）

- **為何需要**：TCP 連線在閒置時可能被中間的 NAT/防火牆/Proxy 靜默斷開，雙方不會收到通知
- **機制**：定期發送 Ping Frame，Server 回應 Pong Frame。若超時未收到 Pong → 判定斷線 → 觸發重連
- **頻率**：通常 15-30 秒一次，取決於網路環境

### 注意事項與 Best Practice

- **斷線重連是必要的**：行動裝置頻繁切換網路（Wi-Fi ↔ 行動網路）、進入電梯/地下室，必須有重連機制
- **前背景處理**：App 進入背景時 WebSocket 可能被系統斷開（尤其 iOS）。進入背景時應主動斷開並在回到前景時重連，離線期間改用推播接收重要訊息
- **訊息順序與冪等**：網路不穩時訊息可能亂序或重複，需用 messageId 去重和排序
- **安全性**：務必使用 `wss://`（加密），不要用 `ws://`。Token 驗證在連線建立時的第一則訊息或 URL 參數中傳遞

---

## 十、圖片載入與快取

**Android — Coil / Glide**
- **Glide**：老牌主流，Google 官方推薦，基於 Java，生態成熟
- **Coil**（Coroutine Image Loader）：Kotlin-first，輕量，Compose 原生支援，已成為新專案首選
- 兩者都支援：記憶體快取 + 磁碟快取、圖片轉換（圓角、模糊）、GIF/WebP/SVG、生命週期感知

**iOS — Kingfisher / SDWebImage**
- **Kingfisher**：Swift 社群最主流，純 Swift，支援 SwiftUI
- **SDWebImage**：Objective-C 時代的老牌庫，仍在維護
- iOS 15+ 原生 `AsyncImage` 功能陽春（無快取控制、無轉換）

**Flutter — cached_network_image**
- **cached_network_image**：社群最主流，底層使用 `flutter_cache_manager` 管理磁碟快取
- Flutter 的 `Image.network()` 有基本記憶體快取但無磁碟快取，在 ListView 中會重複載入

### 三平台圖片載入對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 主流套件 | Coil（新專案）/ Glide（舊專案） | Kingfisher | cached_network_image |
| 記憶體快取 | 內建（LRU Cache） | 內建（NSCache） | 內建（ImageCache） |
| 磁碟快取 | 內建（DiskLruCache） | 內建 | flutter_cache_manager |
| Compose/SwiftUI 支援 | `AsyncImage`（Coil） | `KFImage` | `CachedNetworkImage` Widget |
| 圖片轉換 | `transformations`（圓角、模糊） | `processor`（圓角、模糊） | 需額外處理 |
| GIF 支援 | 內建 | 內建 | 需用其他方案 |
| SVG 支援 | Coil SVG decoder | SVGKit（第三方） | flutter_svg |
| 預載 | `ImageRequest.Builder.preload` | `KingfisherManager.prefetch` | `precacheImage()` |

---

## 十一、安全性

**Android — ProGuard/R8 / Keystore**
- **R8**（取代 ProGuard）：程式碼混淆 + 縮減 + 優化，Release build 預設啟用
- 混淆將類別/方法名改為 `a.b.c`；縮減（Tree Shaking）移除未使用程式碼
- 需撰寫 ProGuard rules 保留 Reflection/序列化用到的類別
- APK 很容易被反編譯（jadx），R8 只增加閱讀難度不是絕對安全
- **Network Security Config**：限制明文 HTTP、設定憑證 pinning

**iOS — ATS / Keychain**
- **ATS（App Transport Security）**：iOS 9+ 預設強制 HTTPS，關閉可能被 Apple 審查退回
- **Keychain**：系統層級加密的安全儲存，即使越獄也難以直接讀取
- iOS App 天生有程式碼保護 — App Store 的 Bitcode + FairPlay DRM 加密
- 反編譯難度高於 Android

**Flutter — 混淆 / 安全儲存**
- `flutter build --obfuscate --split-debug-info=...` 混淆 Dart 程式碼（只混淆 Dart 層）
- 原生層（Kotlin/Swift）需各自處理
- `flutter_secure_storage` 跨平台安全儲存
- 環境變數用 `--dart-define` 注入，不要寫死 API Key

### 三平台安全性對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 程式碼混淆 | R8（預設啟用於 release） | Bitcode + FairPlay DRM（自動） | `--obfuscate` flag |
| 安全儲存 | EncryptedSharedPreferences / KeyStore | Keychain | flutter_secure_storage |
| 網路安全 | Network Security Config | ATS | 各平台原生設定 |
| 憑證 Pinning | OkHttp CertificatePinner | URLSession delegate | dio_http2_adapter / 原生設定 |
| 簽章驗證 | APK Signature Scheme v2/v3 | Code Signing（強制） | 各平台各自處理 |
| Root/越獄偵測 | Play Integrity API | 手動偵測 / 第三方 | flutter_jailbreak_detection |
| 反編譯難度 | 低（APK 容易反編譯） | 高（iOS binary 較難分析） | 中（Dart AOT 增加難度） |

---

## 十二、效能優化

**Android — Baseline Profiles / Compose 優化**
- **Baseline Profiles**：預編譯關鍵路徑的機器碼，加速冷啟動 30-40%
- **Compose 優化**：穩定性標記（`@Stable` / `@Immutable`）避免不必要 recomposition；`remember` / `derivedStateOf` 快取運算；`LazyColumn` 正確設定 key
- **LeakCanary**：記憶體洩漏偵測
- **Android Profiler / Perfetto**：CPU、記憶體、網路、耗電分析

**iOS — Instruments / MetricKit**
- **Instruments**：iOS 最強大的效能分析工具（Time Profiler、Allocations、Core Animation）
- **MetricKit**（iOS 13+）：收集使用者裝置上的效能指標（啟動時間、hang 率）
- **SwiftUI 優化**：`@Observable`（iOS 17+）細粒度觀察比 `@ObservedObject` 更精確；`LazyVStack` 延遲載入

**Flutter — DevTools / Widget 重建優化**
- **Flutter DevTools**：官方效能分析工具（Performance、Memory、CPU Profiler）
- **Widget 重建優化**：`const` 建構子跳過重建（最簡單也最有效）；`RepaintBoundary` 隔離重繪範圍；拆分小 Widget 最小化 `setState` 影響
- **Impeller**（Flutter 3.16+）：新渲染引擎，減少 shader compilation jank
- Profile mode（`flutter run --profile`）才能準確測量效能

### 三平台效能優化對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 效能分析工具 | Android Profiler / Perfetto | Instruments | Flutter DevTools |
| 啟動優化 | Baseline Profiles | Pre-main optimization | Deferred Components |
| 列表優化 | `LazyColumn` + key | `LazyVStack` | `ListView.builder` + key |
| 避免不必要重建 | `@Stable`、`remember` | `@Observable`、`EquatableView` | `const`、`RepaintBoundary` |
| 記憶體洩漏偵測 | LeakCanary | Instruments Allocations | DevTools Memory tab |
| 使用者端效能數據 | Firebase Performance | MetricKit | Firebase Performance |

---

## 十三、測試

**Android — JUnit / Espresso / Compose Testing**
- **Unit Test**（JUnit + MockK）：測試 ViewModel、UseCase、Repository 等純邏輯，JVM 上執行速度快
- **UI Test**（Espresso）：測試 XML View 的 UI 互動，需要模擬器
- **Compose Testing**（`compose-ui-test`）：測試 Compose UI，比 Espresso 更直覺
- **Robolectric**：在 JVM 上模擬 Android 環境，不需模擬器

**iOS — XCTest / XCUITest**
- **XCTest**：Apple 官方測試框架，涵蓋 Unit Test 和 UI Test
- **Swift Testing**（Swift 5.10+）：新一代框架，語法更簡潔（`@Test`、`#expect` 取代 `XCTAssertEqual`）
- **XCUITest**：UI 測試，模擬使用者操作，速度慢，只測關鍵流程

**Flutter — Widget Test / Integration Test**
- **Unit Test**（`package:test`）：測試純 Dart 邏輯
- **Widget Test**：Flutter 的殺手級特色 — 不需要模擬器就能測 UI，速度接近 Unit Test
- **Integration Test**：在真實裝置上跑完整流程
- **Golden Test**：截圖對比測試，確保 UI 不意外改變

### 三平台測試對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| Unit Test 框架 | JUnit 4/5 + MockK | XCTest / Swift Testing | package:test + Mockito |
| UI Test 框架 | Espresso / Compose Testing | XCUITest | Widget Test |
| 整合測試 | Espresso + AndroidJUnitRunner | XCUITest | integration_test |
| Mock 框架 | MockK / Mockito-Kotlin | 手動 Mock / Mockingbird | package:mockito / mocktail |
| 截圖測試 | Paparazzi（Compose） | Swift Snapshot Testing | Golden Test |
| 測試速度（Unit） | 快（JVM） | 快 | 快（Dart VM） |
| 測試速度（UI） | 慢（需要模擬器） | 慢（需要模擬器） | 快（Widget Test 不需模擬器） |
| CI 友善度 | 高（Gradle task） | 中（需 macOS） | 高（flutter test） |

---

## 十四、CI/CD

**Android — Gradle / GitHub Actions**
- **Gradle**：建置系統，定義編譯、測試、打包、簽章的完整流程
- **GitHub Actions**：最普及的 CI/CD 平台，用 YAML 定義 workflow
- **Fastlane**：自動化工具，簡化上傳 Google Play 的流程
- Keystore 和 signing config 存在 CI/CD secrets，不提交到 Git

**iOS — Xcode Cloud / Fastlane**
- **Xcode Cloud**（2022）：Apple 原生 CI/CD，與 Xcode 深度整合，自動管理簽章
- **Fastlane**：社群最主流（match 管理憑證、gym 建置、deliver 上傳）
- iOS CI/CD 最大挑戰是簽章管理 — Fastlane match 用 Git repo 儲存憑證是業界最佳實踐
- GitHub Actions 需要 macOS runner，成本較高

**Flutter — Codemagic / GitHub Actions**
- **Codemagic**：Flutter 專屬 CI/CD，內建 Flutter 環境、自動管理 iOS 簽章
- GitHub Actions 用 `subosito/flutter-action` 設定環境
- 兩平台建置需在一個 workflow 中處理，iOS 建置必須在 macOS runner

### 三平台 CI/CD 對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 官方方案 | 無（Gradle 是建置工具） | Xcode Cloud | 無 |
| 主流 CI/CD | GitHub Actions | Fastlane + GitHub Actions | Codemagic / GitHub Actions |
| Runner 需求 | Linux / macOS | macOS（必須） | Linux（Android）+ macOS（iOS） |
| 建置指令 | `./gradlew bundleRelease` | `xcodebuild` / `fastlane gym` | `flutter build appbundle/ipa` |
| 簽章管理 | Keystore + secrets | Fastlane match / Xcode Cloud 自動 | 各平台各自處理 |
| 上傳商店 | Fastlane supply / Google Play API | Fastlane deliver / pilot | Codemagic 內建 / Fastlane |
| 費用 | GitHub Actions 免費額度充足 | macOS runner 較貴 | Codemagic 有免費額度 |

---

## 十五、模組化架構

**Android — Multi-module**
- Gradle 原生支援多模組，用 `include` 和 `implementation project(":module")` 定義依賴
- 常見分法：按層分（`:app` / `:data` / `:domain`）或按功能分（`:feature:home` / `:core:network`）
- **Convention Plugins**：統一多模組的 Gradle 設定
- 模組未改動時不需重編譯（增量編譯），大型專案效果顯著
- 模組間依賴必須單向（`feature → domain → core`），禁止循環依賴

**iOS — SPM Modules / Framework**
- **SPM（Swift Package Manager）**：Apple 官方模組化方案，用 `Package.swift` 定義
- **Tuist**：社群工具，用 Swift DSL 定義專案結構，解決 `.xcodeproj` 衝突問題
- 動態 Framework 減少 binary 大小但增加啟動時間，靜態反之
- SPM 與 Xcode 的整合仍有 rough edges（如 Resources bundle 處理）

**Flutter — Packages / Plugins**
- **Package**：純 Dart 模組；**Plugin**：包含原生程式碼的模組
- **Melos**：管理多 package 的 Monorepo 工具（版本管理、指令廣播、依賴管理）
- Dart 的存取控制依賴命名慣例（`src/` 下的檔案不應被外部引入），不像 Kotlin/Swift 有 `internal`

### 三平台模組化對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 官方方案 | Gradle Multi-module | SPM | Dart Package / Plugin |
| 模組定義 | `build.gradle.kts` | `Package.swift` | `pubspec.yaml` |
| 依賴管理 | `implementation project(":module")` | Xcode Add Local Package | `path: packages/module` |
| Monorepo 工具 | Convention Plugins | Tuist | Melos |
| 增量編譯 | 支援（模組級別） | 支援（SPM target 級別） | 部分支援 |
| 存取控制 | `internal`（模組內可見，預設） | `internal`（預設） | `src/` 資料夾（慣例私有） |
| 動態載入 | Dynamic Feature Module | 不支援 | Deferred Components（實驗性） |
| DI 跨模組 | Hilt `@InstallIn` | 手動注入 / Swinject | get_it 全域 / Riverpod |

---

## 十六、總結對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 非同步 | Coroutines + Flow | async/await + Actor | async/await + Isolate |
| 本地儲存（輕量） | DataStore | UserDefaults | shared_preferences |
| 本地儲存（資料庫） | Room | SwiftData / CoreData | sqflite / Hive |
| 導航 | Compose Navigation | NavigationStack | GoRouter |
| 登入 | Credential Manager + Firebase Auth | AuthenticationServices + Firebase Auth | google_sign_in + firebase_auth |
| IAP | Google Play Billing Library | StoreKit 2 | in_app_purchase / RevenueCat |
| 推播 | FCM | APNs（+ FCM） | firebase_messaging |
| Deep Link | App Links | Universal Links | GoRouter + 原生設定 |
| WebRTC | libwebrtc | libwebrtc | flutter_webrtc |
| WebSocket | OkHttp WebSocket | URLSessionWebSocketTask | web_socket_channel |
| 圖片載入 | Coil / Glide | Kingfisher | cached_network_image |
| 程式碼保護 | R8 混淆 | Bitcode + FairPlay | `--obfuscate` |
| 安全儲存 | EncryptedSharedPreferences | Keychain | flutter_secure_storage |
| 效能工具 | Android Profiler | Instruments | DevTools |
| Unit Test | JUnit + MockK | XCTest / Swift Testing | package:test |
| UI Test | Compose Testing / Espresso | XCUITest | Widget Test |
| CI/CD | GitHub Actions + Gradle | Fastlane / Xcode Cloud | Codemagic / GitHub Actions |
| 模組化 | Gradle Multi-module | SPM | Dart Package + Melos |
