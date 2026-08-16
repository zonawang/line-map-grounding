# LINE Bot 查地圖要等 30 秒：Loading Animation 背後，還藏著 Webhook 200 的陷阱

前三篇，我和 Codex 已經完成 LINE Location Action、Vertex AI Google Maps Grounding、繁中轉譯與來源卡片。

本機測試也成功了。當時我真的以為，接下來只要部署到 Cloud Run，就可以漂亮收尾。

結果真正拿起手機測試時，位置送出去，聊天室卻安靜了二三十秒。

沒有錯誤、沒有回覆，也沒有任何「正在處理」的提示。

我知道後端可能正在查 Google Maps、整理來源和翻譯，但使用者哪知道？只會想：「剛剛有送成功嗎？Bot 是不是壞了？」然後再按一次、再傳一次，甚至直接離開。

所以最後這篇，我想解決的不只是「怎麼放一個 Loading Animation」，而是更實際的問題：

> 當 AI 任務需要時間，怎麼讓使用者知道系統還在工作，也讓後端真的可靠地把工作做完？

---

## ☁️ 在動畫出現以前，我先卡在 Google Cloud 門口

不過，在手機上看到動畫以前，我們先被 Google Cloud 的登入與權限擋了一輪。

本機程式要呼叫 Vertex AI，需要 ADC。名字有點硬，可以把它想成程式拿來進 Google Cloud 的工作證。登入畫面明明顯示成功，第一次呼叫卻直接收到：

```text
Permission 'aiplatform.endpoints.predict' denied
```

一查才發現，我確實有登入，只是登成了另一個看不到這個 project 的 Google 帳號。好不容易換對帳號，下一關又跳出來：Cloud Resource Manager API 還沒啟用，所以 quota project 設不起來。

把這兩關處理完，本機才終於能呼叫 Vertex AI。接著部署到 Cloud Run，我們又替服務準備自己的 runtime service account，只給它工作需要的角色：

- `roles/aiplatform.user`
- `roles/serviceusage.serviceUsageConsumer`

走完這段，我最大的感想是：畫面顯示登入成功，不代表你登對帳號；部署指令顯示完成，也不代表服務真的拿得到需要的資源。

處理完這些身分問題後，Cloud Run 的 `/health` 回 200，LINE 官方的 Webhook Verify 也顯示成功。每個燈號都是綠的，這次總該好了吧？

---

## 😶 最大的問題不是慢，而是「不知道發生什麼事」

結果我拿起手機傳送位置，聊天室就這樣安靜了二三十秒。

其實後端完全沒閒著。使用者送出位置後，它要一口氣做完：

1. 接收 LINE location event
2. 將經緯度傳給 Vertex AI
3. 等待 Google Maps Grounding
4. 整理店家與來源
5. 將英文回答翻成繁體中文
6. 組成 LINE Flex Message
7. 把結果送回聊天室

整個流程可能需要 20～40 秒。

如果畫面明確告訴我「正在處理」，等 30 秒還可以接受；但它完全不動時，才過 5 秒，我就已經開始懷疑剛才是不是沒有按成功。

Codex 看完 handler 後，先提了一個很直覺的改善：使用 LINE 官方的 `showLoadingAnimation`。

白話來說，就是讓 Bot 先表現出「我收到位置了，現在正在找」。

---

## ⏳ Loading Animation 真的只有幾行

呼叫動畫本身很簡單：

```typescript
await lineClient.showLoadingAnimation({
  chatId: targetId,
  loadingSeconds: 60
});
```

`chatId` 決定動畫要出現在哪個聊天室，`loadingSeconds` 則是它最多顯示多久。

不過 LINE event 可能來自一對一聊天、群組或聊天室，ID 不完全一樣。Codex 先用 helper 統一取得 `userId`、`groupId` 或 `roomId`，後面的動畫和推送只要認同一個 `targetId` 就好。

還有一個小地方很重要：動畫只是讓體驗更好，不是主角。如果動畫 API 剛好失敗，總不能連咖啡廳也不找了。

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

這就像餐廳的叫號螢幕壞了。螢幕壞掉當然不理想，但廚房還是要繼續做餐。

---

## ⚠️ 只加動畫還不夠，真正的問題藏在 Webhook 生命週期

動畫加上去以後，我們才發現事情還沒結束。LINE webhook request 是 200，Cloud Run health 也正常，使用者卻還是收不到結果。

Codex 打開 application logs，裡面沒有 Maps Grounding 開始、完成或訊息送出的紀錄。也就是說，LINE 的請求有進來，但後面的工作可能根本沒跑完。再回頭看 webhook route，終於看到一個很可疑的順序：

