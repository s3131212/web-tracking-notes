---
weight: 3
title: "Browser Fingerprinting 的防禦"
---

# Browser Fingerprinting 的防禦
在過去幾個章節中，每次介紹完 browser fingerprinting 的攻擊手段，都多少有提到一些防禦手段，所以這篇文不會著重在對於特定 fingerprinting 技術的防禦，而是對於防禦的整體策略分析，也就是防禦策略的大方向，並討論這些方向可能遭遇的問題，為何成功或失敗。如果讀者對特定 browser fingerprinting 技術的防禦有興趣，仍然應該回去檢視前面的章節。

## 為何 Anti-Fingerprinting 這麼難？
首先我們可能得先問，為什麼「有效的防禦」這麼難？或是說，anti-fingerprinting 可以如何搞砸？

試想，有個 tracker 用 `screen.width` 與 `screen.height` 做 fingerprinting，我的螢幕是 1920x1200，這個大小不算特別常見，為了避免被 browser fingerprinting，我決定改變 screen 的數值，於是我隨便選一個數值，像是 1234x5678。這下 tracker 拿不到我的真實螢幕大小了，它只會拿到我造假的資訊。這是個有效的防禦嗎？

過去我們討論過 surprisal 與 anonymity set 的概念，那時有提到一個很重要的觀念：一個 fingerprint 要有效，它必須足夠罕見，與足夠少人重複，也就是 surprisal 夠大，anonymity set 夠小。試問 1234x5678 比較獨特，還是 1920x1200 比較獨特？顯然是前者吧。Browser fingerprint 的重點不在於正確或是合理，而是足夠獨特。就算是錯的，只要錯得很獨特，一樣可以拿來做 fingerprinting。

既然不能隨便造假一個數值，那如果偽造成最多人使用的 1920x1080，會比較好嗎？的確我們躲在更大的 anonymity set 中，但很多正常功能也都會用 `screen`，像是用於網頁排版，於是這些網站也就跟著一起壞掉了。

既然固定一個數值會使一些功能無法正常運作，那如果每次都隨機生成一個假的數值，如此則每次 fingerprint 都不同，即便獨特也不持久，然後只要隨機的值不要離真的值太遠，就不會跑版了。乍看之下這個方法好像可行，但 tracker 只要重複取樣很多次並計算平均值，便可以還原出真正的數值。

假設我們找到一個方法可以讓改 screen 的同時不破壞既有功能，也不必然可以高枕無憂。例如，我宣稱自己是 1920x1080 結果 user-agent 說這是一台手機，手機好像通常不會有 1920x1080 這種大小；或我宣稱自己是 1920x1080 結果滑鼠座標可以移動到超過 1080 的地方。這些都透露了我正在騙人。更何況，幾年後最主流的螢幕大小很可能不是 1920x1080 了。

這不是一個完整的列表，但至少至此我們已經可以看到 anti-fingerprinting 並不是個容易的問題。

## Browser Fingerprinting 的防禦手段

接下來我們來討論幾個 anti-fingerprinting 的策略。值得注意的是，這些策略的分類並非涇渭分明，有些方法可能同時使用多個策略。

### 方向一：混淆 Browser Fingerprint 來源的值
第一種，也是最常見也最容易想到的方式是，直接去改 browser fingerprinting 來源的值，如此則 tracker 拿不到真正的 browser fingerprint，可能是大家的 fingerprint 都一樣，無法分辨，也可能是每次 fingerprint 結果都不一樣，無法做有效的長期追蹤。例如前面舉例的修改 `screen` 成 1920x1080，討論 canvas fingerprinting 時討論的增加 noise 等，都屬於這類。
可以進一步把這一類型的防禦再區分成兩種策略。

第一種是直接改數值，可能是改成預設的一個數值，例如大家的 `screen` 都改成 1920x1080，或是隨機產生一個值，例如隨機生成 `screen` 數值，藉此掩蓋真實的數值，讓使用者隱藏在一大群 anonymity set 裡面。前面提過的 `screen`、修改 user agent 等，都屬於這類。其中最早的嘗試之一是 PriVaricator。為了避免破壞使用者體驗，他們設計了一系列的 policy，只有在明顯是在做 fingerprinting 時才會啟動防禦。此舉雖然可能被發現，但就算被發現了，也難以使 fingerprint 持續存活。

