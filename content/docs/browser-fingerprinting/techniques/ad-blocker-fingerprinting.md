---
weight: 8
title: "Ad Blocker Fingerprinting"
---

# Ad Blocker Fingerprinting

曾經我們討論過，阻擋 tracker 的方法中，最廣泛被使用的是 filter list，也就是去列出我想要阻擋的黑名單，然後 ad blocker 會根據 filter list 去阻擋 tracker。Ad blocker 不但可以改善使用體驗，畢竟沒人喜歡看廣告，還可以藉由阻擋 tracker 來減少隱私侵害。然而，有研究者發現，Ad blocker 的行為可以被用於 fingerprinting：哪些 element 會被隱藏或移除，哪些請求會被阻擋，都會因為 ad blocker 的設定而有所不同，因此只要去檢驗 ad blocker 的行為，就有機會增加 fingerprinting 的 entropy。

值得注意的是 ad blocker 除了基於 filter list 的方法以外，也有基於 Machine Learning 或 Heuristic 的，但因為主流的還是 filter list，所以此文還是直接稱呼這些基於 filter list 的 ad blocker 為 ad blocker。另外，這篇文章假設讀者讀過此系列中有關 [Content Blocking]() 的文章，如果還沒讀過，可以先過去看看。

在 Ad blocker 的世界中，並不存在一個世界統一的 filter list，雖然會有幾個最大咖的（e.g. Easylist），但也會有眾多不那麼知名的但同樣好用的（e.g. Peter Lowe’s Ad and tracking server list）、針對特定語系的（e.g. AdGuard Japanese）、針對特定類型內容的（e.g. Online Malicious URL Blocklist）。如果全部 filter list 都啟用，可能造成效能不彰，所以使用者會根據自己需求去挑選他需要的 filter list，或是 ad blocker 可能會根據系統語言去自動地啟用相對應的 filter list。於是作為中文使用者，我會使用針對中文網站的 filter list（e.g.  Easylist China），但不會去用針對德文網站的 filter list（e.g. EasyList Germany），反正我也不會瀏覽德文的網站。

![](/images/ublock-origin-filter-lists.png)

至此我們已經可以看出問題所在了：如果每個使用者啟用的 filter list 不一樣，則可以根據使用者啟用了哪些 filter list 去識別使用者。當然我們可以想像多數使用者都不會去改 ad blocker 的設定，而且同個語系的人使用的 filter list 大概也都差不多，但記得正如一再強調的重點：Fingerprinting 的重點不是一個方法識別出所有人，而是很多不同技術個別貢獻一些 entropy 而達成的。

## 如何從 Ad Blocker 獲得資訊
正如之前介紹過的，ad blocker 主要分成兩種不同策略：阻擋 requests 與隱藏/刪除 elements。所以能提供資訊的，大概也就是兩種策略：去檢視哪些 request 被阻擋了（resource blocking），以及去檢視哪些 elements 被隱藏了（element hiding）。

### 從 resource blocking 取得資訊
流程也很簡單，假設我們現在有 A, B, C 三個 filter list，想要找出每個使用者用了哪些 filter list，可以這麼做：
1. 預處理：找出 A, B, C 分別有哪些 URI 是只有該 filter list 有阻擋，其他都沒有的。
2. 嘗試發一個 request 到該 URI，看會不會成功
3. 如果 request 成功，則該 filter list 沒有啟用，反之，如果 request 被阻擋則代表可能有啟用該 filter list

如果沒有一個 URI 是只存在於唯一 filter list，也還是可以用排容原理來找。

不過這方法有個顯然的缺點：發 HTTP request 很耗時、耗能、佔用頻寬，而且使用者如果看到一個網站一直在載入各種資源，大概也會起疑。如果每個 filter list 都有一個唯一的 URI 是別人沒有的，那也許還可以接受，但如果要排容，所需的請求量很快就會飆升上去了。另外，請求失敗的原因很多，可能是網路正好斷掉了、撞到防火牆等，不一定是 ad blocker 造成的。所以一般來說不會用 request blocking 來做 fingerprinting，畢竟我們有效率更好的方法。

### 從 element hiding 取得資訊
既然送 request 這條路走不通，那就換一個方法。另一條路是利用 element hiding 取得資訊，也就是去檢驗哪些 element 會被隱藏。

