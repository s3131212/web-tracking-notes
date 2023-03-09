---
weight: 8
title: "Ad Blocker Fingerprinting"
---

# Ad Blocker Fingerprinting

曾經我們討論過，阻擋 tracker 的方法中，最廣泛被使用的是 filter list，也就是去列出我想要阻擋的黑名單，然後 ad blocker 會根據 filter list 去阻擋 tracker。Ad blocker 不但可以改善使用體驗，還可以藉由阻擋 tracker 來減少隱私侵害。Ad blocker 的運作方式已經在前文討論過，簡單來說，ad blocker 會阻擋 filter list 上指定的 HTTP request 以及隱藏指定的 DOM element，而這些改變是可以被有效偵測的。

另外，在 Ad blocker 的世界中，並不存在一個世界統一的 filter list，雖然會有幾個最大咖的（e.g. Easylist），但也會有眾多不那麼知名的但同樣好用的（e.g. Peter Lowe’s Ad and tracking server list）、針對特定語系的（e.g. AdGuard Japanese）、針對特定類型內容的（e.g. Online Malicious URL Blocklist）。如果全部 filter list 都啟用，可能造成效能不彰，所以使用者會根據自己需求去挑選他需要的 filter list，或是 ad blocker 可能會根據系統語言去自動地啟用相對應的 filter list。於是作為中文使用者，我會使用針對中文網站的 filter list（e.g. Easylist China），但不會去用針對德文網站的 filter list（e.g. EasyList Germany），反正我也不會瀏覽德文的網站。

![](/images/ublock-origin-filter-lists.png)

值得注意的是 ad blocker 除了基於 filter list 的方法以外，也有基於 Machine Learning 或 heuristic 的，但因為主流的還是 filter list，所以此文還是直接稱呼這些基於 filter list 的 ad blocker 為 ad blocker。

至此已經可以看出問題所在：如果每個使用者啟用的 filter list 不一樣，tracker 則可以根據使用者啟用了哪些 filter list 去識別使用者，而且哪些 filter list 有被啟用，可以使用網頁行為改變的狀態來得知，於是這就成了可供 fingerprinting 的資訊來源。當然我們可以想像多數使用者都不會去改 ad blocker 的設定，而且同個語系的人使用的 filter list 大概也都差不多，但這仍然是個資訊洩漏。

## 如何從 Ad Blocker 獲得資訊
正如之前介紹過的，ad blocker 主要分成兩種不同策略：阻擋 requests（resource blocking）與隱藏/刪除 elements（element hiding）。這兩者也就對應到兩種偵測策略：去檢視哪些 request 被阻擋了（resource blocking），以及去檢視哪些 elements 被隱藏了（element hiding）。

### 從 resource blocking 取得資訊
如果我們現在有 A, B, C 三個 filter list，想要找出每個使用者用了哪些 filter list，可以這麼做：
1. 預處理：找出 A, B, C 分別有哪些 URI 是只有該 filter list 有阻擋，其他都沒有的。
2. 嘗試發一個 request 到該 URI，看會不會成功
3. 如果 request 成功，則該 filter list 沒有啟用，反之，如果 request 被阻擋則代表可能有啟用該 filter list

如果沒有一個 URL 是只存在於唯一 filter list，也還是可以用排容原理來找出哪些 filter list 有被啟用。可能已經有讀者注意到：其實我們根本不需要還原出哪個 filter list 有被啟用，只需要注意哪些 URL 有被阻擋即可。但這麼做的壞處是，如果 filter list 內容有所變動時，以 filter list 啟用狀態作為判別依據的 fingerprint 將不會改變，以 URL 阻擋狀態作為判別依據的 fingerprint 則會出現系統性的偏移。

不過這方法有個顯然的缺點：發 HTTP request 很耗時、耗能、佔用頻寬，而且使用者如果看到一個網站不斷在載入各種資源，也很可能會起疑。如果每個 filter list 都有一個唯一的 URI 是別人沒有的，那也許還可以接受，但如果要使用排容原理，所需的請求量很快就會飆升上去。另外，請求失敗的原因很多，可能是網路不穩定、遭防火牆攔截等，不一定是 ad blocker 造成的。所以一般來說不會用 request blocking 來做 fingerprinting，畢竟我們有效率更好的方法。

### 從 element hiding 取得資訊
第二種策略是利用 element hiding 取得資訊，也就是去檢驗哪些 element 會被隱藏。

