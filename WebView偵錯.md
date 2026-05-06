# WebView 偵錯教學

本文介紹如何在 Android 和 iOS 平台上偵錯 WebView，包括 JavaScript Interface 測試。

---

# Android — Chrome DevTools

在 Android 開發中，WebView 透過 `@JavascriptInterface` 提供原生方法給 Web 端呼叫。以下介紹如何使用 Chrome DevTools 連接裝置上的 WebView 進行即時測試。

## 前置條件

- Android 裝置已開啟 **USB 偵錯模式**
- 裝置透過 USB 連接到電腦
- 電腦已安裝 Chrome 瀏覽器
- App 為 **Debug 版本**（Release 版本預設關閉 WebView 偵錯）

## 步驟一：確認 WebView 偵錯已啟用

App 的 WebView 必須啟用偵錯功能。在 Debug 版本中通常已預設開啟：

```kotlin
if (BuildConfig.DEBUG) {
    WebView.setWebContentsDebuggingEnabled(true)
}
```

如果你的專案沒有這段程式碼，需要在 `Application.onCreate()` 或 WebView 初始化前加入。

## 步驟二：在裝置上開啟目標 WebView 頁面

1. 安裝 Debug 版 APK 到裝置
2. 在 App 中導航到包含 WebView 的頁面（例如 Minikorn 聊天室）

## 步驟三：用 Chrome DevTools 連接 WebView

1. 在電腦的 Chrome 瀏覽器網址列輸入：

```
chrome://inspect/#devices
```

2. 頁面會列出已連接裝置上所有可偵錯的 WebView：

```
Devices
  Samsung SM-A5660 - 16
    WebView in com.showyouapp.ekkorn (59.0.3071.125)
      https://dev-h5.showyouapp.com/v3/twpj.html#/
      [inspect]  [pause]
```

3. 點擊目標 WebView 下方的 **inspect** 連結，會開啟 DevTools 視窗

## 步驟四：在 Console 中測試 JavaScript Interface

DevTools 開啟後，切換到 **Console** 分頁。

### 確認 Interface 是否存在

先輸入 interface 物件名稱確認它已被注入（名稱對應 `addJavascriptInterface` 的第二個參數）：

```javascript
window.MyBridge
// 預期輸出：Object（非 undefined）
```

### 列出所有可用方法

```javascript
Object.getOwnPropertyNames(window.MyBridge)
// 預期輸出：["getToken", "goBack", "showToast", ...]
```

### 呼叫測試

#### Android 端定義

```kotlin
class JsBridge(private val context: WeakReference<Activity>) {

    // 無參數，有回傳值
    @JavascriptInterface
    fun getToken(): String {
        return sessionManager.token
    }

    // 有參數，無回傳值
    @JavascriptInterface
    fun showToast(message: String) {
        context.get()?.runOnUiThread {
            Toast.makeText(context.get(), message, Toast.LENGTH_SHORT).show()
        }
    }

    // 無參數，無回傳值
    @JavascriptInterface
    fun goBack() {
        context.get()?.runOnUiThread {
            context.get()?.finish()
        }
    }
}

// 注入（第二個參數 "MyBridge" 即為 Web 端呼叫時使用的物件名稱）
webView.addJavascriptInterface(JsBridge(WeakReference(this)), "MyBridge")
```

#### Web 端呼叫方式

Web 端透過 `window.MyBridge` 存取 Android 注入的方法：

```javascript
// 無參數，有回傳值
function getToken() {
    if (window.MyBridge) {
        return window.MyBridge.getToken()
    }
}

// 有參數，無回傳值
function showToast(message) {
    if (window.MyBridge) {
        window.MyBridge.showToast(message)
    }
}

// 無參數，無回傳值
function goBack() {
    if (window.MyBridge) {
        window.MyBridge.goBack()
    }
}
```

> **注意**：呼叫前務必用 `if (window.MyBridge)` 檢查物件是否存在，避免在非 Android 環境（如瀏覽器、iOS）中報錯。

