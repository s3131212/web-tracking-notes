---
weight: 2
title: "如何達成 Cross-site Tracking：Third Party Cookie & Cookie Syncing"
---

# 如何達成 Cross-site Tracking：Third Party Cookie & Cookie Syncing
在前一篇文章，我們討論了如何儲存 identifier，介紹了數個方法。其中幾個方法的介紹中，有提到這可以用來做 cross-site tracking，包含 cookie, HSTS supercookie, document cache, HTTP redirection cache 與 ETag。除了 Cookie 以外，在前文也都討論過可以如何防止了。此文將著重討論 cookie 如何作到 cross-site tracking，如何防禦。此外，我們將介紹一個與之相關的技術稱為 cookie syncing。

首先複習一下，在上一篇文章提過，cookie 以 domain 為單位區分，也就是 Same-origin Policy (SOP)，`blog.example` 和 `news.example` 的 cookie 是獨立、互不干擾也不能互相存取的；此外，cookie 會被自動夾帶於 HTTP requests，且可以用 HTTP response 設定。這兩個特性非常重要，之後會不斷用到。

## First-party & Third-party Cookie
Cookie 可以被分為兩種：first-party cookie 與 third-party cookie。兩者其實沒什麼根本性的不同，都是在瀏覽器儲存一些 key-value pairs。他們的差異在於 first-party cookie 是直接儲存於使用者造訪的 domain 底下，而 third-party cookie 則是儲存於其他 domain 底下。

例如使用者正在瀏覽 `blog.example`，則 `blog.example` 的 cookie 是 first-party cookie，在這個網站裡面包含了 `social.example` 的資源，因為 cookie 的特性，送請求給 `social.example` 時會自動夾帶 `social.example` 底下的 cookie，這就是 third-party cookie。因此我們也可以看出，first-party 或 third-party 並不是 cookie 本身的性質，而是取決於我的使用情境，當我在瀏覽 `social.example` 時，`social.example` 的 cookie 就變成 first-party cookie 了。

以下做個簡單的舉例說明 third-party cookie 可以被如何運用。假設現在有個線上支付網站，我們就稱之為 PalPay 好了（？），它提供一個功能是，網站管理員可以在自己的網站插入一個贊助按鈕，使用者點下去，網站就會發一個請求給 PalPay 的伺服器，PalPay 的伺服器就會偵測這是哪個使用者，並自動贊助一筆錢給網站管理員。這應該是個很常見的情境。一般來說，我們會把登入狀態相關資訊儲存在 cookie 中，可能是 session ID 或是 JWT。當使用者點擊贊助按鈕，網站發請求給 PalPay 的伺服器時，PalPay 的 cookie 便會被自動帶入 HTTP request，如此則 PalPay 可以知道目前登入的 PalPay 使用者是誰，或是發現這個使用者根本沒登入 PalPay。在這個故事中，PalPay 的 cookie 就是 third-party cookie，因為使用者正在瀏覽其他網站，不是 Palpay 自己的網站。

同樣的，像是按讚之類的功能也都是這樣做到的：Facebook 需要藉由 HTTP request 自動帶入的 cookie，才能知道現在按讚的人是誰。

## Third-party Cookie 如何做到 Cross-site Tracking
藉由 browser 會自動在 HTTP request 夾帶 cookie 的性質，可以做到 cross-site tracking。

試想現在我正在瀏覽 `blog.example`，而有個追蹤者 `tracker.example` 想要追蹤我的瀏覽紀錄，這個追蹤者與 `blog.example` 的網站管理員合作，把 tracker 放在 `blog.example` 的網頁中。

於是當我第一次瀏覽 `blog.example` 時：
1. 瀏覽器存取 `blog.example`
2. `blog.example` 包含 `tracker.example` 的資源，所以瀏覽器再發請求給 `tracker.example`
3. `tracker.example` 發現請求沒有 cookie，所以生成一個新的 identifier 並在 response header 夾帶 `Set-Cookie: tracker_id=<identifier>`
4. 瀏覽器收到回應，儲存該 cookie 到 `tracker.example` 底下

當我重複造訪 `blog.example` 時：
1. 瀏覽器存取 `blog.example`
2. `blog.example` 包含 `tracker.example` 的資源，所以瀏覽器再發請求給 `tracker.example`，並夾帶 cookie `tracker_id=<identifier>`
3. `tracker.example` 發現請求有 cookie，於是知道擁有這個 identifier 的人再次造訪 `blog.example` 了

