# App 進階知識 — iOS / Android / Flutter 三平台對照

## 一、執行緒與非同步處理

**Android — Coroutines**
- Kotlin Coroutines 是 Android 官方推薦的非同步方案，基於**結構化並行（Structured Concurrency）**設計
- 核心概念：`suspend fun`（掛起函式）、`CoroutineScope`（作用域）、`Dispatcher`（指定執行緒）
- `Dispatchers.Main`：主執行緒（UI）；`Dispatchers.IO`：I/O 操作；`Dispatchers.Default`：CPU 密集運算
- 搭配 `viewModelScope`、`lifecycleScope` 自動在元件銷毀時取消

```kotlin
// ViewModel 中使用
viewModelScope.launch {
    val user = withContext(Dispatchers.IO) {
        userRepository.fetchUser() // 在 IO 執行緒執行
    }
    _uiState.value = UiState.Success(user) // 自動回到主執行緒
}

// Flow — 冷流，類似 RxJava 的 Observable
userRepository.observeUser()
    .flowOn(Dispatchers.IO)
    .collect { user -> _uiState.value = user }
```

**iOS — async/await + GCD**
- Swift 5.5 引入 `async/await`，是現代 iOS 的首選非同步方案
- **GCD（Grand Central Dispatch）**：底層執行緒管理，用 `DispatchQueue` 派發任務
- **Actor**：Swift 的並行安全機制，保證同一時間只有一個任務存取 Actor 的狀態
- `@MainActor`：標記必須在主執行緒執行的程式碼

```swift
// async/await
func fetchUser() async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// Task — 啟動非同步任務
Task {
    let user = try await fetchUser()
    self.user = user // @MainActor 自動在主執行緒更新
}

// GCD（舊寫法，仍常見於舊專案）
DispatchQueue.global(qos: .background).async {
    let data = self.loadData()
    DispatchQueue.main.async {
        self.updateUI(data)
    }
}
```

**Flutter — Isolate + async/await**
- Dart 是**單執行緒**模型，透過 Event Loop 處理非同步（類似 JavaScript）
- `async/await` + `Future`：處理非同步操作（網路、檔案），不會阻塞 UI
- `Stream`：處理持續性資料（類似 Kotlin Flow / Swift AsyncSequence）
- `Isolate`：Dart 的獨立執行緒，擁有自己的記憶體空間，適合 CPU 密集運算（JSON 解析大檔案、圖片處理）
- `compute()`：簡化版 Isolate，一次性任務用

```dart
// async/await
Future<User> fetchUser() async {
  final response = await dio.get('/user');
  return User.fromJson(response.data);
}

// Stream
Stream<int> countStream() async* {
  for (int i = 0; i < 10; i++) {
    await Future.delayed(Duration(seconds: 1));
    yield i;
  }
}

// Isolate（CPU 密集運算）
final result = await Isolate.run(() {
  return heavyJsonParsing(rawData);
});
```

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
| 取消機制 | `Job.cancel()`、scope 自動取消 | `Task.cancel()`、`withTaskCancellationHandler` | `Future` 不可取消；需用 `CancelableOperation` |

### 注意事項與 Best Practice

- **Android**：永遠使用 `viewModelScope` / `lifecycleScope` 而非自建 `GlobalScope`，避免記憶體洩漏。Flow 收集用 `repeatOnLifecycle` 確保只在前景時收集
- **iOS**：新專案優先用 `async/await`，避免 GCD callback hell。注意 `@MainActor` 的隱式繼承 — SwiftUI View 的 `body` 已是 `@MainActor`
- **Flutter**：大多數場景不需要 Isolate（網路/檔案 I/O 已是非同步）。只有 CPU 密集運算（>16ms 導致掉幀）才需要 Isolate

---

## 二、本地儲存

**Android — DataStore / Room**
- **SharedPreferences**（已過時）→ **DataStore**（官方推薦替代）：鍵值對儲存，支援 Flow 非同步讀取
  - `Preferences DataStore`：簡單 key-value，取代 SharedPreferences
  - `Proto DataStore`：用 Protocol Buffers 定義型別安全的結構化資料
- **Room**：SQLite 的 ORM 抽象層，Google 官方推薦的本地資料庫方案
- **MMKV**（騰訊開源）：高效能 key-value 儲存，基於 mmap，讀寫速度遠勝 SharedPreferences

```kotlin
// DataStore
val Context.dataStore by preferencesDataStore(name = "settings")

// 寫入
suspend fun saveToken(token: String) {
    context.dataStore.edit { prefs ->
        prefs[stringPreferencesKey("token")] = token
    }
}

// 讀取（Flow）
val tokenFlow: Flow<String?> = context.dataStore.data
    .map { prefs -> prefs[stringPreferencesKey("token")] }

// Room
@Entity
data class UserEntity(
    @PrimaryKey val id: Int,
    val name: String
)

@Dao
interface UserDao {
    @Query("SELECT * FROM UserEntity")
    fun getAll(): Flow<List<UserEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: UserEntity)
}
```

**iOS — UserDefaults / CoreData / SwiftData**
- **UserDefaults**：簡單 key-value 儲存（等同 SharedPreferences）。適合少量設定
- **CoreData**：Apple 的 ORM 框架，功能強大但學習成本極高（NSManagedObjectContext、NSFetchRequest）
- **SwiftData**（iOS 17+）：CoreData 的現代化替代，用 `@Model` macro 宣告，語法大幅簡化
- **Keychain**：安全儲存（加密），適合存放 token、密碼等敏感資料

```swift
// UserDefaults
UserDefaults.standard.set("value", forKey: "key")
let value = UserDefaults.standard.string(forKey: "key")

// SwiftData（iOS 17+）
@Model
class User {
    var name: String
    var age: Int
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

// 查詢
let descriptor = FetchDescriptor<User>(
    predicate: #Predicate { $0.age > 18 },
    sortBy: [SortDescriptor(\.name)]
)
let users = try modelContext.fetch(descriptor)

// Keychain（使用 KeychainAccess 套件簡化）
let keychain = Keychain(service: "com.app.service")
keychain["token"] = "abc123"
let token = keychain["token"]
```

