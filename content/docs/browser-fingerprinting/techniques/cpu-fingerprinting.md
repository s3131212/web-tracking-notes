---
weight: 9
title: "CPU Fingerprinting"
---

# CPU Fingerprinting
如果前面兩篇 browser extension 的 fingerprinting 還沒有嚇倒你的話，從這一篇開始，我們會進入一個很瘋狂的狀態，一種「天啊這樣也行喔？」的狀態。

那，首先就從 CPU fingerprinting 開始講起吧。這篇文章會用到一些計算機結構的基礎知識，都是非常簡單的東西，所以只要是我在修計算機結構時聽過的詞彙，就不會多加解釋，畢竟我金魚腦，這麼多年前修課而且現在還記得，應該就代表多數有修過計結的都會記得... 吧？

## 何謂 CPU Fingerprinting
雖然說是 CPU fingerprinting，但我們不是要去 fingerprint 每個 CPU 之間的些微差異，而是要去識別一個 CPU 的重要特徵，像是幾個 core、cache 多大等，藉此作為 fingerprint 的一部分。

CPU fingerprint 除了拿來做 web tracking 以外，也有其他用途，像是情蒐之類的，例如這幾年很多基於 speculative execution 和 out-of-order execution 的各種 side channel attack，如果可以了解對象所使用的 CPU，便可以發動攻擊。

不過 CPU fingerprint 之所以說是瘋狂，就在於它很難！當代的 browser 基本上都有很嚴密的 sandbox，可以執行的指令很受限，盡可能避免 browser 上執行的任何程式可以獲得過多的資訊。基本上在 browser 上能做的也就只有 Javascript 與 WebAssembly，頂多運用 Just-in-time compiler 的特徵來做更多事情，但仍非常受限。此外，如果要在 browser 上做針對 CPU 的 timing attack 或其他 side-channel attack，會受到許多影響，畢竟 browser 自己就有非常受限的 scheduling 的機制、OS 再一層 scheduling，再加上各種 noise，許多有用的資訊都被掩埋在雜訊中了。所以，要在 browser 上執行一些程式來洩漏 CPU 資訊沒有那麼 trivial。


## CPU Fingerprinting 技術簡介
接下來我們將開始介紹幾個可以 fingerprint CPU 的技巧。

### CPU Core 數量
CPU Core 數量理應可以直接用 `navigator.hardwareConcurrency` 取得，不過問題沒這麼簡單。當代的 CPU 多半會透過一些方式讓一個 core 可以偽裝的很多個 core，例如常常聽到有人介紹一個 CPU 是 `4C8T`，便意味著這個 CPU 其實只有四個 core 但偽裝成八個 core。我們稱真正的、物理的 core 為 physical processor cores，偽裝出來的為 logical processor cores，一個 4-core CPU 可能有八個 logical processor cores，也就是可以同時跑八個 thread 而不需要做 context switch。`navigator.hardwareConcurrency` 回報的是 logical processor cores 的數量。比較麻煩的是，這個數字有時會騙人，例如當使用者想要避免 browser 佔用太多 CPU 時可能就會故意低報 logical processor cores 的數量，藉此限制 Worker 數量。

等等，Worker 是什麼？眾所皆知的，Javascript 是個 single thread 的語言，之所以感覺很多事情一起做，只是 event loop（Javascript 的 concurrency model）在跑而已，並非真的使用多個 thread。但總有情境是真的需要 multi-thread，Worker 便因此而生。反正重點是我們可以用 Worker 作到 multi-thread。

一個簡單但不那麼完美的作法是，一開始先 spawn 一堆 worker，把 CPU 佔滿，於是為了要做 multithreading，不得不讓一些 worker 先躺回去 idle state 排隊等待，大家輪流使用有限的 thread。於是，如果我們生出來的 worker 數量少於 CPU logical processor core 的數量，理論上 worker 應該跑得差不多快，但如果生出來的 worker 太多，超過 CPU logical processor core 的數量了，那因為大家要輪流跑，worker 的完成時間就會不太一樣，而且會變長。此外，因為 logical processor core 終究是偽裝出來的，一個 physical processor core 跑一個或兩個 logical processor core 時的執行速度不一樣，所以一樣可以藉由執行時間觀察出 physical processor core 的數量。

不過因為出來的數字很亂，所以還需要做很多後續處理，這就不是這篇文章可以涵蓋到的部份了。

### Cache Size
不同的 CPU 會有不同的 cache 大小，所以藉由偵測 cache 大小一樣可以來 fingerprint CPU。舉例來說，通常 ARM 只會有兩層 cache（L1, L2），而 x86 有三層 cache（L1, L2, L3），尤其 L3 大小因 CPU 而有很大的差異。

正如過去針對 cache 的攻擊八九不離十是 timing attack，我們現在要做的也是去偵測 cache 存取速度來判斷其大小。若要偵測 L1 cache 大小，可以嘗試不斷寫資料，同時回頭讀以前的資料，看相隔多遠時速度開始變慢了，便知道 L1 cache 有多大。同樣的，當 L1 與 L2 都滿了的時候，速度會再次變慢，當 L3 也滿了，就也會再次變慢，如果超過合理大小時都沒有變慢，大概就是沒有 L3 cache。

