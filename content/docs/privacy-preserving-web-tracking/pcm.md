---
weight: 4
title: "Private Click Measurement"
---

# Apple 的嘗試：Private Click Measurement
這個章節我們將會討論 Apple 的 Private Click Measurement（PCM）如何作到 privacy-preserving 的 ad attribution 或 conversion measurement，並將討論 PCM 潛在的問題與挑戰。PCM 本身其實不是什麼高科技，也沒有像 FLoC 或 Topics API 有著很有趣的演算法（以及攻擊），而是一些基本甚至有些無聊的 heuristics。之所以選擇介紹它，主要在於它是少數已經被實際部屬的 privacy-preserving web tracking 技術，基本上只要是用 macOS 上的 Safari 或是 iOS，很可能都已經內建PCM，甚至被 PCM 追蹤過，因此它是個值得花時間下去了解的技術。

## PCM 要解決什麼？
PCM 的核心目標是作到不洩漏隱私資訊的 ad attribution。Ad addtribution 指追蹤使用者點擊廣告之後的操作，例如使用者在 blog.example 點擊了廣告，被帶到 shop.example，如果他在 shop.example 做了像是註冊帳號、消費等行為，則 blog.example 會收到 shop.example 的通知 。類似的需求不只在於廣告，所以有個指涉更廣泛且中性的用詞是 "conversion measurement"。

### 傳統的 Conversion Measurement 如何運作
傳統的 conversion measurement 往往仰賴 third-party cookie 與/或 link decoration 之類的方式達成。

舉例來說，如果使用者在 blog.example 上點擊 shop.example 的連結而被帶過去，shop.example 會知道使用者是從 blog.example 來的，這通常是用 link decoration 達到的，例如一開始的 link 放 shop.example/?...&campaign_id=[id]，當使用者連入 shop.example 之後，網站就會讀取該 URL query 以辨認使用者是從哪邊來的。

接著，當使用者在 shop.example 上有什麼操作時，shop.example 就會不斷通知 blog.example 發生什麼事了。一個常見的方法是用 tracking pixel，也就是在 shop.example 上放一個極小的圖檔 `blog.example/pixel.gif?event=[...]`，以確保 blog.example 可以追蹤 conversion 的狀態，例如當使用者消費時 blog.example 就會收到通知。然後利用 third-party cookie，blog.example 就可以辨認出目前正在操作的是哪個使用者。

但這麼做可能會造成 cross-site tracking。在沒有封鎖 third-party cookie 時，載入 blog.example/pixel.gif 時會一起帶入它的 cookie，如此則 blog.example 可以知道目前正在 shop.example 上操作的是誰。更糟糕的是，就算使用者沒有點過 blog.example 上的廣告，因為 tracking pixel 的存在，blog.example 還是會學到使用者在 shop.example 的行為。可是如果直接封鎖 third-party cookie，則 conversion measurement 就變得不可能了，這不是個好的解法。

PCM 正是要嘗試解決此問題。

## PCM 如何運作
PCM 的核心邏輯是，把 conversion measurement 移動到瀏覽器裡面來做，網站可以要求瀏覽器為使用者在自己與另一個網站的操作產出一個 attribution report，記載著使用者在另一個網站的各種 event。

### 點擊
假設現在有兩個網站 blog.example 與 shop.example，後者要在前者投放廣告。於是在 blog.example 上，通往 shop.example 的廣告連結本來可能會用 link decoration 的方式帶入 identifier，現在改成這樣：

```html
<a href="https://shop.example/"
   attributionsourceid="[8-bit source ID]"
   attributiondestination="https://shop.example">
	Something here
</a>
```

`attributionsourceid` 就是傳統所說的 ad campaign ID，簡言之就是「這是哪個廣告」。`attributiondestination`（以前為 `attributeon`）則是指要把 attribution 送給誰。

如果使用者點擊了該連結，他會被帶到 shop.example，且 browser 會記住從 blog.example 點擊而跳到 shop.example 的連結的 `attributionsourceid`。目前設計是每筆紀錄會被儲存七天。

### 觸發事件
如果使用者在 `shop.example` 做了任何會觸發 attribution 的動作（像是消費、訂閱、註冊帳號等），`shop.example` 接著會送一個 GET request 給：

```
blog.example/.well-known/private-click-measurement/trigger-attribution/[4-bit trigger data]/[priority (6 bits, optional)]
```

其中 `[trigger data]` 可以想像成 event ID，也就是是哪個事件被 trigger 了；`priority` 用於表示事件的優先序，舉例來說，購買商品可能比註冊帳號重要，如果要做 attribution 時應該以前者為主，則前者的 priority 會比較高，在最後製作 attribution report 時也會優先把前者放進去。這裡的設計模擬了過去 tracking pixel 的功能，只需要把 tracking pixel 的網址改成上述網址就可以了，希望藉此減少改用 PCM 的技術成本。

