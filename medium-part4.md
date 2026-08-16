# LINE Bot 查地圖要等 30 秒：Loading Animation 背後，還藏著 Webhook 200 的陷阱

前三篇，我和 Codex 已經完成 LINE Location Action、Vertex AI Google Maps Grounding、繁中轉譯與來源卡片。

本機測試也成功了。我原本以為接下來只要部署到 Cloud Run，這個系列就可以漂亮收尾。

結果真正拿起手機測試時，位置送出去，聊天室卻安靜了二三十秒。

沒有錯誤、沒有回覆，也沒有任何「正在處理」的提示。

身為開發者，我知道後端可能正在查 Google Maps、整理來源和翻譯；但使用者看不到這些。他只會懷疑 Bot 是不是壞了，然後重按、重傳，或直接離開。

所以這篇最後要解決的，不只是怎麼顯示一個 Loading Animation，而是：

> 當 AI 任務需要時間，怎麼讓使用者知道系統還在工作，也讓後端真的可靠地把工作做完？

---

## ☁️ 上線前，先讓 Cloud Run 拿到正確身分

在看到動畫之前，我們先遇到一段 Google Cloud 的登入與權限插曲。

本機呼叫 Vertex AI 使用 ADC，也就是給本機程式使用的 Google Cloud 工作證。登入雖然成功，第一次呼叫卻收到：

```text
Permission 'aiplatform.endpoints.predict' denied
```

一查才發現，我登入的是另一個看不到這個 project 的 Google 帳號。換成正確帳號後，又因為 Cloud Resource Manager API 尚未啟用，無法設定 quota project。

這兩個問題修好後，本機才真正能呼叫 Vertex AI。到了 Cloud Run，我們則替服務建立專用的 runtime service account，並只給它工作需要的角色：

- `roles/aiplatform.user`
- `roles/serviceusage.serviceUsageConsumer`

這段最重要的提醒是：登入成功，不代表帳號正確；部署成功，也不代表服務拿到了正確權限。

處理完這些身分問題，Cloud Run 的 `/health` 回 200，LINE 官方的 Webhook Verify 也顯示成功。所有燈號都是綠的，我真的以為完成了。

---

## 😶 最大的問題不是慢，而是「不知道發生什麼事」

Google Maps Grounding 不是回傳一句固定文字。使用者送出位置後，後端要完成：

1. 接收 LINE location event
2. 將經緯度傳給 Vertex AI
3. 等待 Google Maps Grounding
4. 整理店家與來源
5. 將英文回答翻成繁體中文
6. 組成 LINE Flex Message
7. 把結果送回聊天室

整個流程可能需要 20～40 秒。

如果畫面有明確的「正在處理」，30 秒還可以理解；但畫面完全靜止時，5 秒就足以讓人懷疑剛才是不是沒有按成功。

Codex 讀完目前的 handler 後，先提出最直接的改善：使用 LINE 官方的 `showLoadingAnimation`。

白話來說，就是讓 Bot 先表現出「我收到位置了，現在正在找」。

---

## ⏳ Loading Animation 的程式其實很短

真正呼叫動畫只需要：

```typescript
await lineClient.showLoadingAnimation({
  chatId: targetId,
  loadingSeconds: 60
});
```

`chatId` 是要顯示動畫的聊天室；`loadingSeconds` 則是動畫最多持續多久。

LINE event 可能來自一對一聊天、群組或聊天室，所以 Codex 先用 helper 統一取得 `userId`、`groupId` 或 `roomId`，後面的動畫與推送都共用同一個 `targetId`。

但 Loading Animation 只是體驗加分，不是主要功能。如果動畫 API 暫時失敗，咖啡廳搜尋仍然應該繼續。

```typescript
if (targetId) {
  try {
    await lineClient.showLoadingAnimation({
      chatId: targetId,
      loadingSeconds: 60
    });
  } catch (error) {
    logger.error('Loading animation failed', {
      error: error instanceof Error ? error.message : String(error)
    });
  }
}
```

這很像餐廳的叫號螢幕壞了。螢幕壞掉當然不理想，但廚房不應該因此停止做餐。

---

## ⚠️ 只加動畫還不夠，真正的問題藏在 Webhook 生命週期

第一次上線時，LINE webhook request 是 200，Cloud Run health 也正常，使用者卻完全收不到結果。

Codex 往 application logs 裡看，沒有找到 Maps Grounding 開始、完成或訊息送出的紀錄。再回頭讀 webhook route，終於看到可疑的順序：

```typescript
res.sendStatus(200);

await Promise.allSettled(
  req.body.events.map(handleWebhookEvent)
);
```

