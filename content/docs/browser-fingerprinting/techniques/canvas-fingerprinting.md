---
weight: 3
title: "Canvas Fingerprinting"
---

# Canvas Fingerprinting
在 2012 年時，Mowery 與 Shacham 發現了一個 browser fingerprinting 方法： HTML5 的 Canvas API 在不同平台上 render 的結果的細微差異，利用這些細微差異，便可以提供足夠的資訊做 fingerprinting。因此，我只要讓 Canvas API 畫出一張圖，然後把圖片拿出來算 hash，就做完 fingerprinting 了。

這個發現徹底改變了 web tracking 的生態，也改變了大家對 browser fingerprinting 的看法。
這可能是第一次有這麼高準確度又 practical 的 browser fingerprinting 方法。

## 前言
首先，什麼是 canvas？Canvas 直譯是「畫布」，顧名思義，這東西大概跟畫圖脫不了關係。我們需要先區分 Canvas API 與 `<canvas>` 兩個看起來很像但其實不同的東西。Canvas API是一個可以讓 Javascript 繪製 2D 圖形的工具（雖然其實沒人會阻止你在 2D 上畫 3D 圖形啦），它提供了許多畫圖的 method，像是畫直線、畫矩形、填充顏色、插入文字等。`<canvas>` 是 HTML 的一個 tag，它就真的是個畫布，讓 Canvas API 在 `<canvas>` 上畫畫。不過不只有 Canvas API 可以在 `<canvas>` 上畫畫，另一個同樣也以 `<canvas>` 作為容器的是 WebGL API，明天我們就會碰到了。

Canvas fingerprinting 字面意義上就是用 canvas 來做 fingerprinting。通常此處的 canvas 是 Canvas API 而不是 HTML 的 `<canvas>`。明天也會介紹 WebGL API 用於 fingerprinting。讀者可能會疑惑，為何需要去區分兩者。原因是，即便同樣是繪圖在 `<canvas>` 上，同樣是利用渲染在 `<canvas>` 上的內容做 fingerprinting，Canvas API 可以被 fingerprint 的成因與 WebGL API 可以被 fingerprint 的成因不完全相同，雖然大同小異就是了。

Canvas fingerprinting 之所以可能，在於 Canvas API 只是給出一堆指示，這些指示需要從瀏覽器送到作業系統再送到顯卡，才能將其繪製出來，而不同的瀏覽器、作業系統、顯卡等，在遇到同樣的指示時，可能會有不太一樣的行為，例如不同作業系統可能有不同的預設字型，顯卡可能使用了不同的演算法實作混色與 anti-aliasing 等，將這些細微實作差異累積起來，最後就會生出略有不同的圖片了，於是便可利用 Canvas API 的輸出做 fingerprinting。

## 為何 Canvas Fingerprinting 存在
剛剛我們提到了，canvas fingerprinting 利用了不同軟硬體設備會讓 canvas 長得不太一樣，但為什麼？明明是一樣的 script，生出來的 canvas 理應是一樣的吧？此段我們將討論一些可能導致了一樣的 script 會 render 出不一樣結果的原因。

### 字型
正如我們在 [Cnavas Fingerprinting]({{< relref "/docs/browser-fingerprinting/techniques/canvas-fingerprinting" >}}) 討論過的，字型存在與否會影響 canvas render 出來的結果。如果 canvas 所指定的字型不存在，browser 會 fallback 回預設字型，如此 render 出來的圖片當然與那些字型存在的 browser render 出來的圖片不一樣。更有趣的是，Mowery 與 Shacham 發現即便是同個字型，可能也有不同的變體，他們觀察到 Linux 上內建的 Arial 有時和 Windows 內建的不太一樣。

