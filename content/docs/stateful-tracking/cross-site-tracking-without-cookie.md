---
weight: 3
title: "殺死 Third-party Cookie，一樣可以做 Cross-site Tracking：CNAME Cloaking, Bounce Trackers, and Link Decorations"
---

# 殺死 Third-party Cookie，一樣可以做 Cross-site Tracking：CNAME Cloaking, Bounce Trackers, and Link Decorations
先前我們介紹了 third-party cookie 如何用於 web tracking，尤其 cross-site tracking。多數瀏覽器都已經封鎖或即將封鎖 third-party cookie，更何況在許久之前便已有不少人藉由 browser extension 或其他工具阻擋 third-party cookie，這使得傳統的 cross-site tracking 技術不再是穩定可用的。為此，追蹤者也開發了數種其他 cross-site tracking 的方法。在此章節，我們將介紹三種不使用 third-party cookie的 cross-site tracking 技術，分別是 CNAME cloaking、bounce trackers 以及 link decoration。

## CNAME Cloaking
### CNAME Cloaking 是什麼
CNAME 是一種 DNS record 的型態，其所表達的意思大概是「我被稱為 `tracker.example`，但其實我就是 `tracker-real.example`」。

如果有用過 GitHub Page 可能就會知道 GitHub Page 的 [custom domain](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-a-subdomain) 便是使用 CNAME 達成，使用者在他的 custom domain `example.com` 加上一筆指向 `<username>.github.io` 的 CNAME record，於是在嘗試 resolve 時，電腦就知道 `example.com` 其實就是 `<username>.github.io`，無論 `<username>.github.io` 指向哪裡，總之電腦知道想要找到 `example.com`，去找 `<username>.github.io` 肯定沒錯。

有些 tracker 利用了 CNAME 的性質，嘗試把 tracker 移動到網站自身的網域底下。例如現在有個追蹤者把 tracker 的資源放在 `tracker.tracking.example`，有個想要使用 tracker 的網站是 `news.example`，一般來說如果直接嘗試與 `tracker.tracking.example` 溝通，會被視為 third-party 而有諸多限制，例如不能傳送 cookie。不過，如果我用 CNAME record 把 `tracker.news.example` 指向 `tracker.tracking.example`，然後在網站引入 `tracker.news.example` 的資源，則瀏覽器會誤以為這是 first-party subdomain 而有比較寬鬆的限制，使用者可能也會因為那是 first-party subdomain 而放下戒心，即便那背後其實是 `tracker.tracking.example`。

稍微岔題一下，讀者可能會疑惑，`tracker.news.example` 和 `news.example` 是不同 origin，理應不能讀到對方的 cookie，所以拉到 first-party subdomain 理論上應該沒有幫助。不過 cookie 有個屬性是 `domain`，其功能就是想達成在 subdomain 之間共享 cookie。假如我在 `news.example` 設定這樣的 cookie：
```
Set-Cookie: key=value; Domain=news.example
```
則在 `subdomain.news.example`，一樣可以讀到該 cookie。

這種藉由 CNAME 來偽裝成 first-party subdomain 的技術稱為 CNAME cloaking。

### CNAME Cloaking 的危害
CNAME cloaking 的危害大概有以下幾點
- 它用於繞過 ad blocker 等所使用的 filter list，就算 tracker 本身被阻擋，我也可以將其移動到自己的 subdomain。
- 如果每個人都可以有自己專屬的 tracking domain，那要加入 filter list 的 tracking domain 沒完沒了。
- 更糟糕的是，在（新版的）Chrome 與 Safari 對於 filter list 的規則數量有所限制，也就是能 block 的 URL 是有限的，如果 tracking domain 無限地多，很快就會撞到上限了。
- 相較於網站管理員只需要幾分鐘就能改 CNAME 設定並創造新的 tracking domain（這個過程甚至可以被自動化），filter list maintainer 可能需要花很長一段時間才會意識到出現新的 tracking domain，且需要時刻注意 tracking domain 會一直改，這大幅增加了維護 filter list 的成本。
- Browser 中有一些機制會因為是 first-party subdomain 而有較為寬鬆的限制，例如前述的共享 cookie，這使得放在 first-party subdomain 的 tracker 可以蒐集到更多資訊。
- 使用者與 filter list maintainer 可能因為 tracking domain 是 first-party subdomain 而放下戒心，沒意識到這是危險的。

