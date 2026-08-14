# 我用 Codex 從空 Repo 做 LINE 定位 Bot：先讓 Bot 正確收到「我在哪裡」

大家哈囉！今天想記錄一個我自己很喜歡的 LINE Bot 實戰系列。

這次我想做的是一個「附近咖啡廳助手」：

> 使用者在 LINE 傳送目前位置，Bot 根據經緯度找出附近咖啡廳，再附上可以直接打開的 Google Maps 來源。

最後完成的版本會用到 Vertex AI、Google Maps、Cloud Run 與 LINE Messaging API。名詞看起來不少，但先不用被嚇到，我們會一層一層把它接起來。

第一篇先不急著碰 AI 模型。我想從最前面開始：**如何和 Codex 從一個空 GitHub repo 出發，先讓 LINE Bot 正確收到使用者的位置。**

這一步看起來沒有那麼炫，但很像蓋房子前先把水電管線排好。如果一開始所有東西都塞在同一個檔案，後面加入地圖搜尋、翻譯與卡片訊息時，很快就會變成一團。

---

## 🧱 我沒有一開始就叫 Codex「全部做完」

目標 repo 一開始是空的。

我先告訴 Codex：我不要一段貼上去看起來會動的範例，我要的是一個能部署、能測試，而且明天還加得動功能的 MVP。

第一階段範圍很明確：

- Node.js + TypeScript + Express
- LINE webhook signature 驗證
- `GET /health`
- 文字訊息顯示操作引導
- LINE 原生 Location Action
- location event 交給獨立 handler
- `.env.example`、Dockerfile、README
- Cloud Run 可部署結構

Codex 沒有直接把全部功能寫進 `server.ts`，而是先參考我過往 LINE Bot 專案的拆法，再建立這個結構：

```text
src/
  app.ts
  server.ts
  handlers/
    webhookEventHandler.ts
  messages/
    cafeMessages.ts
  routes/
    webhook.ts
  services/
    lineClient.ts
  utils/
    env.ts
    logger.ts
```

這次我很喜歡 Codex 的地方，是它不只在聊天室裡回答「可以怎麼做」。它真的進到 repo 裡建立檔案、安裝套件、跑型別檢查，遇到 SDK 不接受的寫法就繼續改。

感覺比較像身旁有一位工程夥伴一起施工，而不是收到一份很長、但不知道能不能跑的答案。

---

## 📍 地址不要用猜的，直接請 LINE 給我們座標

最直覺的做法，是請使用者打地址。

但地址可能長這樣：

- 台北市信義區市政府附近
- 101 旁邊
- 我現在這裡
- 松高路那一帶

後端不只要猜使用者在說哪裡，還要再把文字地址轉成座標。這個轉換通常叫 geocoding，但白話來說，就是「把人類講的位置翻譯成地圖看得懂的數字」。

既然 LINE 本身就有地圖選擇器，最省事的方法就是直接用原生 `location` action：

```typescript
const locationQuickReply = {
  items: [
    {
      type: 'action',
      action: {
        type: 'location',
        label: '傳送目前位置'
      }
    }
  ]
};
```

使用者點擊後，LINE 會直接開啟地圖介面。送出時，webhook 收到的是乾淨的：

```typescript
event.message.latitude
event.message.longitude
```

這跟我之前做 Camera Action、Datetime Picker 時的心得一樣：

**LINE 已經有原生元件時，讓使用者用點的，通常比叫他自己打更可靠。**

---

## 💬 第一版互動先保持簡單

文字訊息不需要進 AI。

使用者傳送「開始」或任意文字時，Bot 只要清楚說明功能並顯示位置按鈕：

```typescript
export function createWelcomeMessage() {
  return {
    type: 'text',
    text: [
      '☕ 我可以用 Google Maps 資料幫你找附近咖啡廳。',
      '',
      '點下面按鈕傳送位置，我會推薦 3–5 間適合坐下來喝咖啡或使用筆電的店。'
    ].join('\n'),
    quickReply: locationQuickReply
  };
}
```

