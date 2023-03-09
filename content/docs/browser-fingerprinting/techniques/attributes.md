---
weight: 1
title: "Passive Fingerprinting"
---

# Passive Fingerprinting
我們首先從 passive fingerprinting 開始談起。Passive fingerprinting 是那些只從請求內容就可以得出的特徵。此章節介紹三種：IP Address、HTTP request header、TLS。這些特徵用於 fingerprinting 時都不會使用到 JavaScript，只需要檢視請求內容即可。

## IP Address
IP address 幾乎可以說是最早用於 web tracking 的特徵，通常藉由 IP address 便可以把使用者定位在很小一個群體了。不過由於 IP address 時常變換，尤其在行動世代，使用者會一直換基地台、換 WiFi AP，偶而掛個 VPN，且大家共享 public IP address 的狀況已經是常態，IP address 已經沒什麼參考價值了。

## User Agent
User agent 是夾帶於 HTTP request header 的字串，其透漏了使用者的作業系統、瀏覽器與其版本等。舉例來說，我所使用的瀏覽器目前的 User Agent 是：

```
Mozilla/5.0 (X11; Linux x86_64; rv:103.0) Gecko/20100101 Firefox/103.0
```

這就透漏了我是 Linux 使用者，windowing system 是 X11，然後瀏覽器是 Firefox 103.0。AmIUnique 的統計顯示只有不到 0.37% 的人跟我有一樣的 user agent。

不過因為 user agent 洩漏過多資訊而可以用來做追蹤已經廣為人知，所以許多瀏覽器開始盡可能讓 user agent不洩漏太多資訊。Chrome 從 2021 年起開始便開始陸續簡化 user agent，將小版本號（minor version number）移除，例如 `Chrome/93.0.1234.56` 變成 `Chrome/93.0.0.0`，並簡化了平台資訊，例如 `Linux; Android 9; SM-A205U` 變為 `Linux; Android 10; K`，藉由移除細部資訊使 user agent 變得不獨特。此外，也有些browser extension 可以修改 user agent 來混淆 tracker。因此 user agent 用於 fingerprinting 的價值如今已經不高。

題外話，user agent 有著非常精彩的歷史，大家可以看看這兩篇文章，感受一下 web 世界的美好：[History of the browser user-agent string](https://webaim.org/blog/user-agent-string-history/), [History of the user-agent string](https://humanwhocodes.com/blog/2010/01/12/history-of-the-user-agent-string/)。

## Accept
rowser 會使用 HTTP request header `Accept` 來向伺服器表達 browser 支援哪些檔案格式，藉此讓伺服器可以決定可以回應哪種檔案格式。對於不同類型的 request 會有不同的 Accept header，而這個值會隨著 browser 以及電腦中有安裝哪些 decoder 之類的而有所影響。

例如我所使用的瀏覽器在請求一般網頁時使用的 `Accept` 值是：
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
為了讓開發者可以根據螢幕的狀況調整網頁介面呈現，JavaScript 與 CSS 都可以存取視窗大小。除此之外，JavaScript 也可以取得螢幕大小。由於使用者通常不會一直換螢幕，而且大家用的螢幕大小多少有些不同，所以螢幕大小可以提供一定程度的穩定性與 entropy。與螢幕長寬不同，視窗大小通常有許多變化，數值非常多元，但相對的也就不是很穩定。更糟糕的是，當視窗不是被放到最大時，因為視窗大小的數字通常足夠獨特，若是搭配其他資訊，則追蹤者有機會將同視窗的兩個不同分頁串連起來，得知這兩個網頁是由同一個使用者所瀏覽，藉此實現 cross-site tracking。

JavaScript 有幾個 API 可以存取螢幕長寬與視窗大小。螢幕長寬分別可以用 `screen.width` 與 `screen.height` 取得。視窗大小則可以用 `window.width` 與 `window.height` 取得。最後則是螢幕可用的範圍，也就是扣除畫面上方 URL bar 之類的，可以用 `screen.availTop` 與 `screen.availLeft` 取得。它與螢幕大小相比，有更多可能數值，但也更不穩定。

關於螢幕資訊還有一些其他可用資訊，例如可以用 `screen.colorDepth` 取得螢幕色彩深度，用 `screen.orientation.type` 取得螢幕方向，不過桌機都只會拿到 `landscape-primary`。

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

## TLS Fingerprinting
Transport Layer Security (TLS) 是個用於將客戶端與伺服器之間的流量加密的協定。HTTPS 是在以 HTTP 為基礎並使用 TLS 加密，所以 HTTPS 幾乎必然使用 TLS。在開啟 TLS 通訊時，客戶端與伺服器會先經過 TLS handshake。TLS fingerprinting 是利用 TLS handshake 時客戶端傳送的訊息，從中尋找足夠獨特的特徵。

TLS handshake 的第一則訊息是客戶端傳給伺服器一個 Client Hello，並帶有 TLS 協定版本（TLS 1.0 至 1.3）、支援的加密演算法（cipher suite）、支援的數位簽章。接下來的細節並不重要，總之最後結論是客戶端與伺服器會共同協商出雙方都可以接受的加密與簽章演算法，並得出相同的 session key。

TLS fingerprinting 的重點在於，不同客戶端支援的加密與簽章演算法並不完全一樣，也有著不一樣的 TLS extensions。Firefox 使用 NSS、Chrome 使用 BoringSSL、Safari 使用 Apple Secure Transport Layer。不同的實作可能會產生不同的結果。

舉例來說，瀏覽同個網站，Chrome 與 Firefox 所選擇使用的加密與簽章演算法並不一樣。Firefox 使用 AES-256 GCM SHA 384，而 Chrome 選擇用 NIST P-256 與 AES-256 GCM。

也因為不同瀏覽器有不同的行為，即便瀏覽器偽造了 user agent，光靠 TLS fingerprinting，仍然可以猜到使用者真正的瀏覽器為何。