**Flutter — SharedPreferences / Hive / sqflite**
- **shared_preferences**：官方套件，簡單 key-value（底層用各平台原生實作）
- **Hive**：輕量 NoSQL 資料庫，純 Dart 實作，速度快。適合中等複雜度的結構化資料
- **sqflite**：SQLite 封裝，適合需要複雜查詢的關聯式資料
- **flutter_secure_storage**：安全儲存（Android 用 EncryptedSharedPreferences，iOS 用 Keychain）
- **Isar**（Hive 作者新作）：高效能 NoSQL 資料庫，支援全文搜尋、自動索引

```dart
// SharedPreferences
final prefs = await SharedPreferences.getInstance();
await prefs.setString('token', 'abc123');
final token = prefs.getString('token');

// Hive
await Hive.initFlutter();
var box = await Hive.openBox('settings');
await box.put('name', 'John');
final name = box.get('name');

// sqflite
final db = await openDatabase('app.db', version: 1,
  onCreate: (db, version) {
    return db.execute(
      'CREATE TABLE users(id INTEGER PRIMARY KEY, name TEXT)',
    );
  },
);
await db.insert('users', {'name': 'John'});
final users = await db.query('users');
```

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

### 注意事項與 Best Practice

- **Android**：新專案用 DataStore 取代 SharedPreferences（後者在主執行緒同步讀寫，可能造成 ANR）。Room 適合結構化資料，搭配 Flow 可自動觀察資料變化
- **iOS**：UserDefaults 不要存大量資料（它會一次載入全部到記憶體）。敏感資料一定要存 Keychain，不要存 UserDefaults。SwiftData 需要 iOS 17+，還需考慮向下相容
- **Flutter**：shared_preferences 底層是各平台原生實作（Android SharedPreferences、iOS UserDefaults），因此有相同限制。Hive 適合取代 SharedPreferences 做較複雜的本地快取

---

## 三、導航與路由

**Android — Navigation Component / Compose Navigation**
- **Navigation Component**（XML 時代）：用 NavGraph（XML 或 Kotlin DSL）定義路由圖，`NavController` 控制跳轉
- **Compose Navigation**：Jetpack Compose 的路由方案，用字串路由或 type-safe route（推薦）
- 支援 Deep Link、動畫轉場、Safe Args 傳參

```kotlin
// Compose Navigation（Type-safe routes，Compose Navigation 2.8+）
@Serializable
data class ProfileRoute(val userId: String)

NavHost(navController, startDestination = HomeRoute) {
    composable<HomeRoute> {
        HomeScreen(
            onNavigateToProfile = { userId ->
                navController.navigate(ProfileRoute(userId))
            }
        )
    }
    composable<ProfileRoute> { backStackEntry ->
        val route = backStackEntry.toRoute<ProfileRoute>()
        ProfileScreen(userId = route.userId)
    }
}
```

**iOS — NavigationStack / UINavigationController**
- **UINavigationController**（UIKit）：傳統 push/pop 導航堆疊
- **NavigationStack**（SwiftUI，iOS 16+）：宣告式導航，用 `NavigationPath` 管理路由堆疊
- **NavigationLink**：SwiftUI 的導航觸發元件，支援 value-based 導航
- **Coordinator Pattern**：常見架構模式，將導航邏輯從 ViewController 抽離

```swift
// SwiftUI NavigationStack（iOS 16+）
struct ContentView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            List(users) { user in
                NavigationLink(value: user) {
                    Text(user.name)
                }
            }
            .navigationDestination(for: User.self) { user in
                ProfileView(user: user)
            }
        }
    }
}

// UIKit（程式碼導航）
let profileVC = ProfileViewController(userId: "123")
navigationController?.pushViewController(profileVC, animated: true)
```

**Flutter — Navigator 2.0 / GoRouter**
- **Navigator 1.0**（命令式）：`Navigator.push()` / `Navigator.pop()`，簡單但不支援 Deep Link 或複雜路由
- **Navigator 2.0**（宣告式）：`Router` + `RouteInformationParser`，功能強大但 API 極其冗長
- **GoRouter**（官方推薦）：Navigator 2.0 的簡化封裝，支援 Deep Link、Redirect、巢狀路由、型別安全
- **auto_route**：社群方案，用 code generation 定義路由，型別安全

```dart
// GoRouter
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomeScreen(),
      routes: [
        GoRoute(
          path: 'profile/:userId',
          builder: (context, state) {
            final userId = state.pathParameters['userId']!;
            return ProfileScreen(userId: userId);
          },
        ),
      ],
    ),
  ],
  redirect: (context, state) {
    final isLoggedIn = authNotifier.isLoggedIn;
    if (!isLoggedIn) return '/login';
    return null;
  },
);

// 導航
context.go('/profile/123');       // 替換整個堆疊
context.push('/profile/123');     // 推入堆疊
context.pop();                    // 返回
```

### 三平台導航對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 官方方案 | Compose Navigation | NavigationStack（SwiftUI） | Navigator 2.0 |
| 主流封裝 | 不需要（官方已足夠） | 不需要（SwiftUI 內建） | GoRouter（官方推薦） |
| 路由定義 | NavGraph（XML/DSL） | navigationDestination | GoRoute tree |
| 傳參方式 | Type-safe route / Safe Args | NavigationLink value | pathParameters / queryParameters |
| Deep Link 支援 | NavGraph 內建 `<deepLink>` | 需另外設定（SceneDelegate） | GoRouter 內建 |
| 動畫 | `AnimatedNavHost` / `EnterTransition` | 系統預設（可自訂 `transition`） | `CustomTransitionPage` |
| 返回堆疊管理 | `popBackStack`、`popUpTo` | `NavigationPath`、`dismiss` | `context.pop()`、`context.go()` |
| 巢狀導航 | nested NavGraph | NavigationSplitView | GoRouter ShellRoute |

### 注意事項與 Best Practice

- **Android**：Compose Navigation 2.8+ 支援 type-safe route（用 `@Serializable` data class），避免用字串路由
- **iOS**：SwiftUI 的 NavigationStack 僅支援 iOS 16+，如需向下相容仍需使用 NavigationView（已 deprecated）或 UIKit
- **Flutter**：避免直接使用 Navigator 2.0 的原生 API（過於冗長），用 GoRouter 或 auto_route 包裝。`context.go()` 會替換堆疊（像 web 的跳頁），`context.push()` 才是堆疊推入

---

## 四、推播通知

**Android — FCM（Firebase Cloud Messaging）**
- 必須整合 Firebase，透過 `google-services.json` 設定
- **FCM Token**：每台裝置對每個 App 的唯一識別碼，需上傳至後端
- 通知類型：
  - **Notification Message**：系統自動顯示通知（App 在背景時）
  - **Data Message**：App 自行處理（可在前景/背景自訂行為）
