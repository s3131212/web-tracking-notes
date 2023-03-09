---
weight: 3
title: "Canvas Fingerprinting"
---

# Canvas Fingerprinting
最早對 browser fingerprinting 都聚焦在一些明顯可見的、分布清楚的各種屬性，例如 user agent 的不同是一翻兩瞪眼的，任何人都看得出來。在 2012 年時，Mowery 與 Shacham 發現了一個 browser fingerprinting 方法： HTML5 的 Canvas API 在不同平台上 render 的結果的細微差異，利用這些細微差異，便可以提供足夠的資訊做 fingerprinting。因此，我只要讓 Canvas API 畫出一張圖，並取出繪圖結果，便可達成一個足夠獨特的特徵。這個發現徹底改變了 web tracking 的生態，也改變了大家對 browser fingerprinting 的看法。這可能是第一次有這麼高準確度又 practical 的 browser fingerprinting 方法，而且其特徵還是如此隱微的、肉眼完全不可見的差異。

## 前言
Canvas 直譯是「畫布」，顧名思義，這東西與畫圖有關。我們需要先區分 Canvas API 與 `<canvas>` 兩個看起來很像但其實不同的東西。Canvas API是一個可以讓 JavaScript 繪製 2D 圖形的工具，它提供了許多畫圖的 method，像是畫直線、畫矩形、填充顏色、插入文字等。`<canvas>` 是 HTML 的一個 tag，它如同其名，真的是個畫布，讓 Canvas API、WebGL API 等在其之上繪圖。

Canvas Fingerprinting 字面意義上便是用 canvas 做  browser fingerprinting。通常此處的 canvas 是 Canvas API 而不是 HTML 的 `<canvas>`。下一章節會介紹 WebGL API 用於 browser fingerprinting。讀者可能會疑惑，為何需要去區分兩者。因為即便同樣是繪圖在 `<canvas>` 上，同樣是利用渲染在 `<canvas>` 上的內容做 browser fingerprinting，Canvas API 可以被 fingerprint 的成因與 WebGL API 可以被 fingerprint 的成因不完全相同，所以區分兩者有助於做更細緻的討論。

Canvas fingerprinting 之所以可能，在於 Canvas API 只是給出一些繪圖的指示，這些指示需要先由瀏覽器與作業系統處理過，才能將其繪製出來，而不同的瀏覽器、作業系統等，在遇到同樣的指示時，可能會有不太一樣的行為，例如不同作業系統可能有不同的預設字型，顯卡可能使用了不同的演算法實作混色與 anti-aliasing 等，將這些細微實作差異累積起來，最後就會生出略有不同的圖片了。這些差異使得追蹤者可以利用 Canvas API 的輸出做 fingerprinting。

## 為何 Canvas Fingerprinting 存在
剛剛我們提到了，canvas fingerprinting 利用了不同軟硬體設備會讓 canvas 長得不太一樣，但為什麼？明明是一樣的 script，生出來的 canvas 理應是一樣的吧？此段我們將討論一些可能導致了一樣的 script 會 render 出不一樣結果的原因。

### 字型
正如我們在 [Canvas Fingerprinting]({{< relref "/docs/browser-fingerprinting/techniques/canvas-fingerprinting" >}}) 討論過的，字型存在與否會影響 canvas render 出來的結果。如果 canvas 所指定的字型不存在，browser 會 fallback 回預設字型，如此 render 出來的圖片當然與那些字型存在的 browser render 出來的圖片不一樣。更有趣的是，Mowery 與 Shacham 發現即便是同個字型，可能也有不同的變體，他們觀察到 Linux 上內建的 Arial 有時和 Windows 內建的不太一樣。

