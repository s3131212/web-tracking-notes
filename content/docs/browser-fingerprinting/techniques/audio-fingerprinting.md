---
weight: 6
title: "Audio Fingerprinting"
---

# Audio Fingerprinting
Audio fingerprinting 是指涉利用 Web Audio API 來做 fingerprinting 的技術。Web Audio API 是個專門用於在 browser 處理音訊的 API。一如往常的，有人發現 Web Audio API 在不同的軟硬體設備上會產生不同的結果，而且在同一台電腦的同一個瀏覽器上，所產生的結果很穩定與持久，所以可以用來做 fingerprinting。

## Web Audio API
在直接討論 audio fingerprinting 前，得先花一些篇幅討論 Web Audio API 的運作機制。

Web Audio API 是一個在 browser 中處理音訊的 API，其可以建立音訊、載入現存音訊、加入特效等。整個操作流程都是在 [AudioContext](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext) 底下完成，以 audio nodes（音訊節點）作為音訊作業的基本單元，這些 audio nodes 可以互相連接，建構出 audio routing graphs（音訊路由圖）。Audio routing graph 起於一個或多個音訊來源（可以是動態產生的或載入現存檔案），音訊來源會將音訊取樣，切成很多個 sample。接著，音訊的 sample 會順著路由經過各種處理，可能是調整音量、加入特效、混音等。當音訊已經走完所有處理程序，最後就會走到整個 audio routing graph 的終點：可能是播放出來或是儲存進一個 buffer。

## Audio Fingerprinting
接著我們把每個細節都拆開來看，看看每個步驟可以怎麼操作，以達到 audio fingerprinting。

