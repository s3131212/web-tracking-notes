---
weight: 4
title: "Browser Fingerprinting 的防禦如何失敗，以及 Privacy Paradox"
---

# Browser Fingerprinting 的防禦如何失敗，以及 Privacy Paradox
在上一篇文章，我們介紹了 browser fingerprinting 的幾個防禦策略，並做了一個巨大的宣稱：anti-fingerprinting 超級難！在這篇文章，我們將對這個宣稱做更多討論，到底為什麼防禦這麼困難，常見的 anti-fingerprinting 策略又是如何失敗的。接著我們將討論 anti-fingerprinting 失敗如何造成為害。不同於常見的 attack & defense，如果 defense 失敗頂多就是 attack 會成功，在 browser fingerprinting 的世界，如果 anti-fingerprinting 失敗，反而會讓 browser fingnerprinting 更容易！這種「保護隱私失敗了反而會讓狀況變得更糟」有時被稱為 Privacy Paradox。

通常 anti-fingerprinting 都是在 browser extension 層級做，畢竟魔改 browser 沒那麼簡單，要大家轉而使用魔改過的 browser 也不容易，browser extension 相對簡單。然而 browser extension 的限制導致 anti-fingerprinting 很難做好。

## 修改數值的挑戰
一種常見的 anti-fingerprinting 策略是直接去修改可以作為 fingerprint 的數值。就只不過是改個數值而已，是可以有多難？寫 Javascript 的第一天不就知道可以用 `=` 賦值了嗎？

```javascript
screen.width = 1920;
```

恩，很完美，直到你遇到有人嘗試把它刪掉。
```javascript
delete screen.width;
```

突然間原始數值就回來了！在原生 `screen` 中，`width` 是在 prototype 中被定義的，用 `=` assign 只是用 object property 掩蓋 prototype 而已，把 property 刪掉 prototype 就重新出現了。那如果我們直接覆蓋 prototype 呢？

```javascript
Object.defineProperty(Screen.prototype, "width", {value: 1920});
```

這下刪不掉了吧？不過如果嘗試取得 property 的 descriptor，會發現原本 `screen.width` 的 getter 是一個函數，現在變成數值了。於是 tracker 就知道他讀到的數值是錯的，有人正在嘗試騙它。
```javascript
Object.getOwnPropertyDescriptor(Screen.prototype, "width");
// original output: Object { get: width(), set: undefined, enumerable: true, configurable: true }
// modified output: Object { value: 1920, writable: false, enumerable: true, configurable: true }
```

好，那我們就也設定一個 getter 總可以了吧。
```javascript
Object.defineProperty(Screen.prototype, "width", {value: () => 1920});
```

這下 descriptor 是對的了。但如果對函數做 `toString`，會發生什麼事？
```javascript
Object.getOwnPropertyDescriptor(Screen.prototype, "width").get.toString()
// original output: "function width() { [native code] }"
// modified output: "() => 1920"
```

老規矩，我們把 `toString` 覆蓋掉就可以了。
```javascript
Object.getOwnPropertyDescriptor(Screen.prototype, "width").get.toString = () => 'function width() { [native code] }';
```

那，如果我們對 `toString` 取一次 `toString`，會發生什麼事？
```javascript
Object.getOwnPropertyDescriptor(Screen.prototype, "width").get.toString.toString()
// original output: "function toString() { [native code] }"
// modified output: "() => 'function width() { [native code] }'"
```

這是因為 `toString` 會繼承所有 function，而 function 本身又有沒被改過的 `toString`。不如我們把 function 的 `toString` 也改掉吧？
```javascript
const original_toString = Function.prototype.toString;

Function.prototype.toString = function () {
	// 'this' refers to the function whose toString is called

	// toString.toString() case: is the function toString itself?
	if (this === Function.prototype.toString) {
		return 'function toString() { [native code] }';
	}

    // toString case: is the function the getter of screen.width?
    if (this === Object.getOwnPropertyDescriptor(Screen.prototype, "width").get) {
		return 'function width() { [native code] }';
	}

	// otherwise, use the original toString
	return original_toString.call(this);
}
```

這下 `toString` 應該沒問題了齁？
```javascript
Object.getOwnPropertyDescriptor(Screen.prototype, "width").get.toString()
// function width() { [native code] }

Object.getOwnPropertyDescriptor(Screen.prototype, "width").get.toString.toString()
// function toString() { [native code] }

Object.getOwnPropertyDescriptor(Screen.prototype, "width").get.toString.toString.toString()
// function toString() { [native code] }
```

看起來很棒，但我們來看看 `Function.prototype.toString.name` 是什麼吧
```javascript
Function.prototype.toString.name
// original: "toString"
// modified: ""
```

這也好解決，為新的 function 取個名字就好了
```javascript
Function.prototype.toString = function toString() {
// ...
```

可是我們自己 assign 的 function 跟 native function 又會有些 behavior 的不同，像是
```javascript
new Function.prototype.toString();
// original: Uncaught TypeError: Function.prototype.toString is not a constructor
// modified: Uncaught TypeError: Function.prototype.toString called on incompatible object
```