```typescript
res.sendStatus(200);

await Promise.allSettled(
  req.body.events.map(handleWebhookEvent)
);
```

原來程式一收到 webhook，就先回 HTTP 200，接著才在背景處理 event。

這就像櫃台拿到申請單，還沒辦理就先蓋上「已完成」的章。對 LINE 來說，這次 webhook 已經成功；但真正要查地圖、翻譯和傳送訊息的工作，其實才剛開始。

這段在本機不一定會出事，因為 Node.js process 還在，response 結束後的 Promise 可能照樣跑完。但放到 Cloud Run，就不能期待 response 都結束了，後面的背景工作還一定會乖乖完成。

解法看起來很小：把順序反過來，工作做完後再回 200。

```typescript
await Promise.allSettled(
  req.body.events.map(handleWebhookEvent)
);

res.sendStatus(200);
```

這時我才懂，Loading Animation 只解決「使用者不知道自己在等什麼」；調整 request 順序，才是確保「後端真的把工作做完」。兩件事缺一不可。

---

## 📤 為什麼結果改用 Push Message？

Maps Grounding 加上翻譯需要時間。如果一直拖到最後才使用最初的 reply token，就得賭它到那時還沒過期。

所以 Codex 把搜尋結果改成 Push Message：

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

動畫不會讓模型變快，但它讓等待有了意義；Push Message 則讓長時間任務不用綁死在最初的 reply token 上。

---

## 🔍 只記錯誤還不夠，成功路徑也要留下腳印

第一次 Bot 沒回覆時，我們只看到 request 200，其他幾乎一片空白，根本不知道搜尋走到哪裡。

所以 Codex 在幾個重要位置加上 log：webhook 收到事件、搜尋開始、訊息送出，以及失敗。每一筆還會帶著 `webhookEventId`、來源數量與處理時間。

```typescript
logger.info('Cafe search reply sent', {
  webhookEventId: event.webhookEventId,
  sourceCount: result.sources.length,
  elapsedMs: Date.now() - startedAt
});
```

下次再遇到「Bot 沒反應」，只要沿著同一個 `webhookEventId` 往下看，就能知道 event 到了沒、搜尋花了多久，以及 Push Message 有沒有送成功。

Logs 就像沿路留下的腳印。沒有腳印時，只知道旅客沒到終點；有了腳印，才知道他停在哪一站。

---

## 🧪 最後一定要拿手機走一次完整流程

Typecheck、單元測試、`/health` 和 Webhook Verify 當然都要做，但它們只能證明各個零件看起來正常，不能證明使用者真的收得到結果。

最後我拿手機重新測試：

1. 傳送「開始」
2. 點擊「傳送目前位置」
3. 確認 Loading Animation 出現
4. 等待繁中摘要與 Maps 卡片
5. 對照 Cloud Logging 的成功路徑

直到動畫出現，接著真的收到咖啡廳卡片，我才敢說：「好，這次真的修好了。」

當然，這還不是正式產品的終點。如果 LINE 因為等太久而重送 webhook，後端還要用 `webhookEventId` 去重，或把長時間工作交給 Cloud Tasks，才不會同一個位置查兩次、推兩次。

這次 MVP 先把眼前最痛的問題解掉，也把下一步記清楚，不假裝所有可靠性問題都已經消失。

---

## 🏆 第四篇實戰總結

Loading Animation 的 code 只有幾行，但它背後其實連著三個不同問題：

- **使用者體驗**：等待時，要讓使用者知道 Bot 正在工作
- **Webhook lifecycle**：不能先結束 response，再期待背景任務一定完成
- **結果傳送方式**：長時間任務適合用 Push Message 與 reply token 解耦

這次 Codex 不只是替聊天視窗加上一個會動的圖示。它從 Cloud Run logs 找出 `200 OK` 背後的問題，再一起調整 request 順序、Push Message、錯誤處理與成功 logs。最後重新部署，還是得回到手機上再傳一次位置。

回頭看，真正讓體驗變好的不只是動畫，而是整條等待流程終於變得清楚：使用者知道 Bot 還在找，我也知道程式走到哪裡。

如果你也在做需要 LLM、外部搜尋、圖片分析或其他長時間任務的 LINE Bot，請記住：

> 最難查的 Bug 往往不是 500，而是每個入口都顯示成功，使用者卻什麼都沒收到。

---

### 📂 專案開源與完整程式碼

👉 **GitHub：[https://github.com/zonawang/line-map-grounding](https://github.com/zonawang/line-map-grounding)**

👉 **更多 AI × LINE Bot 實作：[https://github.com/zonawang/zona-ai-learning-lab](https://github.com/zonawang/zona-ai-learning-lab)**