- Android 13+ 需要 `POST_NOTIFICATIONS` 執行時權限

```kotlin
// 取得 FCM Token
FirebaseMessaging.getInstance().token.addOnSuccessListener { token ->
    // 上傳 token 到後端
}

// 接收推播
class MyFirebaseService : FirebaseMessagingService() {
    override fun onMessageReceived(message: RemoteMessage) {
        val title = message.notification?.title
        val data = message.data["key"]
        showNotification(title, data)
    }

    override fun onNewToken(token: String) {
        // Token 更新，重新上傳到後端
    }
}

// 建立通知頻道（Android 8+）
val channel = NotificationChannel(
    "channel_id", "Channel Name",
    NotificationManager.IMPORTANCE_HIGH
)
notificationManager.createNotificationChannel(channel)
```

**iOS — APNs（Apple Push Notification service）**
- Apple 的推播基礎建設，所有 iOS 推播都經過 APNs
- 需要在 Apple Developer 設定 Push Notification capability
- 可搭配 FCM 使用（FCM 作為中間層，底層仍走 APNs）
- iOS 10+ 支援 **Rich Notification**（圖片、影片、自訂 UI）
- **Notification Service Extension**：在通知顯示前攔截修改內容

```swift
// 請求推播權限
UNUserNotificationCenter.current().requestAuthorization(
    options: [.alert, .badge, .sound]
) { granted, error in
    guard granted else { return }
    DispatchQueue.main.async {
        UIApplication.shared.registerForRemoteNotifications()
    }
}

// AppDelegate 取得 Device Token
func application(_ app: UIApplication,
    didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
    // 上傳 token 到後端（或透過 FCM: Messaging.messaging().apnsToken = deviceToken）
}

// 處理前景推播
func userNotificationCenter(_ center: UNUserNotificationCenter,
    willPresent notification: UNNotification) async -> UNNotificationPresentationOptions {
    return [.banner, .badge, .sound]
}
```

**Flutter — firebase_messaging**
- 統一封裝 FCM（Android）和 APNs（iOS）
- 背景處理需要 `@pragma('vm:entry-point')` 標記的頂層函式
- 搭配 `flutter_local_notifications` 處理前景通知顯示

```dart
// 請求權限
final settings = await FirebaseMessaging.instance.requestPermission();

// 取得 FCM Token
final token = await FirebaseMessaging.instance.getToken();

// 前景推播處理
FirebaseMessaging.onMessage.listen((RemoteMessage message) {
  final title = message.notification?.title;
  // 使用 flutter_local_notifications 顯示
});

// 背景推播處理（必須是頂層函式）
@pragma('vm:entry-point')
Future<void> _firebaseBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  // 處理背景推播
}

// 註冊背景處理
FirebaseMessaging.onBackgroundMessage(_firebaseBackgroundHandler);

// 點擊通知開啟 App
FirebaseMessaging.onMessageOpenedApp.listen((RemoteMessage message) {
  // 導航到對應頁面
});
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

### 注意事項與 Best Practice

- **Android**：Android 8+ 必須建立 NotificationChannel，否則通知不會顯示。注意區分 Notification Message 和 Data Message — 前者在背景時系統自動處理，App 無法控制
- **iOS**：首次推播權限請求被拒絕後，只能引導使用者到設定中開啟。建議在適當時機（非啟動時）請求權限，附上說明
- **Flutter**：背景處理函式必須是頂層函式（不能是類別方法），且需要 `@pragma('vm:entry-point')` 避免被 tree-shaking 移除

---

## 五、深度連結（Deep Link）

**Android — App Links**
- **Deep Link**：自訂 scheme（`myapp://path`），任何 App 都能註冊，可能衝突
- **App Links**（Android 6+）：使用 `https://` scheme + Digital Asset Links 驗證，確保只有你的 App 能處理。需在網站放置 `assetlinks.json`
- 在 `AndroidManifest.xml` 的 `<intent-filter>` 中宣告

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity">
    <!-- App Links（驗證過的 https 連結） -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https"
              android:host="www.example.com"
              android:pathPrefix="/profile" />
    </intent-filter>
</activity>
```

```kotlin
// 處理 Deep Link
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    intent?.data?.let { uri ->
        val userId = uri.getQueryParameter("id")
        navigateToProfile(userId)
    }
}

// 搭配 Navigation Component
navController.handleDeepLink(intent)
```

**iOS — Universal Links**
- **URL Scheme**（自訂 scheme）：`myapp://path`，任何 App 都能註冊，iOS 9 前的做法
- **Universal Links**（iOS 9+）：使用 `https://` + Apple App Site Association（AASA）檔案驗證。需在網站根目錄放置 `apple-app-site-association`
- 需在 Xcode 的 Associated Domains 加入 `applinks:domain.com`

```json
// apple-app-site-association（放在 https://domain.com/.well-known/）
{
  "applinks": {
    "apps": [],
    "details": [{
      "appID": "TEAMID.com.example.app",
      "paths": ["/profile/*", "/post/*"]
    }]
  }
}
```

```swift
// SceneDelegate 處理 Universal Link
func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL else { return }
    handleDeepLink(url)
}

// SwiftUI
.onOpenURL { url in
    handleDeepLink(url)
}
```

**Flutter — Deep Linking**
- 底層依賴各平台原生設定（Android intent-filter / iOS Associated Domains）
- 官方支援：`MaterialApp.router` + GoRouter 處理 Deep Link
- **uni_links** / **app_links**（社群套件）：簡化 Deep Link 接收

```dart
// GoRouter 自動處理 Deep Link
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/profile/:id',
      builder: (context, state) {
        final id = state.pathParameters['id']!;
        return ProfileScreen(id: id);
      },
    ),
  ],
);

// 手動監聽（使用 app_links 套件）
final appLinks = AppLinks();
appLinks.uriLinkStream.listen((Uri uri) {
  // 處理 Deep Link
  router.go(uri.path);
});
```

