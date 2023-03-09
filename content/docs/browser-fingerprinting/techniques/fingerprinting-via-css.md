---
weight: 12
title: "Fingerprinting via CSS"
---

# Fingerprinting via CSS (JS-less Fingerprinting)
過去我們討論 browser fingerprinting 時，總是離不開 JavaScript，無論是蒐集 JavaScript 本身的性質或是利用 JavaScript 去蒐集瀏覽器其他的性質，都建立在我們可以使用 JavaScript，事實上多數 browser fingerprinting 都需要 JavaScript 才可以執行。但如果沒有 JavaScript，還有可能做 browser fingerprinting 嗎？這篇文章將會討論，如果沒有 JavaScript，如何單純使用 CSS 達到 browser fingerprinting。

不過可能要先澄清一下，「停用 JavaScript 可以避免 browser fingerprinting」這樣的建議沒有錯，有鑑於幾乎所有 tracker 都還是仰賴 JavaScript，這是個完全合理且有效的策略。我只是要展示如何作到沒有 JavaScript 的 browser fingerprinting，沒有要反駁這個建議。甚至我認為本章節所介紹的各種 Javascript-less Fingerprinting 技術主要還是有趣而已，實用價值遠不如前面提過的其他 browser fingerprinting 方法。

Browser fingerprint 的過程可以分為三大步驟：1) 讀取各個 feature 的值；2)聚集起來製成 fingerprint；3) 回傳給 server。第一步是否需要 JavaScript 取決於是什麼特徵。至於第二步，乍看之下，似乎需要 JavaScript 才能作到這樣的複雜運算，只有 CSS 大概是做不太到，但是如果把聚合成 fingerprint 的環節移動到 server-side，客戶端只負責蒐集好資訊並回傳給伺服器，則似乎就可以不用到 JavaScript 了。至於第三步，雖然除了 JavaScript 以外還有許多方式發請求，但如果沒有 JavaScript 在中間幫忙傳遞數值，可以使用的方法也會相對受限。在以下討論中，我們並不會特別區分三個步驟，但讀者可以尤其注意每個部分分別對應到上面三個步驟的哪一步。

## 選擇性執行 CSS Rules
### 使用 background 回傳資訊

在 CSS 的世界，欲觸發請求，最簡單的方式是使用背景圖片。

```css
.probe {
	background: url('/<token>/some-feature)
}
```

其中 token 是伺服器在產生網頁時預先帶入的。伺服器只需要稍等幾秒，檢查它收到哪些請求，便知道哪些 CSS rules 被執行了。於是下一個任務是，令 CSS rule 根據想要蒐集的 feature 選擇性執行。

### Media Query
如果有寫過網頁前端的，肯定對 CSS 的 `@media` 不陌生。`@media` 旨在使不同環境下可以套用不同的 CSS rules，例如螢幕很寬跟很窄時使用不同的規則。通常 media query 的結構大致如下：

```css
@media (feature: value) {
  /* ... */
}
```

其中 `feature` 的部份主要可以分成 media type 與 media features，我們主要在乎的是後者。

如果寫過 RWD（Responsive Web Design），大概會知道 media feature 通常用於根據顯示裝置的性質（例如螢幕大小）套用不同的 CSS rule，例如寬度在某個範圍中要套用特定的 CSS rule。於是我們可以利用 `@media` 中偵測寬度的功能來取代 `screen.width`，探測使用者的螢幕大小。

```css
 @media (min-width: 600 px) {
    #probe {
        background: url (/<token>/width-600);
    }
}
@media (min-width: 601 px) {
    #probe {
        background: url (/<token>/width-601);
    }
}
@media (min-width: 602 px) {
    #probe {
        background: url (/<token>/width-602);
    }
}
/* ... */
```

藉由選擇性套用 CSS rules，只有特定的 background 會被執行，於是只要檢視伺服器收到哪個請求，便可以知道使用者的螢幕寬度在哪個範圍中。


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

由於不同裝置的 media query 結果不一樣，有些裝置支援 HDR 有些裝置不支援，因此可以用於製成 fingerprint。除了 media query 結果直接不一樣以外，對 media query 的支援狀況也一樣可以製成 fingerprint。例如 Firefox 100 支援 `@media (video-dynamic-range: standard)`，但 Firefox 99 並不支援，這也會造成可用於 fingerprinting 的差別。

## 字型
過去我們討論過何謂 [fonts fingerprinting]({{< relref "/docs/browser-fingerprinting/techniques/fonts-enumeration" >}})，也提過如何使用 `@font-face` 的 fallback 機制達成 fonts fingerprinting，詳細討論可以回去看先前章節。簡言之，如果字型存在，CSS 會直接讀取系統內建字型，如果不存在，可以指定一個 fallback 的字型，又這個 fallback 字型可以是從遠端下載的。於是，只要看到客戶端來請求某個特定字型，便可以知道該字型並未內建於客戶端的裝置中。

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
另一個相對罕見的 CSS 功能是 feature query （`@supports`），它用於偵測瀏覽器是否支援各種功能。例如 `@supports (transform-origin: 5% 5%)` 可以用於偵測瀏覽器是否支援 `transform-origin: 5% 5%`，也就是支援 `transform-origin` 而且其值可以接受百分比。

因為同個瀏覽器支援的東西都大同小異，所以 feature query 比較適合用於檢測瀏覽器類型與版本。舉例來說：
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
一開始我們展示了數個從 CSS 取得的 feature，這些 feature 拿來做 fingerprinting 可能並不足以建構出足夠獨特的 fingerprint，畢竟 OS 與 browser 的組合也沒有很多。雖然可以把過去介紹過的 passive fingerprinting 也納入，但因為 passive fingerprinting 的許多特徵（尤其 HTTP request headers）也依賴於瀏覽器的類型與版本，所以和 CSS feature 並不完全獨立。因此這些技術比較屬於有趣以及火力展示，至少現階段來看，實際用途並不明顯。

這邊有個完全不使用 JS 的 fingerprinting 實作，如果有興趣可以參考看看：[No-JS fingerprinting](https://noscriptfingerprint.com/)

## 參考資料
[Demo: Disabling JavaScript Won’t Save You from Fingerprinting](https://fingerprint.com/blog/disabling-javascript-wont-stop-fingerprinting/)
[@supports - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@supports)
[@media - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media#media_features)
[JavaScript-less HSTS super-cookie PoC](http://hsts.nevkontakte.com)