---
weight: 3
title: "Unified ID 2.0"
---

# Unified ID 2.0
之前介紹的幾個技術都是嘗試在不需要識別出特定使用者的情況下推測使用者興趣（例如 FLoC 或 Topics API）或完成 conversion measurement（例如 PCM 與 IPA）。這些技術的目標都是想要用隱私保護的方式達成以前需要 web tracking 才能達成的功能。但此章要介紹的 Unified ID 2.0 採取了一條完全不同的路徑，他們認為傳統 third-party cookie 追蹤的問題在於這些追蹤往往並未獲得使用者同意，即便有同意，使用者也難以管理繁雜的隱私設定，因此新世代的 web tracking 該達成的是只有在使用者同意下可以進行追蹤，而且必須提供使用者足夠簡單且統一的方式管理他們的隱私設定。

Unified ID 2.0 (UID2) 是一個開源框架，可供網站、廣告商和數位廣告平台使用，以建立身分識別。隨著Google宣布計劃在2022年在 Chrome 上封鎖 third-party cookie，The Trade Desk迅速開啟了Unified ID 2.0的開發，嘗試在沒有 third-party cookie 的世界繼續達成 cross-site tracking。與傳統 third-party cookie 不同的是，Unified ID 2.0 將要求使用者在網站管理者建立 UID2  identifier 之前，需要明確地同意網站的隱私權政策（或其他條款），並提供其 email address。此外，會有一個集中化的管理單位負責 key management，而且使用者可以直接在這個單位上登記希望退出（opt-out）追蹤，已達成更友善的隱私管理。

## Unified ID 2.0 如何運作？
UID2 的角色除了傳統廣告產業就有的那些以外，還額外有 UID2 administrator 與 UID2 operator。UID2 administrator  是一個集中化的服務，負責管理 UID2 ecosystem。他們的工作包括 key management，即產生與發布 key 給 UID2 operator 與 DSP。此外，UID2 administrator  還需要處理使用者的退出要求，並通知 UID2 operator 和 DSP。UID2 運營商則是負責生成和管理 UID2 及 UID2 token 所需基礎設施的組織。UID2 operator 從 UID2 administrator 那裡拿到 key 與 salt 。網站發布者可以可以通過 UID2 operator 的 API 請求 UID2 或 UID2 token。

具體來說，整個運作流程大致如下：

![](/images/Unified-ID-2.0.jpg)

當使用者訪問一個部屬 UID2 的網站時，發佈商會向使用者解釋使用者為何需要允許追蹤、如此可以得到什麼（例如此時可以要求使用者同意追蹤以換取免費存取網站內容，或是付費存取網站內容），並要求使用者提供 email address。網站接著會把這個 email address 傳送給 UID2 operator，並在 UID2 operator 經過加鹽和 hash 過後，轉換成一個唯一的 identifier 並回傳給網站，網站會將其儲存在 first-party cookie 或是其他儲存空間中，我們稱之為這個唯一的 identifier 為 UID2。於是以後使用者再次造訪時，這個網站都可以從其 cookie 中得知使用者身分。

UID2 是廣告商會實際看到與用於追蹤與廣告投放的 identifier，不過因為是拿 email address 加鹽後 hash 過的，而且只有 UID2 administrator 與 UID2 operator 知道 salt 為何，所以其他單位（例如 SSP 與 DSP）無法還原出 email address 本身。

當網站想要推送廣告時，它會將使用者的 email address 傳送給UID2 service，由 UID2 operator 生成一個一次性的 UID2 token 並回傳，這個 UID2 token 是直接將 UID2 加密，也就是 Enc(UID2) = `Enc(Hash(salt, e-mail))`（白話文來說，將 email address 加鹽後 hash 再加密） 。接著，網站將該 UID2 token 連同其他競標資訊傳送給潛在的廣告商 SSP，SSP 會再將競標資訊與 UID2 token 轉送給 DSP。DSP 得到 UID2 token 後向 UID2 operator 請求 key 來解密，還原出 UID2，並以此作為 identifier 來追蹤或是做廣告競標。

為了讓使用者更容易管理隱私設定，UID2 使用一個集中化的平台管理使用者的追蹤意向，我們稱之為 UID2 administrator。如果使用者希望特定平台停止使用他的資料，他可以提供自己的 email address，UID2 administrator會通知 UID2 operator，然後 UID2 operator 會將 hash 過的 email address 儲存下來，若以後有網站請求這個 email address 的 UID2 與 UID2 token 時，則不會回應真正的數值。此外，UID2 administrator 也會通知 DSP特定 UID2 已經選擇退出追蹤，DSP 若再遇到這個 UID2 時（例如網站可能有在使用者退出前便 cache 下來的 UID2 token），應該將其移除在競標邏輯之外。