### 三平台 Deep Link 對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 驗證型連結 | App Links（https + assetlinks.json） | Universal Links（https + AASA） | 依各平台原生設定 |
| 自訂 scheme | `myapp://`（intent-filter） | `myapp://`（URL Scheme） | 兩邊都要設定 |
| 驗證檔案位置 | `https://domain/.well-known/assetlinks.json` | `https://domain/.well-known/apple-app-site-association` | 兩邊都要放 |
| 處理入口 | `Activity.intent.data` | `SceneDelegate` / `.onOpenURL` | GoRouter 自動 / `app_links` |
| 未安裝 App 時 | 開啟瀏覽器（Deferred Deep Link 需另外實作） | 開啟瀏覽器 | 開啟瀏覽器 |
| 搭配導航 | Navigation Component `handleDeepLink` | NavigationPath 手動處理 | GoRouter 自動匹配路由 |

### 注意事項與 Best Practice

- **Android**：`autoVerify="true"` 需要 Digital Asset Links 驗證成功，否則系統會彈出選擇器讓使用者選要用哪個 App 開啟
- **iOS**：AASA 檔案必須是 valid JSON 且 Content-Type 為 `application/json`。Apple 會快取 AASA，更新後可能需要重新安裝 App 或等待 CDN 刷新
- **Flutter**：Deep Link 的原生設定（AndroidManifest / Info.plist）仍需手動處理，Flutter 只負責接收和路由。GoRouter 是目前最簡單的 Deep Link 整合方案

---

## 六、圖片載入與快取

**Android — Coil / Glide**
- **Glide**：老牌主流，Google 官方推薦，基於 Java，生態成熟
- **Coil**（Coroutine Image Loader）：Kotlin-first，輕量，Compose 原生支援，已成為新專案首選
- 兩者都支援：記憶體快取 + 磁碟快取、圖片轉換（圓角、模糊）、GIF/WebP/SVG、生命週期感知

```kotlin
// Coil（Compose）
AsyncImage(
    model = ImageRequest.Builder(context)
        .data("https://example.com/image.jpg")
        .crossfade(true)
        .build(),
    contentDescription = null,
    modifier = Modifier.size(128.dp),
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error),
)

// Glide（XML View）
Glide.with(context)
    .load("https://example.com/image.jpg")
    .placeholder(R.drawable.placeholder)
    .error(R.drawable.error)
    .circleCrop()
    .into(imageView)
```

**iOS — Kingfisher / SDWebImage**
- **Kingfisher**：Swift 社群最主流的圖片載入庫，純 Swift，支援 SwiftUI
- **SDWebImage**：Objective-C 時代的老牌庫，仍在維護，支援 SwiftUI
- iOS 15+ 原生 `AsyncImage`：系統內建，但功能陽春（無快取控制、無轉換）

```swift
// Kingfisher（SwiftUI）
KFImage(URL(string: "https://example.com/image.jpg"))
    .placeholder { ProgressView() }
    .resizable()
    .fade(duration: 0.3)
    .scaledToFit()
    .frame(width: 128, height: 128)

// 原生 AsyncImage（iOS 15+，功能有限）
AsyncImage(url: URL(string: "https://example.com/image.jpg")) { phase in
    switch phase {
    case .success(let image): image.resizable().scaledToFit()
    case .failure: Image(systemName: "photo")
    case .empty: ProgressView()
    @unknown default: EmptyView()
    }
}
```

**Flutter — cached_network_image**
- **cached_network_image**：社群最主流，底層使用 `flutter_cache_manager` 管理磁碟快取
- **fast_cached_network_image**：輕量替代，快取機制更簡單
- Flutter 的 `Image.network()` 有基本記憶體快取但無磁碟快取

```dart
// cached_network_image
CachedNetworkImage(
  imageUrl: "https://example.com/image.jpg",
  placeholder: (context, url) => const CircularProgressIndicator(),
  errorWidget: (context, url, error) => const Icon(Icons.error),
  width: 128,
  height: 128,
  fit: BoxFit.cover,
)

// 搭配 CacheManager 自訂快取策略
CachedNetworkImage(
  imageUrl: url,
  cacheManager: CacheManager(Config(
    'customCacheKey',
    stalePeriod: const Duration(days: 7),
    maxNrOfCacheObjects: 100,
  )),
)
```

### 三平台圖片載入對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 主流套件 | Coil（新專案）/ Glide（舊專案） | Kingfisher | cached_network_image |
| 記憶體快取 | 內建（LRU Cache） | 內建（NSCache 為基礎） | 內建（ImageCache） |
| 磁碟快取 | 內建（DiskLruCache） | 內建 | flutter_cache_manager |
| Compose/SwiftUI 支援 | `AsyncImage`（Coil） | `KFImage` | `CachedNetworkImage` Widget |
| 圖片轉換 | `transformations`（圓角、模糊等） | `processor`（圓角、模糊等） | 需額外處理 |
| GIF 支援 | 內建 | 內建 | 需用 `FadeInImage` 或其他方案 |
| SVG 支援 | Coil 有 SVG decoder | SVGKit（第三方） | flutter_svg |
| 預載 | `ImageRequest.Builder.preload` | `KingfisherManager.prefetch` | `precacheImage()` |

### 注意事項與 Best Practice

- **Android**：Coil 是 Kotlin-first 且 Compose 原生支援，新專案建議使用。Glide 仍在 XML View 生態中佔主導地位
- **iOS**：Kingfisher 幾乎是 Swift 社群的標準選擇。原生 AsyncImage 適合簡單場景，但缺乏快取控制和圖片處理能力
- **Flutter**：避免在 `ListView` 中使用不帶快取的 `Image.network()`，會導致滾動時重複載入。`cached_network_image` 的預設快取策略（7 天）適用大多數場景
- **通用**：注意圖片尺寸 — 載入遠大於顯示尺寸的圖片會浪費記憶體。三個平台都支援 resize/downsample 功能

---

## 七、安全性

**Android — ProGuard/R8 / Keystore**
- **R8**（取代 ProGuard）：程式碼混淆 + 縮減 + 優化。Release build 預設啟用
  - 混淆：將類別/方法名改為 `a.b.c`，增加反編譯難度
  - 縮減（Tree Shaking）：移除未使用的程式碼，減少 APK 大小
  - 需撰寫 ProGuard rules 保留必要的類別（Reflection、序列化用到的）
- **Keystore**：App 簽章金鑰，用於確認 App 身份。遺失無法更新 App
- **EncryptedSharedPreferences**：Android Jetpack 提供的加密儲存
- **Network Security Config**：限制明文 HTTP 連線、設定憑證 pinning

