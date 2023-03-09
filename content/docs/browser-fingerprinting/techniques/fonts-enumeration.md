---
weight: 2
title: "Fonts Fingerprinting"
---

# Fonts Fingerprinting
每個人的電腦都裝有不同的字型，也可能有不同的預設字型，這所造成的影響，除了成為所有前端工程師的惡夢以外，也成為 browser fingerprinting 的 entropy 來源：既然每個人都有不同的字型，而且大家不會頻繁地安裝新的字型，那麼使用者設備所配備的字型便成為可用的特徵。我們將於此章介紹幾個可以找到瀏覽器裝有哪些字型的方法，以及 Fonts Fingerprinting 的危害。

在遠古時代，Flash 可以直接列舉出所有字型，但顯然現在沒有人在用 Flash 了。目前並沒有任何 API 可以直接枚舉出所有字型，也沒有 API 可以直接查詢一個字型是否存在，所以能做的只有，先列出所有我們想測試是否存在的字型，然後利用各種 side-channel 去依序測試哪些字型真的存在。

## 篩選待測字型
列出字型乍看之下並非難事，然而如何最有效率地找到足夠獨特的特徵，則不是個簡單的任務。核心要點是，所選的字型必須具有代表性。

一般而言，可以將網頁上的字型分為三類：
- 網頁字型：網頁自帶的字型。因為其不來自使用者的設備，這沒辦法用於 fonts fingerprinting。然而網頁字型的功能本身可以協助 fingerprint 其他字型。
- 系統字型：作業系統自帶的字型，通常使用該作業系統的裝置都會搭載該字型。雖然這可能有些用，但作業系統往往可以從其他方式做推論（例如 user agent），所以這個幫助也很有限。然而，在未來討論到一些有問題的防禦手段時，針對系統字型的 fingerprinting 可以派上用場。此外，部分作業系統會根據不同語言自動安裝不同字型，這使得特殊語言的字型可以用於 fingerprinting。
- 使用者自己安裝的字型：有些字型是使用者自己安裝的，無論是主動安裝，或是因為安裝一些軟體而被安裝（例如有些辦公軟體會自帶一些字型）。這些往往是最有 fingerprinting 價值，因為其通常足夠獨特。

因此，系統字型如「蘋方」（Apple 針對中文市場內建的字型）、Roboto（Android 與 Chrome OS 內建字型）等可以提供一些關於作業系統的資訊。一些使用者必須自行安裝的字型，或是隨著如 Microsoft Office、Adobe CC 等套裝軟體而一起安裝的字型，則因足夠獨特而成為 fingerprinting 的首選。像是 Arial 或 Times New Roman 等大家都有的字型，則不會有什麼幫助而可以直接忽略。

此外，除了字型本身是否存在以外，另一個特徵是，是否有多個通常不會一起出現的字型同時出現了。例如當 Windows 的語言設定為日文或阿拉伯文時，會下載該語言的字型，但很少有裝置同時有日文的字型與阿拉伯文的字型，於是雖然兩者都不獨特，世界上有非常多日文與阿拉伯文的使用者，但兩者搭配在一起便十分獨特。


## 檢查字型是否存在
下一步是檢查字型是否存在，這邊我會討論幾個不同方法。

### 檢查邊框變化
不同字型會有不同的字寬。如下圖所示，同樣的字串如果用了不同字型會有不一樣的寬度。換言之，偵測邊界框（bounding box）改變是個測試字型是否存在的可行方法。

![](/images/font-width.png)

因此我們可以這樣測試一個字型是否存在：
1. 給定預設字串，先使用預設字型，並該字串的外框寬度（`offsetWidth`）
2. 嘗試改變該字串所使用的字型（使用 CSS 的 `font-family`）
3. 如果外框寬度改了，那就代表字型存在
4. 如果外框沒有變化，可能是字型不存在所以 fallback 回預設字型（也就是什麼有沒有變），也有可能真的就這麼剛好預設字型與測試字型的寬度一樣，造成誤判

藉由這個方式，便可以有效率地測出哪些字型存在。需要注意的是，雖然存在誤判可能性，但因為同個使用者每次都會在同個字型誤判，所以還是可以作為 fingerprint，畢竟 fingerprinting 只要求數據獨特，沒有要求數據合理。

那，預設字串應該是什麼？最常見的是 `mmmmmmmmmmlli`，因為在沒有固定寬度的英文字型中，通常 `m` 是最寬的、`l` 是最高的，所以對 bounding box 影響最大（`i` 是來增加 entropy 的）。

