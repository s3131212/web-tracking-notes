---
weight: 5
title: "Privacy-preserving Web Tracking"
---

# Web Tracking 可以是 Privacy-preserving 的嗎？
接著，我們要進入最後一個篇章。在討論過 web tracking 的各種技術，並見識到躲避追蹤有多困難之後，我們不禁要問：難道就沒有兼顧隱私的 web tracking 嗎？為什麼我們不得不在許多 Web 的美好功能以及隱私之間擇一呢？所以接下來我們就要來檢視這個問題了。

## 何謂 Privacy-preserving
Privacy-preserving 並沒有一個嚴謹的定義，而是視情況有不同定義。舉例來說：同樣是「Alice 點了 A 廣告然後也點了 B 廣告」，最簡單的定義可能是只要不知道是 Alice，只知道「有個使用者點了 A 廣告然後也點了 B 廣告」；稍微更強烈一點，可能是「有個使用者點了 A 廣告和 B 廣告，但不知道順序」，也可以是「有個使用者點了 A 廣告，他可能也點了 B 廣告，但我不確定」；更進一步，也可以是「這 10 個人都點了 A，那 8 個人都點了 B，其中最多有 4 人兩者都點了，至於 Alice 在哪，我不知道」，甚至「這 10±3 個人都點了 A，那 8±3 個人都點了 B，其中有 2±2 人兩者都點了，至於具體數字是多少、Alice 在哪，我不知道」；如果有多個人參與資料蒐集，還可以是「我知道 Alice 可能點了 A，你知道 Alice 可能點了 B」，這些都可以是 privacy-preserving 的定義。至此我們也可以看出，其定義取決於我們到底想達成什麼目標、有多大的野心、想保護資料到什麼程度。因此在接下來的文章，可以特別注意這些方法是如何定義何謂隱私資料，又提供怎麼樣的保護。

此外，隱私保護也不是越多越好。最極致的隱私保護就是一點資訊都沒有：「我不知道 Alice 是誰，不知道哪個使用者點了什麼廣告，我什麼都不知道」，但這個資料一點幫助都沒有啊。因此，我們必須在隱私（privacy）與有用（usefulness）之間取得一個平衡。


## Privacy-preserving Web Tracking？
Tracking 和 privacy-preserving 聽起來就是衝突的，怎麼可能有 privacy-preserving web tracking？某種意義上來說的確是如此，所以這個名字的確有些誤導。我們想問的問題其實是：對於那些原本透過 web tracking 達成的目標，有可能用 privacy-preserving 的方法達成一樣目標嗎？這些方法可能符合 web tracking 的定義（還是有 identifier 並透過它來連結使用者不同時間的操作），也可能完全跳脫傳統 web tracking 的框架。接下來所介紹的方法，多數都屬於前者，也就只不過是稍微更保護隱私一點的 web tracking 方法，但有些其實算是後者，它本質上不再是種 web tracking 了。

細心的讀者可能已經發現了，此章和過去兩章所著重的面向其實很不一樣。過去我們花了很多篇幅講各種 web tracking 的技術，而不討論 web tracking 本身到底是為了什麼，只有在第一章快速帶過。或是更白話來說，當我們在一開始說了「廣告商透過具隱私侵害性的 web tracking 技術追蹤使用者的廣告點擊」後，就一直討論所以何謂具隱私侵害性的 web tracking 技術，又可以如何反制這些技術，卻不討論所以廣告商拿到這些資料之後他到底能幹麻。接下來我們要討論的正是，廣告商拿到這些資料之後想要做什麼，有沒有可能其實不使用一些具隱私侵害性的技術也可以達到一樣的效果。

## 有哪些問題需要解決？
Web tracking 目前於廣告產業最主要的運用是廣告的個人化投放與聯盟行銷，也就是需要知道使用者喜歡看怎樣的內容，並偵測他們點了哪些廣告，到了目標位置之後又做了哪些操作。在 privacy-preserving web tracking 中，我們同樣想達成這些目標。個人化廣告投放的技術包含 Google 的 FLoC 與 Topics API 等，追蹤使用者的廣告點擊，則有 Mozilla 與 Meta 的 Interoperable Private Attribution 、Apple 的 Private Click Measurement、Google 的 Attribution Reporting API 等。其他 web tracking 的運用還包含 web analytic 等，這些也都有相關的 privacy-preserving 的技術，但因為篇幅因素，可能無法介紹到他們了。

## 為什麼需要 Privacy-preserving Web Tracking
可能有些讀者會很疑惑，為什麼我們需要 privacy-preserving web tracking？何不直接消滅 web tracking？我認為 web tracking 的許多目的本身其實是合理的。例如，我們想要有免費的服務，總得有人來幫我們付錢，這時廣告商為我們付錢的同時想要最大化廣告投遞效益並追蹤成效，是個合理的要求。問題不在於他們想最佳化投遞並追蹤成效，而在於他們達成這些目的所使用的手段對使用者的隱私侵害過於巨大。也許我們會期待網路世界可以有個新的商業模式而不是完全靠廣告產業撐起，也許我們希望可以完全終結監控資本主義。然而，在那之前，我們需要有方式與廣告產業共存，privacy-preserving web tracking 便是要這樣的目標。我們不必也可能無法消滅 web tracking，但至少我們可以使 web tracking 的目標得以達成的同時保護自身隱私，更甚至，在達成隱私保護的前提下，仍能享受 web tracking 所能帶來的益處。

