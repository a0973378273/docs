# App 基礎知識 — iOS / Android / Flutter 三平台對照

## 一、畫面單位

**Android — Activity / Fragment**
- **Activity**：一個獨立的畫面，擁有完整生命週期。App 至少有一個 Activity
- **Fragment**：依附在 Activity 內的子畫面，可重用、可組合。常見用法是一個 Activity 搭配多個 Fragment 切換（Single Activity 架構）
- **Jetpack Compose**：宣告式 UI 框架，用 Kotlin 函式描述畫面，取代傳統 XML layout

**iOS — UIViewController / SwiftUI View**
- **UIViewController**：等同 Android 的 Activity，管理一個畫面的生命週期與內容
- **UINavigationController**：管理畫面堆疊的容器，提供 push/pop 導航
- **SwiftUI View**：宣告式 UI（等同 Compose），用 struct 描述畫面

**Flutter — Widget**
- **Widget**：Flutter 的一切皆 Widget，包含畫面、佈局、樣式、手勢等
- **StatelessWidget**：無狀態，純展示用（像一個固定的 Text）
- **StatefulWidget**：有狀態，可以因互動改變內容（像一個計數器）

---

## 二、生命週期（三平台交叉對應）

### Android Activity 生命週期

```
onCreate → onStart → onResume → [使用中] → onPause → onStop → onDestroy
                ↑                                        |
                └────────── onRestart ───────────────────┘
```

| 狀態 | 說明 | 對應 iOS | 對應 Flutter |
|------|------|----------|-------------|
| `onCreate` | Activity 被建立，初始化 UI、綁定資料。整個生命週期只呼叫一次 | `viewDidLoad` | `initState` |
| `onStart` | Activity 變為可見，但尚未取得焦點（使用者還不能互動） | `viewWillAppear` | 無直接對應 |
| `onResume` | Activity 取得焦點，使用者可以互動。此時為「前景」狀態 | `viewDidAppear` | `AppLifecycleState.resumed` |
| `onPause` | 失去焦點但仍可見（如彈出半透明對話框、多視窗模式下另一個 App 取得焦點） | `viewWillDisappear` | `AppLifecycleState.inactive` |
| `onStop` | 完全不可見（按 Home 鍵、切到另一個 Activity）。可能被系統回收 | `viewDidDisappear` | `AppLifecycleState.paused` |
| `onRestart` | 從 onStop 回到前景時觸發，接著進入 onStart | 無對應（直接走 `viewWillAppear`） | 無對應（直接走 `resumed`） |
| `onDestroy` | Activity 被銷毀（使用者按返回鍵、系統回收記憶體、呼叫 finish()）。釋放所有資源 | `deinit` | `dispose` |

### Android Fragment 生命週期

Fragment 依附在 Activity 內，因此比 Activity 多了與宿主 Activity 的綁定/解綁步驟。

```
onAttach → onCreate → onCreateView → onViewCreated → onStart → onResume
    → [使用中] →
onPause → onStop → onDestroyView → onDestroy → onDetach
```

| 狀態 | 說明 | 對應 iOS | 對應 Flutter |
|------|------|----------|-------------|
| `onAttach` | Fragment 與宿主 Activity 建立關聯，可取得 Context | 無對應（iOS ViewController 不需顯式綁定父容器） | 無對應 |
| `onCreate` | Fragment 物件初始化，還沒有建立 View。適合接收傳入參數（arguments） | 無對應（iOS 在 `init` 階段處理） | `createState`（概念上） |
| `onCreateView` | 建立 Fragment 的 View 階層（inflate layout XML 或 Compose） | `loadView` | 無對應（Flutter 在 `build` 統一處理） |
| `onViewCreated` | View 已建立完成，適合初始化 UI 元件、設定 Listener、觀察 ViewModel | `viewDidLoad` | `initState` |
| `onStart` | Fragment 可見但尚未可互動 | `viewWillAppear` | 無直接對應 |
| `onResume` | Fragment 可見且可互動 | `viewDidAppear` | `AppLifecycleState.resumed` |
| `onPause` | Fragment 失去焦點（被另一個 Fragment 覆蓋、對話框彈出） | `viewWillDisappear` | `AppLifecycleState.inactive` |
| `onStop` | Fragment 完全不可見 | `viewDidDisappear` | `AppLifecycleState.paused` |
| `onDestroyView` | View 被銷毀，但 Fragment 物件還在（如加入 BackStack 後被替換） | 無直接對應（iOS ViewController 的 view 可被卸載但現代開發中很少發生） | `deactivate`（概念上） |
| `onDestroy` | Fragment 物件被銷毀 | `deinit` | `dispose` |
| `onDetach` | Fragment 與宿主 Activity 解除關聯 | 無對應 | 無對應 |

