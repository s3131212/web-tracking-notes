---
weight: 11
title: "Scheme Flooding"
---

# Scheme Flooding

之前我們看到可以藉由使用者安裝的 browser extension 來做 fingerprinting，於是不禁想問：電腦上裝的應用程式有沒有可能也拿來做 fingerprinting？或是更重要的，有辦法在 browser 上知道系統有哪些應用程式嗎？今天想來介紹一個很有趣的 Browser Fingerprinting 技術稱為 scheme flooding。許多應用程式有所謂的 "custom URL scheme" 或是 "custom protocol handler"，例如 `skype:`, `spotify:`, `itunes:` 等，當瀏覽器看到這些 protocol handler 時，便會嘗試打開相對應的應用程式。這個功能看似好用，但可能打開新的 attack vector。假如我可以窮舉所有 protocol handler，看哪些會有反應，似乎就可以知道使用者安裝了哪些應用程式，並藉由這項資訊來做 profiling 與 tracking。這就是所謂的 scheme flooding。

大家可以嘗試到這個網站看看你的瀏覽器有沒有出賣你：https://schemeflood.com/

但在開始之前，我想要先針對一些用字與定義做一些說明。在此文中我把 scheme flooding 定位成一種 browser fingerprinting 技術，但這樣的定義會有兩種問題。首先，scheme flooding 在發現者眼中是一種「可以達成 fingerprinting 的漏洞」，而不是單純的 fingerprinting 技術，在[有些媒體](https://www.ithome.com.tw/news/144424)上也稱之為漏洞，此處我無意開啟何謂漏洞的討論，而是只想把重點著重在：這個特性可以被用來做 fingerprinting，至於它本身是不是漏洞，並非此文討論的重點。此外，scheme flooding 是 fingerprinting 技術，但它算是 browser fingerprinting 技術嗎？這部份涉及了 browser fingerprinting 的嚴格定義，不過考慮到 scheme flooding 符合 (1) 只需要在 browser 上做運算，而沒有儲存任何內容，也就是 stateless 的，以及 (2) 其所刺探的資訊與 browser 所運行的 OS 有關（嚴格來說，OS 之中安裝了哪些程式），所以我還是想將其定義為 browser fingerprinting。如果讀者不認同我的定義，一來我很期待能聽到不同意見，二來即便此文在讀者眼中可能用字不精確，但我相信還是會有些幫助的。

## 何謂 Custom Protocol Handler

我們可以簡單地設想一個情境：當使用者瀏覽網頁時，如果可以直接打開應用程式的指定畫面，那對使用者體驗會有所改善。舉例來說，當我在某個社群媒體上看到有人分享一首歌，可以點一下就打開 Spotify 並播放這首歌，將會很方便。因此就有了 "custom protocol handler"，其運作方法是，應用程式可以註冊一個 protocol handler，例如 `spotify:`，以後只要瀏覽器看到這個 protocol handler，就會打開指定的應用程式，並且把 URL 轉給應用程式，讓應用程式去完成剩下的事情。

![](/images/custom-protocol.png.png)


## Scheme Flooding 如何運作？
Scheme Flooding 的核心精神在於：嘗試窮舉所有可能的 protocol handler，然後找出那些存在的 protocol handler。

聽起來很簡單，但有個大魔王是 Same Origin Policy，以及 browser 的開發者也不是笨蛋，例如 Chrome 就設下了一些限制。所以接下來來討論一些常見 browser 可以怎麼做 scheme flooding。

### Chrome
Chrome 的防禦大概是所有裡面做的最好的，如果要啟用 custom protocol handler，需要 user gesture 來啟動，例如需要透過使用者點擊等。這基本上抵禦了 scheme flooding，畢竟不可能要使用者幫追蹤者點一堆連結吧。

然而 Chrome 卻開了一個後門：extension 可以設定一個 flag 繞過 user gesture 直接打開 custom protocol handler。而其中有個預設就有的 extension 是 PDF Viewer。於是只要先打開一個 PDF 檔案再打開 protocol handler，就可以繞過 user gesture 的限制。

目前這個 bug 還沒有被修正，有興趣的人可以追蹤相關 issue： [Issue 1208903: External Handler detection technique allows reliable cross-browser fingerprinting](https://bugs.chromium.org/p/chromium/issues/detail?id=1208903)

不過蠻有趣的是 Linux 反而不受影響，因為 Chrome 使用 `xdg-open` 來處理 custom protocol handler，可是在真的召喚 xdg-open 之前 Chrome 並不會知道哪些 custom protocol handler 存在。這算是種意外的收穫嗎？

### Firefox
Firefox 的偵測方式也很有趣，算是一種意外的 side channel 吧。當一個 custom protocol 存在時，Firefox 會打開 `about:blank` 並跳出一個彈跳方塊問使用者是否要打開應用程式，而 `about:blank` 可以從任意 origin 存取，所以如果我在一個網頁裡面加上 `iframe` 指向特定 custom protocol，則可以讀取到 `iframe` 的內容。可是，如果一個 custom protocol 不存在，Firefox 會顯示錯誤畫面，這個錯誤畫面沒有辦法被其他 origin 讀取。於是，我只要看 `iframe` 的內容能不能被讀取即可，如果可以，代表現在 `iframe` 裡面是 `about:blank`，所以 custom protocol 存在，如果不行，則代表 `iframe` 裡面是錯誤畫面，custom protocol 不存在。

Safari 也是用類似的方式偵測，所以就不贅述了。

## Scheme Flooding 的危害
相較於其他 browser fingerprinting 的技術，scheme flooding 就如同 browser extension fingerprinting 一樣有更進一步的危害。藉由使用者有安裝什麼應用程式，除了可以拿來做 fingerprinting 以外，還可以更進一步做 profiling。舉例來說，如果發現使用者有安裝某些 IDE，可能就代表該使用者是個工程師，如果使用者有裝 Slack，可能代表這台電腦是工作用的電腦等。

## 防禦
目前看來大家對於如何防禦 Scheme Flooding 還沒有太多共識。一個可能的解法是讓 custom protocol handler 存在與不存在時的反應一模一樣，但這可能還是有些問題：首先是從反應時間之類的 side channel 有機會還原出一個 custom protocol handler 是否存在；二來是這對開發者來說很不友善，他們很可能無法知道自己的 code 為何無法召喚 custom protocol handler，因為不會有任何 error 或特殊 behavior 出現（不過我還在想有沒有可能還是有 console error，但這些錯誤是 JS 看不到的）。


## 參考資料
[Exploiting custom protocol handlers for cross-browser tracking in Tor, Safari, Chrome and Firefox](https://fingerprint.com/blog/external-protocol-flooding/)