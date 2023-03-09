---
weight: 5
title: "如何對付傳統追蹤技術 (II)：如何繞過基於 Filter List 的 Content Blocking"
---

# 如何對付傳統追蹤技術 (II)：如何繞過基於 Filter List 的 Content Blocking
正如任何資安領域，只要有攻擊，就會有防禦，只要有防禦，就會有繞過。既然 content blocking 會阻擋 tracking script，也就有人會嘗試繞過。如同前面所提到的，content blocker 在識別 tracking script  與廣告的手段中，最主流的是使用黑名單。因此，如何繞過黑名單自然會是最受人矚目的問題。此文將整理數個技術上可行的辦法，有些很常見，有些則是可遇不可求甚至很奇葩的。

稍微複習一下。Filter list 可以分成兩種型態：
- Resource blocking：封鎖整個 domain 或特定 URL path，讓 request 無法發出
- Element hiding：隱藏或刪除特定的 DOM element
因為後者主要是做 ad blocking，對於防禦 tracker 沒什麼幫助，所以本文會著重在 resource blocking 的繞過。

## 常見的繞過方式
### 更改 URL paths
一個很直覺的方法是，既然 URL path 會被封鎖，那我就把 URL path 改掉，不就繞過 filter list 了？舉例來說，假設 `tracker.example/tracker.js` 被封鎖了，那就改成 `tracker.example/tracker2.js`，如果又被封鎖了，就繼續改。出乎意料的是，這方法超級有效。因為多數 filter list 是靠社群夥伴通報與維護，如果追蹤者更改 URL path，往往要等到有人發現並通報後，maintainer 才能將其加入 filter list。改檔案位置只需要花幾分鐘的時間，但 filter list maintainer 可能要好幾天甚至好幾個月後才會發現 filter list 被繞過了。

甚至，其實根本不用一直換檔名，這樣太麻煩了，直接讓任意 URL path 都會回應同一個檔案不就好了？例如，我們之前觀察到一個 domain，只要 path 是四個英文字母加上 .js（e.g. `tracker.example/abcd.js` 與 `tracker.example/efgh.js` ），都會回傳一模一樣的 script，如此則連換檔案都不用了，client side 直接隨機選一個 path 存取即可。或是我們也有觀察到那種明顯是隨機生成的 path，像是 `/i2dl71/921livp0m30yh8q/867vuq/768kypp.php`。

可能讀者會很疑惑，為什麼不直接封鎖該 domain 就好？許多 filter list 的 policy 要求盡可能避免 false positive，並認為 false positive 比 false negative 嚴重，寧可縱放不可誤殺。也許該 domain 上有些檔案不是 tracker，直接封鎖整個 domain 會造成誤殺。某種意義上，追蹤者大概就是在運用這個特性，才能濫用 URL paths 吧。

### 更改 Domains
既然可以改 URL paths，當然也可以改 domain。如果 filter list 把 `tracker.example` 整個封鎖了，就改成用 `tracker2.example`。如同 URL paths 的困境，filter list maintainer 往往要隔一段時間才能發現 tracking domain 被改掉了。

改 domain 的策略很多。第一種是移到 first-party subdomain，例如之前提過的 CNAME cloaking，其造成的挑戰是，原本只有 `tracker.example` 需要封鎖，現在有成千上萬個不同 first-party subdomain 需要封鎖。更多討論可以讀針對 [CNAME cloaking]({{< relref "/docs/stateful-tracking/cross-site-tracking-without-cookie#cname-cloaking" >}}) 的討論。第二種策略是洗一堆 CDN 的 domain，像是 `*.cloudfront.net` 之類的，然後不斷換不同 domain，反正 domain 是 CDN 的，filter list 總不可能把整個 cloudfront 都封鎖。第三種策略是改 subdomain，像是從 `tracker.blog.example` 改成 `tracker2.blog.example`，因為 domain 都是自己的，這麼做沒什麼成本，又因為 `blog.example` 是正常的東西，filter list 不能直接把該 domain 完全阻擋掉，只能繼續貓追老鼠。