**Fragment 特有的重要觀念：**
- `onDestroyView` ≠ `onDestroy`：Fragment 加入 BackStack 後，返回時只銷毀 View（`onDestroyView`），Fragment 物件仍存活。再次顯示時從 `onCreateView` 重新開始，不會再走 `onAttach` / `onCreate`
- `viewLifecycleOwner`：Fragment 有兩個生命週期 — Fragment 本身的和 View 的。觀察 LiveData/Flow 時應使用 `viewLifecycleOwner` 而非 `this`，避免 View 銷毀後仍收到更新

### iOS UIViewController 生命週期

```
init → loadView → viewDidLoad → viewWillAppear → viewDidAppear
    → [使用中] →
viewWillDisappear → viewDidDisappear → deinit
```

| 狀態 | 說明 | 對應 Android Activity | 對應 Android Fragment | 對應 Flutter |
|------|------|----------------------|----------------------|-------------|
| `init` | ViewController 物件被建立，分配記憶體 | 物件建構子 | 物件建構子 | `createState` |
| `loadView` | 手動建立 view 階層（不用 Storyboard 時才需覆寫） | `setContentView()` | `onCreateView` | 無對應 |
| `viewDidLoad` | View 載入完成，初始化 UI、綁定資料。整個生命週期只呼叫一次 | `onCreate` | `onViewCreated` | `initState` |
| `viewWillAppear` | View 即將顯示在螢幕上，可在此更新資料或 UI 狀態。每次出現都會呼叫 | `onStart` | `onStart` | 無直接對應 |
| `viewDidAppear` | View 已完全顯示，使用者可以互動。適合啟動動畫、開始計時器 | `onResume` | `onResume` | `AppLifecycleState.resumed` |
| `viewWillDisappear` | View 即將被遮蓋或移除，適合暫停動畫、儲存暫存資料 | `onPause` | `onPause` | `AppLifecycleState.inactive` |
| `viewDidDisappear` | View 已完全不可見 | `onStop` | `onStop` | `AppLifecycleState.paused` |
| `deinit` | ViewController 被釋放，釋放所有資源 | `onDestroy` | `onDestroy` + `onDetach` | `dispose` |

**iOS 特有的重要觀念：**
- iOS 沒有 `onRestart` 的概念，畫面從不可見回到可見直接走 `viewWillAppear → viewDidAppear`
- iOS NavigationController 的 push/pop 行為：push 新頁時舊頁走到 `viewDidDisappear`，pop 返回時舊頁從 `viewWillAppear` 重新開始（不會再走 `viewDidLoad`）。這與 Fragment BackStack 的行為不同 — Fragment 會重走 `onCreateView`

### Flutter StatefulWidget 生命週期

```
createState → initState → didChangeDependencies → build
    → [使用中] →
deactivate → dispose
```