#### 在 Console 中測試

```javascript
// 無參數 — 取得回傳值
window.MyBridge.getToken()
// 預期輸出："eyJhbGciOiJIUzI1NiIs..."

// 有參數 — 觸發原生行為
window.MyBridge.showToast("Hello from DevTools")
// 預期：裝置上顯示 Toast

// 無參數 — 觸發頁面操作（注意：會關閉 WebView）
window.MyBridge.goBack()
```

## 常見問題

### Q1：chrome://inspect 看不到裝置

- 確認 USB 偵錯模式已開啟
- 確認已信任電腦的 USB 偵錯授權（裝置上會彈出確認對話框）
- 嘗試拔插 USB 線重新連接
- 執行 `adb devices` 確認裝置已被識別

### Q2：看得到裝置但看不到 WebView

- 確認 App 是 Debug 版本
- 確認已呼叫 `WebView.setWebContentsDebuggingEnabled(true)`
- 確認 WebView 頁面已載入完成

### Q3：Console 中 window.XXX 是 undefined

- 確認 interface 名稱正確（區分大小寫）
- 確認已完全重啟 App（不只是按返回，要從最近任務中滑掉再重新開啟）
- `addJavascriptInterface` 是在 Activity `onCreate` 時注入的，如果 Activity 從快取恢復，新增的方法可能不會生效

### Q4：方法存在但呼叫時出現 "is not a function"

- **參數型別問題**：`@JavascriptInterface` 的 `Boolean` 參數在某些 WebView 版本中無法正確對應 JS 的 boolean。建議改用 `String` 接收再轉換：

```kotlin
// 避免
@JavascriptInterface
fun myMethod(flag: Boolean) { ... }

// 推薦
@JavascriptInterface
fun myMethod(flagStr: String) {
    val flag = flagStr.toBoolean()
    ...
}
```

- **App 未重啟**：安裝新版 APK 後必須完全結束 App 再重新開啟，確保新的 interface 被注入

### Q5：Chrome DevTools Console 無法貼上程式碼

首次在 Console 貼上程式碼時，Chrome 會顯示安全警告：

```
Warning: Don't paste code into the DevTools Console that you don't 
understand or haven't reviewed yourself.
```

在 Console 中輸入 `allow pasting` 按 Enter，之後就可以正常貼上了。

---

# iOS — Safari Web Inspector

在 iOS 開發中，WKWebView 透過 `WKScriptMessageHandler` 提供原生方法給 Web 端呼叫。以下介紹如何使用 Safari Web Inspector 連接裝置上的 WebView 進行即時測試。

## 前置條件

- iOS 裝置透過 USB 連接到 Mac
- Mac 已安裝 Safari 瀏覽器
- App 為 **Debug 版本**

## 步驟一：開啟相關設定

### Mac 端

打開 Safari → 偏好設定 → 進階 → 勾選「**在選單列中顯示『開發』選單**」

### iOS 裝置端

設定 → Safari → 進階 → 開啟「**Web Inspector**」

### App 端（iOS 16.4+）

WKWebView 需要在程式碼中啟用 `isInspectable`：

```swift
let webView = WKWebView(frame: .zero, configuration: configuration)

#if DEBUG
if #available(iOS 16.4, *) {
    webView.isInspectable = true
}
#endif
```

> **注意**：iOS 16.4 之前的 debug build 預設可以 inspect，不需要額外設定。16.4+ 之後 Apple 預設關閉了，必須明確開啟。

## 步驟二：在裝置上開啟目標 WebView 頁面

1. 安裝 Debug 版 App 到裝置
2. 在 App 中導航到包含 WebView 的頁面

## 步驟三：用 Safari Web Inspector 連接 WebView

1. 用 USB 線將 iPhone 連接 Mac
2. 在 iPhone 上打開 App，進入有 WebView 的頁面
3. Mac 上打開 Safari → 選單列「**開發**」→ 找到你的裝置名稱 → 選擇對應的 WebView 頁面
4. 會彈出 Web Inspector 視窗

