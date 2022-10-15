---
weight: 6
title: "如何對付傳統追蹤技術 (III)：Apple’s Intelligent Tracking Prevention (ITP)"
---

# 如何對付傳統追蹤技術 (III)：Apple’s Intelligent Tracking Prevention (ITP)
在介紹完最傳統的 content blocking 後，接下來我們來看點比較新穎的技術。傳統 content blocking 在乎的是盡可能不要載入 tracker，不過現在較為新穎的技術在乎的是，即便真的載入 tracker 了，也要把傷害降到最低。首先我認為最值得拿出來討論的是 Safari 的 Intelligent Tracking Prevention (ITP)。如果平常有在關注蘋果的發表會，應該會發現他們很喜歡強調 Safari 防止 web tracking 的機制做得多好，其中很重要的一環就是 ITP。故此文我們將介紹 ITP 的運作方式。

在開始前想提醒一下，因為 ITP 更新的速度其實蠻快的，在我撰文時 ITP 的最新版是 3.0（其實 3.0 不是官方稱呼，不過官方好像也沒給個版本號），如果有數年後回來的讀者，可能要注意一下有些東西是過時的。原本想說這篇文章可以從我前年的筆記改寫就好，結果 ITP 改的東西多到不如重寫一篇 QQ

## 背景介紹
從 2017 年起，Safari 內建了一款名為 Intelligent Tracking Prevention (ITP) 的隱私保護機制。ITP 的主要目標是解決 cross-site tracking，至於 same-site tracking，雖然也有一些防護機制，但那不是重點。因此 ITP 與之前介紹的 content blocking 的目標略有不同，content blocking 目標是所有 web tracking 機制，無論 same-site 或 cross-site。因為 ITP 內建於 Safari，而 macOS 與 iOS 又有很高的市占率，尤其 iOS 上只有 Safari 可以用（其他都是包著 Safari 皮的瀏覽器，感謝 Apple 的壟斷地位），因此 ITP 對於廣告與追蹤產業的影響力不容小覷。