CNAME cloaking 目前被非常廣泛地使用，例如追蹤公司 Criteo 擁有一個 domain `dnsdelegation.io`，可以看看有多少[根本不屬於 Criteo 的 domain 用 CNAME 指向他們的網域](https://securitytrails.com/list/cname/dnsdelegation.io)。

### CNAME Cloaking 的解法
CNAME cloaking 從[第一次](https://medium.com/nextdns/cname-cloaking-the-dangerous-disguise-of-third-party-trackers-195205dc522a)被 security community 發現至今也才不過短短三年，因為其重要性、普遍程度以及影響力，已經吸引許多研究者做深入的探究，也促使各個 anti-tracking 的工具提出相對應的解方。

解決 CNAME cloaking 有個看似簡單的解法：讓 content blocker 去檢驗每個 domain 的 DNS resolve 出來有沒有指向已經在 filter list 裡面的 tracking domain。然而問題是，多數的 browser 都沒有給 extension 查詢 DNS 的權限，唯一的例外是 [Firefox](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/dns)，所以目前 Firefox 上的 uBlock Origin 已經可以抵禦 CNAME cloaking。另一個作法是直接把防禦機制放在 browser 內部，就可以有足夠的權限看到 DNS query 了，這就是目前 Safari 的作法，即我們以後會介紹的 ITP。

## Bounce Trackers
Bounce Tracking 是一種非常常見的 web tracking 技術，其特色在於，所有 tracking 都是在 first party 完成的，就算沒 third-party cookie 一樣能運作。

當使用者在 `blog.example` 上點擊一個連向 `shop.example` 的連結，理應會直接把使用者帶到 `shop.example`，但 bounce tracker 會很快速的把使用者依序帶到 `tracker1.example`, `tracker2.example`, `tracker3.example`, ...，並在每次轉跳時都帶上目的地的 URL，最後才轉到真正的目的地（`shop.example`）。當使用者被轉到 `tracker1.example` 時，`tracker1.example` 知道了這個使用者嘗試造訪 `shop.example`，而且因為 `tracker1.example` 的 cookie 在此時是 first-party cookie，所以 `tracker1.example` 可以輕鬆讀到自己 cookie 內的 identifier。以此類推，`tracker2.example` 與 `tracker3.example` 也都可以知道使用者要造訪 `shop.example` 並且知道使用者在自己手上的 identifier。

當同個使用者在 `another-blog.example` 上又點擊了 `social.example` 的連結，一樣一開始會把使用者依序帶到 `tracker1.example`, `tracker2.example`, `tracker3.example`, ...，最後才轉到目的地。於是 `tracker1.example` 因為可以讀到 identifier，於是知道這個使用者不只之前造訪過 `shop.example`，現在還準備造訪 `social.example`。

一個很常見的例子是，讀者可能也有發現過，在 Facebook 或許多其他網站上，當使用者點擊連結時，並不是直接連結過去，而是先連到類似 `social.example/link?url=...` 的地方，再轉跳出去，這就是一種 bounce tracker。

![](/images/bounce-tracker.png)

目前針對 Bounce Trackers 有最多討論的是 Apple's ITP 與 Brave。ITP 的方法會於幾天後有專文討論。Brave 則是提供一個很酷的功能是 "[debouncing](https://brave.com/privacy-updates/11-debouncing/)"，它會去猜使用者真正的目的地為何（通常都直接夾帶於 URL parameter 裡面），直接把使用者帶過去，跳過中間一堆 bounce tracker。

## Link Decorations
Link Decorations 與 Bounce Trackers 類似，都是讓 tracker 可以在不使用 third-party cookie 下拿到 identifier。

當使用者在某個社群平台 `social.example` 點擊連結時，`social.example` 可能會希望使用者離開自己網站後仍然可以被它追蹤，於是就在連結後面加上一個 URL parameter，例如把 `blog.example/post/123` 改成 `blog.example/post/123?click_id=foobar`，而 `blog.example` 上可能又有 `social.example` 的 script（例如留言板、分享按鈕等），於是 `social.example` 的 script 在回傳使用者行為時，就可以夾帶 `click_id`，而後端因為知道 `click_id=foobar` 是誰，於是 `social.example` 就可以了解使用者在 `blog.example` 的行為了。

![](/images/link-decoration.png)

使用者可能注意過，在 Facebook 上連出去的文章很多都帶有神秘的 URL parameter `fbclid` ，這就是 link decoration，如果目標網站有引入 Facebook 的 script（像是按讚、分享按鈕），Facebook 就可以在該網站繼續追蹤使用者的行為，並且 Facebook 可以知道這個使用者是誰。其他常見的 link decoration 所使用的 URL parameter 還包含 Google Analytics 的 `utm_*` 之類的。

使用 Chrome 或 Firefox Desktop 的人，可以使用 [ClearURL](https://clearurls.xyz) 來移除這些用於 tracking 的 URL parameter。

除了直接儲存於目標 URL 以外，另一個方法是用 HTTP Referrer 來帶資料。HTTP referrer 是一個夾帶於 request header 的資訊，用於表達「請求從哪個 URL 來的」。例如我在 `https://blog.example/post/1` 點擊 `https://shop.example`，則對後者的請求會夾帶 `Referrer: https://blog.example/post/1`。除了 HTTP request header 以外，也可以用 Javascript 的 `document.referrer` 取得 referrer。Referrer 也可以被用於 link decoration，如果使用者在 `social.example` 上點擊連結，他可能先被帶到 `https://social.example/link?url=blog.example...&click_id=foobar`，然後 `social.example/link` 再把使用者帶到 `blog.example`。當使用者抵達時，`social.example` 在 `blog.example` 上的 tracker 便可以利用 `document.referrer` 拿到其 referrer 是 `https://social.example/link?url=blog.example...&click_id=foobar`，並還原出 `foobar`。使用者不會在 URL 上看到怪怪的 parameter，也可能沒注意到中間多了一層轉跳，但 tracker 已經拿到 identifier 了。

題外話，其實 Bounce Tracking 與 Link Decortation 這一系列基於 navigation 的追蹤方法，有個稍微學術一點的詞彙叫做 "navigation based tracking"。

## 參考資料
[CNAME Cloaking, the dangerous disguise of third-party trackers](https://medium.com/nextdns/cname-cloaking-the-dangerous-disguise-of-third-party-trackers-195205dc522a)
[Fighting CNAME Trickery](https://brave.com/privacy-updates/6-cname-trickery/)
[Intelligent Tracking Prevention 2.0](https://webkit.org/blog/8311/intelligent-tracking-prevention-2-0/)
[Intelligent Tracking Prevention 2.2](https://webkit.org/blog/8828/intelligent-tracking-prevention-2-2/)