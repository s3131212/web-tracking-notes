---
weight: 7
title: "Browser Extensions Fingerprinting"
---

# Browser Extensions Fingerprinting
Browser extension（有時譯為「擴充套件」）是指像是 ad blocker、MetaMask 等附加於 browser 之上，可以與之互動且可能改變網頁呈現與功能的程式。這些程式大大地擴充了 browser 的功能，藉由 browser extension，可以作到很多純網頁本身做不到，但 browser 卻又沒有內建的事情。然而，browser extension 既然可以改變網頁呈現與擴充功能，如果我們可以偵測到網頁的行為或呈現方式改變了，是不是就意味著可以偵測到一個 browser extension 存在？這就是 browser extensions fingerprinting 的精神：去偵測哪些 browser extensions 存在，然後將其製為 fingerprint。

## Browser Extension 簡介
在一頭栽進 fingerprinting 技術之前，還是得先知道 browser extension 到底是什麼。

先做個簡單的釐清，browser extension 不是 browser plugin，後者就讓它死在歷史洪流中吧。

Browser plugin 的主要功能是，它可以註冊一些它支援的 content type，當瀏覽器遇到這種 content type 時，就會召喚 browser plugin 出來 render 資料。Browser plugin 通常是直接跑在作業系統的 executable。早期通常使用 NPAPI（Netscape Plugin API），一個比 IE 還要古早的東西，基本上死得差不多了，取而代之的有 Chrome 的 PPAPI 和 IE 的 ActiveX，但兩者也都死了。Browser plugin 常見的應用有像是 Flash Player、WebATM、自然人憑證程式之類的。

Browser extension 則是可以擴充與修改 browser 行為的程式，通常用瀏覽器給定的 API 建立（Chrome 是 extension API，Firefox 是 WebExtension API，兩者兼容），且就跟一般網頁一樣是由 HTML, CSS 與 Javascript 撰寫成，執行於 browser 的環境中，頂多比一般網頁有多一點點的權限，但終究是在 sandbox 內。常見的像是 ad blocker、Privacy Badger 等，很多人可能會用來操作加密貨幣錢包的 MetaMask，讓 Youtube 好用一點的 Enhancer for YouTube，各種密碼管理器之類的。基本上現在大家使用的都是 browser extension，很少會遇到 browser plugin 了。

Browser extension 可以做到的事情大概有幾個[面向](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/What_are_WebExtensions)。第一，它可以改變網頁呈現，藉此提供額外功能，像是如果有裝 Grammarly 的外掛，就會在每個輸入框看到 Grammarly 的 icon 以及文法檢查來提醒我我英文有多爛。再來，延續上一點，它可以增加、移除或修改一些內容，像是 ad blocker 用於移除廣告與 tracker，Dark Reader 可以把任何網站變成黑色主題。也有一些是增加新功能，像是 MetaMask 使網頁可以操作電子錢包。

Browser extension 大概可以分成幾個部份：
- background scripts：只要 browser 存在時就會在背景運作的 script，它會在背景常駐（雖然可能會被丟進 idle state，但總之它會在），background scripts 可以用所有（它有權限的）WebExtension API，可以對它有權限操作的 host 做 cross-origin requests，但不能讀到網頁內容，這需要 content scripts 協助。
- content scripts：當使用者開啟特定網頁時，會在背景偷偷載入的 script，這個 script 可以讀到網頁內容，而且可以在該 context 下執行各種指令；content scripts 不同於一般用 `<script>` 引入的 script 在於，前者可以對它有權限操作的 host做 cross-orign requests，可以用一部分的（它有權限的） WebExtension API，而且可以跟 background script 互動，後者做不到這三點。
- Web-accessible resources：可以讓網頁與 content scripts 讀取的 resource，可以是任意檔案，包含 HTML、CSS、Javascript 等。以 Chrome 來說，是 `chrome-extension://<extension id>/<path>`
- Sidebars, popups, and options pages：就如其名，browser extension 可以建立三種不同樣態的 UI 供使用者操作。

例如，如果我想要在特定網頁的頂部加上一個倒數計時器，就會需要把倒數計時器的 icon，以及其 CSS 與 JS 放到 web-accessible resources，等使用者開啟特定網站時，content scripts 就會載入，它可以在網頁的 DOM tree 中插入倒數計時器的 code，網頁在執行時就會去存取 web-accessible resources 以拿到倒數計時器的 CSS、JS 與 icon，並在網頁上 render 出來。

## 如何偵測 Browser Extension 是否存在
在這個章節，我們將簡單討論幾個偵測 browser extension 是否存在的方法。