但仔細想想，會發現其實就算不是 `blog.example`，也會有一樣的性質。試想我現在存取 `news.example`：
1. 瀏覽器存取 `news.example`
2. `news.example` 包含 `tracker.example` 的資源，所以瀏覽器再發請求給 `tracker.example`，並夾帶 cookie `tracker_id=<identifier>`
3. `tracker.example` 發現請求有 cookie，於是知道擁有這個 identifier 的人造訪 `news.example` 了

於是 `tracker.example` 不只知道使用者造訪了 `blog.example`，還知道同一個使用者也造訪了 `news.example`。如此則達成 cross-site tracking 了。在直覺上，我們可能會以為 `blog.example` 與 `news.example` 是兩個不同網站，應該不會有人發現我同時瀏覽了 `blog.example` 與 `news.example`，但藉由 third-party cookie 的功能，追蹤者便可得到這樣的資訊。

![](/images/cookie-cross-site-tracking.png)

這麼做對追蹤者而言有諸多好處，例如，假設網站 A 與電商 B 都有與追蹤者合作，如果我在網站 A 瀏覽了筆電的開箱文，又到了電商 B 買鞋子，因為追蹤者知道我曾經在網站 A 瀏覽過筆電的開箱文，也知道我正在瀏覽電商 B，於是他可以把「我對筆電有興趣」這項資訊賣給電商 B，電商 B 就可以利用這項資訊推播筆電相關的廣告給我看。

## SameSite cookies
Cookie 的特性會帶來一個問題是，在那些我們其實不需要 cookie 的請求中，也都會把那些用不到的 cookie 塞進去。例如，假設有個電商網站把購物車內的商品儲存在 cookie 之中，則當使用者瀏覽其他網站時，如果該網站包含了電商網站上的圖片，則在該圖片的 HTTP request 中也會夾帶購物車內容的 cookie，但那就只是一張圖片，為什麼需要知道購物車內容？這只是在浪費網路流量啊！

另一個情境是 CSRF（Cross Site Request Forgery）的防禦。假設現在有個部落格，其刪除文章的連結是 `GET /removePost?id=[id]`，需要在管理員登入下才能操作。攻擊者可以建立一個網頁，上面包含了 `<img src="https://blog.example/removePost?id=[id]" />`，然後誘導管理員去訪問該網頁。當管理員訪問該網頁時，瀏覽器就對 `https://blog.example/removePost?id=[id]` 發出請求，並自動夾帶 `blog.example` 的 cookie（其中包含管理員的登入狀態），於是伺服器收到請求，看看 cookie 發現這的確是管理員的 cookie，就把文章刪掉了！

為了解決這些問題，便為 Cookie 新增了一個屬性：SameSite。SameSite 屬性可以指名該 cookie 只有在怎樣的 context 下可以被夾帶。SameSite 屬性有三種不同的值：Strict, Lax, None。在同個網站下，HTTP request 一定會夾帶 cookie，例如我在 `example.com` 對 `example.com` 發出請求（e.g. 載入自己伺服器上的圖片），無論在哪種模式下都會夾帶 cookie。但在不同模式下，跨網站（或更嚴格來說，cross-origin）的請求會有不同的表現，以下將依序介紹。

`SameSite: Strict` 要求，只有在 first-party 的 context 下可以傳送 cookie，從其他網頁點進來（也就是 top-level navigation），或是別的網頁引入這個 domain 的資源，都不會夾帶 cookie。更直白來說，只要是從 third-party 發起的請求，無論是載入資源，或使用者點擊連結，都不會夾帶 cookie。

沿用剛剛電商的例子，當使用者從別的網頁點進電商網站時，瀏覽器對電商網站的 HTTP request **不會**夾帶 cookie，於是電商網站就不知道使用者的登入狀態、購物車內容等。但當使用者在電商網站裡面又點了一個連結或是引入電商網站的其他資源（例如載入圖片），因為此時是 first-party context 了，所以這些請求**會**夾帶 cookie。這麼做的好處在於，如果是變更密碼之類的連結，可以確保使用者不會一點進去密碼就被改掉了。

但這聽起來很不方便，對吧？

`SameSite: Lax` 是弱化的版本，一方面如同 `Strict`，如果在別的網頁引入這個 domain 的資源，**不會**夾帶 cookie，可是如果是 top-level navigation，則**會**夾帶 cookie。繼續沿用電商的例子，如果使用者從別的網頁點連結進入電商網站時，HTTP request 會夾帶 cookie，所以使用者點進電商網站時，可以順利維持登入狀態與購物車內容，但其他網站如果有載入電商網站的圖片，就不會夾帶 cookie 了。

