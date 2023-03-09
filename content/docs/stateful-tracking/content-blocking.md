---
weight: 4
title: "如何對付傳統追蹤技術 (I)：Content Blocking (Ad Blocking)"
---

# 如何對付傳統追蹤技術 (I)：Content Blocking (Ad Blocking)
在所有反制 web tracking 的方法中，最古早也最普遍被使用的方法是 content blocking。正如其名，content blocking 的邏輯很簡單，就是直接把 tracker 阻擋掉。假設現在 `example.com/tracker.js` 是個用於 web tracking 的 script，它會偷偷把使用者的瀏覽紀錄傳回 `example.com`，我們當然可以用各種方式（像是阻擋 third-party cookie）讓它沒辦法正確地辨識出我是誰，但有個更簡單的方式是：一開始不要讓 `example.com/tracker.js` 載入，不就沒問題了？這就是 content blocking 的核心精神。

Content blocking 聽起來像是一個很複雜的技術名詞，但有個與之密切相關且大家都聽過的東西叫做 "ad blocker"，現在主流的 ad blocker 幾乎都是藉由 content blocking 達到，也就是直接把取得廣告用的 Javascript 或其他資源（e.g. 圖片、影片等）的 request 阻擋掉。

在此文我不會嚴格區分 content blocker 與 ad blocker，因為兩者非常類似，很難界定。例如，幾乎所有的知名 ad blocker 同時都會阻擋 tracker，即便 tracker 並沒有在投放廣告。為了避免讀者誤以為 ad blocker 只是在阻擋可見的廣告，我將使用 content blocker 來稱呼這一系列的工具。

## Content Blocker 的不同阻擋樣態
如同前文所述，content blocker 的運作通常是直接阻擋廣告或 tracker 資源的請求。然而在許多情境下，直接阻擋請求不一定可行，或有時是不必要的，因此有更彈性的作法。此段將介紹三種 content blocker 阻擋廣告或是 tracker 的策略。

第一種，也是最常見的是，直接阻擋相關資源的請求。這通常是最理想的，因為不消耗無謂的網路流量，不會下載與執行可能造成隱私或資安風險的 script，當然惱人的廣告也完全不會顯示。我們通常稱此為 resource blocking。

第二種策略是隱藏特定的 element，通常稱之為 element hiding。這通常用於阻擋可見的廣告。當封鎖請求不太可行，會造成其他困擾時，雖然我們無法阻止瀏覽器載入 script 或其他資源，但至少可以避免其內容被顯示出來。Element hiding 通常是藉由插入特定的 CSS code 來隱藏（例如 `display: none`），或是直接把該 element 從 DOM tree 移除（例如 `removeChild`）。

第三種策略則主要是針對 tracker，阻止 tracking script 存取特定功能，例如使用 cookie。有時直接封鎖 tracker 可能會造成網站功能壞掉。例如，如果有個影片播放器的程式內建 tracker，直接把整個影片播放器封鎖大概不是個好方法，但如果限制它不能使用 cookie，將至少可以避免基於 cookie 的 web tracking。使用此方法最知名的 content blocker 是 Privacy Badger。

## Content Blocker 如何阻擋 requests
Content blocker 最核心的功能還是在於阻擋不期待的 HTTP requests，此段將討論 content blocker 可以如何阻擋。主流 ad blocker 大致可以分成兩類：browser extension 與 network firewall。

