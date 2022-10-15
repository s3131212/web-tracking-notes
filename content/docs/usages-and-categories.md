---
weight: 2
title: "Web Tracking 的角色、用途與分類"
---

# Web Tracking 的角色、用途與分類

在開始討論具體技術之前，我還想額外討論 web tracking 常見的用途以及試圖解決的問題。畢竟 web tracking 這個領域存在各種不同技術是一回事，它們為何存在、當初發明它是想解決什麼，又是另一回事。我希望藉由這篇文章去讓大家了解到 web tracking 到底想解決什麼問題，而這個問題又為何會重要，甚至是需要被解決。

從不同使用情境，我們又可以把 web tracker 做分類，不同類型的 web tracker（追蹤器）可能有不同的特性與危害。雖然這與 web tracking 技術沒有直接相關，但可以讓我們更了解 web tracking 的生態。

## Web Tracking 的角色
在 web tracking 的資料中，大概可以分成四個角色：使用者、網站管理員、追蹤者、資料使用者。

使用者是資料的來源，他們在網站上的使用行為會被記錄下來。網站管理員就是網站管理員（？），通常他們會和追蹤者合作，在網站中埋入追蹤者的 tracker，讓使用者在他的瀏覽器中執行 web tracking 的 code。網站管理員可能基於許多誘因與追蹤者合作，例如藉此使用追蹤者提供的服務（e.g. Facebook 的 Like 按鈕、Google Analytics），或是有經濟誘因。追蹤者則是主要開發與部屬 tracker 的單位，他們蒐集到資料後，會賣給或分享給資料使用者，也就是最後需要用這些資料的單位。資料使用者則會利用這些資料去投放廣告或提供服務。在許多情境中，資料使用者和追蹤者可能是同一個單位，一方面蒐集資料一方面使用資料，像是 Facebook。甚至可能資料使用者就是網站管理員自己，藉由這些追蹤者從網站上拿到的資料，網站管理員可以更加了解使用者如何操作自己的網站，網站可以如何改善。

![](/images/tracking-party.png)
另外，因為分析者可能需要很多不同來源的資料，希望有人可以幫忙 aggregate 好整包一起賣，於是有個產業叫做 data broker（資料仲介）。Data broker 可能會跟追蹤者買資料，可能他們自己就同時也是追蹤者，他們蒐集到或買到資料後會為每個使用者建立 profile，把這些資料賣給需要資料的人，像是廣告商。