至於傳統的運作方式則是 `SameSite: None`，或是直接不設定，如此則只要有 HTTP request 就一定會夾帶 cookie。

SameSite 屬性可以在 HTTP response 中如此設定：
```
Set-Cookie: key=value; SameSite=Lax
```
或是在 Javascript 中如此設定：
```
document.cookie = "key=value; SameSite=Lax";
```

如果設定 cookie 時沒有指定 SameSite，目前主流 browser（Chrome, Firefox, Edge, Safari）都預設把 cookie 的 SameSite 屬性設定為 Lax。

## Third-party Cookie 的輓歌... 嗎？
Third-party cookie 作為最古老的 tracking 技術，若要解決 web tracking，自然也是首要敵人。大概從五六年前起，主流 browser 便陸陸續續地說要對 third-party cookie 宣戰，最初是期待大概在 2020 年左右要全面封殺 third-party cookie，也就是不再容許 `SameSite=None`。Chrome 一開始是說 2020 全面禁用，不過因為各種原因，主要大概是對網路生態影響太大，這個死線就一直被往後推，目前的[新聞](https://www.theverge.com/2022/7/27/23280905/google-chrome-cookies-privacy-sandbox-advertising)是說 2024 年才會全面。[Firefox](https://blog.mozilla.org/en/security/firefox-rolls-out-total-cookie-protection-by-default-to-all-users-worldwide/) 則是在不久前的 2022 年六月封鎖 third-party cookie，不過他們不是單純禁止，而是把不同 domain 下的 third-party cookie 分開儲存，沿用之前的舉例， `blog.example` 中 `tracker.example` 的 cookie 與 `news.example` 中 `tracker.example` 的 cookie 不一樣也不能互相存取。[Safari](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/) 在 2020 年就全面禁用 third-party cookie 了。我們將於 Day 8 與 Day 9 分別討論 Safari 與 Firefox 的防禦方式。

其實早在瀏覽器禁止 third-party cookie 之前，就已經有 Privacy Badger 等 extension 來阻擋 third-party cookie。瀏覽器禁止 third-party cookie，比起說是一種創新或是進步，更像是還在努力跟上資安社群的腳步，嘗試在隱私與方便找個大家都能接受的平衡。

雖然 third-party cookie 看起來是差不多死了，但主流瀏覽器中還是有市占率最高的 Chrome 支援，所以，大概是還沒死透吧。

## Cookie Syncing
在為 third-party cookie 上香之前，我想要再介紹一個 third-party cookie 時代很有趣的技術：cookie syncing。

試想現在有兩個追蹤者 A（`trackerA.example`）與 B（`trackerB.example`），他們約定好要共享使用者資料，也就是 B 要知道這個使用者在 A 那邊的 identifier 是什麼，如此則 B 可以維護一張 A, B 之間 identifier 的 mapping。但問題是，因為 SOP，`trackerA.example` 與 `trackerB.example` 的 cookie 沒辦法互相讀取，所以 B 拿不到 A 的 cookie，反之亦然。那，該怎麼解決呢？Cookie syncing 的目標便在於，讓 A, B 可以拿到對方的 cookie。

假設 A 的 cookie 是 `trackerA_id=abc`，在 B 的 cookie 是 `trackerB_id=123`，如果 A 想要讀到 B 的 cookie：
1. 網站向 `trackerB.example` 請求一個資源：`https://trackerB.example/tracker.gif`，這個請求會夾帶 `trackerB.example` 的 cookie
2. `trackerB.example` 回傳 `307 Temporary Redirect` 並導向 `https://trackerA.example/tracker.gif?trackerB_id=123`
3. 瀏覽器收到回應，向 `trackerA.example` 請求資源 `https://trackerA.example/tracker.gif?trackerB_id=123`，這個請求會夾帶 `trackerA.example` 的 cookie
4. `trackerA.example` 收到請求，包含 URL 中的 `trackerB_id=123` 以及 HTTP request header 中自己的 cookie `trackerA_id=abc`
5. A 於是知道 A 眼中的 `abc` 就是 B 眼中的 `123`，於是 A 與 B 之間 identifier 的 mapping 關係就建立起來了

藉由 cookie syncing，可以有效讓不同 domain 之間的 cookie 得以同步。

## 參考資料
[What’s the Difference Between First-Party and Third-Party Cookies?](https://clearcode.cc/blog/difference-between-first-party-third-party-cookies/)
[SameSite cookies explained](https://web.dev/i18n/en/samesite-cookies-explained/)
[What is Cookie Syncing and How Does it Work?](https://clearcode.cc/blog/cookie-syncing/)