```kotlin
// build.gradle — 啟用 R8 混淆
android {
    buildTypes {
        release {
            isMinifyEnabled = true       // 啟用混淆 + 縮減
            isShrinkResources = true     // 移除未使用的資源
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}

// proguard-rules.pro — 保留規則
-keep class com.example.dto.** { *; }  // 保留 DTO（Gson 反射需要）
-keepclassmembers class * {
    @com.google.gson.annotations.SerializedName <fields>;
}

// EncryptedSharedPreferences
val prefs = EncryptedSharedPreferences.create(
    context, "secret_prefs",
    MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC),
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
```

**iOS — ATS / Keychain**
- **ATS（App Transport Security）**：iOS 9+ 預設強制 HTTPS，可在 Info.plist 例外設定但 Apple 審查嚴格
- **Keychain**：iOS 的安全儲存機制，由系統層級加密，即使越獄也難以直接讀取
- **Code Signing**：Apple 的強制簽章機制，確保 App 未被竄改
- iOS App 天生就有程式碼保護 — App Store 的 Bitcode + FairPlay DRM 加密

```swift
// Keychain 存取（使用 KeychainAccess 套件）
import KeychainAccess

let keychain = Keychain(service: "com.example.app")
    .accessibility(.whenUnlockedThisDeviceOnly) // 安全等級
try keychain.set("secret_token", key: "authToken")
let token = try keychain.get("authToken")

// Certificate Pinning（URLSession）
class PinningDelegate: NSObject, URLSessionDelegate {
    func urlSession(_ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        // 驗證伺服器憑證
        guard let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        // 比對憑證指紋...
    }
}
```

**Flutter — 混淆 / 安全儲存**
- **Dart 混淆**：`flutter build --obfuscate --split-debug-info=build/debug-info`，混淆 Dart 程式碼
- **flutter_secure_storage**：跨平台安全儲存（Android 底層用 EncryptedSharedPreferences / KeyStore，iOS 底層用 Keychain）
- **flutter_jailbreak_detection**：偵測越獄/Root 裝置
- 原生層的安全設定（Android ProGuard、iOS ATS）仍需個別處理

```dart
// 建置時混淆
// flutter build apk --obfuscate --split-debug-info=build/debug-info

// flutter_secure_storage
final storage = FlutterSecureStorage();
await storage.write(key: 'token', value: 'secret_value');
final token = await storage.read(key: 'token');
await storage.delete(key: 'token');

// 環境變數隱藏 API Key（使用 --dart-define）
// flutter build apk --dart-define=API_KEY=xxx
const apiKey = String.fromEnvironment('API_KEY');
```

### 三平台安全性對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 程式碼混淆 | R8（預設啟用於 release） | Bitcode + FairPlay DRM（自動） | `--obfuscate` flag |
| 安全儲存 | EncryptedSharedPreferences / KeyStore | Keychain | flutter_secure_storage |
| 網路安全 | Network Security Config | ATS（App Transport Security） | 各平台原生設定 |
| 憑證 Pinning | OkHttp CertificatePinner | URLSession delegate | dio_http2_adapter / 原生設定 |
| 簽章驗證 | APK Signature Scheme v2/v3 | Code Signing（強制） | 各平台各自處理 |
| Root/越獄偵測 | SafetyNet / Play Integrity API | 手動偵測 / 第三方 | flutter_jailbreak_detection |
| 反編譯難度 | 低（APK 容易反編譯） | 高（iOS binary 較難分析） | 中（Dart AOT 增加難度） |

### 注意事項與 Best Practice

- **Android**：APK 很容易被反編譯（用 jadx 即可），R8 混淆只增加閱讀難度不是絕對安全。API Key 等機密不要寫死在程式碼中，應用 BuildConfig + CI/CD 注入
- **iOS**：不要關閉 ATS（`NSAllowsArbitraryLoads = YES`），Apple 審查可能被退回。Keychain 的 accessibility 等級要根據需求設定（`whenUnlockedThisDeviceOnly` 最安全）
- **Flutter**：`--obfuscate` 只混淆 Dart 層，原生層（Java/Kotlin/Swift）需各自處理。flutter_secure_storage 在 Android 上的行為取決於 Android 版本（API 23 以下用 RSA + AES，以上用 AES-GCM）

---

## 八、效能優化

**Android — Baseline Profiles / Compose 優化**
- **Baseline Profiles**：預編譯關鍵路徑的機器碼，加速 App 冷啟動（提升 30-40%）和流暢度
- **Compose 優化**：
  - 穩定性（Stability）：標記 `@Stable` / `@Immutable`，避免不必要的 recomposition
  - `remember` / `derivedStateOf`：快取運算結果，減少重組
  - `LazyColumn` key 設定：正確設定 key 避免不必要的重建
- **LeakCanary**：記憶體洩漏偵測工具
- **Android Profiler**（Android Studio）：CPU、記憶體、網路、耗電分析

```kotlin
// Baseline Profiles（build.gradle）
dependencies {
    implementation("androidx.profileinstaller:profileinstaller:1.3.1")
    baselineProfile(project(":baselineprofile"))
}

// Compose — 避免不穩定參數導致不必要的 recomposition
@Immutable
data class UserUiState(
    val name: String,
    val avatar: String
)

// derivedStateOf — 減少不必要的重組
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}

// LazyColumn 設定 key
LazyColumn {
    items(users, key = { it.id }) { user ->
        UserItem(user)
    }
}
```

**iOS — Instruments / MetricKit**
- **Instruments**（Xcode）：iOS 最強大的效能分析工具，支援 CPU、記憶體、GPU、網路、動畫等
  - **Time Profiler**：分析 CPU 使用熱點
  - **Allocations**：追蹤記憶體分配
  - **Core Animation**：分析畫面渲染效能
- **MetricKit**（iOS 13+）：收集使用者裝置上的效能指標（啟動時間、hang 率、記憶體峰值）
- **SwiftUI 優化**：
  - `@Observable` 細粒度觀察（比 `@ObservedObject` 更精確）
  - `EquatableView`：避免不必要的 view 更新
  - `LazyVStack` / `LazyHStack`：延遲載入（等同 RecyclerView / LazyColumn）

