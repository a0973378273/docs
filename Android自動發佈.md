# Android 自動發佈

本文件涵蓋兩種自動發佈方式：
1. **Firebase App Distribution** — 發佈測試版本給內部測試團隊
2. **Google Play 內部測試** — 上傳 AAB 到 Google Play Console

---

# 一、Firebase App Distribution

透過 Firebase App Distribution 插件，從命令列自動 build 並上傳 APK/AAB 給測試群組。

## 1. 取得必要檔案

### firebase-service-account.json

1. 前往 [Firebase Console](https://console.firebase.google.com/) → **專案總覽** → **專案設定**
2. 切換到 **服務帳戶** 分頁
3. 選擇 **Java**，點擊 **產生新的私密金鑰**
4. 下載後放到**專案根目錄**

### google-services.json

1. 前往 [Firebase Console](https://console.firebase.google.com/) → **專案總覽** → **專案設定**
2. 下載 `google-services.json`（若尚未新增應用程式，先點「新增應用程式」）
3. 依照環境放置：
   - `根目錄/app/src/prod/google-services.json`
   - `根目錄/app/src/uat/google-services.json`

## 2. Gradle 設定

### build.gradle (Project)

```groovy
plugins {
    id 'com.google.firebase.appdistribution' version '5.0.0' apply false
}
```

### build.gradle (Module)

#### 新增插件

```groovy
plugins {
    id 'com.google.firebase.appdistribution'
}
```

#### 設定各環境的 Firebase App Distribution

```groovy
android {
    productFlavors {
        uat {
            firebaseAppDistribution {
                artifactType = "APK"
                releaseNotesFile = "app/src/main/assets/release-notes.txt"
                groups = "Dashflix,group2,group3"  // 測試群組名稱
                serviceCredentialsFile = "${rootProject.projectDir}/firebase-service-account.json"
            }
        }

        prod {
            firebaseAppDistribution {
                artifactType = "AAB"
                releaseNotesFile = "app/src/main/assets/release-notes.txt"
                groups = "Dashflix"
                serviceCredentialsFile = "${rootProject.projectDir}/firebase-service-account.json"
            }
        }
    }
}
```

> UAT 為測試環境（上傳 APK），prod 為正式環境（上傳 AAB）。

#### 新增上傳任務

```groovy
// 建立 release notes 檔案
task createReleaseNotes {
    group 'deployment'
    description '建立 release notes 檔案'

    doLast {
        def releaseNotesDir = file("src/main/assets")
        if (!releaseNotesDir.exists()) {
            releaseNotesDir.mkdirs()
        }

        def releaseNotesFile = file("src/main/assets/release-notes.txt")
        def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone("GMT+08:00"))
        def gitHash = "git rev-parse --short HEAD".execute().text.trim()

        releaseNotesFile.text = """
Release Build
建置時間: ${timestamp}
Git Commit: ${gitHash}
版本: ${android.defaultConfig.versionName} (${android.defaultConfig.versionCode})

主要更新:
- 修正某某問題
- 新增某某功能
        """.trim()

        println "Release notes 已建立: ${releaseNotesFile.absolutePath}"
    }
}

// 建置並上傳 UAT Release APK
task buildAndUploadUatRelease {
    group 'deployment'
    description '建置 UAT Release 版本並上傳到 Firebase App Distribution'

    dependsOn 'createReleaseNotes', 'assembleUatRelease'
    finalizedBy 'appDistributionUploadUatRelease'

    doFirst {
        println "開始建置 UAT Release 版本..."
    }

    doLast {
        println "UAT Release 版本建置完成並上傳到 Firebase App Distribution"
    }
}

// 建置並上傳 Prod Release AAB
task buildAndUploadProdRelease {
    group 'deployment'
    description '建置 Prod Release AAB 版本並上傳到 Firebase App Distribution'

    dependsOn 'createReleaseNotes', 'bundleProdRelease'
    finalizedBy 'appDistributionUploadProdRelease'

    doFirst {
        println "開始建置 Prod Release AAB 版本..."
    }

    doLast {
        println "Prod Release AAB 版本建置完成並上傳到 Firebase App Distribution"
    }
}

// 同時建置並上傳 UAT + Prod
task buildAndUploadBothReleases {
    group 'deployment'
    description '同時建置並上傳 UAT Release APK 和 Prod Release AAB'

    dependsOn 'buildAndUploadUatRelease', 'buildAndUploadProdRelease'

    doFirst {
        println "開始同時建置 UAT Release APK 和 Prod Release AAB..."
    }

    doLast {
        println "UAT Release APK 和 Prod Release AAB 都已建置完成並上傳"
    }
}
```

## 3. 使用方式

```bash
# 上傳 UAT 測試版
./gradlew buildAndUploadUatRelease

# 上傳 Prod 版本
./gradlew buildAndUploadProdRelease

# 同時上傳 UAT + Prod
./gradlew buildAndUploadBothReleases
```

## 4. .gitignore

```
/firebase-service-account.json
/google-services.json
```

---

# 二、Google Play 內部測試

使用 [Gradle Play Publisher](https://github.com/Triple-T/gradle-play-publisher) 插件，從命令列自動 build AAB 並上傳到 Google Play 內部測試 track。

## 1. 取得 Google Play Service Account JSON

### 建立 Service Account

1. 前往 [Google Cloud Console](https://console.cloud.google.com/) → **IAM 與管理** → **服務帳戶**
2. 選擇你的 Google Cloud 專案（需與 Google Play Console 連結的同一個專案）
3. 點擊 **建立服務帳戶**
4. 填寫名稱（例如 `google-play-deploy`），點擊 **建立並繼續**
5. 角色可跳過（權限在 Google Play Console 設定），點擊 **完成**

### 產生 JSON 金鑰

1. 在服務帳戶列表中，點擊剛建立的帳戶
2. 切換到 **金鑰** 分頁
3. 點擊 **新增金鑰** → **建立新金鑰** → 選擇 **JSON**
4. 下載後放到**專案根目錄**，命名為 `google-play-service-account.json`

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

### 在 Google Play Console 授予權限

1. 前往 [Google Play Console](https://play.google.com/console) → **設定** → **API 存取權**
2. 連結你的 Google Cloud 專案（如果尚未連結）
3. 在 **服務帳戶** 區塊找到剛建立的帳戶
4. 點擊 **管理權限**，授予 **版本管理員** 權限
5. 點擊 **邀請使用者** 並確認

> **注意**：新授權的 Service Account 可能需要等待數小時才能生效。

## 2. Gradle 設定

### Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
play-publisher = "3.12.1"

[plugins]
play-publisher = { id = "com.github.triplet.play", version.ref = "play-publisher" }
```

### build.gradle.kts (Project)

```kotlin
plugins {
    // ... 其他插件
    alias(libs.plugins.play.publisher) apply false
}
```

### build.gradle.kts (Module)

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

## 3. 使用方式

```bash
# 上傳 prod AAB 到內部測試
./gradlew publishProdReleaseBundle

# 上傳所有 variant 的 AAB
./gradlew publishBundle

# 將內部測試版本推進到下一個 track
./gradlew promoteArtifact
```

### Track 選項

在 `play {}` 區塊中，`track` 可設定為：

| Track | 說明 |
|-------|------|
| `"internal"` | 內部測試（預設） |
| `"alpha"` | 封閉測試 |
| `"beta"` | 開放測試 |
| `"production"` | 正式版 |

## 4. .gitignore

```
/google-play-service-account.json
```

> **重要**：絕對不要將 Service Account JSON 提交到 Git，私鑰洩漏需要立即輪換。

---

# 三、常見問題

### Q: Firebase 上傳失敗，找不到 service account 檔案
確認 `firebase-service-account.json` 放在專案根目錄，且路徑與 `serviceCredentialsFile` 設定一致。

### Q: Google Play 上傳失敗，顯示 403 Forbidden
Service Account 權限不足或尚未生效。確認已在 Google Play Console 授予「版本管理員」權限，並等待數小時後重試。

### Q: 上傳失敗，顯示版本號重複
`versionCode` 必須大於已上傳的版本號。修改 `libs.versions.toml` 中的 `appVersionCode` 後重新上傳。

### Q: 找不到 publishProdReleaseBundle task
確認 `com.github.triplet.play` 插件已正確設定，執行 `./gradlew tasks --group=publishing` 檢查可用的 task。

---

# 四、參考資料

- [Firebase App Distribution Gradle Plugin](https://firebase.google.com/docs/app-distribution/android/distribute-gradle)
- [Gradle Play Publisher GitHub](https://github.com/Triple-T/gradle-play-publisher)
- [Google Play Developer API](https://developers.google.com/android-publisher)