Trampert 等人提出了一個有趣的方法是：先隨機生成大小為 k 的 circular linked list，從 head 開始，接著往前走 N 個 node（N 是常數，N 會遠大於 circular linked list 的 node 數量），看 k 多大時存取時間開始變長了。這個方法之所以有效，在於其迫使這中間每個 node 都一定要被存取，所以這些 node 不得不被放入 cache。如果所有 node 可以被塞進 L1 cache，理應走 N 個 node 都會一直是 cache hit（撇除第一次讀取），但如果塞不進 L1 cache，那路途中會一直遇到 cache miss，導致執行時間變長。於是同樣是 N 步，k < cache size 時跑很快，k > cache size 時跑很慢，就可以藉此偵測 cache size。

### Cache Associative
上述的演算法除了偵測 cache size 以外，其實也可以偵測 (L1 data-)cache associative。如果我們讓每個 node 之間都相隔一個 cache-size，那在寫入 cache 時就會寫到一樣的地方（因為 mod 結果一樣），於是，一開始往前走時會一直 cache hit，因為 cache 中該 set 還沒有滿，然而一旦超過 associativity，該 set 也就滿了，會 flush 掉東西，開始發生 cache miss，導致時間增加。藉由 k 為多少時執行時間開始變長，便可推論 cache associative。

### TLB Size
又是同個概念，用上述演算法，但每個 node 相隔一個 page-size 以上，當 linked list 所使用的 page 數量小於 TLB 大小，則 TLB 沒有被塞滿，應該會一直 cache hit，但如果 linked list 用太多個 page，TLB 滿了，就會開始出現 cache miss 拉長執行時間。藉此便可知道 TLB 大小。

### Page Size
基本上也就一樣是不斷去戳固定間隔（通常是 256B）的 address，藉由執行時間就可以知道有沒有發生 page fault。

### AES 加速
早期許多高階 CPU 會有內建的 AES 加速（例如 Intel 的 AES-NI），於是可以利用 AES 的執行速度判斷有沒有硬體加入。嚴格來說，可以藉由比較 AES 速度與一個已知的計算（例如 Saito 等人用 Monte Carlo 算圓周率來比較）的速度比例來偵測，對沒有 AES 硬體加入的 CPU 來說，CPU 越快，做已知計算與 AES 的速度應該會一起變快，所以比值差不多，但如果一個 CPU 有 AES 硬體加速，則 AES（相較於已知計算）會異常地快，比值變超大，因此可以藉由兩個計算的時間比值判斷有沒有硬體加速。

但因為當今幾乎所有 CPU 都內建 AES 加速，所以這偵測已經沒什麼用了，反正大家都有，沒啥 entropy 可言。

### Benchmarking
顧名思義，直接執行一段 code 看要跑多久，來偵測 CPU 的效能。因為 multi-core 的效能比較難在瀏覽器上偵測，所以通常會用 single-core 的效能來判斷。

此外，如果有使用一些 dynamic frequency scaling 的功能（像是 Intel Turbo Boost 或 AMD PowerTune），會因為不同 task 透過 dynamic frequency scaling 得到的優勢不同，於是有沒有 dynamic frequency scaling 會讓 benchmark 的結果分佈不太一樣，於是就可以藉此看出一個 CPU 是否有使用 dynamic frequency scaling 技術。

## 如何有個精準的 Timer
前面的各種討論其實都有個隱含的前提：有個精準的 timer。但這其實不太簡單，Javascript 內建的 timer（例如傳統的 Date API 與新的 performance API）多半難以提供足夠準確的 timer，Date API 基於 Unix timestamp 本來就很受限，Performance API 為了避免用於攻擊，刻意降低精準度，可能只有 microsecond 級別，不足以拿來做 cache-timing attack。

一個意料之外但非常準確的 timer 是 Worker 的 `SharedArrayBuffer`。它原先是用於 thread 之間的 memory sharing，也就可以讓 main thread 和 worker thread 可以共享資料。但如果在 worker 裡面跑個無限迴圈去遞增 `SharedArrayBuffer` 裡面的一個數值，然後再從 main thread 去讀該記憶體空間，就可以透過數值大小來推測執行時間。於是這樣我們就有個很精準的 timer 了。

更多資訊可以參考 [XS-Leaks Wiki](https://xsleaks.dev/docs/attacks/timing-attacks/clocks/) 上的討論。

## 防禦與討論
很遺憾的，就我所知目前並沒有防禦 CPU fingerprinting 的手段。雖然移除精準的 timer 可能有些幫助，但要避免出現意料之外的 timer 也沒有這麼容易（看看 `SharedArrayBuffer`）。

不過換個角度想，CPU fingerprinting 真的這麼有效嗎？對於要做攻擊的人來說，知道一個 CPU 可能受到 Spectre 影響，是很有價值的資訊，但對於做 web tracking 的人而言，有數百萬計的 CPU 都是一樣的參數啊！CPU core 數量大概也就是那些，L1 TLB 通常是 64 個 4KB 的 page，cache associative 通常都是 8，幾乎所有人都有 AES 硬體加入與 dynamic frequency scaling，知道這些資訊能貢獻的 entropy 其實很少。（有趣的是 Apple Silicon 系列的數字通常與眾不同）

所以對於 CPU fingerprinting 也不用過於憂心，雖然技術有趣，但其所貢獻的 entropy 其實不高。

## Reference
Trampert, Leon, Christian Rossow, and Michael Schwarz. "Browser-based CPU Fingerprinting." _European Symposium on Research in Computer Security_. Springer, Cham, 2022.
Saito, Takamichi, et al. "Estimating CPU features by browser fingerprinting." 2016 10th International Conference on Innovative Mobile and Internet Services in Ubiquitous Computing (IMIS). IEEE, 2016.
https://xsleaks.dev/docs/attacks/timing-attacks/clocks/