在執行上也和 request blocking 類似，假設我們現在有 A, B, C 三個 filter list，想要找出每個使用者用了哪些 filter list：
1. 預處理：找出 A, B, C 分別有哪些 element 是只有該 filter list 有阻擋，其他都沒有的。通常我們會拿到一個 HTML class 或 id 或 tag name 等資訊。
2. 嘗試在網頁上建立符合條件的空白 element（例如賦予指定的 class 或 id 等），並檢驗該 element 有沒有被隱藏
3. 如果該 element 正常顯示，則該 filter list 沒有啟用，反之，如果 element 被隱藏了，則代表可能有啟用該 filter list

Tracker可以利用 `offsetParent` 檢驗一個 element 是否正常顯示，當一個 element 或是其 parent 被隱藏時，它的 `offsetParent` 會是 null。

不過這個方法並不是萬無一失的。網站可能自己有 CSS 或是一些 script 去處理這些 element，可能會導致錯誤判斷。例如，如果 CSS 裡面把特定 element 設定為 `display: none`，則會造成 false positive，tracker 誤以為 filter list 有啟用，但其實是網站自身把該 element 隱藏了。一個簡單的解法是準備很多個預期會被阻擋的唯一 element，如果這些 elements 之中超過一定比例被隱藏或顯示，就判斷說該 filter list 有被啟用或沒有被啟用

## 優勢與限制
將 Ad blocker 的行為作為 fingerprinting 的資訊來源，對追蹤者來說有幾個優勢。首先，因為安裝 Ad blocker 的人佔有相當比例，所以許多人都會被影響到。此外，如果在隱私模式下有啟用 Ad blocker，則因為在一般模式與隱私模式下 Ad blocker 的行為應該是一樣的，所以在兩種模式下得到的 fingerprint 理應是一樣的，進而可以追蹤那些使用隱私模式的使用者。

然而 Ad blocker 也有幾個限制。首先，許多瀏覽器（Chrome Desktop 與 Firefox Desktop）預設禁止 browser extension 在隱私模式下運作，所以 Ad blocker 在隱私模式下是預設關閉的。不過 Safari 就是預設隱私模式下一樣可以執行 extension，所以不會被影響到。另一個限制是，filter list community 很頻繁地在更新 filter list，所以追蹤者也要積極地更新欲探測的 request 或 element 列表，雖然使用前述的比例法（只有超過一定比例時才認定 filter list 有啟用）可以延長一份 element 列表可用的壽命，但終究必須隨著 filter list 改動而有所更新。

## 防禦
目前要防禦 ad blocker fingerprinting 主要還是靠 content blocking，也就是一開始就把 tracker 阻擋掉。這會陷入一個很諷刺的狀況：如果阻擋 tracker 失敗，則阻擋 tracker 這個行為本身反而會幫助 tracker 拿到更多資訊。後面在系統性地討論 browser fingerprinting 防禦時，會討論到所謂的 privacy paradox，此處的狀況正是一種 privacy paradox 的案例。
除了最廣泛的防禦手段以外，也有針對 ad blocker fingerprinting 的防禦手段。如果仔細檢視，會發現「以 ad blocker 做 fingerprinting」與「偵測 ad blocker 的存在」其實是非常類似的問題，只是 ad blocker fingerprinting 還額外要求了偵測各個 filter list 的啟用狀態。於是，我們可以把問題簡化成：如何避免 ad blocker 被發現，畢竟只要連 ad blocker 存在都無法被偵測到，就不可能知道有哪些 filter list 正在被使用了。此處可以回到之前討論 [browser extensions fingerprinting]({{< relref "/docs/browser-fingerprinting/techniques/browser-extensions-fingerprinting" >}}) 的防禦，畢竟 ad blocker 也是一種 browser extension，也就可以使用 browser extension fingerprinting 的防禦手段阻擋 ad blocker fingerprinting 。

最後，除了消極地避免被偵測到以外，也可以積極的防禦：動態地去偵測那些可能是在探察 Ad blocker 是否存在的 script，然後積極地封鎖。這裡的動態偵測可以是用一些 heuristic 去判斷，也可以建立 filter list 來阻擋，例如 [Anti-Adblock Killer](https://github.com/reek/anti-adblock-killer)，不過它看起來已經沒有在維護了。