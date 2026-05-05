# Android WebView JavaScript Interface 測試教學

在 Android 開發中，WebView 透過 `@JavascriptInterface` 提供原生方法給 Web 端呼叫。本文介紹如何使用 Chrome DevTools 連接裝置上的 WebView 進行即時測試。

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

先輸入 interface 物件名稱確認它已被注入：

```javascript
window.AndroidShow
// 預期輸出：Object（非 undefined）
```

### 列出所有可用方法

```javascript
Object.getOwnPropertyNames(window.AndroidShow)
// 預期輸出：["goToProfile", "goToSearch", "showSoundStateToast", "onForceLogout", ...]
```

### 呼叫測試

```javascript
// 測試帶參數的方法
window.AndroidShow.showSoundStateToast("true")   // 應顯示「聲音已關閉」Toast
window.AndroidShow.showSoundStateToast("false")  // 應顯示「聲音已開啟」Toast

// 測試無參數的方法（注意：此方法會關閉 WebView）
window.AndroidShow.onForceLogout()
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