### Web-accessible resources
第一個方法是利用 web-accessible resources 是否存在來偵測。前面提過，browser extension 可以選擇 expose 一些檔案供網頁存取，那我只要嘗試存取看看這些檔案是否存在，不就解決了？

舉例來說，[Google Translate](https://chrome.google.com/webstore/detail/google-translate/aapbdbdomjkkjkaonfhkkikfgjllcleb) 的 web-accessible resources 包含一個檔案 `popup_css_compiled.css`，其路徑是 `chrome-extension://aapbdbdomjkkjkaonfhkkikfgjllcleb/popup_css_compiled.css`，於是追蹤者只需要嘗試存取看看這個路徑，檢視其是否存在，便可以知道使用者有沒有安裝 Google Translate。

```javascript
fetch('chrome-extension://aapbdbdomjkkjkaonfhkkikfgjllcleb/popup_css_compiled.css')
.then(() => {
  console.log('Google Translate detected');
})
.catch(() => {
  console.log('Google Translate not detected');
})
```

### Web-accessible resources 的 Timing Attack
可是，並不是所有 browser extension 都有 web-accessible resources，那其他 browser extension 怎麼偵測？

我們可以仔細想想，browser 會怎麼實作 web-accessible resources 的存取機制，其八九不離十是這樣實作：收到請求 -> 檢查 browser extension 是否存在，不存在就跳出 -> 檢查該資源是否存在且可存取，如果不能就跳出 -> 回傳該資源。換言之，browser extension 存在與不存在的情境中，執行路徑是不同的，存在時執行路徑比較長，不存在時執行路徑比較短，這不正是 Timing Attack 的要件嗎！？

於是，我們可以直接檢測存取一個 web-accessible resource 耗時多久，來判斷該 extension 是否存在。

雖然 [timing attack](https://hitcon.org/2022/agenda/df4755c9-65d3-40c2-b647-08b1ba05971b) 有許多統計檢定黑魔法，但因為在 browser extensions fingerprinting 的情境下，攻擊完全在 local 執行，所以執行時間其實蠻穩定的，直接拿出來比大小就好。我們的策略是：比較目標 extension 以及不存在的 extension 下同一個 path 的執行時間是否一樣，如果 extension 存在，它的執行時間應該會比較長，如果它不存在，那兩者執行時間應該差不多。

```javascript
const REPEAT = 200;
const THRESHOLD = 0.67;
const sum = (arr) => arr.reduce((ps, a) => ps + a, 0);

// 測試執行時間
const fetchTiming = async (url) => {
	const start = performance.now();
	try {
		await fetch(url);
	} catch (e) {
		// pass
	} finally {
		return performance.now() - start;
	}
};

// driver
const testExtension = async (extensionId, extesionFile) => {
	const treatmentUrl = `chrome-extension://${extensionId}/${extesionFile}`;
	const controlUrl = `chrome-extension://aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa/${extesionFile}`;

	// 測試兩個路徑的執行時間
	const fasterCnt = sum(await Promise.all(Array.from(Array(REPEAT)).map(async () => {
		const treatmentTime = await fetchTiming(treatmentUrl);
		const controlTime = await fetchTiming(controlUrl);
		return treatmentTime > controlTime ? 1 : 0;
	})));

	return fasterCnt / REPEAT > THRESHOLD;
};

// 測試 uBlock origin 是否存在
(async () => {
	console.log(await testExtension('cjpalhdlnbpafiamejdnhcphjbkeiagm', 'web_accessible_resources/noop.html'));
})();
```

### 檢測被修改的 DOM
許多 browser extension 會修改 DOM，像是 Grammarly 會在輸入框插入一個 icon，1Password 會在密碼輸入框下面加上自動帶入按鈕，此時這些改變對網頁而言是可見的，網頁只要有個 script 去偵測自己的 DOM 有沒有被以特定方式修改過，或是被插入特定的 element，就可以知道 browser extension 是否存在。

舉例來說，我在 Chrome 上安裝了 [Video Speed Controller](https://chrome.google.com/webstore/detail/video-speed-controller/nffaoalbilbmmfgbnbgppjihopabppdk)與 [Grammarly](https://chrome.google.com/webstore/detail/grammarly-grammar-checker/kbfnbcaeplbcioakkpcpgfkobkghlhen?hl=en)，便可以看到兩邊的 DOM tree 差了多少。仔細檢視，會發現 Video Speed Controller 插入了 `.vsc-controller`，Grammarly 則是在四處都插入了一堆東西，例如 `grammarly-desktop-integration`。

![](/images/browser-extension-dom.png)
（左邊有安裝 browser extension，右邊無安裝）

所以只需要很簡單地去看特定 element 是否存在即可。以 Grammarly 來說，可以看 `grammarly-desktop-integration`，因為這個 element 無論 Grammarly 的 icon 是否有出現，都會默默地出現在 DOM tree 中；Video Speed Controller 則可以看 `.vsc-controller`。

```javascript
setTimeout(() => {
	if (document.querySelector('.vsc-controller')) {
		console.log('Video Speed Controller detected');
	} else {
		console.log('Video Speed Controller not detected');
	}
	if (document.querySelector('grammarly-desktop-integration')) {
		console.log('Grammarly detected');
	} else {
		console.log('Grammarly not detected');
	}
}, 1000);
```

![](/images/browser-extension-dom-detection.png)
（左邊有安裝 browser extension，右邊無安裝）

可能會有讀者尖叫：等等，怎麼可以沒討論到 ad blocker！這可是最常見的 browser extension 欸。恩對，的確是該討論 ad blocker，但它能討論的東西比較多，所以我想拉成獨立一篇文章討論，應該幾天後就會發了。

或也許，這篇文章根本沒有讀者嗚嗚嗚嗚嗚嗚嗚嗚嗚嗚。

至於這方法的缺點，大概就是，許多 extension 只有在遇到特定網域時觸發，例如 Youtube 相關的 browser extension 可能只會在 `youtube.com` 上會動，不像 Grammarly 或 Video Speed Controller 在任何網域都可以動，對於這種 extension，就沒辦法用 DOM tree 做偵測了，畢竟我們沒辦法 hijack `youtube.com`。

### 利用被插入的 stylesheets
有些 browser extension 在插入 DOM element 的同時也會想加上 CSS rules，所以可能插入一些 stylesheets。於是，我們可以先檢查一個 browser extension 插入什麼樣的 stylesheet，找一個特定的 selector 出來實驗。如果在 DOM tree 新增一個符合剛剛找到的 selector 的 element，然後用 `getComputedStyle` 檢查其 CSS property 是否跟預設一樣，如果不一樣，就代表存在一個不屬於網頁的 stylesheet，也就是 browser extension 存在。

舉例來說，[Wikiwand](https://chrome.google.com/webstore/detail/wikiwand-wikipedia-modern/emffkefkbkpkgpdeeooapgaicgmcbolj/related) 插入的 stylesheet 中有一個 rule 會匹配到這個 element：
```html
<div id="ww_hovercard">
	<div class="ww_image">
		<img trigger="yes" />
	</div>
</div>
```

於是我們可以新增一個符合條件的 element，然後用 `getComputedStyle` 檢查它的 style 有沒有被改過。

![](/images/browser-extension-style.png)
（左邊有安裝 browser extension，右邊無安裝）

此方法的限制跟偵測 DOM 修改一樣，只能偵測到適用於所有 host 的 browser extension，只在特定 host 啟用的 browser extension 則無法偵測。

### PostMessage
在有些情境中，可能會出現網頁需要跟 browser extension 溝通的狀況，例如，當某個服務導引使用者安裝他們的 browser extension 之後，可能會讓 browser extension 跳出 popup 或是特定頁面繼續完成使用教學導引。通常這樣的功能是利用 [postMessage](https://developer.chrome.com/docs/extensions/mv3/content_scripts/#host-page-communication) 達成的。換言之，只要去監聽有沒有 extension 傳來的 postMessage，就可以知道 browser extension 是否存在。

### 偵測特殊的 HTTP Requests
有些 browser extension 在執行過程中會送 HTTP requests，其可能是藉由 background scripts 或 content scripts 去發送，也可能是直接插入在網頁的 DOM tree 中，在網頁 render 時自動送（例如插入 `<img>`），前兩者因為是在 browser extension 中執行的，所以網頁看不到請求，但最後一個情境，因為是假借網頁的名義送的請求，網頁本身看得到這個請求。所以，如果網頁去偵測自己發出的 HTTP requests 中有沒有不該發送的，便可以猜測這是 browser extension 發送的。

Performance Timeline API 是個用來分析 client-side latency 的工具，其中一個功能 `performance.getEntriesByType("resource")` 是列出所有 requests 以及其 latency，我們不在乎 latency，重點是這個功能可以列出所有網頁發出的 HTTP requests！於是我們可以先用 `performance.getEntriesByType("resource")` 去列出所有 HTTP request，然後尋找是否有出現某些 browser extension 可能會發出的 requests，藉此推論 browser extension 是否存在。

例如[趨勢科技防詐達人](https://chrome.google.com/webstore/detail/trend-micro-check-browser/imhhfjfjfhjjjgaedcanngoffjmcblgi/related?hl=zh-TW)觸發時會請求 `https://mm-static.mustcheck.com/tmc/image/extension/icon_card_safe@2x.png`，於是只要把 `performance.getEntriesByType("resource")` 列出來看有沒有這個 URL 就知道使用者有沒有安裝該 browser extension 了。

![](/images/browser-extension-http-requests.png)
（左邊有安裝 browser extension，右邊無安裝）

## 如何做 Fingerprinting
至此我們都只討論了怎麼偵測 browser extension 存在，那我們該如何做 fingerprinting 呢？其實也蠻簡單的，就把 browser extension 的列表這些當成一個 feature 塞進 fingerprint 即可。

此外，還可以根據使用者安裝了哪些 browser extension 來做更進一步的 profiling。例如，如果偵測到有安裝一些 dev tool，那使用者大概是工程師。或甚至，如果有偵測到一些防毒軟體用於提醒惡意網站的 extension，則可能透漏使用者啟用的防毒軟體，這對於做滲透時是很有用的資訊。

## 防禦
### Web-accessible resources
可以存取 web-accessible resources 的一個重要原因是，我們知道 browser extension 的 ID。如果不知道 ID，是不是就沒辦法存取了？事實上這正是[目前改進的方向](https://developer.chrome.com/docs/extensions/mv3/manifest/web_accessible_resources/)，讓 browser extension 的 ID 在每次 session 開始時都隨機地產生，稱之為 dynamic ID。Timing attack 也一樣可以藉由此方法改善，因為攻擊者不知道 dynamic ID，也就無法生成有效的 web-accessible resources 的 path，所以當兩個請求時間一樣時，他不知道是他猜錯 dynamic ID 還是該 browser extension 真的不存在。

然而這也不是萬無一失的，如果有 browser extension 有插入 DOM element，tracker 仍然可以從這些 DOM element 中引用 web-accessible resources 的 URL 還原出 dynamic ID，但如果都可以找到 DOM element 了，我想大概也不需要使用 web-accessible resources 來判別 extension 是否存在了。

### DOM 與 stylesheets 插入
對此的防禦大概有兩種途徑。

第一種途徑是，可以讓 DOM element 與 stylesheet 都隨機化，例如 `.vsc-controller` 變成 `.serkjvxtyu`，然後每次都重新 randomize，如此追蹤者便沒辦法從 selector 資訊看出什麼。然而，如果 browser extension 新增的 element 符合特定規律（例如 `div` 包兩個 `img` 與一個 `p`，其中 `p` 裡面有一個 `span`），還是可以用這種規律找出該 element，藉此推論 browser extension 存在。其中最早的嘗試是 [CloakX](https://github.com/sefcom/cloakx)。

第二種途徑是，改成維護兩份 DOM tree（都初始化成一開始從伺服器收到的樣子），一份是可以受到 extension 修改並顯示給使用者的，另一份是給網站 Javascript 看的，於是使用者看到的是 extension 修改過的 DOM tree 所 render 出來的畫面，而網頁上的 Javascript 只會看到最原本的樣子。


## 參考資料
[What are extensions? - MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/What_are_WebExtensions)
[Anatomy of an extension - MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Anatomy_of_a_WebExtension)
[Manifest - Web Accessible Resources](https://developer.chrome.com/docs/extensions/mv3/manifest/web_accessible_resources/)
[z0ccc / extension-fingerprints](https://github.com/z0ccc/extension-fingerprints)
Starov, Oleksii, and Nick Nikiforakis. "Xhound: Quantifying the fingerprintability of browser extensions." _2017 IEEE Symposium on Security and Privacy (SP)_. IEEE, 2017.
Sanchez-Rola, Iskander, Igor Santos, and Davide Balzarotti. "Extension breakdown: Security analysis of browsers extension resources control policies." _26th USENIX Security Symposium (USENIX Security 17)_. 2017.
Sjösten, Alexander, Steven Van Acker, and Andrei Sabelfeld. "Discovering browser extensions via web accessible resources." _Proceedings of the Seventh ACM on Conference on Data and Application Security and Privacy_. 2017.
Laperdrix, Pierre, et al. "Fingerprinting in style: Detecting browser extensions via injected style sheets." _30th USENIX Security Symposium (USENIX Security 21)_. 2021.
Trickel, Erik, et al. "Everyone is different: client-side diversification for defending against extension fingerprinting." _28th USENIX Security Symposium (USENIX Security 19)_. 2019.
Karami, Soroush, et al. "Carnus: Exploring the Privacy Threats of Browser Extension Fingerprinting." _In Proceedings of the 27th Network and Distributed System Security Symposium (NDSS)_. 2020.