```javascript
function font_enumeration() {
	const fonts = ["Microsoft YaHei", "Microsoft Yi Baiti", "MingLiU", "MingLiU-ExtB", "Times New Roman Baltic", "Times New Roman CYR", "Times New Roman", "Ubuntu", "Noto Sans CJK TC", "TW-MOE-Std-Kai", "Droid Sans", "fake-font-this-should-not-exist"];
	const containerDiv = document.createElement('div');
	containerDiv.style.cssText = 'position: absolute;  display: block !important';
	document.body.appendChild(containerDiv);

	const pDefault = document.createElement('p');
	pDefault.innerText = 'mmmmmmmmmmlli';
	pDefault.style.cssText = `display:inline !important; width:auto !important; font-size: 10px !important;`;
	containerDiv.appendChild(pDefault);

	const ps = fonts.map((font, i) => {
		const p = document.createElement('p');
		p.innerText = 'mmmmmmmmmmlli';
		p.style.cssText = `display:inline !important; width:auto !important; font: 10px "${font}" !important`;
		containerDiv.appendChild(p);
		return p;
	});

	const results = fonts.filter((font, i) => ps[i].offsetWidth != pDefault.offsetWidth);

	return results;
}
console.log(font_enumeration())
```

### 嘗試寫到 Canvas
Canvas API 讓 Javascript 也有基本的繪圖功能，常用於動畫、遊戲、資料視覺化等。作為繪圖功能，當然會允許在畫布上放置文字。第二種方法是直接把文字寫到 canvas 並指定字型，然後比較該 canvas 是否與使用預設字型的 canvas 一樣。通常此時會用的預設字串是 `Cwm fjordbank glyphs vext quiz`，也就是英文的 pangram，以確保每個字母的不同都會被偵測到。

有時我們會把這歸類到 canvas fingerprinting，也就是後續的主題，屆時會再做更深入的討論。

### 使用 `@font-face` 的 Fallback 機制
如果前面都要用 Javascript 覺得太無聊了，在此提供一個純 CSS 的解法。CSS 的 `@font-face` 用於自訂字型，其特色是字型可以從 local 拿也可以從 remote 下載，且可以提供 fallback 字型，也就是如果位置 A 拿不到就去位置 B 拿。舉例來說：
```css
@font-face {
	font-family: 'Helvetica';
	src: local('Helvetica'),
		 url('https://fonts.example/Helvetica.ttf')
		 format('truetype');
}
```

如此則如果 local 找不到 `Helvetica`，就會去 remote 拿。

於是這就造成一個 side-channel：如果伺服器發現有個客戶端來下載字型，便意味著這個客戶端並未安裝該字型。

所以追蹤者可以這樣做。首先，當瀏覽器請求網頁時，伺服器隨機生一個 token 並在回傳的網頁：
```html
<div style="font-face: 'Helvetica';">a</div>
<style>
  @font-face {
	font-family: 'Helvetica';
	src: local('Helvetica'),
		 url('https://fonts.example/<token>/Helvetica.ttf')
		 format('truetype');
  }
</style>
```
接著，如果 `fonts.example` 收到對 `/<token>/Helvetica.ttf` 的請求，就知道被分配到該 token 的 browser 並沒有內建 Helvetica。


## 防禦
目前有兩個作法。第一種是，可以把可以使用的字型限制在一個白名單內，例如只能使用系統字型，拒絕任何使用者自行安裝的字型。由於是作業系統內建的字型，基本上使用這個作業系統的人看起來會一模一樣，或至少會和其他使用相同語言的人一樣。另一個是要求所有字型都必須要用下載的，不能使用內建的，但這個方法的可行度較低，因為可能使用更多網路流量。

偵測 fonts fingerprinting 則稍微簡單一些。大量存取各種字型的行為非常明顯，除了 fingerprinting 外也沒什麼理由需要這麼做，所以雖然無法完全避免，但至少還能偵測到追蹤者正在嘗試攻擊。

## 參考資料
D. Cassel, S.-C. Lin, A. Buraggina, W. Wang, A. Zhang, L. Bauer, H.-C. Hsiao, L. Jia, T. Libert. OmniCrawl: Comprehensive Measurement of Web Tracking With Real Desktop and Mobile Browsers. In Privacy Enhancing Technologies Symposium (PETS), July 2022.
Fifield, David, and Serge Egelman. "Fingerprinting web users through font metrics." _International Conference on Financial Cryptography and Data Security_. Springer, Berlin, Heidelberg, 2015.
[Font Enumeration Fingerprinting](https://www.darkwavetech.com/index.php/device-fingerprint-blog/fonts-fingerprinting)
[Demo: Disabling JavaScript Won’t Save You from Fingerprinting](https://fingerprint.com/blog/disabling-javascript-wont-stop-fingerprinting/)
[Font fingerprinting defenses roadmap](https://gitlab.torproject.org/tpo/applications/tor-browser/-/issues/18097)