| 狀態 | 說明 | 對應 Android Activity | 對應 Android Fragment | 對應 iOS |
|------|------|----------------------|----------------------|----------|
| `createState` | 建立 State 物件，由 Framework 自動呼叫 | 物件建構子 | `onAttach` + `onCreate` | `init` |
| `initState` | State 初始化，只呼叫一次。適合初始化控制器、訂閱串流 | `onCreate` | `onViewCreated` | `viewDidLoad` |
| `didChangeDependencies` | 依賴的 InheritedWidget 改變時觸發（如 Theme、Locale 改變）。首次 `initState` 之後也會呼叫一次 | 無直接對應（Configuration change 會重建 Activity） | 無直接對應 | 無直接對應 |
| `build` | 建構 Widget Tree，每次狀態改變都會重新呼叫。Flutter 的宣告式核心 | 無對應（Android 不會重複呼叫 `onCreate`） | 無對應 | 無對應（SwiftUI 的 `body` 類似） |
| `didUpdateWidget` | 父 Widget 重建導致此 Widget 的設定改變時觸發 | 無對應 | 無對應 | 無對應 |
| `deactivate` | Widget 從 Widget Tree 中移除（但可能被重新插入，如 GlobalKey 轉移） | `onStop` | `onDestroyView` | `viewDidDisappear` |
| `dispose` | State 被永久銷毀，釋放資源（取消訂閱、關閉控制器） | `onDestroy` | `onDestroy` + `onDetach` | `deinit` |

**Flutter App 層級生命週期**（透過 `WidgetsBindingObserver.didChangeAppLifecycleState`）：

| AppLifecycleState | 說明 | 對應 Android Activity | 對應 iOS UIApplication |
|-------------------|------|----------------------|----------------------|
| `resumed` | App 在前景且可互動 | `onResume` | `applicationDidBecomeActive` |
| `inactive` | App 可見但不可互動（如來電覆蓋、控制中心拉出） | `onPause` | `applicationWillResignActive` |
| `paused` | App 完全不可見（進入背景） | `onStop` | `applicationDidEnterBackground` |
| `detached` | App 仍在執行但沒有任何 View（即將被銷毀） | `onDestroy` | `applicationWillTerminate` |

**Flutter 特有的重要觀念：**
- Widget 層級的生命週期（`initState` → `dispose`）只管 Widget 自身的建立與銷毀，不感知 App 前背景切換
- 要監聽 App 前背景切換，必須另外 mixin `WidgetsBindingObserver` 並監聽 `AppLifecycleState`
- `build` 會被高頻呼叫（任何 `setState` 都觸發），這是 Flutter 與 Android/iOS 最大的差異 — Android/iOS 的初始化方法只呼叫一次，UI 更新透過命令式操作（`setText()`、`.text = ""`），Flutter 則是整個 Widget Tree 重新宣告

### 三平台生命週期關鍵差異總結

| 差異點 | Android | iOS | Flutter |
|--------|---------|-----|---------|
| View 與物件生命週期分離 | Fragment 有（`onDestroyView` vs `onDestroy`） | 無（view 與 ViewController 同生共死） | 無（Widget 與 State 綁定，但 State 可被 GlobalKey 保留） |
| 返回前頁時重建 View | Fragment BackStack：重走 `onCreateView` | 不重建，從 `viewWillAppear` 恢復 | 取決於路由實作（預設 pop 會 dispose） |
| App 層級 vs 頁面層級 | Activity/Fragment 生命週期同時反映兩者 | ViewController 只管頁面；App 層級另有 `UIApplicationDelegate` | 嚴格分離：Widget lifecycle 管頁面，`AppLifecycleState` 管 App |
| 組態變更（旋轉螢幕） | 預設銷毀重建整個 Activity | 不銷毀，觸發 `viewWillTransition` | 不銷毀，觸發 `didChangeMetrics` |

---

## 三、狀態管理

| 平台 | 方案 | 概念 |
|------|------|------|
| Android | ViewModel + StateFlow | ViewModel 存活於組態變更（如旋轉螢幕），StateFlow 推送狀態給 UI |
| iOS | @Observable + SwiftUI | 標記 @Observable 的物件變更時自動刷新 View |
| Flutter | BLoC / Riverpod / Provider | BLoC 用事件驅動狀態，Provider/Riverpod 用依賴注入式管理 |