程式一收到 webhook，就先回覆 HTTP 200，然後才在背景處理 event。

這就像櫃台拿到申請單，還沒辦理就先蓋上「已完成」的章。對 LINE 來說，這次 webhook 已經成功；但真正要查地圖、翻譯和傳送訊息的工作，其實才剛開始。

在本機，Node.js process 還在，response 結束後的 Promise 可能照樣跑完；但到了 Cloud Run，不能把 response 結束後的背景工作當成可靠保證。

最後我們把順序反過來：

```typescript
await Promise.allSettled(
  req.body.events.map(handleWebhookEvent)
);

res.sendStatus(200);
```

Loading Animation 解決的是「使用者不知道正在等什麼」；保持 request 則是確保「後端真的把工作做完」。兩件事缺一不可。

---

## 📤 為什麼結果改用 Push Message？

Maps Grounding 加上翻譯需要時間。如果一直等到最後才使用最初的 reply token，流程會更依賴 token 的有效時間。

所以 Codex 將搜尋結果改成 Push Message：

```typescript
const result = await findNearbyCafes(latitude, longitude);
const messages = createCafeResultMessages(result);

await lineClient.pushMessage({
  to: targetId,
  messages
});
```

最後的使用體驗變成：

1. 使用者傳送位置
2. LINE 顯示 Loading Animation
3. 後端查詢 Vertex AI 與 Google Maps
4. Bot 主動推送繁中摘要與來源卡片

動畫不會讓模型變快，但它讓等待變得可以理解；Push Message 則讓長時間任務不需要綁死在原始 reply token 上。

---

## 🔍 只記錯誤還不夠，成功路徑也要留下腳印

第一次沒回覆時，我們只看到 request 200，卻不知道搜尋究竟走到哪裡。

所以 Codex 在 webhook 收到事件、搜尋開始、訊息送出與失敗等關鍵節點加入 log，並記下 `webhookEventId`、來源數量與處理時間。

```typescript
logger.info('Cafe search reply sent', {
  webhookEventId: event.webhookEventId,
  sourceCount: result.sources.length,
  elapsedMs: Date.now() - startedAt
});
```

現在只要沿著同一個 `webhookEventId` 往下看，就能知道 event 有沒有抵達、搜尋花了多久，以及 Push Message 是否成功。

Logs 就像沿路留下的腳印。沒有腳印時，只知道旅客沒到終點；有了腳印，才知道他停在哪一站。

---

## 🧪 最後一定要拿手機走一次完整流程

Typecheck、單元測試、`/health` 與 Webhook Verify 都很重要，但它們只能證明各個零件看起來正常，不能證明使用者真的收到結果。

最後我拿手機重新測試：

1. 傳送「開始」
2. 點擊「傳送目前位置」
3. 確認 Loading Animation 出現
4. 等待繁中摘要與 Maps 卡片
5. 對照 Cloud Logging 的成功路徑

看到動畫出現，接著真的收到咖啡廳卡片，這次才算修好。

正式產品還需要再處理 webhook retry。如果 LINE 因等待時間較長而重送 event，後端應使用 `webhookEventId` 去重，或把長時間工作交給 Cloud Tasks，避免重複查詢與推送。

這次 MVP 先把下一步清楚留下，沒有假裝所有可靠性問題都已經消失。

---

## 🏆 第四篇實戰總結

Loading Animation 的 code 只有幾行，但它背後其實連著三個不同問題：

- **使用者體驗**：等待時，要讓使用者知道 Bot 正在工作
- **Webhook lifecycle**：不能先結束 response，再期待背景任務一定完成
- **結果傳送方式**：長時間任務適合用 Push Message 與 reply token 解耦

這次 Codex 不只是替聊天視窗加上一個會動的圖示。它從 Cloud Run logs 找到 `200 OK` 背後的真正問題，再一起修改 request 順序、Push Message、錯誤處理與成功 logs，最後重新部署，請我用手機再測一次。

回頭看，真正讓體驗變好的不是動畫本身，而是整條等待流程終於對使用者與開發者都變得清楚。

如果你也在做需要 LLM、外部搜尋、圖片分析或其他長時間任務的 LINE Bot，請記住：

> 最難查的 Bug 往往不是 500，而是每個入口都顯示成功，使用者卻什麼都沒收到。

---

### 📂 專案開源與完整程式碼

👉 **GitHub：[https://github.com/zonawang/line-map-grounding](https://github.com/zonawang/line-map-grounding)**

👉 **更多 AI × LINE Bot 實作：[https://github.com/zonawang/zona-ai-learning-lab](https://github.com/zonawang/zona-ai-learning-lab)**
