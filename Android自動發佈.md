# Android 自動發佈到 Google Play 內部測試

使用 [Gradle Play Publisher](https://github.com/Triple-T/gradle-play-publisher) 插件，從命令列自動 build AAB 並上傳到 Google Play 內部測試 track。

---

## 一、取得 Google Play Service Account JSON

### 1. 建立 Service Account

1. 前往 [Google Cloud Console](https://console.cloud.google.com/) → **IAM 與管理** → **服務帳戶**
2. 選擇你的 Google Cloud 專案（需與 Google Play Console 連結的同一個專案）
3. 點擊 **建立服務帳戶**
4. 填寫名稱（例如 `google-play-deploy`），點擊 **建立並繼續**
5. 角色可跳過（權限在 Google Play Console 設定），點擊 **完成**

### 2. 產生 JSON 金鑰

1. 在服務帳戶列表中，點擊剛建立的帳戶
2. 切換到 **金鑰** 分頁
3. 點擊 **新增金鑰** → **建立新金鑰** → 選擇 **JSON**
4. 下載後妥善保存，這就是 `google-play-service-account.json`

金鑰格式如下：
```json
{
  "type": "service_account",
  "project_id": "your-project-id",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n",
  "client_email": "google-play-deploy@your-project.iam.gserviceaccount.com",
  "client_id": "...",
  ...
}
```

### 3. 在 Google Play Console 授予權限

1. 前往 [Google Play Console](https://play.google.com/console) → **設定** → **API 存取權**
2. 連結你的 Google Cloud 專案（如果尚未連結）
3. 在 **服務帳戶** 區塊找到剛建立的帳戶
4. 點擊 **管理權限**，授予以下權限：
   - **版本管理員**（至少需要這個，才能上傳 AAB 和管理 track）
5. 點擊 **邀請使用者** 並確認

> **注意**：新授權的 Service Account 可能需要等待數小時才能生效。

---

## 二、設定 Gradle Play Publisher 插件

### 1. Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
play-publisher = "3.12.1"

[plugins]
play-publisher = { id = "com.github.triplet.play", version.ref = "play-publisher" }
```

### 2. Root `build.gradle.kts`

```kotlin
plugins {
    // ... 其他插件
    alias(libs.plugins.play.publisher) apply false
}
```

### 3. App `build.gradle.kts`

```kotlin
plugins {
    // ... 其他插件
    id("com.github.triplet.play")
}

// 在檔案最後加入
play {
    serviceAccountCredentials.set(file("${rootProject.projectDir}/google-play-service-account.json"))
    track.set("internal")          // 上傳到內部測試 track
    defaultToAppBundles.set(true)  // 使用 AAB 而非 APK
}
```

### 4. 放置金鑰檔案

將 `google-play-service-account.json` 放到專案根目錄。

### 5. 加入 `.gitignore`

```
/google-play-service-account.json
```

> **重要**：絕對不要將 Service Account JSON 提交到 Git，私鑰洩漏需要立即輪換。

---

## 三、使用方式

### 上傳 AAB 到內部測試

```bash
./gradlew publishProdReleaseBundle
```

此指令會自動：
1. Build prod release AAB
2. 上傳到 Google Play 內部測試 track

### 其他可用指令

| 指令 | 說明 |
|------|------|
| `./gradlew publishProdReleaseBundle` | Build + 上傳 prod AAB 到內部測試 |
| `./gradlew publishBundle` | 上傳所有 variant 的 AAB |
| `./gradlew promoteArtifact` | 將內部測試版本推進到下一個 track |

### Track 選項

在 `play {}` 區塊中，`track` 可設定為：

- `"internal"` — 內部測試（預設）
- `"alpha"` — 封閉測試
- `"beta"` — 開放測試
- `"production"` — 正式版

---

## 四、常見問題

### Q: 上傳失敗，顯示 403 Forbidden
Service Account 權限不足或尚未生效。確認已在 Google Play Console 授予「版本管理員」權限，並等待數小時後重試。

### Q: 上傳失敗，顯示 APK/AAB 版本號重複
`versionCode` 必須大於 Google Play 上已有的版本號。修改 `libs.versions.toml` 中的 `appVersionCode` 後重新上傳。

### Q: 找不到 publishProdReleaseBundle task
確認插件已正確設定在 `build.gradle.kts` 中，執行 `./gradlew tasks --group=publishing` 檢查可用的 task。

---

## 五、參考資料

- [Gradle Play Publisher GitHub](https://github.com/Triple-T/gradle-play-publisher)
- [Google Play Developer API 文件](https://developers.google.com/android-publisher)
