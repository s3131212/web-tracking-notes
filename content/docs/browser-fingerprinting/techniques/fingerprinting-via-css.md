---
weight: 12
title: "Fingerprinting via CSS"
---

# Fingerprinting via CSS (JS-less Fingerprinting)
過去我們討論 browser fingerprinting 時，總是離不開 Javascript，無論是蒐集 Javascript 本身的性質或是利用 Javascript 去蒐集 browser 其他的性質，都建立在我們可以使用 Javascript，事實上多數 browser fingerprinting 都需要 Javascript 才可以執行，於是許多人會說如果停用 Javascript 就可以避免被 browser fingerprinting 了。這篇文章將會討論，如果沒有 Javascript，如何單純使用 CSS 達到 browser fingerprinting。

不過可能要先澄清一下，「停用 Javascript 可以避免 browser fingerprinting」這樣的建議沒有錯（雖然挺不方便的），有鑑於幾乎所有 tracker 都還是仰賴 Javascript，這是個完全合理且有效的策略。我只是要展示如何作到沒有 Javascript 的 browser fingerprinting，沒有要反駁這個建議。

## 如何利用 CSS 蒐集資訊
Fingerprint 很重要的一個環節是，讀取各個 feature 的值，然後聚集起來當成 identifier，回傳給 server，但這要 Javascript 才能作到這樣的複雜運算，只有 CSS 大概是做不太到。（我很期待哪天看到什麼奇葩 hack 可以作到就是了）

不妨換個想法，我們把聚合成 identifier 的環節移動到 server-side，client 只負責蒐集好資訊想辦法傳給 server。在 CSS 上發送請求其實蠻簡單的，一個 `background-image` 就可以解決了。

```css
.probe {
	background: url('/<token>/some-feature)
}
```

其中 token 是伺服器在 serve 網頁時預先帶好的。伺服器只需要稍等幾秒，看他收到哪些請求，就知道哪些 CSS rules 被執行了。理論上所有 rules 都會被執行，所以下一個任務是：想辦法依照情況只執行我們期待的 rule。

## Media Query
如果有寫過網頁前端的，肯定對 CSS 的 `@media` 不陌生。`@media` 旨在使不同環境下可以套用不同的 CSS styles，例如螢幕很寬跟很窄時使用不同的 style。通常 media query 長這樣：

```css
@media (feature: value) {
  /* ... */
}
```

`feature` 的部份主要可以分成 media type 與 media features，我們主要在乎的是後者。

如果寫過 RWD，大概會知道 media feature 可以用於在有著不同特徵的顯示裝置（主要是不同螢幕大小）上套用不同的 rule，例如寬度在某個 range 中要套用怎樣的 rule。於是我們可以利用 media type 來取代 `screen.width`，探測使用者的螢幕大小！

```css
@media (max-width: 349.99px) {
	.probe {
		background: url('/<token>/screen-width/:350');
	}
}
@media (min-width: 350px) and (max-width: 767.99px) {
	.probe {
		background: url('/<token>/screen-width/350:768');
	}
}
@media (min-width: 768px) and (max-width: 1279.99px) {
	.probe {
		background: url('/<token>/screen-width/768:1280');
	}
}
@media (min-width: 1280px) and (max-width: 1439.99px) {
	.probe {
		background: url('/<token>/screen-width/1280:1439');
	}
}
@media (min-width: 1440px) and (max-width: 1919.99px) {
	.probe {
		background: url('/<token>/screen-width/1440:1920');
	}
}
/* ... */
```

於是只要看伺服器收到哪個請求，就知道使用者的螢幕大概在哪個 range。


Media feature 除了用於偵測螢幕寬度以外，還有很多其他功能。例如：
- `hover`、`any-hover` ：輸入設備是否支援 hover，分為 `hover` 與 `none`，例如 `@media (hover: hover)`
- `pointer`、`any-pointer`：輸入設備是否是個 pointing device（例如滑鼠），以及是否足夠精細（分為 `fine`、`coarse`），例如 `@media (pointer: fine)`
- `aspect-ratio`、`min-aspect-ratio`、`max-aspect-ratio`：螢幕的長寬比，例如 `@media (aspect-ratio: 1/1)`
- `color-gamut`、`color-index`：螢幕色域，例如 `@media (min-color-index: 15000)`、`@media (color-gamut: srgb)`
- `prefers-color-scheme`：偏好深色或淺色系，有 `light` 與 `dark`，例如 `@media (prefers-color-scheme: dark)`
- `prefers-contrast`：是否偏好增加對比度，分為 `no-preference`、`more`、`less`、`custom`
- `dynamic-range`：是否支援 HDR，分為 `standard`、`high`
- `resolution`：就... resolution，例如 `@media (resolution: 150dpi)`

