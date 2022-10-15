---
weight: 4
title: "WebGL Fingerprinting"
---

# WebGL Fingerprinting
討論完 canvas fingerprinting 之後，我們要來討論它的近親：WebGL Fingerprinting。WebGL Fingerprinting 分成兩個部份：WebGL Report Fingerprinting 與 WebGL Canvas Fingerpinting，後者基本上就是 canvas fingerprinting 的 WebGL 版本，所以才說他們是近親。WebGL Fingerprinting 利用了每台電腦在軟硬體上的不同，尤其是顯卡與顯示驅動不同，來做 fingerprinting。有趣的是，因為同一台電腦配備的顯卡一樣，所以 WebGL Fingerprinting 有可能用於跨瀏覽器的 tracking，不過這部份的討論會放到其他專文。

## 何謂 WebGL
一言以蔽之，WebGL 是 OpenGL 的 Web 版本。

WebGL 提供了一個 interface 讓開發者可以藉由 Javascript 操作 OpenGL ES 的 API，因此得以借助此 API 來在 browser 裡面實現 2D 與 3D 的繪圖，並輸出至 `<canvas>`（與 Canvas API 無關，請見昨天的文），其中不需要使用外掛程式。WebGL 的程式大概包含幾個部份：Javascript 控制主要邏輯，GLSL（OpenGL Shading Language）處理著色器的邏輯，也就是各種電腦圖形課會有的 shading 黑魔法，然後 GPU 執行著色器的程式碼。

## WebGL Report Fingerprinting
WebGL 可以讀取許多關於顯卡以及驅動的參數，通常我們稱呼這些資訊為 WebGL report，其用於了解該設備是否符合需求。參數的種類很多，可以到 [WebGL Report](https://webglreport.com/?v=2) 看到，這些資訊都可以用於 browser fingerprinting。

![](/images/webgl-report.png)

取得 WebGL 的參數通常可以使用 `WebGLRenderingContext` 的 `gl.getExtension` 與 `gl.getgetParameter`。舉例來說，在 WebGL 裡面，可以直接拿到顯卡的 vendor 與 renderer。其中可以分為一般的跟 unmasked 的，後者需要存取 debug 資料（`WEBGL_debug_renderer_info`），所以不一定每個 browser 都有。
```javascript
const canvas = document.getElementById('canvas');
const gl = canvas.getContext('webgl');
const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');

console.log(`${gl.getParameter(gl.VENDOR)}, ${gl.getParameter(debugInfo.UNMASKED_VENDOR_WEBGL)}`);
console.log(`${gl.getParameter(gl.RENDERER)}, ${gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL)}`);
```

