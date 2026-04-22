# Throttle 與 Debounce 的區別

## 概述

Throttle（節流）和 Debounce（防抖）都是用來控制函式執行頻率的技術，常用於處理高頻觸發的事件（如滾動、輸入、視窗縮放等），但它們的行為邏輯不同。

---

## Throttle（節流）

**定義**：在指定的時間間隔內，**最多只執行一次**。不管事件觸發多頻繁，都會以固定的頻率執行。

**行為**：事件持續觸發時，每隔固定時間就會執行一次回呼函式。

**時間軸示意**：

```
事件觸發:  x x x x x x x x x x x x x x x
時間軸:    |-------|-------|-------|-------|
執行:      o       o       o       o
```

**適用場景**：

- 視窗滾動（scroll）事件
- 視窗縮放（resize）事件
- 滑鼠移動（mousemove）事件
- API 頻率限制（rate limiting）

**程式碼範例**：

```kotlin
// Kotlin 實作
fun <T> throttle(intervalMs: Long, action: (T) -> Unit): (T) -> Unit {
    var lastExecutionTime = 0L
    return { param: T ->
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastExecutionTime >= intervalMs) {
            lastExecutionTime = currentTime
            action(param)
        }
    }
}
```

```javascript
// JavaScript 實作
function throttle(fn, delay) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= delay) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}
```

---

## Debounce（防抖）

**定義**：事件觸發後，**等待指定時間不再觸發**，才執行一次。如果在等待期間內再次觸發，則重新計時。

**行為**：只有在事件停止觸發一段時間後，才會執行回呼函式。

**時間軸示意**：

```
事件觸發:  x x x x x x x x              x x x
時間軸:    |-------|-------|-------|       |-------|
執行:                               o                o
                                    ^                 ^
                              停止觸發後才執行    停止觸發後才執行
```

**適用場景**：

- 搜尋框即時搜尋（等使用者停止輸入後才發送請求）
- 表單驗證（等使用者停止輸入後才驗證）
- 按鈕防重複點擊
- 自動儲存草稿

**程式碼範例**：

```kotlin
// Kotlin 實作（搭配 Coroutine）
fun <T> CoroutineScope.debounce(delayMs: Long, action: (T) -> Unit): (T) -> Unit {
    var debounceJob: Job? = null
    return { param: T ->
        debounceJob?.cancel()
        debounceJob = launch {
            delay(delayMs)
            action(param)
        }
    }
}
```

```javascript
// JavaScript 實作
function debounce(fn, delay) {
  let timer = null;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}
```

---

## 比較表

| 比較項目 | Throttle（節流） | Debounce（防抖） |
|---------|-----------------|-----------------|
| 執行時機 | 固定間隔執行一次 | 停止觸發後才執行 |
| 執行次數 | 持續觸發期間會多次執行 | 通常只在最後執行一次 |
| 回應性 | 較即時，定期回應 | 延遲回應，等待穩定後才執行 |
| 適合場景 | 需要持續但有節制地回應 | 只關心最終結果 |

---

## 一句話總結

- **Throttle**：「不管你多急，我每隔一段時間才做一次。」
- **Debounce**：「你先忙完，等你不動了我再做。」
