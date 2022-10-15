---
weight: 1
title: "那些可以洩漏資訊的 Attributes"
---

# 那些可以洩漏資訊的 Attributes
第一篇討論技術的文章可能稍微無聊一些些，我們要來細數有哪些伺服器端或 Javascript 可得的 attribute 可以作為 browser fingerprinting 的資料來源，並討論當今有哪些解法。

## User Agent
User Agent 是夾帶於 HTTP request header 的字串，其透漏了使用者的作業系統、瀏覽器與其版本等。

舉例來說，我目前的 User Agent 是：
```
Mozilla/5.0 (X11; Linux x86_64; rv:103.0) Gecko/20100101 Firefox/103.0
```
這就透漏了我是 Linux 使用者，windowing system 是  X11，然後瀏覽器是 Firefox 103.0。

[AmIUnique](https://amiunique.org/fp) 的統計顯示只有不到 0.37% 的人跟我有一樣的 user agent，大概是因為我用 Linux 吧。

除了在 HTTP request header 以外，也可以用 `navigator.userAgent` 拿到。另外，如果只要拿 OS，也可以用 `navigator.platform`。

不過 user agent 因為洩漏過多資訊而可以用來做追蹤已經廣為人知，所以許多瀏覽器開始盡可能讓 user agent 縮短並盡可能不洩漏太多資訊，以及有些 anti-tracking browser extension 也會竄改 user agent。所以在 2022 年，user agent 可能沒什麼幫助。

題外話，user agent 有著非常精彩的歷史，大家可以看看這兩篇文章，感受一下 web 世界的美好：[History of the browser user-agent string](https://webaim.org/blog/user-agent-string-history/), [History of the user-agent string](https://humanwhocodes.com/blog/2010/01/12/history-of-the-user-agent-string/)。

## IP Address
恩... 好像也不知道怎麼解釋，就 IP address，通常藉由 IP address 就可以把使用者定位在很小一個 group 了。不過由於 IP address 常常會變，尤其在行動世代，我會一直換基地台、換 WiFi AP，偶而掛個 VPN，IP address 已經因為不夠穩定而沒什麼參考價值了。

## Accept
Browser 會使用 HTTP request header `Accept` 來向伺服器表達 browser 支援哪些檔案格式，藉此讓伺服器可以決定可以回應哪種檔案格式。對於不同類型的 request 會有不同的 `Accept` header，而這個值會隨著 browser 以及電腦中有安裝哪些 decoder 之類的而有所影響。

舉例來說，我的 browser 在請求 HTML 檔案時使用的是：
```
Accept:text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
```
[AmIUnique](https://amiunique.org/fp) 的統計顯示只有三成的人跟我有一樣的 `Accept` 值。

使用 `<img>` 請求圖片時是：
```
Accept: image/avif,image/webp,*/*
```

使用 `<link rel="stylesheet" href="..." />` 請求 CSS 檔案時是：
```
Accept: text/css,*/*;q=0.1
```

我不清楚 `Accept` header 在 2022 年多有效，畢竟現在大家都照標準走了，可能只有有沒有啟用一些特殊格式的 decoder 會有影響。但在十年前，那時 Safari 和 IE 的 `Accept` [都在亂寫](https://www.newmediacampaigns.com/blog/browser-rest-http-accept-headers)（恩，IE，有多久沒聽到這名字了？），所以可以使這兩款 browser 在 fingerprinting 上格外地有特色。

## Accept-Encoding
另一個與 `Accept` 密切相關的是 `Accept-Encoding`，表達瀏覽器支援哪些壓縮格式。

例如我的是：
```
Accept-Encoding: gzip, deflate, br
```

## Accept-Language
再一個 `Accept` 家族的成員，`Accept-Language`是使用者偏好什麼語言，這個的 entropy 就比前面其他 `Accept` 家族成員都還要高，畢竟不同人有不同的語言偏好。

例如我的是：
```
Accept-Language: zh-TW,en-US;q=0.7,en;q=0.3
```

JS 可以用 `navigator.language` 或 `navigator.languages` 拿到。

[AmIUnique](https://amiunique.org/fp) 的統計顯示只有不到 0.01% 的人跟我有一樣的 `Accept-Language` 值。看來用 `zh-TW` 的人很少呢。

## 螢幕資訊
為了讓 Javascript 可以根據螢幕的狀況調整 UI 呈現，有個 API 是 `screen`，可以存取與螢幕有關的一些資訊，像是長、寬、可用範圍、color depth 之類的。

- 螢幕寬：`screen.width`
- 螢幕高：`screen.height`
- color depth: `screen.colorDepth`
- 網頁可用的範圍（也就是扣除畫面上方 URL bar 之類的）：`screen.availTop`, `screen.availLeft`
- 螢幕方向：`screen.orientation.type`（不過桌機都只會拿到 `landscape-primary`）

也許讀者會覺得這很廢，但其實大家的螢幕大小與其他性質的多樣性可能比直覺得猜想還要更大，因此光是透過螢幕資訊，其實就可以獲得不少資訊。至少我自己的螢幕資訊在 AmIUnique 上又是小於 0.01% 啦，即便我是用非常標準的 4K 螢幕。

更多資訊可以到 [MDN 的文件](https://developer.mozilla.org/en-US/docs/Web/API/Screen)上看。

## Timezone
就... 時區。Javascript 可以從 `new Date().getTimezoneOffset()` 拿到。

## DNT (Do Not Track)
DNT 是一個 HTTP request header 中的 entry，它用於向追蹤者表達「我不想被追蹤」，就這樣而已，它沒辦法阻止追蹤者嘗試追蹤，只能夠單純地表達使用者的意願而已，如果追蹤者執意要追蹤，DNT 什麼也做不了。

DNT 在 HTTP request header 裡面是：
```
DNT: 0
DNT: 1
```
在 Javascript 中可以用 `navigator.doNotTrack` 取得。

嘲諷的是，因為設定 DNT 的人太少了，所以使用 DNT 反而有可能讓使用者在人群中特別突出（也就是給出很大的 surprisal）。因此 DNT 現在已經 deprecated 了，也不推薦大家開啟 DNT，不過基本上主流瀏覽器幾乎都還是支援 DNT，只有 Safari 已經拿掉了。

## 特殊功能啟用狀態
有點難定義何謂特殊功能，但總之對於有些不是全部人都會開啟的功能，可以拿這些資訊來做 fingerprinting。例如 cookie 可以用 `navigator.cookieEnabled` 來檢查是否有啟用，Local Storage 可以嘗試[直接寫個東西看會不會噴 exception](https://stackoverflow.com/questions/16427636/check-if-localstorage-is-available)。

可能有些讀者會很疑惑，大家不都在使用 cookie 與 local storage 嗎？不過如同前面 DNT 時的討論，正是因為大家都在用，所以不使用的人會特別突出，於是如果有人不使用，這個 feature 就會貢獻很大的 surprisal。


這篇文章我們細數了一些常常用來做 fingerprinting 的 browser attributes。當然，正如一再強調的，這些資訊還不足以提供足夠好的 fingerprinting，接下來幾篇文章會再介紹更多 browser fingerprinting 的技術。