## 功能
此段我將介紹 ITP 的一些功能。內容架構主要來自 ITP 的[官方文件](https://webkit.org/tracking-prevention/#intelligent-tracking-prevention-itp)，不過因為官方文件有時缺少一些重要的細節或歷史脈絡，所以我會適時地補充。

### 判斷一個 domain 是否是具備 cross-site tracking 的能力
作為一個隱私保護的機制，ITP 的首要任務是判斷出哪些 domain 具有 cross-site tracking 的能力。其總共考慮了三種不同的樣態，其中一個通過了就會被認定為具備 cross-site tracking 能力的 tracking domain。

第一種樣態是，如果這個 domain 的資源頻繁地出現在其他 domain，作為其他網站的 third-party resource，那它很可能是在做 cross-site tracking（舉例來說，幾乎所有網站都能看到 Google Analytics 的蹤影）。其使用了一個 ML classifier 來判斷，考慮的 feature 包含：
- 該 domain 作為 third-party resource 出現在有多少獨立的網站
- 該 domain 作為 third-party iframe 出現在多少獨立網站
- 該 domain 有多少獨立網站會 redirect 過來

第二種樣態是先前提過的 bounce tracking。ITP 會去計算該 domain 做了幾個 redirection，如果做太多 redirection，看起來就很可疑，就可能被 flag 為 tracking domain。

第三種樣態是偵測 tracker collusion，也就是，如果 `trackerA.example` 被認定為是 tracking domain，而 `trackerB.example` 曾經 redirect 到 `trackerA.example`，代表他們可能在分享 tracking 的資訊，因此 `trackerB.example` 也會被標注為 tracking domain。

最後，雖然本文我都籠統地用了 "domain" 這個詞，但其實 ITP 的 tracking domain 辨認是針對 eTLD+1，也就是 `example.com`, `example.co.uk`, `s3131212.github.io` 這種公開地允許大家註冊的 domain。因為與本文無關所以不展開 eTLD+1 的嚴格定義，總之就想像成「我可以找到一個網站，然後就註冊下來的 domain」即可，這種 domain 的特色在於 domain 本身與其 parent 的擁有者不同，例如我擁有 `allenchou.cc` 但不擁有 `cc`，擁有 `s3131212.github.io` 但不擁有 `github.io`。

### 自動刪除 Tracking Domain 的資料
如果 `tracker.example` 被認定為是 tracking domain，則如果使用者在 30 天內沒有造訪過 `tracker.example`，也沒有使用 Storage Access API 明文地同意 `tracker.example` 儲存資料（後文會介紹何謂 Storage Access API），那 ITP 就會在 30 天後把 `tracker.example` 的資料刪除，包含 cookie, Local Storage, IndexedDB 等。

### Tracking Domain 防止 Cache-based Tracking 的方法
我們討論過 [cache 可以被用於 web tracking]({{< relref "/docs/stateful-tracking/storing-identifier#document-cache" >}})。為了阻止 cache-based tracking，每一筆來自 tracking domain 的 cache 在一開始會先被 flag 起來，註記為「待驗證」，如果七天內再次載入該資源（cache hit），則會重新請求該資源，檢查該資源與 cache 內的資源是否一樣，如果 cache 內的資源與新下載的資源是一樣的，則推測這筆 cache 沒有被用於 web tracking，所以就把 flag 拿掉，以後可以直接使用 cache。反之，如果 cache 內的資源與新的資源不一樣，就會把原有的 cache 刪掉，加入新的 cache 並 flag 起來，註記為「待驗證」，如此一方面可以避免 cache-based tracking，又可以避免因為資源更新而被永久地誤判為是在 tracking。

### Bounce Tracking 防禦
在前面已經提過如何找到正在做 bounce tracking 的 tracking domain。對於 bounce tracking 的主要防禦還是在於 ITP 會把這些 tracking domain 的資料清空。除此之外，另一個防禦機制是，ITP 會把 tracking domain 上的 cookie 全部強制設定為 `SameSite=Strict`。如同討論[封鎖 third-party cookie]() 時提過的，`Strict` 的意義是，第一次從其他 domain 跳到該 domain 時，cookie 不會被自動帶入，所以這樣轉跳時路過的 tracking domain 就拿不到自己的 cookie 了。

### Link Decoration 偵測
如果 `social.example` 已經被標注為 tracking domain，則當使用者點擊 `social.example` 上的任一連結時，如果是連往其他 domain 而且該連結含有 query string 或 fragment（URL 的格式大概是 `https://blog.example/post/1?query=string#fragment`），則 `blog.example` 上藉由 Javascript 設定的 cookie 會在一天後被刪掉。

雖然這無法避免 `blog.example` 當下的瀏覽被 `social.example` 追蹤，但至少這樣的追蹤是不持久的，因為如果 `social.example` 的 tracking script 被 `blog.example` 引入後，想要嘗試把 identifier 寫進 `blog.example` 的 first-party cookie，會在一天後被 ITP 刪掉。

### CNAME Cloaking 防禦
ITP 判定 CNAME cloaking 的方式是，如果 first-party subdomain 被 resolve 到 third party（任何不是 first-party domain 的位置），就假定它是 CNAME cloaking，然後把 cookie 限制在 7 天後清空。

### 阻擋所有的 Third-party Cookie
基本上就如同多數其他主流 browser，ITP 預設會阻擋 third-party cookie，這應該已經是所有隱私保護機制必備的功能了，詳情請回頭看討論封鎖 [third-party cookie]({{< relref "/docs/stateful-tracking/cross-site-tracking-with-cookie" >}}) 的文章。

如果想要存取 Third-party domain，也不是不可能，但是需要藉由 [Storage Access API](https://developer.mozilla.org/en-US/docs/Web/API/Storage_Access_API) 得到使用者的同意。Storage Access API 主要是用於解決有時在網頁中引入 cross-origin 的資源會因為沒有帶 third-party cookie 而壞掉的問題，或稍微嚴謹一點來說，third-party resources 會因為沒有辦法讀取自己的 first-party cookie 而壞掉。常見的情境包含 federated login 之類的，至少就我所知會被 third-party cookie blocking 影響到的，除了真的是 cross-site tracking 以外，主要都是做 authentication。

為了解決這個問題，Apple 就提出了 Storage Access API。Third party 可以使用 `document.requestStorageAccess()` 來請求允許讀取自己的 cookie，或是用 `document.hasStorageAccess()` 檢查自己有沒有權限讀取 cookie。

此外，`iframe` 也可以藉由以下屬性使其可以存取 cookie。
```html
<iframe sandbox="allow-storage-access-by-user-activation
                 allow-scripts
                 allow-same-origin">
</iframe>
```

當 third party 請求使用 storage 時，就會跳出這樣的通知，請求使用者同意：
![](https://privacycg.github.io/storage-access/images/storage-access-prompt.png)
（圖片取自 [The Storage Access API](https://privacycg.github.io/storage-access/)）

講個小故事。最早的 ITP 其實不是阻擋所有 third-party cookie，而是使用前述的 ML classifier 來判斷是否該封鎖。如果 classifier 判斷此 domain 成是 tracking domain，則會禁止該 third party 存取自己的 cookie，反之，如果 ITP 認為這不是 tracking domain，就會遵守 Cookie 的 `SameSite` 規範走。仔細整理一下思緒，存取 `example.com` 時的 behavior（有沒有帶 cookie）會因為是否有在 ITP 的 tracking domain 名單裡面而有所不同，且我可以藉由在很多不同 domain 引入 `example.com` 的資源來使 ITP 誤判 `example.com` 是 tracking domain，這不正是 HSTS supercookie 發生的情境嗎！？只要刻意做一些操作讓有些 domain 被放進 ITP 的 tracking domain 名單，就可以藉此儲存 identifier 了。這個攻擊最先被 Google 的研究員發現。為了解決該問題，ITP 就改成一律封鎖了。

### 防止 HSTS Supercookie
為了阻止 HSTS supercookie，ITP 要求只能對 first-party domain 以及 eTLD+1 設定 HSTS。例如我在 `my.blog.example` 時，`tracker.example` 沒辦法將 `a.tracker.example` 或 `tracker-a.example` 加入 HSTS list，唯二可以加入 HSTS list 的是 `my.blog.example` (first-party domain) 與 `blog.example` (eTLD+1)。

### 截斷 HTTP Referrers
HTTP referrer 夾帶於 request header，用於表達「請求從哪個 URL 來的」。但 referrer 可能被用於 link decoration，而且有時 URL 會夾帶一些機密資訊（如 token），所以 ITP 預設把 HTTP referrer truncate 到只剩下 domain，也就是 `https://blog.example/`。除了 HTTP referrer 以外，其實從 Javascript 一樣可以拿到 referrer，一開始 ITP 並沒有修改 `document.referrer`，但之後發現有追蹤者改從 Javascript 拿到 referrer，所以現在是 Javascript 的 referrer 一樣會被截斷。

## 廣告商如何繞過 ITP
ITP 從 2017 出現至今，歷經了五年的演變，許多新增的功能其實都在封鎖那些嘗試繞過 ITP 的追蹤者。此段我將簡短地介紹幾個曾經出現過的或現在仍在使用的繞過方法。

在 ITP 剛出現時，因為 third-party cookie 不可用了，許多追蹤者轉而使用 link decoration 與 CNAME cloaking。前者在 ITP 2.3 (2019) 時被封鎖了，後者則在 2020 年被封鎖了。另外，最一開始只有封鎖 cookie，於是大家就跑去用 Local Storage 與 IndexedDB 等其他儲存空間作為 workaround。於是 ITP 2.3 把所有 Javascript 能寫入的空間都納入管理了，就算寫進 Local Storage 一樣會被刪掉。

Optimizely 是一個專門做 A/B test 的 web analytics 服務提供者。他們建議把 cookie 建立移動到 server side，用 HTTP 的 `Set-Cookie` response header 來處理，因為會被 ITP 的多數功能都是針對那些用 Javascript 建立的 cookie，所以藉由 HTTP 來設定 cookie 可以避免被 ITP 砍掉。此外，Optimizely 提供了一個功能是，網站管理員可以自己產生 identifier，然後再通知 Optimizely 去讀取該 identifier，如此則 identifier 會成為 first-party cookie or local storage，不會被 ITP 針對。另外很有趣的是，Optimizely 還建議使用者可以考慮直接把 Safari 從實驗結果中移除，避免實驗數據被 ITP 污染。

總之，套一句 Adobe 在他們的[文件](https://developer.adobe.com/target/before-implement/privacy/apple-itp-2x/?lang=en)中的哀號：Apple 阻擋 third-party cookie 已經讓廣告商與科技公司夠頭痛了，他們現在又更進一步封鎖更多東西了！

看起來他們也蠻絕望的。

## 參考資料
Intelligent Tracking Prevention：[spec](https://webkit.org/tracking-prevention/#intelligent-tracking-prevention-itp), [1.0](https://webkit.org/blog/7675/intelligent-tracking-prevention/) [1.1](https://webkit.org/blog/8142/intelligent-tracking-prevention-1-1/) [2.0](https://webkit.org/blog/8311/intelligent-tracking-prevention-2-0/) [2.1](https://webkit.org/blog/8613/intelligent-tracking-prevention-2-1/) [2.2](https://webkit.org/blog/8828/intelligent-tracking-prevention-2-2/) [2.3](https://webkit.org/blog/9521/intelligent-tracking-prevention-2-3/)
[CNAME Cloaking and Bounce Tracking Defense | WebKit](https://webkit.org/blog/11338/cname-cloaking-and-bounce-tracking-defense/)
[Preventing Tracking Prevention Tracking | WebKit](https://webkit.org/blog/9661/preventing-tracking-prevention-tracking/)
[Full Third-Party Cookie Blocking and More | WebKit](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/)
[Intelligent Tracking Prevention and Optimizely](https://help.optimizely.com/Set_Up_Optimizely/Intelligent_Tracking_Prevention_and_Optimizely#External_Mitigations_(full))