這邊還有更多：[@media - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media#media_features)。

很難想像原來光是 CSS 就可以 leak 這麼多資訊出來齁。

## 字型
在討論 [fonts fingerprinting]({{< relref "/docs/browser-fingerprinting/techniques/fonts-enumeration" >}}) 時其實已經聊過如何只用 CSS 做字型是否存在的偵測，詳細討論可以回去看當時的文章。反正主要概念是：如果字型存在，CSS 會直接讀系統內建字型，如果不存在，可以指定一個 fallback 的字型，而這個 fallback 字型可以是從遠端下載的。於是，只要看到 client 來請求某個特定字型，就知道該字型並未內建於 client 的裝置中。

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

## Feature Query
另一個相對罕見的 CSS 功能是 feature query （`@supports`），它用於偵測瀏覽器是否支援各種功能。例如 `@supports (transform-origin: 5% 5%)` 可以用於偵測 browser 是否支援 `transform-origin: 5% 5%`（必須支援 `transform-origin` 而且其值可以接受百分比）。

因為同個 browser 支援的東西都大同小異，所以 feature query 比較適合用於檢測 browser 類型。
- `-moz-perspective: 10px` 只對 Firefox 與其他 Gecko-based browser 有效
- `-apple-pay-button-type: plain` 只對 Safari 有效
- `-webkit-touch-callout: default` 只對 iOS 上的 Safari 有效（iOS 上的其他 browser 都只是 Safari 換皮）
- `-moz-osx-font-smoothing: grayscale` 只對 macOS 上的 Firefox 有效

```css
@supports (-moz-perspective: 10px) {
	.probe {
		background: url('http://example.com/123/is-firefox/true');
	}
}

/* 也可以用 not 反轉結果 */
@supports not (-moz-perspective: 10px) {
	.probe {
		background: url('http://example.com/123/is-firefox/false');
	}
}
```

甚至，在 [appearance](https://developer.mozilla.org/en-US/docs/Web/CSS/appearance) 有許多只有特定 browser 支援的屬性，這些也全都可以拿來做 fingerprinting。

## 同場加映：HSTS SuperCookie with CSS
之前我們介紹過 [HSTS supercookie]({{< relref "/docs/stateful-tracking/storing-identifier#hsts-supercookie" >}})，當時介紹的方法需要 Javascript 協助。nevkontakte 提出了一個很有趣的方法，讓整個過程可以只用 CSS 做到。雖然這其實不太算是 browser fingerprinting，但我覺得蠻有趣的，就拉進來一起討論了。如果還不知道 HSTS supercookie 是什麼，建議先回去看之前介紹 [HSTS supercookie]({{< relref "/docs/stateful-tracking/storing-identifier#hsts-supercookie" >}}) 的文章。

當使用者第一次造訪網頁時：
1. 請求 `http://tracker.example/fingerprinting.css`
2. 伺服器發現是 `http`，重新導向到 `https://tracker.example/setup.css`
3. 瀏覽器請求 `https://tracker.example/setup.css`
4. 伺服器生成 identifier，並根據 identifier 建構 CSS：若 kth bit 為 1 則 `@import url("https://[k].tracker.example/1.css`，反之則引入 `@import url("https://[k].tracker.example/0.css`，回傳 CSS 時加上 HSTS header
5. Browser 會根據 `setup.css` 去載入所有被 import 的檔案
6. 如果檔案是 `1.css` 則伺服器加上 HSTS header，`0.css` 則不加上
7. Browser 寫入 HSTS 的要求

當使用者再次造訪時：
1. 請求 `https://tracker.example/fingerprinting.css`（注意因為有 HSTS，所以會直接用 `https`）
2. 伺服器發現是 `https`，重新導向到 `https://tracker.example/read.css`
3. 瀏覽器請求 `https://tracker.example/read.css`
4. 伺服器生成 token，並以此建構 CSS：`@import url("http://[k].tracker.example/[token].css`
5. Browser 依序請求 `http://[k].tracker.example/[token].css`，其中如果有設定過 HSTS，會直接 rewrite 成 `https`
6. 伺服器依照哪些請求使用 `http` 哪些使用 `https` 來判斷 identifier 的哪些 bit 為 0（HTTP）或 1（HTTPS）

其缺點基本上繼承了所有（有用到 JS 的）HSTS supercookie 的缺點，可以直接回之前介紹 [HSTS supercookie]({{< relref "/docs/stateful-tracking/storing-identifier#hsts-supercookie" >}}) 的文章看看。

## 結語
一開始我們展示了數個從 CSS 取得的 feature，這些 feature 拿來做 fingerprinting 可能不太夠用，畢竟 OS 與 browser 的組合也沒有很多。如果把在討論 [attributes]({{< relref "/docs/browser-fingerprinting/techniques/attributes" >}}) 所提過的各種 HTTP Request Header 也放進來，就能大大增加 entropy。最後則是討論了只用 CSS 做 HSTS fingerprinting，雖然直接 assign identifier 可以做到更針對性的追蹤，但這也繼承了之前提過的 HSTS fingerprinting 的各種限制。

這邊有個完全不使用 JS 的 fingerprinting 實作，如果有興趣可以參考看看：[No-JS fingerprinting](https://noscriptfingerprint.com/)

## 參考資料
[Demo: Disabling JavaScript Won’t Save You from Fingerprinting](https://fingerprint.com/blog/disabling-javascript-wont-stop-fingerprinting/)
[@supports - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@supports)
[@media - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media#media_features)
[JavaScript-less HSTS super-cookie PoC](http://hsts.nevkontakte.com)