> **提示**：Xcode Simulator 也可以使用，Safari「開發」選單同樣會出現模擬器的選項。

## 步驟四：Web Inspector 功能介紹

- **Elements** — 檢視/修改 DOM 結構和 CSS
- **Console** — 執行 JS、查看 log、測試 JS Bridge 呼叫
- **Network** — 監控 WebView 內的網路請求
- **Sources** — 設中斷點除錯 JavaScript
- **Storage** — 檢查 cookies、localStorage 等

## 步驟五：在 Console 中測試 JavaScript Interface

### iOS 端定義（WKScriptMessageHandler）

```swift
class WebViewController: UIViewController, WKScriptMessageHandler {

    func setupWebView() {
        let config = WKWebViewConfiguration()
        let contentController = WKUserContentController()

        // 註冊 message handler（"MyBridge" 即為 Web 端呼叫時使用的名稱）
        contentController.add(self, name: "MyBridge")
        config.userContentController = contentController

        let webView = WKWebView(frame: view.bounds, configuration: config)

        #if DEBUG
        if #available(iOS 16.4, *) {
            webView.isInspectable = true
        }
        #endif
    }

    // 接收 Web 端訊息
    func userContentController(
        _ userContentController: WKUserContentController,
        didReceive message: WKScriptMessage
    ) {
        guard message.name == "MyBridge",
              let body = message.body as? [String: Any],
              let action = body["action"] as? String else { return }

        switch action {
        case "showToast":
            let msg = body["message"] as? String ?? ""
            showToast(msg)
        case "goBack":
            navigationController?.popViewController(animated: true)
        default:
            break
        }
    }
}
```

### Web 端呼叫方式

Web 端透過 `window.webkit.messageHandlers` 存取 iOS 注入的 handler：

```javascript
// 有參數 — 觸發原生行為
function showToast(message) {
    if (window.webkit?.messageHandlers?.MyBridge) {
        window.webkit.messageHandlers.MyBridge.postMessage({
            action: "showToast",
            message: message
        })
    }
}

// 無參數 — 觸發頁面操作
function goBack() {
    if (window.webkit?.messageHandlers?.MyBridge) {
        window.webkit.messageHandlers.MyBridge.postMessage({
            action: "goBack"
        })
    }
}
```

> **注意**：與 Android 不同，iOS 的 `postMessage` 是單向的（Web → Native），無法直接取得回傳值。如需回傳，Native 端需透過 `webView.evaluateJavaScript()` 回呼。

### 在 Console 中測試

```javascript
// 確認 handler 是否存在
window.webkit.messageHandlers.MyBridge
// 預期輸出：MessagePort 物件（非 undefined）

// 觸發原生行為
window.webkit.messageHandlers.MyBridge.postMessage({
    action: "showToast",
    message: "Hello from Safari Inspector"
})
// 預期：裝置上顯示 Toast

// 觸發頁面操作（注意：會關閉 WebView）
window.webkit.messageHandlers.MyBridge.postMessage({
    action: "goBack"
})
```

## 常見問題

### Q1：Safari「開發」選單看不到裝置

- 確認 iOS 裝置的 Safari → 進階 → Web Inspector 已開啟
- 確認 USB 線已正確連接，裝置已信任此電腦
- 嘗試重新拔插 USB 線
- 確認 Mac 的 Safari 版本為最新

### Q2：看得到裝置但看不到 WebView

- 確認 App 是 Debug 版本
- iOS 16.4+ 需確認已設定 `webView.isInspectable = true`
- 確認 WebView 頁面已載入完成

### Q3：Console 中 window.webkit.messageHandlers.XXX 是 undefined

- 確認 handler 名稱正確（區分大小寫）
- 確認 `WKUserContentController.add(_:name:)` 已正確呼叫
- 確認已完全重啟 App