三者共通概念：UI 層只負責「顯示」與「發送事件」，狀態邏輯放在獨立的層（ViewModel / BLoC / Store），實現單向資料流。

---

## 四、依賴注入（DI）

### 主流方案比較

| 面向 | Android — Hilt | iOS — Swinject / 手動注入 | Flutter — get_it / Riverpod |
|------|---------------|--------------------------|----------------------------|
| 類型 | 編譯時期 DI（Annotation Processor） | 執行時期 DI / 手動注入 | 執行時期 DI / 編譯時期檢查（Riverpod） |
| 設定方式 | 用 `@Inject`、`@Module`、`@Provides` 等 annotation 宣告 | Swinject 用 container 註冊；手動注入則直接在 init 傳入 | get_it 用全域 container 註冊；Riverpod 用 Provider 宣告 |
| 學習曲線 | 高（需理解 Hilt 的 Component 階層、Scope、Annotation） | 低（手動注入無框架）/ 中（Swinject） | 中（get_it 簡單）/ 中高（Riverpod 概念較多） |
| 編譯時安全 | 高 — 缺少依賴在編譯時就會報錯 | 低 — 手動注入靠人；Swinject 執行時才知道有沒有註冊 | get_it 低（執行時才報錯）；Riverpod 高（編譯時檢查） |
| 效能 | 好 — 編譯時生成程式碼，無反射開銷 | 好 — 手動注入零開銷；Swinject 有少量執行時開銷 | 好 — 兩者都是輕量 |
| 生命週期管理 | 自動管理（`@Singleton`、`@ActivityScoped` 等，跟 Android 元件綁定） | 需手動管理或靠 Swinject 的 scope 設定 | get_it 支援 Singleton/Factory；Riverpod 自動 dispose |
| 測試便利性 | 好 — 提供 `@TestInstallIn` 替換依賴 | 好 — 手動注入天生好測試（直接傳 mock） | 好 — 都能輕鬆替換為 mock |

### 優缺點摘要

**Android Hilt**
- 優點：Google 官方推薦、編譯時安全、與 Android 生命週期深度整合
- 缺點：學習成本高、annotation 多、編譯時間增加、錯誤訊息難讀

**iOS 手動注入 / Swinject**
- 優點：手動注入簡單直觀零依賴；iOS 社群主流偏好簡潔
- 缺點：手動注入在大型專案中 init 參數會爆炸；Swinject 缺乏編譯時安全

**Flutter get_it / Riverpod**
- 優點：get_it 極簡上手快；Riverpod 結合 DI + 狀態管理，功能強大
- 缺點：get_it 是 Service Locator 模式（全域存取，測試時需注意清理）；Riverpod 學習曲線陡峭

---

## 五、網路請求

### 主流方案比較

| 面向 | Android — Retrofit + OkHttp | iOS — URLSession / Alamofire | Flutter — dio / http |
|------|---------------------------|------------------------------|---------------------|
| 定位 | Retrofit 是型別安全的 API 客戶端，OkHttp 是底層 HTTP 引擎 | URLSession 是系統內建；Alamofire 是社群封裝 | http 是官方基礎套件；dio 是功能完整的社群套件 |
| API 定義方式 | 用 interface + annotation 宣告（`@GET`、`@POST`），編譯時生成實作 | URLSession 手動建立 Request；Alamofire 用鏈式呼叫 | 手動呼叫方法（`dio.get()`、`dio.post()`） |
| 序列化 | 搭配 Gson / Moshi / Kotlinx Serialization 自動轉 DTO | `Codable` 協定（Swift 內建），自動 JSON ↔ Struct | 手動 `jsonDecode` 或搭配 `json_serializable` 程式碼生成 |
| 攔截器 | OkHttp Interceptor — 強大且成熟（加 Header、Log、重試、Token 刷新） | URLSession 無原生攔截器；Alamofire 有 `RequestInterceptor` | dio 有完整的 Interceptor 機制（類似 OkHttp） |
| 非同步模型 | Kotlin Coroutines（`suspend fun`） | Swift async/await 或 Combine | Dart Future / async-await |
| 快取 | OkHttp 內建 HTTP 快取 | URLSession 內建 URLCache | 需手動實作或用 dio_cache_interceptor |
| 檔案上傳 | `@Multipart` + `@Part` annotation | Alamofire `upload(multipartFormData:)` | `dio.post()` + `FormData` |