更重要的是：
```javascript
'prototype' in Function.prototype.toString;
// original: false
// modified: true
```

沒救了，魔改 `toString` 這條路走不通，不如換一條路。其實追根究底來說，就是我們用的 getter 是個新建立的函數，然後函數的 `this` 在搞事，所以把 `this` 拿掉就可以了。

```javascript
Object.defineProperty(Screen.prototype, "width", {get: (() => 1920).bind(null)});
```

但在這個頁面把 `screen.width` 改掉了，不代表在 iframe 裡面的其他頁面也會一起被改到。

```javascript
const frame = document.createElement("iframe");
document.body.appendChild(frame);
console.log(screen.width, frame.contentWindow.screen.width);
```

如果要讓 iframe 裡面的頁面被改到，要在 extension 的 manifest 設定 `"all_frames": true, "match_about_blank": true`。需要後者的原因是，當 iframe 沒有指定 location 時，會預設開啟 `about:blank`。

至此我們應該已經可以看出，要在 Javascript 層級去修改一個數值有多困難。所以像是 PriVaricator 等防禦手段是在 browser 層級做的，無法只靠 browser extension 做到。

## 不一致的 OS
有些 anti-fingerprinting 技術會嘗試修改 OS 的資訊，例如仿造 user agent。然而，光是這樣還不夠，有很多管道可以洩漏真正的 OS 資訊，其中不少是我們過去介紹過的，在這邊做個總整理。

### Font
之前討論過 [fonts fingerprinting]({{< relref "/docs/browser-fingerprinting/techniques/fonts-enumeration" >}}) 技術，當時提過有些字型只內建於特定 OS，如果用 Android 就一定有 Roboto，換言之，假設找不到 Roboto，那這大概不是 Android。我們可以用這個特性來偵測 browser 有沒有嘗試騙人：如果一個 browser 宣稱自己是 Android，卻找不到 Roboto，那它很可能其實不是 Android。

### WebGL Report Fingerprinting
WebGL report 有提供 vendor 與 renderer 資訊。如果一個宣稱自己在 Mac 之上的 browser，其 renderer 是 Direct3D，這不太合理，便可知道 browser 在騙人。

### Feature Query
在討論 [Fingerprinting via CSS]({{< relref "/docs/browser-fingerprinting/techniques/fingerprinting-via-css" >}}) 提過可以使用 feature query `@supports` 來檢測 browser 是否支援特定功能。其中有些功能僅限於特定 OS，像是 `-moz-osx-font-smoothing: grayscale` 只能在 macOS 上使用，如果有宣稱自己是 Windows 但 feature query 卻回應該功能可用的，那它很可能其實是 Mac。

### Navigator
除了 user agent，另一個可以取得 OS 資訊的是 Javascript 的 `navigator.platform`。有些 anti-fingerprinting extension 會忘記改這個數值。

## 不一致的 Browser
除了 OS 資訊以外，有些 anti-fingerprinting 技術也會嘗試掩蓋自己的 browser 類型，例如 Chrome 宣稱自己是 Firefox 等，姑且不論這樣修改 user agent 可能會讓一些網站壞掉，其也很難掩蓋到完全不被發現。

### Error Message
不同的 browser 可能會在同個情境噴出不太一樣的 error（無論是 type of error 或 error message），利用這種資訊，便可以猜出 browser 類型。

例如同樣是 `null[0]`，在 Firefox 上的 error message 是 "Uncaught TypeError: null has no properties"，在 Chrome 上是 "Uncaught TypeError: Cannot read properties of null (reading '0')"。

![](/images/error-message-on-different-browsers.png)
（上圖為 Chrome，下圖為 Firefox）

### Function 的 String Representation
文章一開始花了很多篇幅討論 `toString`，這東西叫做 string representation，也就是用字串表達一個 function。有趣的是在不同 browser，同一個 function 可以有不同的 string representation。

例如同樣是 `eval`，Chrome 上的沒有換行，Firefox 上的有換行，這導致了兩者的字串長度不同。

![](/images/string-representation-on-different-browsers.png)
（左圖為 Chrome，右圖為 Firefox）

如果讀者疑惑為何不能透過修改 `toString` 來使其行為一致，可以回頭看看前文來了解魔改 `toString` 有多困難。

### Feature Query
CSS 除了專屬於特定 OS 的規則以外，也有專屬於特定 browser 的規則，可以藉由 feature query 來偵測 browser 是否支援它理應支援的功能。像是 `-moz-perspective: 10px` 只對 Firefox 有效，如果有個 browser 宣稱自己是 Chrome 卻支援它，就很可疑。

### Navigator 的長度與順序
不同的 browser 有長得不太一樣的 `navigator`，甚至同個 browser 不同版本都可能長得不太一樣。差異主要在於有哪些 property 以及 property 排列的順序。可以藉此推測 browser 類型與版本。

![](/images/navigator-object-on-different-browsers.png)
（上圖為 Chrome，下圖為 Firefox）