此外，比一般字型更有用的是 emoji。不同瀏覽器對 emoji 有不同的支援狀態，也可能 render 出不一樣的樣子。例如 `😀` 與 `☺` 在 Chromium 與 Firefox 上 render 的樣子，不用算 hash 了，用肉眼看就知道不一樣了吧。
![](/images/emoji-chromium-firefox.png)
關於 Emoji 的災難，可以看這篇精彩的介紹：[The struggle of using native emoji on the web](https://nolanlawson.com/2022/04/08/the-struggle-of-using-native-emoji-on-the-web/)

以上都還只是「字型本應該長怎樣」，接下來要進到「字型如何被渲染出來」。

當代字型幾乎都是只有外框的向量圖，如此設計雖然可以使字型呈現很精準，但光是有外框，還不足以呈現在螢幕上，因為精準的向量圖必須以有限的像素網格繪製，而直接沿著外框繪製，效果並不理想。在字型光柵化（font rasterization）的技術中，有兩個重要角色：字型微調（font hinting）與抗鋸齒（anti-aliasing）。

字型微調（font hinting）是指藉由一些指令去微調字型的輪廓，使其在像素網格上可以對齊網格，以提升字型的清晰度。左圖是沒有 hinting，右圖是有 hinting，便可看出，藉由對齊網格，字型變得更為清楚。

![](/images/font-hinting.png)
（取自：https://de.wikipedia.org/wiki/Hint）

抗鋸齒（anti-aliasing）的功能在於，藉由在邊緣插入一些淡色，使文字或線條的邊緣不要有明顯的鋸齒。當高解析度的訊號（例如向量圖呈現的完美弧線）以解析度相對有限的顯示器表示時，會經過取樣（sampling）使高解析度的資料以較少資料點表示，這個過程中會損失一些資料，使得結果失真，在圖形繪製上，這種失真會展現在圖形邊緣出現鋸齒。抗鋸齒正是為了解決這種鋸齒，藉由一些演算法（通常是套用濾波器），將邊緣鋸齒處調補一些淡色。

無論 font hinting 與 antialiasing 都是為了使 canvas render 出來更好看，但因為不同設備的渲染方式可能不太一樣（這與瀏覽器與作業系統等都有關），導致每台電腦渲染出來的 canvas 有著些微的不同，成為可以被 fingerprint 的對象。

### Sub-pixel Rendering
圖片是 discrete 的，每個 pixel 都只能有一種顏色，顏色本身也是有所限制的，並非每種顏色都表現得出來。因此，如果要在 canvas 上畫出一條曲線，不可能畫出完美的曲線，頂多在有限的、discrete 的一堆 pixel 中去盡可能地擬合出這條曲線，而這個擬合的過程就稱為 sub-pixel rendering，去討論如何運用有限的 pixel 去 render 出比一個 pixel 還精細的內容。

![](/images/canvas-sub-pixel-rendering.png)

Sub-pixel rendering 中一個重要的技術是 anti-aliasing（反鋸齒），其功能在於藉由在邊緣插入一些淡色來使文字或線條的邊緣不要有鋸齒。
![](https://upload.wikimedia.org/wikipedia/commons/a/ae/Anti-aliasing.jpg?20070304134757)
（由 MD 製作，取自 https://commons.wikimedia.org/wiki/File:Anti-aliasing.jpg ）

無論 font hinting 與 antialiasing 都是為了使 canvas render 出來更好看，但因為不同電腦的渲染方式可能不太一樣（這與瀏覽器、顯卡、驅動等都有關），導致每台電腦渲染出來的 canvas 都不一樣。

除了上面提到的幾點以外，顏色疊加、混色等，也都和顯卡如何實作這些功能有密切相關，甚至即便使用同個演算法，也可能因為浮點數的關係而有差異（記得 `a+b+c != c+b+a`），不同顯卡可能渲染出有著細微差異的圖片。

## 如何產生 Canvas Fingerprint
產生 canvas fingerprinting 的方式非常直關：畫出一張足夠複雜的圖，並取得繪製的結果。通常繪製結果的取得方式是用 `toDataURL` 取得圖片內容後，將其輸入雜湊函數以取得較小的、可以凸顯差異的雜湊，並將其作為 fingerprint。

以下 code 修改自這個[範例](https://codepen.io/jon/pen/rwBbgQ)。

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
防禦 canvas fingerprinting 出乎意料地困難。

乍看之下有個很直觀的防禦方法是：在每個 canvas 繪製結果加上一點隨機的雜訊。這麼做的問題在於，只要多取幾次 fingerprint 並取平均，便可將雜訊抵銷掉。
即便把雜訊改成固定的，也不一定會有所幫助。這樣只會提升fingerprint 的獨特性而已，因為原本兩個裝置的 canvas fingerprint所呈現的差異只來自繪製的差異，加上固定雜訊之後，差異就有了來自繪製的與來自雜訊的，反而使 fingerprint 更為獨特。而且，其實攻擊者只要繪製一張全白的圖，便可以直接還原出固定的雜訊了，甚至因為這個雜訊因使用者而異，於是固定的雜訊可以直接作為fingerprint。

然而這也不代表固定雜訊就毫無用處。在前面的章節提過，browser fingerprinting 的重點在於當 cookie 被刪除時可以還原 cookie，以及達成 cross-site tracking。換言之，防禦手段不需要完全使 canvas fingerprinting 變得無效，只需要使其無法達成上述兩點即可。因此，Brave Browser 的防禦手段是：針對不同的 origin 加上不同的固定雜訊，並且在清除儲存空間時重置雜訊。由於不同 origin 有不同雜訊，因此無法藉由 canvas fingerprinting 達成 cross-site tracking。又由於清除儲存空間時雜訊會被重置，因此無法藉此還原被刪除的 cookie，至於沒有重置儲存空間時，本就可以使用 cookie 進行 first-site tracking，因此此時允許 canvas fingerprinting 並沒有增加風險。

另一個策略是，確保大家的執行環境是一樣的。既然顯卡不能確保繪製結果都一樣，那就回歸 software rendering 的時代，並建立一個字型白名單，要求所有人都必須配備白名單內的字型，且不能使用白名單外的字型。這雖然可能有效，但如此會變得很不方便。

當然，最極端的方法是直接封鎖 Canvas API，或是要求使用者同意後才可以讓 JavaScript 讀到 `<canvas>` 的內容。Tor Browser 與開啟 browser fingerprinting 防禦的 Firefox 便是採取此策略。但這對使用者體驗可能不是太好。

最後，值得一提的是 canvas fingerprinting 雖然在十多年前剛提出時非常有效，但十年後的現在，大家都使用行動裝置上網，瀏覽器幾乎都是 Chrome 和 Safari，所有同世代 iPhone 的硬體都是一樣的，iOS版本分布也很集中，Android 手機雖然選擇多，但特定幾個裝置仍然佔有最高的市場份額。在軟硬體都欠缺多樣性的情況下，canvas fingerprinting 在行動瀏覽器上能提供的資訊不一定有過往那麼多了。

## 參考資料
Mowery, Keaton, and Hovav Shacham. "Pixel perfect: Fingerprinting canvas in HTML5." Proceedings of W2SP 2012 (2012).
[Everything you need to know about canvas fingerprinting](https://multilogin.com/blog/everything-you-need-to-know-about-canvas-fingerprinting/)（注意此文對於防禦的討論可能有些問題）
[How Does Canvas Fingerprinting Work?](https://fingerprint.com/blog/canvas-fingerprinting/)
https://codepen.io/jon/pen/rwBbgQ
