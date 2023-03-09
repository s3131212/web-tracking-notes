---
weight: 4
title: "WebGL Fingerprinting"
---

# WebGL Fingerprinting
WebGL Fingerprinting 分成兩個部份：WebGL Report Fingerprinting 與 WebGL Canvas Fingerpinting，後者基本上就是 canvas fingerprinting 的 WebGL 版本，所以才說他們是近親。WebGL Fingerprinting 利用了每台電腦在軟硬體上的不同，尤其是顯卡與顯示驅動不同，來做 fingerprinting。有趣的是，因為同一台電腦配備的顯卡一樣，所以 WebGL Fingerprinting 有可能用於跨瀏覽器的 tracking，不過這部份的討論會放到其他專文。

## 何謂 WebGL
一言以蔽之，WebGL 是 OpenGL 的 Web 版本。WebGL 提供了一個介面（interface）讓開發者可以藉由 JavaScript 操作 OpenGL ES 的 API，因此得以借助此 API 在瀏覽器中實現 2D 與 3D 的繪圖，並輸出至 `<canvas>`，其中不需要使用瀏覽器外掛。WebGL 的程式大概包含幾個部份：JavaScript 控制主要邏輯，GLSL（OpenGL Shading Language）處理著色器的邏輯，然後 GPU 執行著色器的程式碼。

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
WebGL canvas fingerprinting 與 canvas fingerprinting 雷同，都是因為軟硬體上的不同，導致瀏覽器使用 WebGL 時可能繪製出不太一樣的圖，利用這個特性，追蹤者便可以用 WebGL 繪圖功能達成 browser fingerprinting。WebGL canvas fingerprinting 在實際使用上也與一般的 canvas fingerprinting 一樣，追蹤者需要繪製一張足夠複雜的圖，並把繪製結果匯出並使用雜湊函數產生 fingerprint。

### 為什麼 WebGL 的 render 結果不同
於是我們又回到一樣的問題：為什麼一段相同的程式碼在不同設備上會繪製出不同的結果。Wu 等人提出了一個有趣的解釋：繪製的差異來自於浮點數運算的差異。幾乎所有 Computer Graphics 的演算法都大量地使用浮點數，當不同平台在浮點數的精準度或是處理上有差異時，便會造成誤差，又因為浮點數運算實在太多了，這些誤差會被不斷放大直到可以被偵測出來。

在那之前，我們必須先理解 WebGL 的繪製流程。WebGL 的繪製流程可以被區分為三個步驟。首先，vertex shader 會執行一系列與頂點相關的操作，例如套用旋轉矩陣等。具體而言，vertex shader 會將 attribute（套用到個別頂點的屬性）與 uniform（套用到所有頂點的屬性）綁定到每個頂點（例如套用 texture 便是在這一步），並輸出轉換後 gl_Position 與 varyings。接著 vertex shader 的輸出會被送入  WebGL 的 rasterization 與varying interpolation 模組，將模型轉換成像素網格，並且利用頂點的 attributes 對 varyings 做內插（interpolation）。上述的輸出會被送入 fragment shader 做上色，其在套用各種相關操作（例如 texture mapping）之後，將結果繪製在 canvas 上。

在這整個過程中，涉及許多浮點數操作。在第一階段都有大量的矩陣操作。舉例來說，在處理頂點時，需要將頂點對應到像素網格時。WebGL 的座標系使用 [-1, 1] 的浮點數。如果有個點在 (0.0, 0.0)，若要繪製到一張 8x8 的 canvas 上，並沒有一個整數點可以剛好對應到正中央，於是放在 (4, 4), (4, 5), (5,4), (5, 5) 都是對的。對於使用無條件捨去、無條件進入、四捨五入的系統，算出來的結果便會有些微的、肉眼不可見但可用於 fingerprinting 的差異。

在第二階段同樣會運用到許多浮點數。例如，在判斷一個像素點是否在圖形內、計算 z-buffer（識別頂點的前後順序）、對 varyings 做內插時，都會用到浮點數。例如在給定一個三角形，問一個像素點是否在三角形內，如果有便繪製三角形的顏色，否則使用底色。在不同實作下，可能會因為浮點數誤差而導致不同結果，導致該像素在不同實作下有不同的顏色。

在第三階段，處理上色時，一樣會遇到浮點數的問題。在 Web 上會使用 hex triplet（三個 8bit 的數字）來表示顏色，但在 WebGL 中，使用三個 0~1 的浮點數來表達。於是當 WebGL 要繪製畫面時，需要把 RGB 中 [0, 1] 的浮點數轉換成 [0, 255] 的 hex triplet。一個常見的實作方法是把 WebGL 的浮點數 256 後取近似的整數。假設在 A 瀏覽器在取近似值時使用無條件捨去，B 瀏覽器使用四捨五入，便會造成誤差。同樣是 (0.5, 0.5, 0.5)，會被分別轉換成 (127, 127, 127) 與 (128, 128, 128)。在視覺上這幾乎完全沒差，但已經足以造成可用於 fingerprinting 的差異。

## 解法
WebGL Report Fingerprinting 的解法，除了完整關掉 WebGL，便是投遞假的數值。例如 Brave 現在會將一些資訊混淆，extension 只回傳 `WEBGL_debug_renderer_info`，renderer 在同個 eTLD+1 同個 session 下回傳同樣的隨機字串。

WebGL Canvas Fingerprinting 的防禦方法 canvas fingerprinting 很像，畢竟都是基於 canvas。基本上防禦 canvas fingerprinting 對 WebGL canvas fingerprinting 都還是有用。Tor Browser 一如往常的直接封鎖 WebGL。另一個有趣的方向是，既然我們知道 WebGL 的繪製差異來自浮點數，這意味著只要浮點數運算可以統一，便能完全消除繪製結果的差異。這正是前面提過的 Wu 等人的團隊之前做過的事情，他們把第一階段的 vertex rendering 移動到 JavaScript 來做，以確保行為統一；第二階段的 rasterization 與varying interpolation，以及第三階段的 fragment rendering，則改用整數去模擬有理數運算。根據他們的實驗，如此可以確保繪製結果一定一樣。

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