這種策略會遇到幾個挑戰。首先，如果同一個資訊可以從多個來源取得，假設有其中一個來源沒有被正確地修改，便會被發現 fingerprinting 結果有誤，例如從 HTTP Request Header 中的 user agent 與 JavaScript 的 `navigator` 物件都可以取得作業系統的資訊，如果前者揭示的是 macOS，後者卻自稱是 Windows，便可發現其中有詐。此外，如果資訊變得太罕見而詭異，可能會顯得獨特，本身就是一個好的 fingerprint，也可能和其他現有資訊衝突（例如 macOS 上的 browser 宣稱自己的顯示卡品牌是高通），也可以藉此發現內容是偽造的。最後，直接改數值往往可能嚴重破壞使用者體驗，畢竟除了做 browser fingerprinting 以外，這些 API 往往有更常見、正常的用途。例如 Firefox 的 Fingerprint Protection，其在許多 API 都會給出假的結果，像是 timezone 全部回報 UTC，這雖然可以感受隱私，但將嚴重影響使用者體驗，使用者將不再能享受網頁能自動調整到正確時區的方便。

第二種策略是增加雜訊，其雜訊小到一般使用者不會注意到，但足以讓 tracker 拿到充滿雜訊的資料而無法有效利用。這雖然可能使 fingerprint 變得獨特，但卻會使 fingerprint 無法延續。這類型的方法主要用於對抗 canvas fingerprint（在 canvas 加上雜訊）與 audio fingerprinting（在 AudioContext 產出的波形加雜訊）。

這類型的方法可能會遇到數個困難。首先，如果每次雜訊都不一樣，多取幾次取個平均便可以還原出真實的數值了。因此，目前多數方法都會有固定的雜訊，可能在同個 eTLD+1 有同樣的雜訊，如此則 browser fingerprinting 只能用於 same-site tracking，也可能是每次開啟新的 session（例如重開 browser、清空 cookie 等）就重新生成 noise，如此則 browser fingerprint 最多只能存活到 session 結束，沒辦法活得比 cookie 還久。然而，除了雜訊本身要夠亂以外，還必須確保其不能被還原。以 canvas fingerprinting 來說，如果直接將雜訊以相加的方式干擾 canvas fingerprinting，只要畫出空白畫布，就可以還原出雜訊，把 canvas 繪製結果減去雜訊就還原出真正的 canvas 了，所以擾動方式肯定不會是直接把繪製結果與雜訊相加。

目前最有影響力的概念來自 FPRandom。在 canvas fingerprinting 防禦上，為了避免上述問題，FPRandom 選擇不在最終結果上做擾動，而是在繪圖時就直接做擾動，例如當指定要畫橘色時，直接把整個色塊做些微的調整，且這些擾動在同個 session 時都是固定的，如此則確保 tracker 沒有辦法做很多次 canvas 來取平均，且也不容易取出雜訊。Brave 的作法則是，加上雜訊時會每個 eTLD+1 有自己的雜訊且加入雜訊的方法會依賴於 canvas 繪製的原始內容，使得雜訊在同一張 canvas 繪製結果上是固定的，但在不同繪製結果上是不一樣的，讓提取雜訊變得困難或不可能。

另外，雖說增加的雜訊可以小到讓使用者無法注意到，但其實有另一個方向是：讓 JavaScript 讀到的與 browser 繪製出來的資料不一樣。例如一張 canvas，可能 browser render 出來的是原本的，JavaScript 藉由 toDataURL 拿到的是有擾動過的，如此可以進一步降低 anti-fingerprinting 對使用者的影響。

### 方向二：使大家的 Fingerprint 都一樣
另一個方向是，讓大家的 fingerprint 都一樣，如此則有超大的 anonymity set，大家都會隱藏在人海之中。

其中做得最好的可能是 Tor Browser。Tor Browser 開發團隊對 Firefox 做了很多改善，甚至讓視窗統一都是 800x600，藉此讓所有 Tor Browser 的 fingerprint 都是一樣的。另一個是 iOS 上的 Safari 有著一樣的 canvas fingerprint，這也是可以理解的，畢竟所有同世代的 iDevice 都有幾乎一樣的硬體，然後 OS 也都是 iOS。

然而這類型的方法有個缺點是，這種「大家都一樣」的 fingerprint 有時可以是有意義的，例如 Tor Browser 的特徵太過於明顯，和其他一般 browser 都不太一樣，但同時「使用 Tor Browser」這件事情有相當程度的意義（畢竟一般使用者不會用 Tor Browser），於是這種大家都一樣的 fingerprint 反而讓網站可以偵測出使用者正在使用 Tor Browser。

此外，如果 fingerprint 都一樣，一旦有一個環節出了錯，fingerprint 就會特別突出。例如 Tor Browser 大家都有一樣的 fingerprint，但如果有個使用者 resize 了他的視窗，從 800x600 變成 1920x1080，則這個使用者會突然變得格外獨特，竟然有某個 Tor Browser 使用者有著與眾不同的視窗大小。相較於 Chrome 或 Firefox 大家本來 fingerprint 都很不一樣，視窗大小也很多元，沒什麼好訝異的，在 Tor Browser 的 fingerprint 之中，視窗大小不一樣是非常重要的資訊。