在執行上也和 request blocking 類似，假設我們現在有 A, B, C 三個 filter list，，想要找出每個使用者用了哪些 filter list：
1. 預處理：找出 A, B, C 分別有哪些 element 是只有該 filter list 有阻擋，其他都沒有的。通常我們會拿到一個 HTML class 或 id 或 tag name 等資訊。
2. 嘗試在網頁上建立符合條件的空白 element（例如賦予指定的 class 或 id 等），並檢驗該 element 有沒有被隱藏
3. 如果該 element 正常顯示，則該 filter list 沒有啟用，反之，如果 element 被隱藏了，則代表可能有啟用該 filter list

要檢驗一個 element 是否正常顯示，可以利用 [offsetParent](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetParent)，當一個 element 或是其 parent 被隱藏時，它的 `offsetParent` 會是 `null`。

不過這個方法並不是萬無一失的。例如網站可能自己有 CSS 或是一些 script 去處理這些 element，可能就會導致錯誤判斷。舉例來說，如果 CSS 裡面把特定 element 設定為 `display: none`，則會造成 false positive，tracker 誤以為 filter list 有啟用，但其實是網站自身把該 element 隱藏了。一個簡單的解法是準備很多個預期會被阻擋的唯一 element，如果這些 elements 之中超過一定比例被隱藏或顯示，就判斷說該 filter list 有被啟用或沒有被啟用，取決於追蹤者認為 false positive 和 false negative 的危害孰重孰輕。

## 優勢與限制
使用 Ad blocker 的 behavior 作為 fingerprinting 的資訊來源，對追蹤者來說有幾個優勢。首先，因為裝 Ad blocker  的人蠻多的（我也不確定有多少，但根據[這篇文章](https://www.insiderintelligence.com/insights/ad-blocking/)，大概有 24%~40% 不等的使用者安裝，在年輕族群更多），所以許多人都會被影響到。此外，如果在隱私模式下有啟用 Ad blocker，則因為在一般模式與隱私模式下 ad blocker 的行為應該是一樣的，所以在兩種模式下得到的 fingerprint 理應是一樣的，進而可以追蹤那些使用隱私模式的使用者。

然而 Ad blocker 也有幾個硬傷。首先，許多瀏覽器（Chrome Desktop 與 Firefox Desktop）預設禁止 browser extension 在隱私模式下運作，所以 Ad blocker 在隱私模式下是預設關閉的。不過 Safari 就是預設隱私模式下一樣可以執行 extension，所以不會被影響到。另一個硬傷是，filter list community 很頻繁地在更新 filter list，所以追蹤者要積極地更新欲探測的 element 列表，雖然使用前述的比例方法（只有超過一定比例時才認定有/沒有 filter list）可以延長一份 element 列表可用的壽命，但終究得要換掉。

## 防禦
簡單來說，老樣子，目前要防禦 ad blocker fingerprinting 還是只能靠 content blocking，也就是一開始就把 tracker 阻擋掉。我一直覺得這件事很嘲諷，如果阻擋 tracker 失敗，則阻擋 tracker 這個行為本身反而會幫助 tracker 拿到更多資訊。

那除了 general 的防禦手段以外，有沒有針對 ad blocker fingerprinting 的防禦手段呢？如果仔細檢視，會發現拿 ad blocker 做 fingerprinting 跟偵測 ad blocker 的存在其實是非常類似的問題，只是 ad blocker fingerprinting 還額外要求了連同有哪些 filter list 正在被使用都要還原出來。於是我們可以把問題簡化成：如何避免 ad blocker 被發現，畢竟只要連 ad blocker 存在都無法被偵測到，就不可能知道有哪些 filter list 正在被使用了。

一種解法是，因為網頁上的 Javascript 在執行位階上比 extension 還要低，而如果要偵測有沒有啟用 ad blocker 又需要使用網頁上的 Javascript，所以只要在每次 Javascript 去 query 一些有關 ad blocker 的資訊時（例如去存取一個已經被 ad blocker 隱藏的 element），ad blocker 就攔截這個 query 並回傳「若沒有 ad blocker 本應有的值」，也就是給假的值。

這個問題其實還可以進一步化約成：如何避免被發現有安裝特定 browser extension。這部份的討論可以回去看 [browser extensions fingerprinting]({{< relref "/docs/browser-fingerprinting/techniques/browser-extensions-fingerprinting" >}})的文章。

最後，除了消極地避免被偵測到以外，也可以積極的防禦：動態地去偵測那些可能是在探察 Ad blocker 是否存在的 script，然後積極地封鎖。這裡的動態偵測可以是用一些 heuristic 去判斷，也可以建立 filter list 來阻擋，例如 [Anti-Adblock Killer](https://github.com/reek/anti-adblock-killer)，不過它看起來已經沒有在維護了。