其他可以用來 fingerprint 的資料還有 WebGL [啟用了哪些 extension](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/getSupportedExtensions)（可以用 `gl.getSupportedExtensions` 取得）以及[浮點數運算的 precision](https://developer.mozilla.org/en-US/docs/Web/API/WebGLShaderPrecisionFormat)（可以用 `gl.getShaderPrecisionFormat` 取得）等。

因為這不是 WebGL 教學文件，我也不懂 Computer Graphics，所以就不細談每個參數的意思，只需要知道這些參數可以用於 fingerprinting。關於實作，可以參考上述 WebGL Report 網站的 [repo](https://github.com/CesiumGS/webglreport)。

## WebGL Canvas Fingerprinting
WebGL Canvas Fingerprinting 就如同昨天提到的 canvas fingerprinting，因為軟硬體上的不同，每一台電腦上的 browser 在使用 WebGL 時，可能渲染出不太一樣的圖，利用這個特性，我們便可以用 WebGL 渲染功能來做 browser fingerprinting。

與 canvas fingerprinting 類似，我們需要做的是渲染一張足夠複雜的圖，然後把圖抓出來算 hash。不過需要注意的是，早在這方法剛提出時，就有人發現 WebGL 的結果其實不太穩定，同一台電腦可能產出不一樣的結果，例如 AmIUnique 的作者就主張 WebGL canvas fingerprinting 爛透了。Cao 等人之後發現問題出 AmIUnique 沒有控制 anti-aliasing 與 canvas size 等重要參數，才導致 render 結果這麼不穩定，藉由控制這些參數，WebGL render 結果足夠穩定以致可以用於 browser fingerprinting。

### 為什麼 WebGL 的 render 結果不同
老問題又出現了：為什麼一段一樣的 code 在不同設備上會 render 出不同的結果。Wu 等人（就是前面 Cao 等人的研究團隊）提出了一個有趣的解釋：render 的差異來自於浮點數運算的差異。幾乎所有 Computer Graphics 的演算法都廣泛地使用浮點數，當不同平台在浮點數的精準度或是處理上有差異時，便會造成誤差，又因為浮點數運算實在太多了，這些誤差會被不斷放大直到可以被偵測出來。

第一種可能的誤差來自取近似值的方法。

在 Web 上我們會使用 hex triplet（三個 8bit 的數字，所以共會吃掉 24 bit）來表示顏色，但在 WebGL 中，使用三個 0~1 的浮點數來表達。於是當 WebGL 要被渲染到畫面上時，需要把 RGB 中 [0, 1] 的浮點數轉換成 [0, 255] 的 hex triplet。一個常見的實作方法是把 WebGL 的浮點數 * 256 後取近似的整數。假設在 A 瀏覽器在取近似值時使用 *floor*（無條件捨去），B 瀏覽器使用 *round*（四捨五入），這便會造成誤差。同樣是 (0.5, 0.5, 0.5)，會被分別轉換成 (127, 127, 127) 與 (128, 128, 128)。在視覺上這幾乎完全沒差，但已經足以拿來作為 fingerprint 了。

同樣的問題也會出現在 texture mapping 時做 linear interpolation，或是計算透明度等，這些操作中也都會出現需要取近似值，而近似值的取法不同便會造成結果不同。

第二種誤差可能來自於浮點數本身的運算誤差。

眾所皆知的，[浮點數的運算並不精準](https://0.30000000000000004.com/)，且每個 implementation 的 precision 也不一定一樣。如果現在有兩種實作，前者是 10-bit floating number，後者是 16-bit floating number，兩者經過一系列的計算之後，顯然結果會不太一樣。這所造成的影響很多。例如，當判斷一個點是否包含在某個區域內時，可能就會因為 precision 不同而在不同實作下有不同的結果。

## 解法
WebGL Report Fingerprinting 的解法通常只能透過完整關掉 WebGL。雖然投遞假的數值並不是不可能，但影響有限，因為透過實際嘗試繪圖，還是能重新找出真正的值。如果把參數限縮某個特定的值，則使用高階電競顯卡的使用者大概會崩潰吧。

WebGL Canvas Fingerprinting 的防禦方法 canvas fingerprinting 很像，畢竟都是基於 canvas。基本上防禦 canvas fingerprinting 對 WebGL canvas fingerprinting 都還是有用。Tor Browser 一如往常的直接封鎖 WebGL。另一個有趣的方向是，既然我們知道 WebGL 的 render 差異來自浮點數，那就讓浮點數的計算是統一的啊，這就是前面提過的 Wu 等人的團隊之前做過的事情，有興趣的讀者可以參考 reference。

## 同場加映：在 browser 上面做 anti-VM
在惡意程式分析的領域中，攻擊者為了避免自己的惡意程式被分析，會偵測自己是不是在 VM 中，如果在 VM 中，就判定這是分析者的實驗環境，所以不要發作，讓分析者找不到惡意程式的行為，畢竟真正的使用者（受害者）用 VM 作為主力機的機率很低。

那，在 browser 上，有可能偵測 VM 嗎？也許可以喔。

試想，上一次你看到一台電腦沒有顯卡沒有內顯，是什麼時候？我人生中用的第一台電腦就已經有內顯了欸。但許多 VM 預設沒有顯卡，無論是模擬或是 GPU passthrough 都需要特別設定。因此，如果 WebGL report 中的 renderer 的值並非顯卡，就很有可能這是在 VM 之中。例如下圖是我在 PVE 中的 Windows 跑出來的結果。

![](/images/webgl-report-vm.png)

或是有時 renderer 名稱就直接暴雷了。例如下圖是我在 VirtualBox 中的 Windows 跑出來的結果。

![](/images/webgl-report-vm-2.png)

當然，做 (anti-)anti-VM 的人大概看了會笑出來，這也太容易繞過了吧。這方法雖然對專家沒用，但在多數情境下，使用 VM 的人可能都不會刻意去改規定，他們大概也不會意識到其實 web 上也可能做到（非常陽春的）VM 偵測，只覺得奇怪怎麼驗證碼一直跳而已。

## 參考資料
Wu, Shujiang, et al. "Rendered Private: Making {GLSL} Execution Uniform to Prevent {WebGL-based} Browser Fingerprinting." _28th USENIX Security Symposium (USENIX Security 19)_. 2019.
Cao, Yinzhi, Song Li, and Erik Wijmans. "(Cross-) Browser Fingerprinting via OS and Hardware Level Features." _NDSS_. 2017.
https://browserleaks.com/webgl