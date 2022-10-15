---
weight: 2
title: "Fonts Fingerprinting"
---

# Fonts Fingerprinting
每個人的電腦都裝有不同的[字型](https://blog.justfont.com/2013/02/some_nouns/)，也可能有不同的預設字型，這所造成的影響，除了我的網站在我的 Mac 筆電上超讚但到了 Windows 上真的醜斃了以外（怨念深刻），這也成為 Browser Fingerprinting 的 entropy 來源：既然每個人都有不同的字型，那就用字型的搭配來 fingerprint 大家吧。此文我們將介紹幾個可以找到瀏覽器裝有哪些字型的方法，以及 Fonts Fingerprinting 的危害。

## 策略
在遠古時代，Flash 可以直接列舉出所有字型。但已經 2022 年了，我們還是把這東西塵封在（不怎麼好的）回憶中就好了。

就我所知，目前並沒有任何 API 可以直接枚舉出所有字型，也沒有 API 可以直接 query 一個字型是否存在，所以能做的只有，先列出所有我們想測試是否存在的字型，然後想辦法利用各種 side-channel 去依序測試哪些字型真的存在。

## 篩選待測字型
列出字型聽起來簡單，但如果要有效率，也是門學問，畢竟把成千上萬個字型丟下去測試，大概不是個好方法。核心要點是，所選的字型必須具有代表性。例如[「蘋方」](https://zh.wikipedia.org/zh-tw/%E8%98%8B%E6%96%B9)是 Apple 針對中文市場內建的字型，Roboto 與 Ubuntu 是 Android 與 Chrome OS 內建字型，「標楷體」之類的字型通常來自中文版 Windows 與 Office，這些字型都具有辨識度，可以提供足夠的 entropy。像是 Arial 或 Times New Roman 這種大家都有的字型，大概就沒什麼幫助。

## 檢查字型是否存在
下一步是檢查字型是否存在，這邊我會討論幾個不同方法。

### 檢查邊框變化
不同字型會有不同的字寬。如下圖所示，同樣的字串如果用了不同字型會有不一樣的寬度。換言之，偵測邊框（bounding box）改變是個測試字型是否存在的可行方法。

![](/images/font-width.png)

因此我們可以這樣測試一個字型是否存在：
1. 給定預設字串，先使用預設字型，並該字串的外框寬度（`offsetWidth`）
2. 嘗試改變該字串所使用的字型（使用 CSS 的 `font-family`）
3. 如果外框寬度改了，那就代表字型存在
4. 如果外框沒有變化，可能是字型不存在所以 fallback 回預設字型（也就是什麼有沒有變），也有可能真的就這麼剛好預設字型與測試字型的寬度一樣，造成誤判

藉由這個方式，便可以有效率地測出哪些字型存在。需要注意的是，雖然存在誤判可能性，但因為同個使用者每次都會在同個字型誤判，所以還是可以作為 fingerprint，畢竟 fingerprinting 只要求數據 unique 沒有要求數據合理（之後專文討論防禦與 privacy paradox 時會討論更多）。

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
[Canvas API](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API) 讓 Javascript 也有基本的繪圖功能，常用於動畫、遊戲、資料視覺化等。作為繪圖功能，自然會允許在畫布上放文字。第二種方法是直接把文字寫到 canvas 並指定字型，然後把 canvas 的圖片拿出來看跟預設字型的圖片是否一樣。通常此時會用的預設字串是 `Cwm fjordbank glyphs vext quiz`（英文的 pangram）。

有時我們會把這歸類到 canvas fingerprinting，也就是明天的主題，屆時會再做更深入的討論。因為跟 font 有關所以我還是想今天先提一下。

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

於是這就造成一個 side-channel：如果伺服器發現有個 client 來下載字型，便意味著這個 client 沒有安裝該字型。

所以可以這樣做：
首先，當 browser 請求網頁時，伺服器隨機生一個 token 並在回傳的網頁：
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


至此我們討論完如何偵測字型是否存在。

## 生成 Identifier
拿到一整串使用者有安裝的字型後，該如何算出一個 identifier 呢？

第一種策略是直接把整個名單 sort 起來算 hash，這是最簡單的方法，暴力而有效。其缺點是對於誤判太敏感了。第二種策略把每個字型存在與否都當成獨立的 feature，如此的好處是就算有個字型不見了或冒出來了，因為其他 feature 還是 consistent 的，所以對整個 fingerprint 影響不大，有機會把兩次 fingerprint 連結起來。不過仔細想想會發現其實兩種策略差不多就是了。

當然，fonts fingerprinting 不會是唯一的 feature，還會綜合許多其他 feature，才能算出有意義的 fingerprint。

## 防禦
目前有兩個作法，一個是把可以使用的字型限制在一個白名單內並要求所有瀏覽器與作業系統都要內建，另一個是要求所有字型都必須要用下載的，不能使用 local 的。兩者都不太方便呢。

偵測 font fingerprinting 則稍微簡單一些，畢竟大量存取各種字型怎麼看都很可疑，所以雖然無法完全避免，但至少還能偵測到這樣的攻擊存在，也許可以提前阻止之。

## 參考資料
D. Cassel, S.-C. Lin, A. Buraggina, W. Wang, A. Zhang, L. Bauer, H.-C. Hsiao, L. Jia, T. Libert. OmniCrawl: Comprehensive Measurement of Web Tracking With Real Desktop and Mobile Browsers. In Privacy Enhancing Technologies Symposium (PETS), July 2022.
Fifield, David, and Serge Egelman. "Fingerprinting web users through font metrics." _International Conference on Financial Cryptography and Data Security_. Springer, Berlin, Heidelberg, 2015.
[Font Enumeration Fingerprinting](https://www.darkwavetech.com/index.php/device-fingerprint-blog/fonts-fingerprinting)
[Demo: Disabling JavaScript Won’t Save You from Fingerprinting](https://fingerprint.com/blog/disabling-javascript-wont-stop-fingerprinting/)
[Font fingerprinting defenses roadmap](https://gitlab.torproject.org/tpo/applications/tor-browser/-/issues/18097)