UID 2.0 設計的一個核心理念是要建造假名（pseudonym），這個假名無法讓廣告商連回使用者真正的身分。因此，UID2 operator 在產生 UID2 時會加鹽，而且 salt 是由一百萬個 salt bucket 中隨機選一個出來的，如此可以確保 UID2 難以被還原回 email address。此外，對於個別使用者而言，每年都會重新抽一個 salt bucket 並以此產生新的 UID2。一個合理的推測是如此可以避免使用者被長時間的追蹤，但這麼做效果如何仍然未知。

## 安全分析
目前的設計足以確保在不考慮 key compromise 的情況下只有獲得授權的單位可以追蹤使用者。由於每次訪問都會提供新的UID2 token，也就是加密 UID2 + random nonce，因此對於那些無法解密 token 的人而言，UID2 token 本身無法用於追蹤， 只有得到授權的參與者可以正確地解密 UID2 token 得到穩定的 UID2。此外，因為有加鹽並 hash，而且定期改變 salt bucket，所以直接從 UID2 還原出 email address 的可能性並不高，藉此確保廣告商永遠都只能看到使用者的假名，不會知道真正身分。

然而，UID 2.0 的設計有幾個可能的安全性風險。首先，因為 UID2 operator 用於加解密 UID2 的 key 都是一樣的，所以一個惡意的 DSP 可以洩漏 decryption key 給其他與之串通的人，使他們可以解密他們本來不該可以解密的 UID2 token。而且目前的設計並沒有辦法辨認出誰是惡意的 DSP 也沒辦法追蹤 UID2 洩漏的狀況。此外，目前的 UID2 token 產生過程欠缺驗證機制，使得任何得到 key 的人都可以偽造 UID2 token。一個理想的情境是 UID2 operator 應該為 UID2 token 提供簽章以保證其完整性與可驗證性。

## 隱私分析
至此應該已經可以了解為何在文章開頭指出，UID 2.0 走了一條完全不一樣的路。他們想做的並不是消滅個人化的追蹤，而是在得到使用者的同意下進行個人化的追蹤，而且還是 cross-site 甚至 cross-device tracking。這種積極取得同意追蹤的措施是否算是一種保護隱私，取決於大家對於隱私的定義。也許和直覺有些衝突，但這種強調個人選擇與獲得同意的方法，正是當代許多國家的立法方向。專精隱私權的知名法學家 Daniel J. Solove 將其稱為 privacy self-management，指法律賦予人們一系列的權利，讓他們能夠自行決定如何權衡蒐集、使用、揭露資料的成本和利益，更甚至，人們的「同意」（consent）幾乎成為了合法蒐集、使用和揭露個人資料的關鍵要素。UID 2.0 正是一種 privacy self-management 的展現，藉由強調賦予使用者控制其個人資料的權利，來促進使用者的隱私。

但是，顯而易見地 privacy self-management 並不能有效地賦予個人對其資料的控制權，社會科學與心理學研究一再顯示了人們不一定可以有效地控制個人資料。我想多數人的生活經驗也會同意這點：幾乎沒人會真的把隱私權政策讀完，而且人們往往無法評估給出個人資料的代價為何。在 GDPR 出現之後，四處橫行的 cookie banner 不只沒有讓大家獲得更多隱私，還非常惱人，如今出現的 UID 2.0，大概也只會是下一個 cookie banner。當然也沒必要完全否定 UID 2.0，它的確（部分地）解決了如今基於 third-party cookie 的 cross-site tracking 通常沒獲得同意且隱私管理困難的問題，在這種意義上，仍然是一種進步。但是我不相信 privacy self-management 可以帶領我們通往更尊重隱私的網路世界。

更令人困擾的是，UID 2.0 的設計藉由一個中心化的服務以及通用的身分識別依據（email address），繞過了目前瀏覽器對於 cross-site tracking 的防禦機制，其他類似提案（例如 SWAN.community ）使用類似 bounce tracker 的方式至少還有偵測與防禦方式，UID 2.0 幾乎只能回歸傳統 content blocking 來攔截相關的 HTTP requests，在瀏覽器隱私的角度來看，這正好顯示了目前如此繁瑣的防禦措施仍然可能被繞過。

## 結語
UID 2.0 的未來會如何，目前還很難說。一方面，這個方案獲得了眾多廣告商與平台的支持，例如 Criteo、Neilson 等廣告產業的重要角色都表達支持，Prebig.org（Header Bidding 技術的提供者）也決定擔任 UID2 operator 的角色，但另一方面，Google 則明確表達拒絕任何個人化追蹤的技術，算是間接地拒絕了 UID 2.0 的提案。在廣告商眼中，這無疑又是個一整群小蝦米對抗 Google 大鯨魚的故事，但在在乎隱私的使用者眼中，他們的資料終究只是無奈地等著被吞食的藻類而已，至於是被小蝦米還是大鯨魚吞食，似乎也沒那麼重要了。