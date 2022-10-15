---
weight: 4
title: "Browser Fingerprinting"
---

# Browser Fingerprinting：如何被獨一無二的 Browser 出賣
在過去數篇文章，我們都在討論 stateful tracking，tracker 需要儲存一些 identifier，然後各種隱私保護機制想辦法避免 identifier 被寫入或是存活太久。如果 identifier 其實可以不用儲存，而是用算出來的呢？那是不是意味著 identifier 沒辦法被刪掉，因為只要重新算一次就好了？這就是所謂的 stateless tracking，其中最大宗的技術是今天要介紹的 browser fingerprinting。

早從 Web 剛出現的遠古時代開始，為了改善使用者體驗，browser 會傳送一些資訊幫助網站提供客製化的網頁結果。例如，為了提供不同語言的網站給不同語言的使用者，browser 會在 HTTP request 裡面夾帶使用者偏好的語言，也可以在 Javascript 使用 `navigator.languages` 拿到偏好語言；為了讓網頁可以根據使用者時區顯示不同時間，可以用 Javascript 從 `new Date().getTimezoneOffset()` 拿到時區；為了讓網頁可以根據不同視窗大小調整 UI，可以用 `screen.height` 與 `screen.width` 取得螢幕長寬。

其中最重要的資訊莫過於 user agent，它會透漏使用者的作業系統、瀏覽器、版本等，如此則當使用者要下載軟體時可以下載到符合其所用作業系統的版本，或是網頁可以針對不同瀏覽器做特別處理，使用一些只有特定瀏覽器支援的功能。User agent 會夾帶於 HTTP requests，也可以用 Javascript 的 `navigator.userAgent` 取得。

另外，不同的執行環境也會影響 browser 的性質。例如，不同顯卡或驅動程式在渲染圖片上可能有些微的差異；當網頁指定某個字型時，電腦有沒有安裝該字型會決定 browser 能不能順利顯示。又或是其他硬體設備的差異，像是電腦有一個或多個麥克風之類的。

上面舉的各種情境，都反映一件事：每個 browser 都多少有一些性質差異。可能我的預設語言是英文，我朋友的是中文；我的時區是 UTC+8，他的是 UTC+9；我的電腦有標楷體，他的沒有。如果把 browser 的各個性質都蒐集起來，便會發現很難在世界上找到兩個完全一模一樣的 browser。這聽起來很像是個 identifier 對吧？我們將其稱為 *Browser Fingerprinting*。

Browser fingerprinting 是一種藉由瀏覽器的各種 feature 去建構獨一無二的 identifier，並以此識別與追蹤使用者的技術。

## Browser Fingerprinting 範例
有個很棒的網站叫做 [AmIUnique](https://amiunique.org/)，可以自動對使用者的瀏覽器做幾個常見的 browser fingerprinting 方法，並算出該結果有多獨特。

下圖是我的電腦用 Chromium 算出來的結果：
![](/images/amiunique.png)

首先可以注意到，我的 user agent 非常罕見：只有 0.02% 的人跟我有一模一樣的 user agent。光是 user agent 就已經洩漏非常多資訊了。此外，有 26% 的使用者用 Linux，挺沒說服力的，可能因為會用這網站的人都是在乎隱私的技術人員吧。只有 0.01% 的人設定的語言是 `en-US,en,zh-TW`。光是從我有截圖的部份，就可以看出我的 browser 有多少 feature 是罕見的。

最糟糕的是，在我撰文時，該資料庫有接近八十萬筆 browser fingerprint，沒有任何一個人跟我有一樣的 fingerprint。換言之，我的 browser 可以被唯一地識別出來，而這過程中完全沒有存取 cookie 或其他 storage，也沒有跟我要任何特殊權限。

## Browser Fingerprinting 的特性
希望上面的範例已經讓讀者感受到 browser fingerprinting 有多可怕。此段將介紹 browser fingerprinting 的幾個重要的特性。

### 同時使用多個 Features
很少會遇到某個 feature 可以唯一地識別使用者。舉例來說，系統語言是波蘭文的人很多，時區是 UTC+11 的很多，作業系統是 Linux 的也有不少，但系統語言是波蘭文且時區是 UTC+11 且作業系統是 Linux 的大概沒多少個。基本上現代的 browser fingerprinting 都是非常多 feature 聚合在一起，才能達到理想的準確率。在下一篇文章，我們會用數學黑魔法來討論每個 feature 可以貢獻多少資訊。

### 不需要特殊權限
此外，browser fingerprinting 不仰賴特殊權限，因為這些資訊在早期都被認為是沒有危害的，可以放心地讓 Javascript 讀取，所以使用者很可能在不知不覺之間就被 fingerprint 了。不過有些在乎隱私的瀏覽器，像是 Tor Browser，開始會在一些很可能被用於 browser fingerprinting 的功能加上權限管制或是取得使用者同意，像是 canvas。

### 不保證唯一與永久不變
Browser fingerprinting 所建構出來的 identifier 無法保證獨一無二。雖然機率不高，但總會遇到真的有兩個 browser 剛好在這些 feature 上都是一樣的。

最後，browser fingerprinting 並不是永久不變的。許多很 minor 的小事都可能讓 fingerprint 改變，例如換了一個螢幕、更新驅動程式、更新瀏覽器、裝了一個新的 extension，這些小事都可能讓 fingerprint 改變。不過這並不意味著就無法用於追蹤，如果改變足夠小，還是有可能把兩個 fingerprint 連結在一起。更何況也不是只有 browser fingerprinting 可以追蹤使用者。假設追蹤者同時使用 cookie 與 browser fingerprinting 追蹤使用者，則如果 browser fingerprint 改變了但 cookie 沒有改變，追蹤者還是能藉由 cookie 繼續追蹤，並把改變前後的 browser fingerprint 連結在一起，反之亦然，如果 cookie 被刪除了但 browser fingerprinting 還在，就能重新把原本 cookie 中的 identifier 放回去。

### 難以防禦
Browser fingerprinting 可說是出了名的難以防禦。如果阻擋 third-party cookie 就已經可以誤殺這麼多正當使用情境，那阻擋可以用於 browser fingerprinting 的 feature 幾乎可以說是毀了整個 web。我們不可能不讓網頁存取使用者的時區與語言偏好，不可能不讓網頁渲染一張圖，不可能不讓網頁使用特殊字型，這些都會使如今非常蓬勃的 web 生態不復存在。

此外，可以 leak 資訊的地方真的太多了，有許多現在非常知名的 browser fingerprinting 技術，在當初發展時根本沒人意識到這會 leak 這麼多資訊。只要 browser 還在演進，就會有越來越多功能，也就有越來越多可能洩漏資訊的地方。

防禦 browser fingerprinting 的手段很多，其中有些已經被證實為無效或甚至有害，有些則是對使用者體驗影響太大，甚至可以說，至今還沒有一個真正足夠有效且不傷害使用者體驗的防禦方式。我們在未來會有數篇專文討論 browser fingerprinting 抵禦的困境。


此文是 browser fingerprinting 系列文章的第一篇，也意味著我們告別了 stateful tracking，來到了新的篇章。期待大家會喜歡接下來的文章。

## 參考資料
[AmIUnique](https://amiunique.org/)
[Fingerprint Randomization](https://brave.com/privacy-updates/3-fingerprint-randomization/)
[Browser Fingerprinting: An Introduction and the Challenges Ahead](https://blog.torproject.org/browser-fingerprinting-introduction-and-challenges-ahead/)