```swift
// MetricKit — 收集效能數據
class MetricSubscriber: NSObject, MXMetricManagerSubscriber {
    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            let launchTime = payload.applicationLaunchMetrics
            // 上報到分析平台
        }
    }
}

// SwiftUI — @Observable 細粒度更新（iOS 17+）
@Observable
class ViewModel {
    var name: String = ""     // 只有用到 name 的 View 會更新
    var count: Int = 0        // 只有用到 count 的 View 會更新
}

// LazyVStack（延遲載入）
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ItemView(item: item)
        }
    }
}
```

**Flutter — DevTools / Widget 重建優化**
- **Flutter DevTools**：官方效能分析工具（Performance、Memory、CPU Profiler、Widget Inspector）
- **Widget 重建優化**：
  - `const` 建構子：標記不會改變的 Widget，Framework 跳過重建
  - `RepaintBoundary`：隔離重繪範圍
  - 拆分小 Widget：將 `setState` 的影響範圍最小化
- **Impeller**（Flutter 3.16+）：新渲染引擎，減少 shader compilation jank
- **Performance Overlay**：即時顯示 UI/Raster 執行緒的幀率

```dart
// const Widget — 避免不必要重建
class MyWidget extends StatelessWidget {
  const MyWidget({super.key}); // const 建構子

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        const Text('Static Text'),  // 加 const，不會重建
        Text('Dynamic: ${DateTime.now()}'), // 每次都重建
      ],
    );
  }
}

// 拆分 Widget 最小化 setState 影響
class CounterPage extends StatefulWidget {
  const CounterPage({super.key});
  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        const ExpensiveHeader(), // 不受 setState 影響
        Text('$_count'),         // 只有這行需要更新
        ElevatedButton(
          onPressed: () => setState(() => _count++),
          child: const Text('Add'),
        ),
      ],
    );
  }
}

// RepaintBoundary
RepaintBoundary(
  child: ComplexAnimation(), // 隔離這個 Widget 的重繪
)
```

### 三平台效能優化對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 效能分析工具 | Android Profiler / Perfetto | Instruments | Flutter DevTools |
| 啟動優化 | Baseline Profiles | Pre-main optimization | Deferred Components |
| 列表優化 | `LazyColumn` + key | `LazyVStack` | `ListView.builder` + key |
| 避免不必要重建 | `@Stable`、`remember` | `@Observable`、`EquatableView` | `const`、`RepaintBoundary` |
| 記憶體洩漏偵測 | LeakCanary | Instruments Allocations | DevTools Memory tab |
| 執行緒卡頓偵測 | StrictMode | Main Thread Checker | Performance Overlay |
| 使用者端效能數據 | Firebase Performance | MetricKit | Firebase Performance |
| 渲染引擎 | Skia / RenderThread | Core Animation / Metal | Impeller / Skia |

### 注意事項與 Best Practice

- **Android**：Baseline Profiles 對冷啟動效果顯著，建議所有正式 App 都加入。Compose 的 recomposition 是效能關鍵 — 用 Layout Inspector 檢查哪些 Composable 被不必要地重組
- **iOS**：Instruments 的 Time Profiler 是找出效能瓶頸的最佳工具。SwiftUI 的 `@Observable`（iOS 17+）比 `@ObservedObject` 更精確，減少不必要的 view 更新
- **Flutter**：`const` 是最簡單也最有效的優化。Profile mode（`flutter run --profile`）才能準確測量效能，Debug mode 包含大量斷言和除錯資訊會嚴重拖慢速度

---

## 九、測試

**Android — JUnit / Espresso / Compose Testing**
- **Unit Test**（JUnit + MockK/Mockito）：測試 ViewModel、UseCase、Repository 等純邏輯
- **UI Test**（Espresso）：測試 XML View 的 UI 互動
- **Compose Testing**（`compose-ui-test`）：測試 Compose UI，用 `ComposeTestRule`
- **Robolectric**：在 JVM 上模擬 Android 環境，不需要模擬器，速度快

```kotlin
// Unit Test（JUnit + MockK）
@Test
fun `fetchUser should return user`() = runTest {
    val mockRepo = mockk<UserRepository>()
    coEvery { mockRepo.getUser("1") } returns User("1", "John")

    val useCase = GetUserUseCase(mockRepo)
    val result = useCase("1")

    assertEquals("John", result.name)
}

// Compose UI Test
@get:Rule
val composeTestRule = createComposeRule()

@Test
fun `click button should show text`() {
    composeTestRule.setContent { MyScreen() }
    composeTestRule.onNodeWithText("Click Me").performClick()
    composeTestRule.onNodeWithText("Hello!").assertIsDisplayed()
}

// Espresso（XML View）
@Test
fun clickButton_showsText() {
    onView(withId(R.id.button)).perform(click())
    onView(withId(R.id.textView)).check(matches(withText("Hello!")))
}
```

**iOS — XCTest / XCUITest**
- **XCTest**：Apple 官方測試框架，涵蓋 Unit Test 和 UI Test
- **XCUITest**：XCTest 的 UI 測試子集，模擬使用者操作
- **Swift Testing**（Swift 5.10+）：新一代測試框架，語法更簡潔（`@Test`、`#expect`）
- **Preview Test**：SwiftUI Preview 可作為快速的視覺測試

```swift
// XCTest Unit Test
class UserViewModelTests: XCTestCase {
    func testFetchUser() async throws {
        let mockRepo = MockUserRepository()
        mockRepo.mockUser = User(id: "1", name: "John")
        let viewModel = UserViewModel(repo: mockRepo)

        await viewModel.fetchUser(id: "1")

        XCTAssertEqual(viewModel.user?.name, "John")
    }
}

// Swift Testing（新語法）
@Test func fetchUser() async throws {
    let viewModel = UserViewModel(repo: MockUserRepository())
    await viewModel.fetchUser(id: "1")
    #expect(viewModel.user?.name == "John")
}

// XCUITest
func testLoginFlow() {
    let app = XCUIApplication()
    app.launch()
    app.textFields["Email"].tap()
    app.textFields["Email"].typeText("test@example.com")
    app.buttons["Login"].tap()
    XCTAssertTrue(app.staticTexts["Welcome"].exists)
}
```

**Flutter — Widget Test / Integration Test**
- **Unit Test**：測試純 Dart 邏輯（`package:test`）
- **Widget Test**：在測試環境中渲染 Widget，模擬互動（不需要模擬器，速度快）
- **Integration Test**：在真實裝置或模擬器上跑完整的 App 流程
- **Mockito**（`package:mockito`）：Mock 依賴
- **golden test**：截圖對比測試，確保 UI 不意外改變