Codex 在這裡沒有過度設計意圖分類，也沒有急著加入資料庫。

它先把最短的使用者路徑跑通：

1. 傳送文字
2. 看見按鈕
3. 分享位置
4. webhook 正確辨識 `location` message

這個小流程其實已經驗證了整條基本通道：LINE 有把訊息送來、後端知道訊息是真的來自 LINE、程式也能分辨文字和位置。

---

## 🧩 Webhook 就像收件櫃台，不要什麼事都叫它做

如果把 webhook 想成公司一樓的收件櫃台，它的工作應該是確認包裹、登記，再交給正確部門，而不是自己拆包、研究內容、回覆客戶。

所以這次架構裡，我特別希望 route 保持乾淨。

`routes/webhook.ts` 只負責：

- LINE middleware 與 signature 驗證
- 取得 events
- 呼叫 event handler
- 回傳 HTTP response

真正判斷文字或位置訊息，放在 `webhookEventHandler.ts`：

```typescript
export async function handleWebhookEvent(event: WebhookEvent) {
  if (event.type !== 'message') {
    return;
  }

  if (event.message.type === 'text') {
    await reply(event.replyToken, [createWelcomeMessage()]);
    return;
  }

  if (event.message.type === 'location') {
    // 下一篇接入 Maps Grounding
  }
}
```

Codex 在產生程式後，還繼續做兩種檢查：

```bash
npm run typecheck
npm test
```

這很重要，因為程式「看起來正確」跟「真的跑得起來」是兩回事。Typecheck 像是在出發前檢查零件規格，smoke test 則是真的把引擎發動一次。

Codex 不是寫完就停，而是把 server build 起來，再做 `/health` smoke test。這讓第一階段不只是 code review 上合理，而是真的可以啟動。

---

## ☁️ 從第一天就把部署條件放進設計

雖然第一篇還沒接 Vertex AI，但專案一開始就加入：

- `Dockerfile`
- `.dockerignore`
- Cloud Run 使用的 `PORT`
- 結構化 JSON logger
- 不提交 secret 的 `.gitignore`
- `/health` endpoint

這也是 Codex 協作帶來的改變。

如果我只是請它「寫一個 location handler」，可能很快就會有答案；但當我明確說最終要部署到 GitHub 與 Cloud Run，它就會從一開始把 runtime、環境變數與驗證方式一起考慮。

部署不是最後一天才突然想到的額外工作。

部署限制應該從第一版架構就進來。

---

## 🏆 第一篇實戰總結

第一階段看起來還沒有 AI，但它已經完成了幾個重要里程碑：

- 從空 repo 建立 TypeScript + Express 專案
- LINE webhook signature 驗證
- 文字訊息與位置訊息分流
- 原生 Location Action 降低輸入摩擦
- route、handler、message、service 分層
- health、build、typecheck 與測試
- Docker 與 Cloud Run 部署骨架

更重要的是，我不是叫 Codex 丟一份範例給我，而是讓它直接參與：

- 讀過往 repo
- 提出架構
- 建立檔案
- 安裝依賴
- 修正型別
- 執行測試
- 把結果推到 GitHub

下一篇，我們會把 location event 的 latitude / longitude 真正交給 Vertex AI Google Maps Grounding，並從 Gemini API key 路線改成 Vertex AI 與 ADC 認證。

👉 **下一篇：從 API Key 轉向 Vertex AI——讓 LINE Bot 用 Google Maps Grounding 找附近咖啡廳**

---

### 📂 專案開源與完整程式碼

👉 **GitHub：[https://github.com/zonawang/line-map-grounding](https://github.com/zonawang/line-map-grounding)**

👉 **更多 AI × LINE Bot 實作：[https://github.com/zonawang/zona-ai-learning-lab](https://github.com/zonawang/zona-ai-learning-lab)**