這個故事做了許多簡化。在真正的廣告產業中，還有非常多不同單位，像是有負責整理廣告 demand 的，有負責處理廣告欄位 supply 的，也幫忙做 bidding（廣告欄位競標）的。[廣告產業的複雜程度](https://lumapartners.com/content/lumascapes/display-ad-tech-lumascape/)簡直是場災難，我只挑出了與 tracking 密切相關的角色，其餘的就被我簡化掉了。

## Web Tracking 的用途
### 廣告投放
Web tracking 最早便是為了投放廣告而出現的。藉由追蹤使用者造訪過哪些網頁，便可推測使用者可能對什麼有興趣，藉此投放廣告。至今 web tracking 最常見的用途仍然是為了蒐集資料做廣告投放，在可預見的未來大概也不會改變。

當今整個廣告產業有很大一部份著重在精準投放，甚至可以說，如果有一天精準投遞消失了，廣告產業雖然不會消失，但肯定和現在很不一樣。舉個簡單的例子，Apple 之前在 iOS 14.5 加上 App Tracking Transparency 功能，就讓 Facebook 跑出來[哀號](https://www.cnbc.com/2022/02/02/facebook-says-apple-ios-privacy-change-will-cost-10-billion-this-year.html)說他們會損失 10B 的利潤。題外話亂扯個，這個案例比起說是在講 tracking 對廣告產業多重要，更重要的可能是 Apple 的壟斷地位到底有多恐怖，一個小功能可以造成這麼大的影響。

### 聯盟行銷（Affliate marketing）
另一個同樣與廣告有關的是聯盟行銷，或有時稱為夥伴計畫（Affiliate / Referral Program）等，其運作方式是，如果使用者可能會在網站 A 上看到某個商品的宣傳，於是他從網站 A 藉由點擊連結或任何其他方式轉跳到某個電商 B，則當使用者在電商 B 消費時，網站 A 可能會獲得分潤之類的。此時電商 B 需要有個方法使其知道使用者是從網站 A 來的，web tracking 在此便可派上用場。

### 個人化推薦
類似於投放廣告，個人化推薦為透過分析使用者瀏覽網頁的行為來找到使用者的偏好並藉此推薦文章、影片、商品、搜尋紀錄等。舉例來說，如果 Facebook 發現我在瀏覽偏好某個政黨的新聞，他可能就會想推薦給我跟支持此政黨有關的貼文等。

### 網站分析（Web Analytics）
對於網站擁有者而言，使用者進入這個網站之前與之後發生了什麼事是很重要的，例如流量來源、逗留時間、轉換率等，都是重要的資訊。為了讓使用者在網站上的行為可以被連貫地分析，例如找到重複瀏覽者等，必須使用 web tracking 技術來輔助。

另外一個與之類似的是 usability tests，在於分析網站的可用性，例如追蹤使用者的滑鼠移動與點擊，藉此了解使用者究竟是如何使用這個網站，他們的使用方式是否符合期待，或是可能網站設計無法使他們正確了解如何操作等。

許多網站會做 A/B test，此時 web analytics 與 usability tests 便至關重要，可以提供許多有用的資訊，協助網站管理員做出好的設計決策。

Web analytics 與 usability tests 雖然會蒐集許多資訊，但可能是無害的，因為這些東西通常只會被用於內部分析，而且也不太會去做跨網站之間的分析與資訊共享。然而這不意味著 web analytics 與 usability tests 就沒有隱私侵害，並不是每個人都想被如此分析，使用者在網頁上的操作也可能洩漏一些預料之外的資訊，例如操作習慣。

### 監控
無論對於政府、司法機構或數據分析相關公司而言，使用者瀏覽什麼資料都是很珍貴的，可以被拿來分析的。

我們已經有[非常多政府與司法機構嘗試監控人民的案例](https://zh.wikipedia.org/zh-tw/%E6%94%BF%E5%BA%9C%E7%9B%A3%E8%81%BD%E9%A0%85%E7%9B%AE%E5%88%97%E8%A1%A8)了，其中以美國 NSA 的 PRISM 最為知名。這裡我無意討論監控的正當性或合法性，僅是指出政府與司法機構有動機與案例監控網路使用。

數據分析相關公司，尤其資料仲介（data broker），對使用者瀏覽網站以及操作方式等也同樣感興趣，因為這些資訊可以被賣給廣告商或需要這些資料的單位。例如保險公司如果知道這個使用者常常瀏覽競速相關資訊，可能就會想提升車禍險的保費；電商如果知道使用者常常瀏覽筆電開箱文，可能會把筆電價格拉高一點。其中最有名的案件莫過於[劍橋分析事件](https://en.wikipedia.org/wiki/Facebook%E2%80%93Cambridge_Analytica_data_scandal)。總之，數據就是資訊世界的貨幣。


## Web Tracker 的分類
在上一段，我們簡介了幾個 web tracking 的用途，所以就想稍微延伸一下，介紹一個我私心覺得蠻不錯的 web tracker 分類。DuckDuckGo 的 [Tracker Radar](https://github.com/duckduckgo/tracker-radar) 是一個分析 tracker 的專案，其中他們提出了一個對於 tracker 的分類。

原文[在此](https://github.com/duckduckgo/tracker-radar/blob/main/docs/CATEGORIES.md)，以下內容有些是翻譯自該文件並加上我自己的附註，不過只會介紹幾個我認為需要知道的。

- Action Pixels：類似於 [Web beacon](https://en.wikipedia.org/wiki/Web_beacon)，用於追蹤發生在 first party 或 third party 的特定事件。
- Ad Motivated Tracking 與 Advertising：為了蒐集資料來投放廣告而部屬的 tracker。
- Analytics：用於網站分析。
- Audience Measurement：更進一步的網站分析，通常會包含一些 demographic 的資料（性別、種族等）以及行為分析。
- Session Replay：這類型的 tracker 會追蹤使用者在網站上的所有行為，像是滑鼠移動、點擊、捲動或甚至網路流量，但它們本質上還是網站分析。
- Federated Login, Online Payment, Single Sign On, Social Network 等：顧名思義，就是與 Federated Login, Online Payment, Single Sign On 等有關的 tracker，這些服務為了做產品使用分析與防止詐欺，通常會部屬一些 tracker


至此我們已經完成對 web tracking 的 high-level overview。我知道這篇文章蠻無聊的，但有這些背景知識，接下來就可以走向較為技術細節的部份了！