---
weight: 3
title: "Stateful Tracking"
---

# Stateful Tracking

在第一章時，我們提過，一般而言可以將 web tracking 技術分為兩大類型：stateful 與 stateless。在 stateful web tracking 中，tracker 會預先決定好一個唯一的 identifier，並儲存在特定位置；在 stateless web tracking 中，tracker 沒辦法儲存東西，所以只能去蒐集現有資訊並算出一個 identifier。

在這個章節中，我們將討論各種 stateful web tracking 技術，討論 tracker 可以如何儲存它的 identifier。

如同前面提過，web tracking 可以根據範圍區分成 same-site 與 cross-site tracking。只要可以儲存 identifier，就一定可以作到 same-site tracking，畢竟網站 A 存進去的 identifier，網站 A 當然讀得到。不過 cross-site tracking 卻不是如此，如果要做到 cross-site tracking，追蹤者需要在不同網站維持同一個 identifier，或至少可以連結不同的 identifier，如此才能知道兩個網站的訪客是同一個人，可是這兩個途徑都不太簡單。這個章節中，我們也會不同儲存 identifier 的方法是否可能達成 cross-site tracking。