# 五個 Maps 來源不等於五間店：繁中轉譯、來源卡片與 `placeId` 去重

上一篇，我和 Codex 已經讓 LINE Bot 把使用者的經緯度交給 Vertex AI Google Maps Grounding，成功取得附近咖啡廳的英文推薦與 Grounding metadata。

但模型「有回答」只是第一步。真正要交給 LINE 使用者時，還有三個產品問題：

- Google Maps Grounding 使用英文，使用者想看繁體中文
- 推薦必須附上可以點回 Google Maps 的來源
- response 裡有 5 個來源，不代表真的有 5 間不同咖啡廳

這一篇要處理的，就是如何把 API response 變成使用者看得懂、點得到，而且不會重複的 LINE 訊息。

---

## 🌏 地圖先用英文查，最後再好好說中文

Google Maps Grounding 文件要求 Grounded prompt 與 response 使用英文，但 LINE 使用者希望看到繁體中文。

所以 Codex 沒有硬逼第一次回答直接變中文，而是設計成兩階段：

```text
LINE 經緯度
    ↓
Vertex AI + Google Maps Grounding
    ↓
英文推薦 + Grounding metadata
    ↓
同一個 Vertex AI client 翻成繁中
    ↓
繁中摘要 + 原始 Maps URLs
```

翻譯 prompt 明確限制：

- 保留店名、數字與 caveat
- 不新增事實
- 不產生 URL
- 只回傳翻譯內容

而 Google Maps URL 完全不經過第二個模型，而是直接從：

```typescript
candidate.groundingMetadata?.groundingChunks
```

取出。

這個分工很重要：模型負責翻譯文字，程式負責保留來源。URL 不需要經過模型重寫，也就不會在翻譯時被改壞或憑空補出來。

Codex 也替翻譯加上備案：如果翻譯這一步失敗，至少先把原始英文回答保留下來，不要讓整個搜尋一起消失。

---

## 🔗 AI 說了什麼，使用者應該能點回去確認

每個 Maps chunk 可能包含：

```typescript
chunk.maps?.title
chunk.maps?.uri
chunk.maps?.placeId
```

我們將來源做成 LINE Flex Message carousel：

- 顯示咖啡廳名稱
- 標示「資料來源：Google Maps」
- 提供「在 Google Maps 查看」按鈕
- 讓使用者自行確認營業時間、照片、評論與導航

這讓推薦不再只是「AI 說的」，而是變成可以驗證的資訊。

使用者不是只能相信 AI，而是可以直接回到來源。

---

## 🧩 真實測試才發現：五張卡片裡，有三張可能是同一家店

TypeScript 編譯與單元測試都通過後，Codex 沒有停在假資料。

它用台北座標實際呼叫一次 Vertex Maps Grounding，只輸出摘要長度、來源數量與來源標題，不輸出敏感資料。

結果確實拿到 5 個來源，但標題中出現：

- 店家 Google Maps 頁面
- `Review of ...` 評論頁
- 同一店家的多個 review URLs

原本程式只看網址是否相同。偏偏同一家店的店家頁、不同評論頁，本來就有不同網址，因此全部被當成不同店家。

也就是說，來源數量和地點數量是兩回事。如果直接把每個 chunk 做成一張卡片，LINE carousel 看起來有很多推薦，實際上可能只是不斷重複同一家店。

Codex 根據真實 response 修改策略：

1. 優先使用 `placeId` 當唯一 key。
2. 沒有 `placeId` 時，使用正規化店名。
3. 移除 `Review of` 與 `- Google Maps`。
4. 同一地點同時有店家頁與評論頁時，保留店家頁。

```typescript
const key = maps.placeId || normalizedTitle || maps.uri;

if (!existing || (existing.isReview && !isReview)) {
  uniqueSources.set(key, {
    title: normalizedTitle,
    uri: maps.uri,
    isReview
  });
}
```

接著 Codex 把這個真實案例補進單元測試，確認 review 先出現、店家頁後出現時，最後仍會保留店家頁。

這一段很能代表我喜歡的 AI 協作方式：Codex 不是宣布「測試通過」就收工，而是真的去看使用者最後會看到什麼。

> 不是只把文件範例寫進專案，而是用真實 API 回應驗證，發現產品問題，再把問題變成測試。

---

## 🏆 第三篇實戰總結

這一篇把原始的 Maps Grounding response 整理成真正適合 LINE 使用者的輸出：

- 用英文完成 Maps Grounding，再用第二次 Vertex 呼叫翻成繁中
- 翻譯失敗時保留原始英文回答
- Maps URL 直接取自 Grounding metadata，不經模型改寫
- 使用 Flex Message 顯示 Google Maps attribution
- 以真實 response 發現評論來源重複
- 用 `placeId`、正規化店名與來源優先順序去重
- 把真實案例補進單元測試

此時本機功能已經能真正找到附近咖啡廳，並把可驗證、不重複的繁中推薦送進 LINE。

但下一篇才是最像正式產品的考驗：Bot 查地圖要等二三十秒，怎麼讓使用者知道它還在工作？而 Cloud Run 明明回了 200，為什麼聊天室仍然什麼都沒收到？

👉 **下一篇：LINE Bot 查地圖要等 30 秒——Loading Animation 背後，還藏著 Webhook 200 的陷阱**

---

### 📂 專案開源與完整程式碼

👉 **GitHub：[https://github.com/zonawang/line-map-grounding](https://github.com/zonawang/line-map-grounding)**

👉 **更多 AI × LINE Bot 實作：[https://github.com/zonawang/zona-ai-learning-lab](https://github.com/zonawang/zona-ai-learning-lab)**