當 `shop.example` 發出上述請求時，browser 會檢查這個請求是否有對應的 `attributionsourceid`，如果有，browser 會把請求攔截並暫存下來，作為之後 attribution report 使用。

### Attribution Report
從使用者第一次觸發事件後，browser 會等待 24~48 小時，然後生成一個 attribution report，並透過 anonymous proxy 送給雙方（網址如下），而且請送出時不夾帶 cookie 等 stateful 的資訊，或更直白來說，在無痕模式下送出。

```
blog.example/.well-known/private-click-measurement/report-attribution/
shop.example/.well-known/private-click-measurement/report-attribution/
```

內容：
```json
{
  "source_engagement_type" : "click",
  "source_site" : "blog.example",
  "source_id" : [8-bit source ID],
  "attributed_on_site" : "shop.example",
  "trigger_data" : [4-bit trigger data],
  "version": 1
}
```

其中 `source_engagement_type` 目前一定是 `click`，但未來可能可以新增其他型態。

簡單來說，瀏覽器會主動跟 blog.example 說：「在 24~48 小時前，有個之前在 blog.example 上點了 shop.example 廣告（該 `source_id` 為 `[8-bit source ID]`）的人，他之後在 shop.example 上觸發了特定事件，該事件的編號為 `[4-bit trigger data]`」。

### 安全性分析
PCM 在設計上有許多小巧思來維護使用者的隱私與安全性。

首先，attribution report 會放置 24~48 小時後再藉由 anonymous proxy 送出，且不夾帶 cookie，如此則可以避免廣告發布者（以前述例子來說是 blog.example）藉由時間、IP 或 cookie 猜到完成 conversion 的使用者是誰。

此外，因為 source id 只有 8 bits，這足以用來識別不同 campaign 或是不同廣告的型態、位置等，但不足用來作為 web tracking 的 identifier，畢竟多數網站應該都有遠多於 256 個訪客。只有 4 bits 的 trigger data 也同理，對正常用途來說堪用，但無法拿來當成 identifer 做 tracking。

而且因為 attribution report 是瀏覽器主動傳送，而非讓網站來蒐集，所以第三方不會收到 attribute report，不像現在 third-party tracker 可以看到使用者的 conversion。又因為如果沒有 `attributionsourceid`，瀏覽器便不會理會請求，所以如果使用者沒有在 blog.example 上點廣告，則 shop.example 送的 attribution 會被忽略，以避免 blog.example 學到他不需要知道的東西。

最後，回報位置是固定的，都在 `/.well-known/private-click-measurement` 底下，如果有 content blocker 想要阻擋掉 conversion measurement，可以輕易阻擋。