AudioContext 是整個處理流程的 context，其要求音訊最後的[去處](https://developer.mozilla.org/en-US/docs/Web/API/BaseAudioContext/destination)必須是個音訊輸出硬體設備，像是音響或是耳機。但在 fingerprinting 的情境下，我們不希望聲音真的被播出來，我只想看到他處理完成的結果。此時可以使用 [OfflineAudioContext](https://developer.mozilla.org/en-US/docs/Web/API/OfflineAudioContext)，它會將音訊處理完之後儲存到 in-memory 的 buffer 內。這個 buffer 名為 [AudioBuffer](https://developer.mozilla.org/en-US/docs/Web/API/AudioBuffer)，其內部使用 linear PCM 並正規化到 -1 ~ 1。

```javascript
const context = new OfflineAudioContext(1, 5000, 44100);
```
（本文所使用的範例 code 與參數主要來自 [fingerprint.js](https://github.com/fingerprintjs/fingerprintjs/blob/3201a7d61bb4df2816c226d8364cc98bb4235e59/src/sources/audio.ts)，會做一些微調）

目前為止我們有 context 也知道音訊的目的地去哪了，接著要回過頭來看看，音訊從哪來的。我們當然可以載入一個現成的檔案，但以 fingerprinting 來說這樣太麻煩了，其實重點只是要有個音訊而已，所以用個 Oscillator（震盪器）去產生一個 sine wave 之類的其實就可以了。

```javascript
const oscillator = new OscillatorNode(context, {
	type: 'sine',
	frequency: 10000
});
```

目前為止的音訊大概長這樣：

![](/images/audio-before.png)

接著，我們想要盡可能對音訊做各種莫名其妙的處理，越多處理，不同軟硬體就有可能產出各種不同的結果。能做的事情很多，不過為求簡單，我們就用個基本的 [Compressor](https://developer.mozilla.org/en-US/docs/Web/API/BaseAudioContext/createDynamicsCompressor) 就好。Compressor 主要用於把音訊中最大聲的部份往下壓，藉此避免爆音，但它一般用於什麼不重要，重點是因為 Compressor 會對音效做一些處理，而我們想要捕捉到這些處理之後微小的落差。所以我就隨便指定一些參數（嚴格來說是直接從開源版本的 fingerprint.js 複製來的），反正只要數字合理就好。

```javascript
const compressor = new DynamicsCompressorNode(context, {
	threshold: -50,
	knee: 40,
	ratio: 12,
	reduction: -20,
	attack: 0,
	release: 0.25
});
```

經過一連串的處理，現在的音訊長這樣：
![](/images/audio-after.png)

最後把兩者串在一起
```javascript
oscillator.connect(compressor);
compressor.connect(context.destination);
```

然後令 audio context 動起來，去取每一個 sample 的值，拿去算出一個 fingerprint。

```javascript
oscillator.start(0);
context.oncomplete = (event) => {
	const samples = event.renderedBuffer.getChannelData(0);
	console.log(getFingerprint(samples));
};
context.startRendering();
```

最後實作一下算 hash 的功能，坦白說也沒什麼黑魔法，取個絕對值的和就可以了。
```javascript
function getFingerprint(samples) {
	return samples.reduce((ps, s) => ps + Math.abs(s), 0);
}
```

同樣是 Firefox，我的 Linux 拿到的數值是 `344.5930904112756`，Windows 拿到的是 `344.5930906422436`。在 Linux 的 Chrome 上則是拿到 `1169.8477467813227`。在三個情境下，無痕模式與一般模式拿到的 fingerprint 都是一樣的。

## 為何 Web Audio API 會產生不同結果
正如我們在 canvas fingerprinting 與 WebGL fingerprinting 時問過的問題：為什麼同樣一段 code 會產生不同結果？

就表象上來說，因為不同軟硬體下，oscillator 產生的波形有非常些微的差異：
![](/images/oscillator-sample.png)
（左邊是 Chrome，右邊是 Firefox，使用相同的電腦產生頻率 100 的 sine wave，取樣率為 50）

基本上主流 browser engine（Chrome 的 Blink、Safari 的 Webkit、Firefox 的 Gecko）目前的 Web Audio API 實作都是基於早期 Webkit 的實作，但隨著時代演變，各個 browser engine 都陸陸續續地加上自己的 implementation，不同的 implementation 可能會有不同結果（試想同樣一個算式，使用浮點數計算時，順序不一樣就有不同結果了），這導致不同 browser 的 behavior 開始有些微的落差。

在硬體層面，為了改善處理速度，有些 browser 會針對特定平台做最佳化。例如 Blink 在 Mac 上有[特殊的 FFT 實作](https://github.com/chromium/chromium/blob/55d9bda1b972b1488b1bb202e01a7ef7b6fff937/third_party/blink/renderer/platform/audio/mac/fft_frame_mac.cc)，也有對不同 CPU architecture 做特別處理，像是 x86 上會使用 [AVX 指令集](https://github.com/chromium/chromium/blob/55d9bda1b972b1488b1bb202e01a7ef7b6fff937/third_party/blink/renderer/platform/audio/cpu/x86/vector_math_avx.cc)。此外，這個計算過程中也大量地使用了浮點數，所以計算精度也一樣會影響結果。

## 解法
Tor Browser 的解法總是簡單而暴力：預設停用 Web Audio API。

Brave 的主要防禦方法是，針對每個不同的 eTLD+1，都加上不同的但固定的 noise。在不同 eTLD+1 使用不同 noise，是為了保證了 audio fingerprinting 無法用於 cross-site tracking。在相同 eTLD+1 必須使用相同 noise，因為如果每次 noise 都不一樣的話，多取樣幾次就平衡回來了。不過這有個問題是，當沒有聲波時，我就可以直接還原 noise，正如在 canvas fingerprinting 的防禦中，只要產生一張空白的 canvas 就能還原出 noise，再把 canvas 的數值減去 noise，就可以還原出真正的、沒被混淆過的 canvas 值。然而這在 audio fingerprinting 上是不可能的，因為數字太小了，浮點數的計算誤差會大到把有用的 fingerprint 蓋過去。

基本上其他防禦方式八九不離十是 randomize 或是盡可能統一 implementation，所以就不贅述了。

## 參考資料
[Web Audio API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
[How the Web Audio API is used for audio fingerprinting](https://fingerprint.com/blog/audio-fingerprinting/)