最後一種策略，也是最奇葩的策略，是用 domain generation algorithms (DGA) 去大量生產 domain，一個被封鎖就改用下一個，這整個過程完全是自動化的，filter list maintainer 只能很辛苦地在後面追趕，把剛出現的 tracking domain 加進 filter list，然後馬上就出現下一個新的 tracking domain，永無止盡。其甚至有個專有名詞稱為 *revolving domain*：會一直轉的 domain。舉例來說，有個廣告商叫做 LuckyAds，他有各種亂數生產的 domain，像是 `qakdki.com` 與 `inpiza.com`，這些都是用於繞過 filter list。

當然 filter list maintainer 也不會就此放棄。目前 EasyList（最常見的 filter list）有兩種策略回應。首先，他們有個 bot 去自動監測那些已知在使用 revolving domain 的網站，如果看到新的 domain 就自動封鎖。此外，對於那些已知在使用 revolving domain 的網站，預設阻擋所有 third-party requests，如此雖然可以阻擋 revolving domain，但可能讓網站功能壞掉。

## 罕見的繞過方式
### 利用沒寫好的例外規則
例外規則，顧名思義，就是指有些 request 就算符合 filter list 裡面的東西，也會因為它是例外而被放行。通常這用於一些誤判（例如多數 `ads.js` 都是廣告，但偏偏有些不是），或是用於繞過 anti-ad blocking（到後面講 ad blocking 如何用於 fingerprinting 時可能會討論到）。

可是，filter list 是人寫的，終究會出錯。於是有人就想到，如果利用沒寫好的例外規則來繞過 filter list，聽起來好像可行。

舉例來說，曾經 EasyList 有一條例外規則是：
```
@@||redtube.com*/adframe.js
```

其意思是，domain name 部份的開頭是 `redtube.com`，且 URL 的結尾是 `/adframe.js`。
這... 聽起來怪怪的吧？於是就有人想到這樣繞過：
```
http://redtube.com.umamdmo.com/.......?q=/adframe.js
```
這符合開頭是 `redtube.com` 結尾是 `/adframe.js`，但這顯然不是一開始想加入白名單的東西吧。

有趣的是，這個案例存活了兩年多才被發現。當然，這種繞過方式是可遇不可求的。

### 把 Tracker 移動到 Inline Script
一直以來，大家都習慣把 tracker 的 Javascript 放在獨立一個檔案，然後用 `<script src="<URL>"` 或其他方式去存取。於是 content blocker 只要把這個 URL 封鎖掉就好。

那，如果我把 tracker 放到 inline script（`<script>...</script>`），是不是就沒辦法封鎖了？於是就有一些追蹤者鼓勵網站管理員把 tracker 放到 inline script，藉此避免被封鎖。EasyList 的回應是，利用 [CSP policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 去 block 這些不想要被執行的 inline script。

## 其他
### WebSocket（曾經）
曾經有很長一段時間，Chrome extension 沒有辦法攔截 WebSocket 的流量，於是有些廣告商與追蹤者就開始用 WebSocket 來傳送廣告與 tracker。不過這個 bug 在 2017 年修掉之後，基本上就沒用了。

### Anti-ad blocker
應該很多人看過有些網站會要求關閉 ad blocker，否則就不允許使用。基本上這不是在繞過 filter list，而是故意讓網站會因為 content blocker 而壞掉，逼迫使用者關閉 content blocker。不過現在也有針對 anti-ad blocker 的 filter list，專門把這些 script 砍掉。


## 參考資料
Mshabab Alrizah, Sencun Zhu, Xinyu Xing, and Gang Wang. 2019. Errors, Misunderstandings, and Attacks: Analyzing the Crowdsourcing Process of Ad-blocking Systems. In Proceedings of the Internet Measurement Conference (IMC '19). Association for Computing Machinery, New York, NY, USA, 230–244. https://doi.org/10.1145/3355369.3355588
Su-Chin Lin, Kai-Hsiang Chou, Yen Chen, Hsu-Chun Hsiao, Darion Cassel, Lujo Bauer, and Limin Jia. 2022. Investigating Advertisers’ Domain-changing Behaviors and Their Impacts on Ad-blocker Filter Lists. In Proceedings of the ACM Web Conference 2022 (WWW '22). Association for Computing Machinery, New York, NY, USA, 576–587. https://doi.org/10.1145/3485447.3512218