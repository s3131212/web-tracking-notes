---
weight: 5
title: "WebRTC Fingerprinting + Media Device Fingerprinting"
---

# WebRTC Fingerprinting + Media Device Fingerprinting
今天要來介紹兩個與 WebRTC 有關的 browser fingerprinting 技術。WebRTC fingerprinting 利用 WebRTC 的設計，有可能洩漏使用者的 public 與 local IP address，即便有 VPN 一樣會洩漏。media device fingerprinting 則是利用了 browser 可以取得攝影機、麥克風等多媒體設備的特性，以此做 fingerprinting。

## WebRTC
WebRTC (Web Real-Time Communication) 是一個允許網頁取得影音媒體、串流影音（或任意資料）的 API。最常見的應用是線上會議軟體，網頁擷取了攝影機與麥克風的輸入，並將其串流出去。另外一個有趣的應用是在瀏覽器上做 P2P（peer-to-peer）的檔案傳輸，例如 [ShareDrop](https://github.com/szimek/sharedrop)。WebRTC 的一大特色是，這是一個 P2P 的傳輸協定，也就是資訊串流不經過伺服器，而是直接在兩個瀏覽器之間互相傳輸。這性質很重要，等等會用到。

## WebRTC Fingerprinting
前面提過 WebRTC 使用 P2P 通訊，為了找到兩端之間的 network path，勢必得知道對方的位置。可是如果 client 躲在 NAT 裡面，就拿不到 public address，也就無法告訴對方自己在哪邊，因而無法建立 WebRTC connection。為了解決此問題，WebRTC 使用了一個名為 STUN（Session Traversal Utilities for NAT）的通訊協定，藉由此協定，client 可以獲得自己的 public address 與（已經打開來的）port，接著透過 WebRTC 共享給對方，並且，以此建立 P2P connection。此外，為了讓內網之間的兩端在建立連線時，可以不用經過外網繞一整圈，WebRTC 也會分享 local IP。這個描述大幅度地簡化了 STUN 的運作方式，甚至其實 STUN 也有很多不同的實作，但總之重點是透過 STUN server，WebRTC 可以拿到 local 與 public address，而且因為其運作方式，有可能繞過 VPN，即便躲在 VPN 後面一樣可以拿到真實的（ISP 的）public address。更完整的討論可以看這篇文章：[真实世界中的WebRTC：STUN, TURN and signaling](https://michaelyou.github.io/2018/08/01/%E7%9C%9F%E5%AE%9E%E4%B8%96%E7%95%8C%E4%B8%AD%E7%9A%84WebRTC/)。

因此，理論上，在建立 WebRTC connection 時，browser 可以獲得 public 與 local IP address。這便可以作為 fingerprinting 的來源。

常見的實作大概會長[這樣](https://ourcodeworld.com/articles/read/257/how-to-get-the-client-ip-address-with-javascript-only)：
```javascript
let pc = new RTCPeerConnection({
	iceServers: []
});

//create a bogus data channel
pc.createDataChannel("");

// create offer and set local description
pc.createOffer(function(sdp) {
	sdp.sdp.split('\n').forEach(function(line) {
		if (line.indexOf('candidate') < 0) return;
		line.match(ipRegex).forEach(console.log);
	});

	pc.setLocalDescription(sdp, () => {}, () => {});
}, () => {});

//listen for candidate events
pc.onicecandidate = function(ice) {
	console.log(ice?.candidate?.candidate);
};
```

例如，browser 可能會收到這樣的東西
```
candidate:4234997325 1 udp 2043278322 192.168.0.56 44323 typ host
```

如果真的執行了這段 code，會發現你沒有收到 local IP。因為這個洞被修掉了，賀！

### 防禦：mDNS
接下來我想討論一下 WebRTC 洩漏 local IP 的防禦：mDNS（Multicast DNS）。如果說 DNS 是想讓整個 global 的網路上可以有個 domain name 與 public IP address 的 mapping ，那 mDNS 的目標就是讓 local 的網路上也可以有個 hostname 與 local IP address 的 mapping，除此之外，我們還希望不要有個集中管理的設施（像是 DNS server），不然維護成本太高了。舉例來說，現在如果要存取內網的 Chromecast，我可能要背起來它的 IP 是 `192.168.1.123`，但如果有 mDNS，我可以將其命名為 `chromecast.local`，以後用這個名字就可以存取 Chromecast 了。

mDNS 的運作模式正如其名，使用了 multicast 的方法。當有台電腦想知道 `chromecast.local` 是誰時，會直接傳訊息給 multicast address，其他設備收到封包後就檢查這是不是要找自己的，而 Chromecast 收到之後發現這是在找它，所以就再 multicast 一次公告自己的 local IP address。透過這種方法，不需要有個 DNS server，也可以讓所有設備透過 hostname 找到彼此。

藉由 mDNS，WebRTC 不再需要知道自己的 local IP address，只需要知道 mDNS 的 hostname 即可。目前主流實作是，每次都會生成新的 hostname，確保不會被 track：
```
candidate:2573043965 1 udp 2113937151 53b8d70d-02af-409d-a584-761589541922.local 51605 typ host generation 0 ufrag DFSx network-cost 999
```

因此透過 local IP address 追蹤已經不可行了。

至於 public IP address 洩漏。在沒有 VPN 的情況下不是個問題，因為追蹤者本來就拿得到 public IP address。在有 VPN 的情況下，重點在於要讓瀏覽器使用系統預設的 public interface，此問題就我所知應該也已經被修得差不多了。



## Media Device Fingerprinting
另一個與 WebRTC 相關的 fingerprinting 技術是 media device fingerprinting。試想，在許多網頁會議軟體（e.g. Google Meet）上，都可以調整輸入與輸出設備，假設我有插耳機，就可以選擇用耳機當作聲音輸出，電腦內建的麥克風作為聲音輸入之類的。通常這種功能是利用 [MediaDevices API](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices) 達成的，此 API 可以列舉所有輸入輸出設備，然後選取特定設備作為訊號來源或是目的地。

例如，若要列出所有設備，可以這麼做：
```javascript
navigator.mediaDevices.enumerateDevices()
.then((devices) => {
	devices.forEach((device) => {
		console.log(`${device.kind}: ${device.label ? device.label : "no label"}, id=${device.deviceId}`);
	});
})
```

其中 `kind` 是輸入設備的類型，`label` 是裝置的名字，`deviceId` 是每個設備有獨一無二的 ID，等等會有更多討論。

不過，如果真的執行上述的 code，讀者可能會發現拿不到 label，如圖所示。

![](/images/mediadevice-no-labels.png)

這是因為只有當取得使用 media device 的權限（例如取得讀取麥克風的權限）時，才可以看到 label。
![](/images/mediadevice-labels.png)

相信讀者已經想到了，既然不同電腦有不同的 media devices，那就拿來做 fingerprinting 啊！這就是 media device fingerprinting 的概念：利用每個使用者都可能有不一樣的 media devices，蒐集所有 media device 的 ID 或 label，便可以生成可能是獨一無二的 fingerprint。

為了避免 media device 列表被用於 fingerprinting，標準制定者做了一些努力：

首先，device ID 並不是固定的。在不同 origin 下會有不同的 ID，例如上面兩張圖就有不一樣的 ID。清除 cookie 會促使 device ID 重新生成，無痕瀏覽下每個 session 也都會有自己的 device ID，並於 session 結束時清空。因此 device ID 最多用於 same-site tracking，無法用於 cross-site tracking，而且存活長度最多就跟 cookie 一樣長而已。這裡運用的邏輯是：反正 cookie 都可以活這麼久了，讓 device ID 活一樣久其實 nothing to lose，追蹤者大可直接用 cookie 追蹤來獲得一樣的效果。不過需要注意的是，device ID 只有透過 browser 內建的方法清除 cookie 時會順便被 reset，如果用其他 anti-tracking extension 移除 cookie，device ID 會被保留。

不過，就算 device ID 不固定，裝置名稱（`label`）還是固定的啊！？於是，就如同剛剛提到的，若要取得裝置名稱，需要獲得存取權限，也就是會直接跳出一個框框問使用者是否同意網站存取麥克風或 webcam。這使得在背後偷偷做 fingerprinting 變得不太可能。但是，如果網站本來就需要麥克風與 webcam 的權限（例如一再出現的線上會議軟體），便還是可以偷偷做 fingerprinting。因此 media device fingerprinting 雖然罕見且有門檻，但仍然是可行的。


WebRTC 相關的 fingerprinting 技術雖然在早期很盛行，但由於多數的洞要不被修補了，要不 exploit 的門檻變高了。不過即便如此，這些 fingerprinting 技術貢獻的 entropy 與其穩定性仍然是有用的，因此至今都還是能看到有人在用這些技術。


## 參考資料
[Fingerprinting WebRTC](https://privacycheck.sec.lrz.de/active/fp_wrtc/fp_webrtc.html#fpWebRTC)
[WebRTC Leak Test](https://browserleaks.com/webrtc)
[真实世界中的WebRTC：STUN, TURN and signaling](https://michaelyou.github.io/2018/08/01/%E7%9C%9F%E5%AE%9E%E4%B8%96%E7%95%8C%E4%B8%AD%E7%9A%84WebRTC/)
[PSA: Private IP addresses exposed by WebRTC changing to mDNS hostnames](https://groups.google.com/g/discuss-webrtc/c/6stQXi72BEU)
[PSA: mDNS and .local ICE candidates are coming](https://bloggeek.me/psa-mdns-and-local-ice-candidates-are-coming/)