```dart
// Unit Test
test('Counter increments', () {
  final counter = Counter();
  counter.increment();
  expect(counter.value, 1);
});

// Widget Test
testWidgets('Click button shows text', (tester) async {
  await tester.pumpWidget(const MaterialApp(home: MyScreen()));
  await tester.tap(find.text('Click Me'));
  await tester.pump(); // 重建 Widget
  expect(find.text('Hello!'), findsOneWidget);
});

// Integration Test（integration_test）
testWidgets('Login flow', (tester) async {
  app.main();
  await tester.pumpAndSettle();
  await tester.enterText(find.byType(TextField).first, 'test@example.com');
  await tester.tap(find.text('Login'));
  await tester.pumpAndSettle();
  expect(find.text('Welcome'), findsOneWidget);
});

// Golden Test（截圖對比）
testWidgets('MyWidget matches golden', (tester) async {
  await tester.pumpWidget(const MyWidget());
  await expectLater(
    find.byType(MyWidget),
    matchesGoldenFile('goldens/my_widget.png'),
  );
});
```

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

### 注意事項與 Best Practice

- **Android**：優先寫 Unit Test（速度快、覆蓋率高），Compose Testing 比 Espresso 更直覺。用 Robolectric 避免依賴模擬器
- **iOS**：Swift Testing 語法比 XCTest 簡潔許多（`#expect` vs `XCTAssertEqual`），新專案建議採用。XCUITest 很慢，只測關鍵流程
- **Flutter**：Widget Test 是 Flutter 的殺手級特色 — 不需要模擬器就能測 UI，速度接近 Unit Test。善用 `pumpAndSettle()` 等待動畫完成

---

## 十、CI/CD

**Android — Gradle / GitHub Actions**
- **Gradle**：Android 的建置系統，定義編譯、測試、打包、簽章的完整流程
- **GitHub Actions**：最普及的 CI/CD 平台，用 YAML 定義 workflow
- **Fastlane**：自動化工具（也適用 Android），簡化上傳 Google Play 的流程
- **Google Play Developer API**：程式化上傳 AAB、管理測試軌道

```yaml
# .github/workflows/android-ci.yml
name: Android CI
on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build
        run: ./gradlew assembleDebug

      - name: Unit Test
        run: ./gradlew testDebugUnitTest

      - name: Lint
        run: ./gradlew lintDebug

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Build Release AAB
        run: ./gradlew bundleRelease

      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT }}
          packageName: com.example.app
          releaseFiles: app/build/outputs/bundle/release/*.aab
          track: internal
```

**iOS — Xcode Cloud / Fastlane**
- **Xcode Cloud**：Apple 原生的 CI/CD（2022 推出），與 Xcode 深度整合，自動管理簽章
- **Fastlane**：社群最主流的 iOS CI/CD 工具（match 管理憑證、gym 建置、deliver 上傳）
- **GitHub Actions**：需要 macOS runner（`runs-on: macos-latest`），成本較高

```ruby
# Fastfile（Fastlane）
default_platform(:ios)

platform :ios do
  desc "Deploy to TestFlight"
  lane :beta do
    match(type: "appstore")           # 自動管理簽章
    increment_build_number
    gym(scheme: "MyApp")              # 建置
    pilot(skip_waiting_for_build_processing: true)  # 上傳 TestFlight
  end

  desc "Deploy to App Store"
  lane :release do
    match(type: "appstore")
    gym(scheme: "MyApp")
    deliver(force: true)              # 上傳 App Store
  end
end
```

```yaml
# GitHub Actions for iOS
jobs:
  build:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - name: Build and Test
        run: |
          xcodebuild test \
            -scheme MyApp \
            -destination 'platform=iOS Simulator,name=iPhone 15' \
            -resultBundlePath TestResults
```

**Flutter — Codemagic / GitHub Actions**
- **Codemagic**：Flutter 專屬的 CI/CD 平台，內建 Flutter 環境、自動管理 iOS 簽章
- **GitHub Actions**：用 `subosito/flutter-action` 設定 Flutter 環境
- **Very Good CLI**（VGD）：社群工具，快速建立包含 CI/CD 模板的 Flutter 專案

```yaml
# GitHub Actions for Flutter
name: Flutter CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'

      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test --coverage

      - name: Build Android
        run: flutter build apk --release

  build-ios:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter build ipa --release --export-options-plist=ExportOptions.plist
```

```yaml
# Codemagic（codemagic.yaml）
workflows:
  android-release:
    name: Android Release
    max_build_duration: 30
    environment:
      flutter: stable
      groups:
        - google_play
    scripts:
      - name: Build AAB
        script: flutter build appbundle --release
    publishing:
      google_play:
        credentials: $GCLOUD_SERVICE_ACCOUNT_CREDENTIALS
        track: internal
```

### 三平台 CI/CD 對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 官方方案 | 無（Gradle 是建置工具） | Xcode Cloud | 無 |
| 主流 CI/CD | GitHub Actions | Fastlane + GitHub Actions | Codemagic / GitHub Actions |
| Runner 需求 | Linux / macOS | macOS（必須） | Linux（Android）+ macOS（iOS） |
| 建置指令 | `./gradlew bundleRelease` | `xcodebuild` / `fastlane gym` | `flutter build appbundle/ipa` |
| 簽章管理 | Keystore 檔案 + secrets | Fastlane match / Xcode Cloud 自動 | 各平台各自處理 |
| 上傳商店 | Fastlane supply / Google Play API | Fastlane deliver / pilot | Codemagic 內建 / Fastlane |
| 測試自動化 | `./gradlew test` | `xcodebuild test` | `flutter test` |
| 費用 | GitHub Actions 免費額度充足 | macOS runner 較貴 | Codemagic 有免費額度 |

### 注意事項與 Best Practice

- **Android**：將 Keystore 和 signing config 存在 CI/CD 的 secrets 中，不要提交到 Git。`./gradlew bundleRelease` 是上傳 Google Play 的標準產物
- **iOS**：iOS CI/CD 最大挑戰是簽章管理 — Fastlane match 用 Git repo 儲存憑證是業界最佳實踐。Xcode Cloud 自動管理簽章，但客製化能力較弱
- **Flutter**：兩平台建置都需要在一個 workflow 中處理。iOS 建置必須在 macOS runner 上。Codemagic 對 Flutter 的支援最友善（預設環境已設定好）

