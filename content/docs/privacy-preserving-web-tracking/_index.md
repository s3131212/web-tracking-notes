---
weight: 5
title: "Privacy-preserving Web Tracking"
---

# Web Tracking 可以是 Privacy-preserving 的嗎？
終於，我們要在最後四天進入一個新的篇章了。在討論過 web tracking 的各種技術，並見識到躲避追蹤有多困難之後，我們不禁要問：難道就沒有兼顧隱私的 web tracking 嗎？所以接下來我們就要來檢視這個問題了。

## 何謂 Privacy-preserving
Privacy-preserving 並沒有一個嚴謹的定義，而是視情況有不同定義。舉例來說，同樣是「Alice 點了 A 廣告然後也點了 B 廣告」，最簡單的定義可能是只要不知道是 Alice，只知道「有個使用者點了 A 廣告然後也點了 B 廣告」，稍微更強烈一點，可能是「有個使用者點了 A 廣告和 B 廣告，但不知道順序」，也可以是「有個使用者點了 A 廣告，他可能也點了 B 廣告，但我不確定」，更進一步，也可以是「這 10 個人都點了 A，那 8 個人都點了 B，其中最多有 4 人兩者都點了，至於 Alice 在哪，我不知道」，甚至「這 10±3 個人都點了 A，那 8±3 個人都點了 B，其中有 2±2 人兩者都點了，至於具體數字是多少、Alice 在哪，我不知道」，如果有多個人參與資料蒐集，還可以是「我知道 Alice 可能點了 A，你知道 Alice 可能點了 B」，這些都可以是 privacy-preserving 的定義。至此我們也可以看出，其定義取決於我們到底想達成什麼目標、有多大的野心、想保護資料到什麼程度。因此在接下來的文章，會需要特別注意這些方法是如何定義何謂隱私資料，又提供怎麼樣的保護。

此外，隱私保護也不是越多越好。最極致的隱私保護就是一點資訊都沒有：「我不知道 Alice 是誰，不知道哪個使用者點了什麼廣告，我什麼都不知道」，但這個資料一點幫助都沒有啊。因此，我們必須在隱私（privacy）與有用（usefulness）之間取得一個平衡。

## Privacy-preserving Web Tracking？
Tracking 和 privacy-preserving 聽起來就是衝突的，怎麼可能有 privacy-preserving web tracking？某種意義上來說的確是如此，所以我覺得這個名字其實不太好。我們想問的問題其實是：對於那些原本透過 web tracking 達成的目標，有可能用 privacy-preserving 的方法達成一樣目標嗎？這些方法可能符合 web tracking 的定義（還是有 identifier 並透過它來連結使用者不同時間的操作），也可能完全跳脫傳統 web tracking 的框架，根本不算是種 tracking 了。多數都是前者，但仍有些其實算是後者。

某種意義上，此章的文章和這系列的其他文章在敘事邏輯上是斷裂的。過去我們花了很多篇幅講各種 web tracking 的技術，而不討論 web tracking 本身到底是為了什麼，只有在介紹 [web tracking 的用途的文章]({{< relref "/docs/usages-and-categories" >}})快速帶過。或是更白話來說，當我說「廣告商透過具隱私侵害性的 web tracking 技術追蹤使用者的廣告點擊」後，就一直討論所以何謂「具隱私侵害性的 web tracking 技術」，又可以如何反制這些技術，卻不討論所以廣告商拿到這些資料之後他到底能幹麻。接下來我們要討論的正是，廣告商拿到這些資料之後想要做什麼，有沒有可能其實不使用一些具隱私侵害性的技術也可以達到一樣的效果。

因此就這種意義上，privacy-preserving web tracking 這個名字其實有點誤導性，可惜我暫時不知道更好的名字。也許 privacy enhancing technology 還不錯？

## 有哪些問題需要解決？
Web tracking 目前於廣告產業最主要的運用是廣告的個人化投放與聯盟行銷，也就是需要知道使用者喜歡看怎樣的內容，並偵測他們點了哪些廣告，到了目標位置之後又做了哪些操作。如果想要對這方面有更多認識，可以回頭看介紹 [web tracking 的用途的文章]({{< relref "/docs/usages-and-categories" >}})。在 privacy-preserving web tracking 中，我們同樣想達成這些目標。個人化廣告投放的技術包含 Google 的 FLoC 與 Topics API 等，追蹤使用者的廣告點擊，則有 Mozilla 與 Meta 的 Interoperable Private Attribution 、Apple 的 Private Click Measurement、Google 的 Attribution Reporting API 等。我們會從中挑選幾個來介紹

其他 web tracking 的運用還包含 web analytic 等，這些也都有相關的 privacy-preserving 的技術，但因為篇幅因素，可能無法介紹到他們了。

## 為什麼需要 Privacy-preserving Web Tracking
可能有些讀者會很疑惑，為什麼我們需要 privacy-preserving web tracking，何不直接消滅 web tracking？我自己的觀點是，web tracking 的許多目的本身其實是合理的。例如，我們想要有免費的服務，總得有人來幫我們付錢，這時廣告商為我們付錢的同時想要最大化廣告投遞效益並追蹤成效，是個合理的要求。問題不在於他們想最佳化投遞並追蹤成效，而在於他們達成這些目的所使用的手段對使用者的隱私侵害過於巨大。當然直接把廣告砍光光可以解決隱私問題，但這對 web 生態長遠來看不見得是好的。我們需要的不是廣告消失（雖然特別惱人的廣告還是消失比較好），而是投遞廣告時不要侵害我們的隱私，privacy-preserving web tracking 便是要這樣的目標。我們不必也可能無法消滅 web tracking，但至少我們可以使 web tracking 的目標得以達成，同時保護自身隱私。

另外值得注意的是，其實也有不少人主張光是想最大化成效這件事情就已經是有問題的了，但我暫時不想在這邊陷入這樣的倫理爭論，或是提出某種對於 Web 生態系的全新想像，只是想要指出上面的論述充滿了需要被更進一步討論的斷言。

今天的文章就先結束在這邊吧，明天我們會從 FLoC 開始談起，討論各種 privacy-preserving web tracking 技術。希望我有時間把文章寫完。