另一個同樣屬於這個策略的是我們在 WebGL Canvas Fingerprinting 中提過，其差異來自於浮點數運算的誤差，只要讓大家的浮點數運算過程都一樣，就會產生一模一樣的 WebGL Canvas Fingerprint。

### 方向三：移除或限制用於 Fingerprinting 的 API
最終極的策略是，直接把用於 fingerprinting 的 API 拿掉或是限制其使用。

有些 API 本身可能很少用，卻可以用於 fingerprinting，既然大家都用不太到，不如就拿掉吧。例如 Battery Status API 真的很少被用到，很可能根本沒幾個網站有真的在有意義的使用它，所以 Firefox 就決定直接把 Battery Status API 拿掉了，Chrome 則是限制可以拿到的資料量多寡，電量只能拿到小數點後兩位，畢竟對於正常用途來說，知道小數點後兩位就綽綽有餘了。

Tor Browser 則是限制許多 API 的使用，需要取得使用者的同意才得以執行，例如是 canvas 與 WebAudio。畢竟對於多數網站來說，就算沒有 canvas 或 WebAudio也不至於不能用，但這些功能也可能有正當用途，沒必要也不適合直接拿掉，不如就預設封鎖，要求使用者積極地同意使用。

前面提過的 Firefox 的 Fingerprinting Protection 也有包含限制 API 的部份，像是 WebSpeech、Performance API 都會被停用，然後 canvas 在呼叫 toDataURL 時需要得到使用者同意。

但無論是直接移除，或是限制 API 使用，對使用者體驗來說都還是有害。其實我們沒必要要求一點資訊都不能洩漏，只需要確保洩漏的資訊量沒有大到可以做有效的 browser fingerprinting 即可，這正是 Google 的 Privacy Budget 的概念：tracker 只有多少「預算」可以存取危險的 API，一旦預算用完，就不能繼續存取這些 API 了，藉此限制 tracker 可以獲得的資訊量。我們後續會再做更多討論。

### 方向四：模擬不同環境
在方向一我們提到了 browser fingerprinting 的防禦可以藉由欺騙 tracker 來達成，然而這樣的防禦機制很容易被偵測出來。簡言之，欺騙 tracker 不難，但要騙到 tracker 不會發現自己被騙，則困難許多，太容易留下各種蛛絲馬跡使 tracker 發現瀏覽器正在提供錯誤資訊。

因此，另一個方向是，比起處心積慮騙過 tracker，不如直接讓 tracker 跑在假的環境，例如讓整個瀏覽器執行在 Virtual Machine 或是 Container 中，如此則 tracker 只能拿到虛擬環境中的參數，只要一直更換 VM 或 container，就可以一直產生出不同的 fingerprint，而且因為這些 fingerprint 在當下都是真的，所以 tracker 很難發現自己其實是跑在刻意創造出來的虛擬環境中。雖然基於顯而易見的效能考量，這種方法很難被大規模部屬。但是有許多市售宣稱可以偽造 browser fingerprinting 的工具，多半是使用這種方式建立的，畢竟在這些工具的使用場景中，效能考量與使用者體驗相對不那麼重要。

### 方向五：直接封鎖 Tracker（Content Blocking）
最後就是我們的老面孔：content blocking。不管有多少方式限制 tracker，都不如一開始不要載入 tracker。基本上就是在 [content blocking]({{< relref "/docs/stateful-tracking/content-blocking" >}}) 討論過的東西，也繼承了之前討論過的[限制]({{< relref "/docs/stateful-tracking/content-blocking-circumvention" >}})，所以就不多談了。



在這篇文章，除了數個防禦的策略以及他們可能的問題與挑戰，我們也討論了 anti-fingerprinting 可能會犯的錯誤，明天則會進一步討論這些錯誤為什麼會出現，以及其有些出乎意料的結果：失敗的 browser fingerprinting 導致使用者更容易被 fingerprint。

## 參考資料
Laperdrix, Pierre, et al. "Browser fingerprinting: A survey." ACM Transactions on the Web (TWEB) 14.2 (2020): 1-33.
Laperdrix, Pierre, Benoit Baudry, and Vikas Mishra. "FPRandom: Randomizing core browser objects to break advanced device fingerprinting techniques." International Symposium on Engineering Secure Software and Systems. Springer, Cham, 2017.
Nikiforakis, Nick, Wouter Joosen, and Benjamin Livshits. "Privaricator: Deceiving fingerprinters with little white lies." Proceedings of the 24th International Conference on World Wide Web. 2015.
https://mozilla.github.io/ppa-docs/privacy-budget.pdf