此外，比一般字型更有用的是 emoji。不同瀏覽器對 emoji 有不同的支援狀態，也可能 render 出不一樣的樣子。例如 `😀` 與 `☺` 在 Chromium 與 Firefox 上 render 的樣子，不用算 hash 了，用肉眼看就知道不一樣了吧。
![](/images/emoji-chromium-firefox.png)
關於 Emoji 的災難，可以看這篇精彩的介紹：[The struggle of using native emoji on the web](https://nolanlawson.com/2022/04/08/the-struggle-of-using-native-emoji-on-the-web/)

以上都還只是「字型本應該長怎樣」，接下來要進到「字型如何被渲染出來」。

當代字型幾乎都是只有外框的向量圖，於是下一個問題是，現在有了外框，如何繪製出點陣圖。在 font rendering 的技術中，有兩個重要角色是 font hinting 與 anti-aliasing。

[Font hinting](https://docs.microsoft.com/zh-tw/typography/truetype/hinting) 是指利用一些指令去微調字型的輪廓，因為如果直接強硬地轉，可能不會那麼好看。左圖是沒有 hinting，右圖是有 hinting。
![](/images/font-hinting.png)
（取自：https://de.wikipedia.org/wiki/Hint）

Anti-aliasing 會於下一段一起討論。

### Sub-pixel Rendering
圖片是 discrete 的，每個 pixel 都只能有一種顏色，顏色本身也是有所限制的，並非每種顏色都表現得出來。因此，如果要在 canvas 上畫出一條曲線，不可能畫出完美的曲線，頂多在有限的、discrete 的一堆 pixel 中去盡可能地擬合出這條曲線，而這個擬合的過程就稱為 sub-pixel rendering，去討論如何運用有限的 pixel 去 render 出比一個 pixel 還精細的內容。

![](/images/canvas-sub-pixel-rendering.png)

Sub-pixel rendering 中一個重要的技術是 anti-aliasing（反鋸齒），其功能在於藉由在邊緣插入一些淡色來使文字或線條的邊緣不要有鋸齒。
![](https://upload.wikimedia.org/wikipedia/commons/a/ae/Anti-aliasing.jpg?20070304134757)
（由 MD 製作，取自 https://commons.wikimedia.org/wiki/File:Anti-aliasing.jpg ）

無論 font hinting 與 antialiasing 都是為了使 canvas render 出來更好看，但因為不同電腦的渲染方式可能不太一樣（這與瀏覽器、顯卡、驅動等都有關），導致每台電腦渲染出來的 canvas 都不一樣。

除了上面提到的幾點以外，顏色疊加、混色等，也都和顯卡如何實作這些功能有密切相關，甚至即便使用同個演算法，也可能因為浮點數的關係而有差異（記得 `a+b+c != c+b+a`），不同顯卡可能渲染出有著細微差異的圖片。

最後，[這個網站](https://browserleaks.com/canvas#how-does-it-work) 上有個 GIF 顯示了 35 個不同設備 render 出來的圖，可以參考看看。

## 如何產生 Canvas Fingerprint
好像講了很多東西，但都還沒講到最關鍵的問題，所以 canvas fingerprinting 到底要怎麼做。

具體實作其實也就只是畫出一些東西到 canvas 上，用 `toDataURL` 把資料拿出來，然後算個 hash。以下 code 修改自這個[範例](https://codepen.io/jon/pen/rwBbgQ)。

```javascript
let canvas = document.body.appendChild(document.createElement('canvas'));
let ctx = canvas.getContext('2d');
canvas.height = 200;
canvas.width = 500;

// English pangram + emojis
var txt = "Cwm fjordbank glyphs vext quiz\n\r 😀😃😄😁😆😅";
ctx.textBaseline = "top";
// Use a commonly-seen font style
ctx.font = "14px 'Arial'";
ctx.textBaseline = "alphabetic";
ctx.fillStyle = "#f60";
ctx.fillRect(125,1,62,20);

// Color mixing
ctx.fillStyle = "#069";
ctx.fillText(txt, 2, 15);
ctx.fillStyle = "rgba(102, 204, 0, 0.7)";
ctx.fillText(txt, 4, 17);

// Canvas blending
ctx.globalCompositeOperation = "multiply";
ctx.fillStyle = "rgb(255,0,255)";
ctx.beginPath();
ctx.arc(50, 50, 50, 0, Math.PI * 2, true);
ctx.closePath();
ctx.fill();
ctx.fillStyle = "rgb(0,255,255)";
ctx.beginPath();
ctx.arc(100, 50, 50, 0, Math.PI * 2, true);
ctx.closePath();
ctx.fill();
ctx.fillStyle = "rgb(255,255,0)";
ctx.beginPath();
ctx.arc(75, 100, 50, 0, Math.PI * 2, true);
ctx.closePath();
ctx.fill();
ctx.fillStyle = "rgb(255,0,255)";

// Canvas winding
ctx.arc(75, 75, 75, 0, Math.PI * 2, true);
ctx.arc(75, 75, 25, 0, Math.PI * 2, true);
ctx.fill("evenodd");


// Calcuate the hash
// https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/digest
async function digestMessage(message) {
	const msgUint8 = new TextEncoder().encode(message);
	const hashBuffer = await crypto.subtle.digest('SHA-256', msgUint8);
	const hashArray = Array.from(new Uint8Array(hashBuffer));
	const hashHex = hashArray.map((b) => b.toString(16).padStart(2, '0')).join('');
	return hashHex;
}

digestMessage(canvas.toDataURL())
.then((digestHex) => document.body.appendChild(document.createTextNode(digestHex)));

```

## 防禦
防禦 canvas fingerprinting 很難，非常難。

一個很直觀的想法是：我就加上一點 random noise 啊。但如果是這樣，其實只要多 fingerprint 幾次然後取個平均，noise 就會被抵銷掉了。此外，藉由 render 一張全白的 canvas，追蹤者可以得知使用者有沒有嘗試防禦 canvas fingerprinting，於是這就又成為一個 feature 了。之後會有一篇專文介紹 browser fingerprinting 的 privacy paradox，屆時會有更多討論。

那如果加上固定 noise 呢？這樣只會提升我和其他人 fingerprint 的不同而已，因為原本兩人的 canvas fingerprint 的差異只來自 rendering 的差異，加上固定 noise 之後，差異就有了來自 rendering 的與來自 noise 的，反而更好做 fingerprinting。而且，其實我只要 render 一張全白的圖，就能直接找到固定的 noise 了，甚至因為這個 noise 可能是因使用者而異（可能是 extension 安裝時隨機生成的），於是固定的 noise 就直接變成 fingerprint 了。聽起來糟透了。喔對，有個號稱可以阻擋 canvas fingerprinting 的 browser extension 叫做 [Canvas Defender](https://addons.mozilla.org/zh-TW/firefox/addon/no-canvas-fingerprinting/)，它就是使用固定 noise，作者是宣稱因為 noise 可以改，所以使用者想「斷尾」時可以直接改 noise，這個手段是否合理就交給大家自行評價吧。

另一個作法是，確保大家的執行環境是一樣的。既然顯卡不能一樣，那我們就回歸 software rendering 的時代，然後建立一個字型白名單，大家都必須配備白名單內的字型，且不能使用白名單外的字型。這雖然可能有效，但如此會變得很不方便。

當然我們也可以放大絕，直接封鎖 Canvas API（像是 Tor），或是要求使用者同意後才可以讓 Javascript 讀到 `<canvas>` 的內容，但這對使用者體驗大概不是太好。



最後，想快速地討論一下，前面我們都在講 canvas fingerprinting 有多恐怖，但那是 10 年前的實驗了。10 年後的現在，大家的 mobile browser 幾乎都是 [Chrome 和 Safari](https://gs.statcounter.com/browser-market-share/mobile/worldwide/)，所有同世代 iPhone 的硬體都是一樣的，OS 更是只有一個 walled garden 可以選，Android 手機雖然選擇多在最主流的也就那幾個，在軟硬體都非常接近的情況下，canvas fingerprinting 在 mobile browser 上能提供的資訊其實也不多，能從 canvas fingerprinting 拿到的 entropy 大概都是 user agent 就有的了（e.g. browser, OS）。雖然我也不知道這是不是好事就是了。

## 參考資料
Mowery, Keaton, and Hovav Shacham. "Pixel perfect: Fingerprinting canvas in HTML5." Proceedings of W2SP 2012 (2012).
[Everything you need to know about canvas fingerprinting](https://multilogin.com/blog/everything-you-need-to-know-about-canvas-fingerprinting/)（注意此文對於防禦的討論可能有些問題）
[How Does Canvas Fingerprinting Work?](https://fingerprint.com/blog/canvas-fingerprinting/)
https://codepen.io/jon/pen/rwBbgQ
