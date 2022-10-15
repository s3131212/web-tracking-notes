---
weight: 7
title: "如何對付傳統追蹤技術 (IV)：Firefox’s Tracking Protection"
---

# 如何對付傳統追蹤技術 (IV)：Firefox’s Tracking Protection
看完 Safari 的隱私保護，下一個來看 Firefox 的保護機制。Mozilla 也同樣在隱私上花了許多心力，開發了許多精彩的防禦機制。Firefox 的保護機制大概分成幾個部份，我們將依序介紹之。

## Enhanced Tracking Protection
Firefox 最主要的防禦機制是 Enhanced Tracking Protection (ETP)。雖然取了一個很潮的名字，但 ETP 本質上就是 content blocking，不過 Mozilla 花了許多心思在確保網站不會因為 content blocking 而壞掉。

ETP 所使用的 filter list 來自 [Disconnect](https://disconnect.me/trackerprotection)，並且會去區分不同性質的 tracker（例如 social media trackers, cross-site tracking cookies, fingerprinters, cryptominers 等），此外，ETP 也有區分模式，Standard 會阻擋多數的 tracker，Strict 則會連那些躲在廣告或影片播放器裡面的 tracker 都擋掉，當然使用者也可以根據自己的需求去選擇哪些 tracker 要阻擋掉。

![](/images/firefox-etp.png)

另外 ETP 有個很重要的功能稱為 SmartBlock，其主要是想在不犧牲太多隱私的情況下提升可用性，盡可能確保網站不會因為 tracker 擋掉而壞掉。例如以 Facebook 登入的功能，許多 content blocker 會因為這有 Facebook 的 tracker 而全部封鎖，但 SmartBlock 會載入那些登入功能的 script 並持續阻擋其他 tracker，所以 Facebook 登入功能可以正常運作。

## Total Cookie Protection
Total Cookie Protection (TCP) 是 ETP 的一個延伸功能。其概念是，把每個 first party 都有專屬於自己的 cookie jar（儲存 cookie 的空間），裡面會有 first-party 與 third-party cookie，且不同 first party 的 cookie jar 是隔開來。

舉例來說，`blog.example` 與 `news.example` 都使用了 `tracker.example` 的 tracker。在 `blog.example` 上，`tracker.example` 的 cookie 中有個 identifier `tracker_id=blog_123`，但是在 `news.example` 上，`tracker.example` 看不到這個 cookie，因為 first party 不同，TCP 就把 cookie 分開來儲存了。當然，如果在 `news.example` 上 `tracker.example` 又 assign 了新的 identifier `tracker_id=news_abc`，回到 `blog.example`，`tracker.example` 還是只會看到專屬於這個 first party 的 cookie，即 `tracker_id=blog_123`。

於是，就算要用 third-party cookie 追蹤，也只能追蹤到個別網站上的行為，基本上就是 reduce 成 same-site tracking 了。這是在 third-party cookie 的隱私危害與方便性之間做的權衡，直接封鎖所有 third-party cookie 可能會讓很多功能壞掉（看看 ITP 讓多少網站開發者崩潰），但顯然開放 third-party cookie 也不是個好選項，藉由 TCP，基於 third-party cookie 的功能還是可以使用，但做不到 cross-site tracking 了。

## Enhanced Cookie Clearing
基本上所有瀏覽器都有清除 cookie 的功能。當我選擇清除 `blog.example` 的 cookie 時，所有屬於 `blog.example` 的 cookie 都會被刪掉。不過傳統清除 cookie 的問題在於，它不會連 third-party cookie 都清掉。舉例來說，如果 `blog.example` 引入了 `tracker.example` 的 tracker，就算 `blog.example` 的 cookie 不見了，`tracker.example` 的 tracker 還在啊，於是當我再次造訪 `blog.example` 時，`tracker.example` 還是知道我是重複訪客。

Enhanced Cookie Clearing 是 TCP 的延伸，其基於 TCP 的特性，嘗試解決這個問題。在 TCP 中，每個 first-party domain 都有自己的 cookie jar 來儲存位於該 domain 的 first-party & third-party cookie，於是，我只要把該 domain 的 cookie jar 清空即可。

沿用 TCP 的例子。當我用 Enhanced Cookie Clearing 把 `blog.example` 的 cookie 清空，會被刪除的不只有 `blog.example` 自己的 cookie，還有 `tracker.example` 在 `blog.example` 上的 cookie（`tracker_id=blog_123`）。可是同時，`tracker.example` 在 `news.example` 上的 cookie（`tracker_id=news_abc`）完全不會被影響到。

藉由這樣的設計，使「清空 cookie」功能變成真正意義上的清空 cookie。

## Redirect Tracking (Bounce Tracking) Protection
Firefox 同樣有針對 bounce tracking 的防禦機制，他們稱之為 Redirection Tracking Protection。首先，ETP 已經有一系列 tracker 的名單，其中有些可能是個 domain，姑且稱之為 tracking domain。Firefox 會去檢查這些 tracking domain 裡面有沒有哪些 domain 是有 cookie 或使用其他 storage，但其實根本沒與其互動過的。試想如果一個 tracking domain 只作為 bounce tracker，使用者一造訪該 tracking domain 就會立刻被 redirect 走，不會與其有互動。因此如果有些 tracking domain 有使用 storage，使用者卻沒與之互動過，大概就很可能是 bounce tracker。Firefox 會把那些超過 45 天沒互動的 tracking domain 的 storage 清空。


## 結語
可能會有讀者疑惑，為什麼我不討論 Brave 的隱私保護機制，明明 Brave 才是花最多心力在阻擋 web tracking 上的瀏覽器。Brave 的確做得很好，不過因為 Brave 的隱私保護機制實在太多而且又太複雜，把它講完可能鐵人賽都比完了，更不用說其所需要的背景知識多到實在沒辦法三言兩語帶過。考慮到 Brave 的市占率比較低，且 Safari 與 Firefox 基本上已經涵蓋了多數主流的 anti-tracking 技術，所以只挑了這兩個 browser 出來討論。至於 Brave 的機制中比較有趣且獨特的部份，有些我在以前的文章中已經提過了，有些我在未來的文章可能也會零碎地提到，雖然可能沒有專文介紹，但應該還是會碰到不少值得一提的技術。

至此，傳統 stateful web tracking 方法及其防禦已經討論得差不多了。從下一篇開始我們會進入全新的單元：browser fingerprinting。

## 參考資料
[Enhanced Tracking Protection in Firefox for desktop](https://support.mozilla.org/en-US/kb/enhanced-tracking-protection-firefox-desktop)
[Firefox rolls out Total Cookie Protection by default to all users worldwide](https://blog.mozilla.org/en/security/firefox-rolls-out-total-cookie-protection-by-default-to-all-users-worldwide/)
[SmartBlock for Enhanced Tracking Protection](https://support.mozilla.org/en-US/kb/smartblock-enhanced-tracking-protection)
[Firefox 91 Introduces Enhanced Cookie Clearing](https://blog.mozilla.org/security/2021/08/10/firefox-91-introduces-enhanced-cookie-clearing/)
[Redirect tracking protection - MDN](https://developer.mozilla.org/en-US/docs/Web/Privacy/Redirect_tracking_protection)