## 瀏覽器的隱私革命
至此應該有個問題，看起來只有一群在乎隱私的倡議者以及很想發論文的學者會對這種東西感興趣吧。的確，最早對於 privacy-preserving web tracking 的討論幾乎都來自學界，甚至早在二十多年前便有相關嘗試，這些發展雖然沒有直接影響瀏覽器的設計，但也為未來的隱私保護技術發展埋下種子與準備好養分。如今這些種子總算開始發芽了。大概從 2018 年左右開始，越來越多主流瀏覽器也開始導入各式各樣的 privacy-preserving web tracking，其中最浩大的（也最有爭議性的）嘗試便是 Google 的 Privacy Sandbox。所以發生什麼事了？一言以蔽之，大家都在嘗試為了 third-party cookie 被消滅後的 web tracking 世界鋪路。

有很長一段時間，網路和Web的設計從未考慮過安全或隱私的問題。但這十多年（或更多）來，安全性的重視已經有顯著的提升，越來越多產品設計是 security by design，在設計初期便把安全性納入考量。隱私在這幾年也開始受到關注，雖然整體發展仍然落後於安全，但距離正在縮小。更重要的是，改善 Web 隱私的進程有 Google、Apple 等大公司的積極參與，這些大公司無疑地在科技發展上佔有舉足輕重的地位，尤其這些公司自己就是 Web 產業最大的利益關係人，無論是依賴廣告產業或是自己有開發瀏覽器，這使得這幾年來的隱私發展得以付諸實現，與過去單純的理論研究有著天壤之別。

在改善瀏覽器隱私的路途中，最大的魔王無疑是 third-party cookie。基本上整個瀏覽器的隱私革命，最大的戰場就是如何封鎖 third-party cookie 的同時不破壞現有生態系。其他戰場還有如何防制各種不同型態的 web tracking，例如幾乎所有瀏覽器都有自己的 bounce tracker 防禦方法，無論已經付諸實現或還是草案。

以 Google 與 Apple 兩家科技公司為例。多年來 Apple 一直提出新的Web API 草案，例如評估廣告成效的 Private Click Measurement 或是控制未加區隔的儲存空間存取的 Storage Access API，Apple 也有提出 Intelligent Tracking Prevention 等防範 web tracking 的機制，藉此限制資料流動與隱私洩漏。Google 的 Privacy Sandbox 則更宏偉且加引人注目，它幾乎根本改變了 Web 的運作，以全新的方法限制與改變資訊流動與利用的方式。Privacy Sandbox 整個計畫從 partitioned cookie（之前提過的 CHIPS）、追蹤使用者興趣（Topics API 與 FLoC）、衡量數位廣告成效（Attribution Reporting API）、詐欺防制（Private State Tokens API）、browser fingerprinting 防護（之前提過的 Privacy Budget）都有所涉及。然而它也伴隨著諸多爭議，並且已經推持數次，目前預計在 2024 年實施。

這場由瀏覽器公司帶頭的隱私革命，也有幾個有趣的特徵。

首先，同樣一個功能往往有多個同實在臺面上的技術，背後很可能又都是不同公司。舉例來說，同樣是衡量數位廣告成效，有 Google 的 Attribution Reporting API、Apple 的 Private Click Measurement、Meta 與 Mozilla 的 Interoperable Private Attribution，又很顯然的 Google 開發的 Chrome 只支援 Attribution Reporting API，Apple 開發的 Safari 只支援 Private Click Measurement，至於 Interoperable Private Attribution，目前還在草案，但短期內大概也只有 Mozilla 開發的 Firefox 有機會實作。每個公司有各自的技術，使得 privacy-preserving web tracking 基本上是被割據的狀態，又由幾乎處於壟斷地位的 Google 佔上風。

但這也不代表 Google 可以推動任意的變革。例如在2020年，Google取消了PIGIN提案，並推出了Turtledove 取代之，之後 Google 又在 2022 年撤回備受爭議的 FLoC，以 Topics API 取代之，同年 Google 也撤回了SameParty cookies。這些技術提案的變化反映出隱私議題現在已經成為 Web 生態討論不可忽視的一環，而且網路標準的討論也不再是只屬於傳統的那些大型科技公司，還有眾多利益關係人參與其中。這也意味著我們很難預料接下來 privacy-preserving web tracking 的世界會長怎樣。

與此同時，傳統的資料監管機構似乎已經不再主導這些關於隱私保護技術的辯論，而是讓市場上的各種利益關係人以及來自學界的聲音做討論。不過雖然資料監管機構並未介入，反壟斷調查倒是很關心 Google 的 Privacy Sandbox，畢竟 Chrome 是擁有最高市佔率的公司，任何風吹草動都足以影響整個產業。另外，目前的隱私法規應該如何跟上這場瀏覽器的隱私革命，例如 cookie 是否還有過去所想像的這麼有害，如何面對新穎 web tracking 方式的崛起等，也是個尚待解決的問題。

總結來說，過去十多年來在隱私保護界的諸多討論與技術，終於在這場 2018 年左右開花結果了，各大瀏覽器開始開發眾多新功能與重新設計，以更進一步保護使用者的隱私。然而如今可能已經有不下 20 個草案被提出，最終哪些會獲勝，得到市場青睞，則還不得而知。更重要也更令人憂心的是，我們還不知道這場革命最後會真的創造一個充分保護隱私的網路生態，或是成為再一個無疾而終的行銷宣傳。

在接下來的章節，我將介紹數個有趣的、值得一提的 privacy-preserving web tracking，有不少已經實作並被使用，也有一些還在草案階段。也必須提醒讀者，如同前面一再提到，這個領域的發展實在太過快速，也許當讀者翻開這本書時，有些資訊已經過時了，故請務必再次查證。