### 優缺點摘要

**Android Retrofit + OkHttp**
- 優點：型別安全、自動序列化、攔截器生態成熟、業界標準
- 缺點：兩層抽象（Retrofit + OkHttp）學習成本較高、annotation 多

**iOS URLSession / Alamofire**
- 優點：URLSession 零依賴系統內建；Alamofire 語法簡潔、社群活躍
- 缺點：URLSession 原生寫法冗長（需手動處理 Request 建構、錯誤處理）；Alamofire 對比 Retrofit 缺少型別安全的 API 定義

**Flutter dio / http**
- 優點：http 輕量適合簡單場景；dio 功能完整（攔截器、取消請求、進度監聽），API 風格直覺
- 缺點：http 功能陽春（無攔截器、無檔案上傳便利 API）；dio 是第三方套件，需注意維護狀態；序列化不如 Retrofit 自動化

---

## 六、版本管控差異

### 版本號定義

**Android**（`build.gradle`）
- `versionCode`：整數，每次上傳必須嚴格遞增，不能重複不能回退，Google Play 用來判斷新舊
- `versionName`：給使用者看的版本字串，格式自訂

**iOS**（`Info.plist` 或 Xcode 設定）
- `CFBundleShortVersionString`：等同 versionName，顯示在 App Store。必須符合 `X.Y.Z` 格式（Apple 強制）
- `CFBundleVersion`：等同 versionCode（Build number）。同版本內可重複上傳不同 build（如 2.1.0 build 1, build 2）

**Flutter**（`pubspec.yaml`）
- 格式為 `version: 2.1.0+42`，`+` 前面是 versionName，後面是 versionCode
- 建置時自動對應到 Android 的 `versionCode`/`versionName` 和 iOS 的 `CFBundleVersion`/`CFBundleShortVersionString`

### 版本策略差異

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 版本號強制格式 | 無（versionName 任意） | 必須 X.Y.Z | 繼承兩平台規則 |
| Build 號重複 | 不允許 | 同版本內允許 | 依平台 |
| 多 Build Variant | `buildTypes` + `productFlavors` | Scheme + Configuration | `--dart-define` + flavor |
| 簽章 | Keystore（.jks/.keystore） | Provisioning Profile + Certificate | 各平台各自簽章 |

---

## 七、發布流程差異

### Android 發布流程

```
程式碼 → Build AAB → 簽章（Keystore）→ Google Play Console 上傳
                                            ├── 內部測試（Internal Testing）
                                            ├── 封閉測試（Closed Testing）
                                            ├── 開放測試（Open Testing）
                                            └── 正式發布（Production）
```

- **AAB**（Android App Bundle）：Google Play 要求的格式，由 Google 依據裝置動態產生最佳化 APK
- **APK**：傳統格式，用於側載或第三方商店
- **審查**：首次上架需審查（約數小時至數天），後續更新通常數小時內
- **階段性發布**：可設定 10%、50%、100% 逐步推送
- **Firebase App Distribution**：內部測試用，不經 Google Play

### iOS 發布流程

```
程式碼 → Archive → 上傳到 App Store Connect
                        ├── TestFlight 內部測試（最多 100 人，免審查）
                        ├── TestFlight 外部測試（最多 10,000 人，需審查）
                        └── App Store 正式發布（需審查）
```

- **Archive**：Xcode 編譯出最終產物（.ipa）
- **Provisioning Profile**：Apple 特有的簽章機制，綁定 App ID + 裝置 + 憑證
- **審查**：每次提交都需 Apple 審查（通常 24-48 小時，可能被退回）
- **TestFlight**：Apple 官方內測平台，比 Android 的 Internal Testing 更成熟
- **階段性發布**：支援 7 天自動逐步推送