## Privacy Paradox：失敗的 Anti-Fingerprinting 技術會使 Fingerprinting 更容易
早在 browser fingerprinting 剛被提出的年代，Peter Eckersley 便指出，anti-fingeprinting 技術可能使 fingerprint 更獨特，他將其稱為 the Paradox of Fingerprintable Privacy Enhancing Technologies。例如在那個年代大家都還在用 Flash，且 Flash 可以用來做 tracking（透過 Flash Cookie），但如果把 Flash 阻擋掉，反而會顯得很獨特，提供很大的 surprisal。現在的 anti-fingerprinting 技術大多有把 entropy 的問題考慮進去，正是為了解決這個問題。

不過我這邊想討論的 privacy paradox 在意義上稍微有點不一樣。並非 anti-fingerprinting 本身使 fingerprint 變獨特，而是如果 anti-fingerprinting 失敗，fingerprint 會變得更獨特。

試想文章一開始舉例的 `screen.width`，假設我的螢幕是 1024x768，想偽裝成 1920x1080，如果我的 anti-fingerprinting 成功，tracker 只會拿到 1920，但如果失敗了，tracker 會同時拿到 1024 與 1920。於是，在使用 anti-fingerprinting 之前，tracker 只拿得到一個 data point，使用之後反而拿到兩個了。偽造 user-agent 也可能遇到一樣的問題，從前面諸多對於 OS 與 browser 特性不一致的討論，應該可以看出要猜出真正的 OS 與 browser 並非難事，於是原本從 user-agent 只知道「他的 OS 是 Mac、browser 是 Firefox」，偽造 user-agent 後，可以得出「他的 OS 是 Mac、browser 是 Firefox，但他假裝自己的 OS 是 Windows、browser 是 Chrome」，這更獨特了。又例如加上固定 noise 的 canvas fingerprinting 防禦，如果 noise 被提取出來，反而會變成超級獨特的 fingerprint，甚至比 canvas 本身還獨特，於是原本只有 canvas 可以做 fingerprinting，使用失敗的防禦手段之後有 canvas 與 noise 可以做 fingerprinting。

以上舉例都假設 tracker 有辦法還原出真正的 fingerprint。這看似困難，但相信前面許多討論已經顯示這並非不可能。由此可以看出，使用錯誤的 anti-fingerprinting 方法其實是在為 tracker 增加更多資料點。

就算無法還原出真正的 fingerprint，光是知道 fingerprint 被偽造就是有意義的了，因為嘗試偽造 fingerprint 這件事可能非常獨特。例如，使用 1024x768 的人可能很多，但會去掩蓋螢幕解析度的人很少，於是在掩蓋螢幕解析度的同時，使用者反而提供更多 surprisal 了。甚至在 [fingerprint.js](https://github.com/fingerprintjs/fingerprintjs)（目前市面上最強的 browser fingerprinting library）就直接把是否有偽造 OS、browser、螢幕解析度等當成 fingerprinting 的 feature。

如果讀者之前很疑惑為什麼要一直強調可以如何發現 fingerprint 被偽造，至此應該可以理解「不發現自己在偽造 fingerprint」為何重要了。

## 結語
至此我們結束了對 browser fingerprinting 的討論。其實還有蠻多想分享的東西，但時間實在是不太夠了，還需要留一點篇幅給 privacy-preserving web tracking，所以就先收束在這邊吧。

稍微回憶一下這系列是怎麼開始的。一開始我們說因為 stateful tracking 很容易被發現並砍掉，所以需要 stateless tracking 輔助，而 stateless tracking 裡面最重要的一系列的技術是 browser fingerprinting。接著我們開始討論 browser fingerprinting 的各種技術，並結束在討論其防禦手段。

Browser fingerprinting 無疑是過去十年來 web tracking 領域最熱門的主題，也是最被廣泛使用的技術。為了避免隱私侵害，許多人在研究如何防禦它，也有不少 solution 已經被實際部屬出去，無論是 browser vendor 直接修改 browser 或是其他研究者透過 browser extension 來達成。然而，這些防禦手段至今仍然有未解的難題，也還有許多可以改善的空間。

我們必須收束在此，不然會寫不完 QQ 之後我會把這系列的文章整理好，屆時再把一些值得提及但實在沒篇幅的主題補回去。

從下一篇開始，我們會進入最後一個篇章：Privacy-preserving Web Tracking。過去我們一直在講 web tracking 好壞壞，因為 web tracking 侵害隱私，糟糕透頂，但又不可否認的 web tracking 可以達成很多事情，因此我們需要一個既能追蹤使用者又能保護使用者隱私的技術。可以怎麼做呢？就繼續看下去吧。

## 參考資料
Laperdrix, Pierre, et al. "Browser fingerprinting: A survey." ACM Transactions on the Web (TWEB) 14.2 (2020): 1-33.
Vastel, Antoine, et al. "{Fp-Scanner}: The Privacy Implications of Browser Fingerprint Inconsistencies." 27th USENIX Security Symposium (USENIX Security 18). 2018.
https://adtechmadness.wordpress.com/2019/03/23/javascript-tampering-detection-and-stealth/
https://palant.info/2020/12/10/how-anti-fingerprinting-extensions-tend-to-make-fingerprinting-easier/