## 問題
PCM 的設計也有遭遇一些隱私上的質疑，其中最大的挑戰來自 Mozilla 的團隊。他們寫了一個很完整的 [report](https://mozilla.github.io/ppa-docs/pcm.pdf)，可以讀讀看。以下我整理了其中幾個有趣的批評。

### 發送時間可能洩漏身份
PCM 的隱私設計之一是 attribution report 會延後 24~48 小時發送，但如果這段時間使用者都沒使用 browser，則會被繼續往後推遲。顯然的，使用者必須要用瀏覽器（或至少允許瀏覽器在背景運作），瀏覽器才可以傳送 report。不過使用者也同樣只有在使用瀏覽器時有辦法造訪網頁。也就是說，如果網站管理者知道使用者何時造訪其網站，就可以知道使用者大概何時在用瀏覽器（這對於一些常用服務，如 Gmail、Facebook 等，而言並非難事，畢竟很多人 Facebook 都掛著），如此則網站管理者就有機會知道一個 attribution report 是否有可能是該使用者的：如果該使用者在當下根本不在線上，則這個 report 不會是他的。如果再額外考慮時區之類的，排除掉大家都不太活躍的時間，則可以增加識別的可能性。

不過 Mozilla 雖然做出這樣的宣稱，卻沒有給出具體攻擊的確可行的證明，例如 PoC 等，所以只能說這在理論上有可能，我們還不知道實際是否可能。

### 作為 Cookie-syncing 媒介的可能性
我們在[提過 cookie syncing]({{< relref "/docs/stateful-tracking/cross-site-tracking-with-cookie" >}})，如果兩個 tracker 要知道某個使用者在彼此的資料中有什麼 identifier，可以使用 cookie syncing，或是反過來說：tracker A 的這個 identifier 所對應到的使用者，在 tracker B 那邊是哪個 identifier。PCM 可以傳遞資訊的量雖然小（8 bits 的 source id、4 bits 的 trigger data），但仍然有作為 cookie syncing 媒介的潛力。

假設現在兩個 tracker 想要確認某個使用者在彼此的資料中是哪個 identifier。他們可以先根據 IP 或 browser fingerprint 等媒介猜測可能哪兩筆資料是對應的，接著使用 PCM 驗證這個猜測。假設基於前述方式，我們猜測  `blog.example` 的 user X 與 `shop.example` 的 user Y 是同個人，可以把使用者引到 `blog.example`，並誘使他點擊 `shop.example` 的連結，該連結帶有 attribution source id，同時 `shop.example` 在遇到 user Y 時 trigger 一個 event。如果等 24~48 小時後有收到 attribution report，代表他們猜測是對的，如果沒收到就是猜錯了。如果他們猜對了，則 `blog.example` 與 `shop.example` 知道他們之間的 identifer 對應關係，並可以開始做資訊交換了。

上述攻擊甚至可以延伸到多個人的情境：如果 `shop.example` 不確定到底是 user Y 還是 user Z 對應到 `blog.example` 的 user X，可以對 user Y 和 user Z 分別 trigger 不同的 event（即不同 trigger data），看最後 attribution report 上的 trigger data 是哪一個即可。反之亦然，如果是 `blog.example` 不確定是 user X 還是 user W，也一樣可以先帶不同 source id，看最後收到的 attribution report 上記載的 source id 是哪一個。

考慮到 source id 與 trigger data 組合有 4096 種，每發動一次測試最多可以同時測 4096 組 identifier pair，看起來蠻強大的。不過因為 report 要等 24~48 小時才會送到，還可能延遲（如果使用者都沒開瀏覽器的話），如果沒有收到 report 其實不知道是猜錯了還是單純還沒收到而已，且 source id 只會儲存 7 天，所以藉此辨認的效率其實稱不上好。

### Source ID 與 Trigger Data 不夠用
在前面我們宣稱 source id（8 bits）與 trigger data（4 bits）在多數時候，於正常用途時，是堪用的。然而，還是存在一些情境使其不敷使用，例如當我希望 trigger data 包含使用者的消費金額，4 bits 很可能不足以儲存消費金額。但同時如果加大 source id 與 trigger data 的大小，又可能使其被當作 identifier 來使用，使 cross-site tracking 變得可能。因此這邊陷入了兩難。

### eTLD+1 的限制過於嚴苛
前面提過 `attributiondestination` （以前面的舉例來說，是 `shop.example`）必須是 eTLD+1。如此的設計是為了避免使用 domain 作為 identifer，例如將 `attributiondestination` 設定為 `[id].shop.example`，如此則 `blog.example` 收到 attribution report 時可以用 `[id]` 識別使用者是誰。

然而，許多電商會使用 subdomain 分別商店，例如 Shopify 商店可能有 domain `shop1.myshopify.com` 與 `shop2.myshopify.com` 之類的（這些 domain 屬於 eTLD+2），他們不全然符合 Public Suffix 的定義，但兩者仍然是不同商店，在使用 PCM 之後，因為只能使用 eTLD+1，於是就不知道 attribution 到底是發生在哪個網站上了，只知道是 `myshopify.com` 的其中一個 subdomain。這樣廣告的帳根本不知道怎麼算啊。

大家想到的絕佳好方法是：把自己的 domain 放到 Public Suffix List，也就是讓 `myshopify.com` 變成 eTLD，讓 `shop1.myshopify.com` 變成 eTLD+1。結果就是 Public Suffix List 被各種新申請洗板，[管理員快抓狂了](https://github.com/publicsuffix/list/issues/1245#issuecomment-812228981)。

## 結語
整體而言 PCM 雖然不完美，但是個不錯的嘗試。Mozilla 的安全性分析固然有提出一些值得深入探究的 attack vector，但目前看來都不算太 practical。然而 PCM 推廣最大的問題其實是 spec 一直都非常不明確，例如 WebKit 部落格和 spec 在小細節上有些落差，至於 [Fruad Prevention](https://webkit.org/blog/12566/private-click-measurement-conversion-fraud-prevention-and-replacement-for-tracking-pixels/) 機制更是完全沒出現在 spec 上，這也導致了其他 browser 難以跟進。另一個問題是，PCM 對於有些廣告商來說可能不敷使用，這也是需要解決的。

題外話，PCM 的 Fraud Prevention 機制還蠻有趣的，不過因為它和 web tracking 的隱私議題比較無關，所以這篇文章就沒有討論，如果有興趣可以看看。

## 參考資料
[Introducing Private Click Measurement, PCM](https://webkit.org/blog/11529/introducing-private-click-measurement-pcm/)
[Privacy Preserving Ad Click Attribution For the Web](https://webkit.org/blog/8943/privacy-preserving-ad-click-attribution-for-the-web/)
[Private Click Measurement: Conversion Fraud Prevention and Replacement For Tracking Pixels](https://webkit.org/blog/12566/private-click-measurement-conversion-fraud-prevention-and-replacement-for-tracking-pixels/)
[Understanding Apple’s Private Click Measurement](https://blog.mozilla.org/en/mozilla/understanding-apples-private-click-measurement/)
[privacycg/private-click-measurement](https://github.com/privacycg/private-click-measurement)
Thomson, Martin. "An Analysis of Apple’s Private Click Measurement." (2022).