---

## 十、模組化架構

**Android — Multi-module**
- Gradle 原生支援多模組，用 `include` 和 `implementation project(":module")` 定義依賴
- 常見分法：
  - **按層分**：`:app`、`:data`、`:domain`、`:presentation`
  - **按功能分**：`:feature:home`、`:feature:profile`、`:core:network`、`:core:ui`
- **Convention Plugins**：統一多模組的 Gradle 設定（避免每個模組重複設定）
- **Build 效能**：模組未改動時不需重新編譯（增量編譯），大型專案效果顯著

```kotlin
// settings.gradle.kts
include(":app")
include(":core:network")
include(":core:ui")
include(":core:data")
include(":domain")
include(":feature:home")
include(":feature:profile")

// feature/home/build.gradle.kts
plugins {
    id("com.android.library")
    id("kotlin-android")
    id("dagger.hilt.android.plugin")
}

dependencies {
    implementation(project(":core:network"))
    implementation(project(":core:ui"))
    implementation(project(":domain"))
    // feature 模組之間不能互相依賴
}

// 依賴方向（單向）
// app → feature:* → domain → core:*
// feature 之間透過 Navigation 跳轉，不直接依賴
```

**iOS — SPM Modules / Framework**
- **SPM（Swift Package Manager）**：Apple 官方的模組化方案，用 `Package.swift` 定義
- **Xcode Project + Framework**：傳統做法，建立多個 Framework target
- **Tuist**：社群工具，用 Swift DSL 定義專案結構，解決 Xcode project 檔衝突問題
- **動態 vs 靜態 Framework**：動態減少 binary 大小但增加啟動時間，靜態反之

```swift
// Package.swift（Local SPM Module）
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "CoreNetwork",
    platforms: [.iOS(.v16)],
    products: [
        .library(name: "CoreNetwork", targets: ["CoreNetwork"]),
    ],
    dependencies: [
        .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
    ],
    targets: [
        .target(
            name: "CoreNetwork",
            dependencies: ["Alamofire"]
        ),
        .testTarget(
            name: "CoreNetworkTests",
            dependencies: ["CoreNetwork"]
        ),
    ]
)

// 主專案引入 Local Package
// Xcode → File → Add Packages → Add Local → 選擇資料夾
```

**Flutter — Packages / Plugins**
- **Package**：純 Dart 模組，不含原生程式碼
- **Plugin**：包含原生程式碼（Android/iOS）的模組，透過 Platform Channel 溝通
- **Monorepo + Melos**：管理多 package 的工具（版本管理、指令廣播、依賴管理）
- 常見結構：`packages/core`、`packages/features/*`、`packages/shared_ui`

```yaml
# packages/core_network/pubspec.yaml
name: core_network
description: Network layer module
version: 1.0.0

dependencies:
  dio: ^5.4.0
  retrofit: ^4.1.0
```

```dart
// packages/feature_home/lib/home_screen.dart
import 'package:core_network/core_network.dart';
import 'package:shared_ui/shared_ui.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: Center(child: Text('Home')),
    );
  }
}
```

```yaml
# 主專案 pubspec.yaml — 引入本地 package
dependencies:
  core_network:
    path: packages/core_network
  feature_home:
    path: packages/feature_home
```

```yaml
# melos.yaml（Monorepo 管理）
name: my_app
packages:
  - packages/**
  - app

scripts:
  analyze:
    exec: dart analyze .
  test:
    exec: flutter test
    packageFilters:
      dirExists: test
```

### 三平台模組化對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 官方方案 | Gradle Multi-module | SPM（Swift Package Manager） | Dart Package / Plugin |
| 模組定義 | `build.gradle.kts` | `Package.swift` | `pubspec.yaml` |
| 依賴管理 | `implementation project(":module")` | Xcode Add Local Package | `path: packages/module` |
| Monorepo 工具 | Convention Plugins | Tuist | Melos |
| 增量編譯 | 支援（模組級別） | 支援（SPM target 級別） | 部分支援 |
| 存取控制 | `internal`（模組內可見，預設） | `internal`（模組內可見，預設） | `src/` 資料夾（慣例私有） |
| 動態載入 | Dynamic Feature Module（Play Feature Delivery） | 不支援 | Deferred Components（實驗性） |
| DI 跨模組 | Hilt 的 `@InstallIn` + Component 階層 | 手動注入 / Swinject | get_it 全域 / Riverpod |

### 注意事項與 Best Practice

- **Android**：模組間依賴必須是單向的（`feature → domain → core`），禁止循環依賴。Convention Plugins 可統一 Gradle 設定，避免「複製貼上 build.gradle」的問題
- **iOS**：SPM 是目前 Apple 推薦的模組化方式，但與 Xcode 的整合仍有 rough edges（如 Resources bundle 處理）。Tuist 可以解決多人協作時 `.xcodeproj` 衝突的問題
- **Flutter**：Dart 的存取控制依賴命名慣例（`src/` 下的檔案不應被外部引入），不像 Kotlin/Swift 有語言級別的 `internal` 修飾子。Melos 是大型 Flutter Monorepo 的必備工具

---

## 十一、總結對照表

| 面向 | Android | iOS | Flutter |
|------|---------|-----|---------|
| 非同步 | Coroutines + Flow | async/await + Actor | async/await + Isolate |
| 本地儲存（輕量） | DataStore | UserDefaults | shared_preferences |
| 本地儲存（資料庫） | Room | SwiftData / CoreData | sqflite / Hive |
| 導航 | Compose Navigation | NavigationStack | GoRouter |
| 推播 | FCM | APNs（+ FCM） | firebase_messaging |
| Deep Link | App Links | Universal Links | GoRouter + 原生設定 |
| 圖片載入 | Coil / Glide | Kingfisher | cached_network_image |
| 程式碼保護 | R8 混淆 | Bitcode + FairPlay | `--obfuscate` |
| 安全儲存 | EncryptedSharedPreferences | Keychain | flutter_secure_storage |
| 效能工具 | Android Profiler | Instruments | DevTools |
| Unit Test | JUnit + MockK | XCTest / Swift Testing | package:test |
| UI Test | Compose Testing / Espresso | XCUITest | Widget Test |
| CI/CD | GitHub Actions + Gradle | Fastlane / Xcode Cloud | Codemagic / GitHub Actions |
| 模組化 | Gradle Multi-module | SPM | Dart Package + Melos |