### Flutter 發布流程

```
程式碼 → flutter build appbundle（Android）→ 走 Android 流程
     → flutter build ipa（iOS）          → 走 iOS 流程
```

Flutter 沒有自己的發布管道，最終產物分別走各平台原生流程。

### 三平台審查差異

| 面向 | Android | iOS |
|------|---------|-----|
| 審查嚴格度 | 較寬鬆 | 嚴格（UI 規範、隱私政策、功能完整性） |
| 審查速度 | 快（數小時） | 較慢（24-48 小時） |
| 退回頻率 | 低 | 較高（常見原因：UI 不符 HIG、缺隱私說明） |
| 熱更新 | 允許（WebView、動態載入） | 限制嚴格（不可改變 App 主要功能） |

---

## 八、跨平台溝通機制

### Flutter ↔ 原生（Platform Channel）

Flutter 提供三種 Channel 與原生層通訊：

**1. MethodChannel — 一次性呼叫（最常用）**

像 HTTP 請求，一問一答。Flutter 端呼叫 `invokeMethod`，原生端收到後回傳結果。

```
Flutter (Dart)  ──── MethodChannel ────  Native (Kotlin/Swift)
   invokeMethod('getBattery')  →→→  收到呼叫，回傳電量
   ←←←  回傳 result: 87
```

**2. EventChannel — 持續串流**

像 WebSocket，原生端持續推送事件給 Flutter。適合感應器資料、位置更新等持續性資料。

```
Native  ══════ EventChannel ══════►  Flutter
  傳感器資料持續推送  →→→  Stream 接收
```

**3. BasicMessageChannel — 雙向自由通訊**

不限格式的雙向訊息傳遞，較少使用。

```
Flutter  ←←←→→→  BasicMessageChannel  ←←←→→→  Native
       雙向傳送任意格式訊息
```

### WebView ↔ 原生（JavaScript Bridge）

Web 內容透過 JavaScript Bridge 與原生層溝通，這是 Ekkorn 專案的核心模式。

```
WebView (JavaScript)  ──── JS Bridge ────  Android (Kotlin)
   AndroidShow.openCamera()  →→→  原生開啟相機
   window.onCameraResult(url) ←←←  回傳結果給 Web
```

| 方向 | Android | iOS |
|------|---------|-----|
| Web 呼叫原生 | `@JavascriptInterface` 標註的方法 | `WKScriptMessageHandler` |
| 原生呼叫 Web | `webView.evaluateJavascript()` | `webView.evaluateJavaScript()` |
| 資料格式 | 字串（通常用 JSON） | 字串或字典 |

### 跨平台溝通對照表

| 場景 | Flutter | Android WebView | iOS WKWebView |
|------|---------|-----------------|---------------|
| 原生呼叫 Web | `webViewController.runJavaScript()` | `webView.evaluateJavascript()` | `webView.evaluateJavaScript()` |
| Web 呼叫原生 | `JavaScriptChannel` | `@JavascriptInterface` | `WKScriptMessageHandler` |
| 原生對原生 | MethodChannel / EventChannel | 不適用（同平台） | 不適用（同平台） |
| 資料格式 | 自動序列化基本型別 | 字串（通常用 JSON） | 字串或字典 |

---

## 九、總結對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 語言 | Kotlin | Swift | Dart |
| UI 框架 | Compose / XML | SwiftUI / UIKit | Widget |
| 建置系統 | Gradle | Xcode + SPM | Flutter CLI |
| 簽章 | Keystore | Provisioning Profile | 各平台各自 |
| 發布產物 | AAB | IPA | AAB + IPA |
| 審查速度 | 快 | 慢 | 取決於平台 |
| 跨平台通訊 | JS Bridge（WebView） | WKScriptMessageHandler | Platform Channel |
