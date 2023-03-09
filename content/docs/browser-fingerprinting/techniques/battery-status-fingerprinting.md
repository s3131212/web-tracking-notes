---
weight: 10
title: "Battery Status Fingerprinting"
---

# Battery Status Fingerprinting
HTML5 提供了一些使網頁開發者得以取得硬體資訊的 API，其中一個是 Battery Status API，其原本是用來讓網站監控裝置的電池狀態，以提供更好的用戶體驗。電池狀態有許多優勢，例如，開發者可以根據使用者的電量來決定要提供高效能版本或省電版本，或當然也有可能用於一些不那麼正當的用途，例如 Uber 發現使用者在手機快沒電時傾向於接受比較高的價格1。這個功能看起來非常無害，甚至標準制訂者一開始也不認為這可能有什麼安全性威脅。然而，2015 年時，Olejnik 等人發現，該 API 也可能被用來進行 browser fingerprinting。更甚至，2016 年時 Englehardt 等人發現真的有 tracker 正在積極地使用 Battery Status API 做 fingerprinting。我們將這種技術稱為 Battery Status Fingerprinting。

## Battery Status API
Battery Status API 可以提供一些關於電池的資訊，包含電量（`level`，0.0 ~ 1.0）、是否在充電（`charging`，布林值）、預期還有多久可以充滿電（`chargingTime`，整數秒）、預期還有多久會消耗完電池（`dischargingTime`，整數秒）。

![](/images/battery-api.png)

```javascript
navigator.getBattery().then(console.log);
```

## Battery Status Fingerprinting
電池資訊看似無害，但我們可以這樣想想：一個裝置的電池狀態，好像應該會是連續的，而且在各個網站上都是一樣的？如果我在充電，我的電池剩餘量一定是上升，反之則一定是下降，而且不管我在看哪個網站，都是用同一個電池，所以狀態也會一樣。換言之，藉由電池狀態好像可以作為某種短期的 identifier。

Olejnik 等人於是就去研究是否可能將電池狀態用於 browser fingerprinting。我們可以觀察到，在一個非常短的時間內（大概 30 秒左右），電池狀態不會有改變，所以 `(chargingTime, dischargingTime, level)` 可以作為 identifier。有鑑於 `chargingTime` 與 `dischargingTime` 只會擇一出現，其實我們只有兩個連續的資料點和一個離散的資料點（是否正在充電），但這樣也足以做 identifier 了。

讀者可能會問：欸不是，一個只能活 30 秒的 identifier 是能幹麻？欸... 其本身不能幹麻。不過不妨這樣想，會需要 browser fingerprinting 的原因之一是 cookie 會被刪除，而追蹤者希望 cookie 被刪除後，可以藉由 browser fingerprint 把原本的 identifier 塞回去。換言之，在一些情境中，browser fingerprint 其實本身不需要是長期穩定的，只需要能從「cookie 中的 identifier 被刪掉」存活到「重新把 identifier 寫進 cookie」即可，而 30 秒去偵測到 cookie 被刪除並重新建立其實綽綽有餘。

## Battery Capacity
雖然剩餘電量沒辦法存活很久，不過電池本身的參數（充飽的滿電量、電壓等）卻是相對穩定的，其中滿電量會依照電池本身規格與消耗狀態受到影響，即便是一模一樣的電池，在經過一段時間的使用，也會有不同的滿電量。換言之，如果能回推出滿電量，就能有比較長效的 identifier 了。

舉例來說，UPower 的電量計算方法是 `100 * (Energy / EnergyFull)`，其中 `EnergyFull` 精準到小數點後四位。

我們會發現 browser 只拿得到 Energy 與 EnergyFull 的比值，還不足以知道 EnergyFull 的數字。但是，因為 UPower 會給出 64-bit double precision 的浮點數，能夠讓相除結果剛好完美地符合這個浮點數的數字組合其實不多，於是我們只要先窮舉許多可能的數字，找到所有有辦法讓 `Energy / EnergyFull` 等於該浮點數的 `EnergyFull`，隔一段時間之後再重複做一次，會找到另外一堆符合條件的潛在 `EnergyFull`，多做幾次然後 intersect 下來，就可以還原出 `EnergyFull`。不過這邊有個很重要的前提是能夠拿到原始的 64-bit double precision 浮點數，如果被處理過（例如四捨五入），數字就會不精準到難以回推出有用的資訊。

## 防禦
防禦大概可以分成兩種：降低精準度、拿掉該功能。最早的 Battery Status API 在有些平台上可以拿到 64-bit double precision 的浮點數，但現在多半只能拿到小數點後兩位（也就是百分比不會有小數），如文章一開始的圖中所示。

另外一些瀏覽器，例如 Firefox，則是直接拿掉這個功能的公開介面，只能使用 browser extension 存取；Safari 在 Olejnik 等人的研究出現時尚未實作 Battery Status API 的公開介面，所以也決定直接不實作該功能。

## 參考資料
Olejnik, Łukasz, et al. "The leaking battery." _Data Privacy Management, and Security Assurance_. Springer, Cham, 2015. 254-263.
Laperdrix, Pierre, et al. "Browser fingerprinting: A survey." _ACM Transactions on the Web (TWEB)_ 14.2 (2020): 1-33.