Browser extension 的 content blocker 包含常見的：[Adblock Plus](https://adblockplus.org/), [uBlock origin](https://ublockorigin.com/), [AdGuard](https://adguard.com/en/welcome.html), [Privacy Badger](https://privacybadger.org/) 等。這類型的 content blocker 是利用 browser extension 來達成的。

在不同瀏覽器有不同的達成方法，以下簡介幾個：
- Chrome 與 Chromium-based browser 有兩種方式可以攔截 requests。早期多半是用 [webRequest](https://developer.chrome.com/docs/extensions/reference/webRequest/)，extension 可以攔截所有 request，每一個都看看是不是應該被阻擋的內容，但這樣似乎給 extension 太大的權力了，如果 content blocker 是惡意的，所有請求會被攻擊者看到。第二種策略，也是未來 Google 打算採取的策略是 [declarativeNetRequest](https://developer.chrome.com/docs/extensions/reference/declarativeNetRequest/)，事先向 browser 註冊哪些 request 應該被攔截，如果匹配到了才會轉給 extension 處理，同時前述的 webRequest 也打算移除了。不過這個方法因為缺乏彈性且可能使現存有些 ad blocker 的功能無法延續，所以有引起一些爭議。
- Firefox 所使用的 extension API 和 chrome 非常雷同，不過因為 Mozilla 對 declarativeNetRequest 很[有意見](https://blog.mozilla.org/addons/2022/05/18/manifest-v3-in-firefox-recap-next-steps/)，所以雖然支援它，但保留了 webRequest。
- Safari 的特色是，content blocker 要預先要給 Safari 一長串應該阻擋的 requests，Safari 看到時就會直接阻擋，不會讓 content blocker 看到請求，所以可以避免惡意的 content blocker 取得使用者的瀏覽資料。但相對應的就是，Safari 的 content blocker 非常欠缺彈性。

第二種策略是 network firewall 的阻擋，其中最知名的是 [Pi-hole](https://pi-hole.net/)。其原理是，在遇到與廣告有關的 domain 時，攔截 DNS query 使其無法 resolve。這類型方法的缺點在於很難做到細微的調整，例如在 A 網站要阻擋 `example.com/tracker.js`，但在 B 網站不阻擋，或是做到阻擋 `example.com/tracker.js` 但不阻擋 `example.com/logo.png`。因此這類型的 content blocker 有式微的傾向。


## 如何偵測廣告與 tracker
至此我們還沒討論到最關鍵的問題：如何正確地偵測哪些請求是正常的，哪些是廣告或 tracker。目前有三種主流方法。

第一個，也是最常見的，是基於 filter list（黑名單）的 content blocking。這種方法仰賴人為地或自動地建立與維護廣告與 tracker 的黑名單。目前主流的幾個 filter list 都是開源社群的夥伴們共同維護。在分類上，有普遍的黑名單，也有針對區域、性質等不同類型的黑名單，例如針對 CryptoMiner 的黑名單。

幾乎所有設計給 browser extension 的主流 filter 都使用 [AdBlock Plus syntax](https://help.eyeo.com/en/adblockplus/how-to-write-filters)，雖然一開始是 AdBlock Plus 在用，但之後已經成為實質上的標準了。在這個標準下，黑名單是一個文字檔，每一行都代表一個不同的 filter，可以是 domain 或 URL path（resource blocking），也可以是 CSS selector（element hiding）。

```
# 封鎖所有 example.com 作為 third-party 時的請求
||example.com^$third-party

# 封鎖 example.net/ads 開頭的 URL 但不封鎖其他 URL
||example.net/ads

# 隱藏所有 class 為 s-ads 的 element
##.s-ads
```

幾個常見的為 browser extension 設計的 filter list 包含：[EasyList 系列](https://easylist.to/)（包含針對 tracker 的 EasyPrivacy 以及針對不同語系的 EasyList China, EasyList Germany 等）、[AdGuard 系列](https://kb.adguard.com/en/general/adguard-ad-filters)等。另外也有針對 Network firewall 的 filter list，例如 [Peter Lowe’s Ad and tracking server list](https://pgl.yoyo.org/adservers/)，不過基本上針對 network firewall 的 filter list 稍做轉換也都可以用於 browser extension。

![](/images/ublock-origin-filter-lists.png)

第二種方法是 heuristic-based 的，也就是基於一個演算法去自動判斷一個資源有沒有可能用於 tracking 或是投放廣告。其中最知名的大概是 Privacy Badger，有興趣的人可以看看他的[演算法](https://github.com/EFForg/privacybadger/blob/master/doc/DESIGN-AND-ROADMAP.md)如何運作。簡言之，Privacy Badger 會去判斷 third-party origin 有沒有設定 cookie、有沒有在存取可能是在做 browser fingerprinting 的 API、有沒有對外傳送 first-party cookie 等，如果看起來太可疑，就推測它是 tracker 並阻擋其存取 cookie 或是完全阻擋。

第三種方法是 Machine-Learning-based 的 content blocking，也就是用 ML 去判斷一個內容是否看起來像是廣告或 tracker。目前這方面主要都還僅限於學術研究中，並沒有被很廣泛的使用，所以就不多加著墨。

往後對 content blocking 的討論，只要沒有明說，基本上都是在說 filter-list-based 的 content blocking，因為這是目前最廣泛被使用的方法。

## 如何不把網站搞壞
讓 tracking script 不要被載入，乍聽之下很簡單，但有時一段 code 不必然只用於 tracking，還可能有其他功能，例如前面提過的，有些影片播放器的程式內建 tracking 功能，或有時一個 tracking script 也可能有一些 API 供網站管理員使用，如果貿然地擋掉 tracker 可能會讓網站的功能壞掉。

```js
<script src= "//tracker.example/track.js"> </script> <!-- blocked by the content blocker -->
<script> 
    tracker(setup); // Uncaught ReferenceError: tracker is not defined
</ script >
```

舉例來說，Google Analytics 的 analytics.js 提供了 [`ga` object]((https://developers.google.com/analytics/devguides/collection/analyticsjs/ga-object-methods-reference)) 供網站管理員使用，網站管理員可以利用 `ga` object 存取一些 Google Analytics 提供的功能。如果貿然阻擋 analytics.js 阻擋，可能會讓網站上原本與 Google Analytics 有關的 code 會因為找不到 `ga` object 而無法正常運作，甚至導致網站的功能完全壞掉。

很遺憾的是，主流的 content blocker 多半只能二元地決定阻擋資源與否，無法做到更細緻的權限控管，例如限制 script 可以執行但不能存取儲存空間，於是使用者只能在「阻擋 tracker 並讓網站壞掉」和「不阻擋 tracker 以換取網站正常運作」之間選擇。

為了解決這個問題，uBlock Origin 的作法是：手動維護一份取而代之的 code！例如前述的 analytics.js，uBlock Origin 不單純只是阻擋 HTTP request，而是直接回應一個假的、只有 interface 沒有功能的 analytics.js1。例如下面一段擷取的程式碼，可以看到所有原本 Google Analytics 的 interface 都被以 no-op 取代了。

為了解決這個問題，uBlock Origin 的作法是：手動維護一份取而代之的 code！例如前述的 `analytics.js`，uBlock Origin 不單純只是阻擋 HTTP request，而是直接回應一個假的、只有 interface 沒有功能的 [analytics.js](https://github.com/gorhill/uBlock/blob/master/src/web_accessible_resources/google-analytics_analytics.js)。例如下面一段擷取的程式碼，可以看到所有原本 Google Analytics 的 interface 都被以 no-op 取代了。

```js
const noopfn = function() {};
// …
Gaq.prototype.Na = noopfn;
Gaq.prototype.O = noopfn;
```

事實上，uBlock Origin 維護了一長串的假 tracking script 來取代真正的 tracking script，以此確保真正的 tracker 被攔截時不會讓網站壞掉。但這麼做的問題也很顯然：它仰賴非常大的人工成本。許多 tracker 的程式碼都有混淆與壓縮過，要逆向這些形同亂碼的程式碼並找出哪些 interface 需要插入 no-op，是個非常繁雜的工作，所以只有那些最常見的 tracker 有可能特別為其撰寫替代的假 tracker。Brave 與 UCSD 的團隊則更進一步提出了自動生成假 tracker 的工具，稱為 [SugarCoat](https://brave.com/privacy-updates/12-sugarcoat/)。他們的方法，約略來說，是透過分析 JavaScript 的 Web API call，從中識別出可能造成 tracking 的 API call，並且用無害的版本（例如 no-op）取代之。總而言之，如何在封鎖 tracker 時不破壞網站功能，並非一件易事。


## 參考資料
[How ad blockers can be used for browser fingerprinting](https://fingerprint.com/blog/ad-blocker-fingerprinting/)
[SugarCoat: Programmatically Generating Privacy-Preserving, Web-Compatible Resource Replacements for Content Blocking](https://brave.com/research/files/sugarcoat-ccs-2021.pdf)