# 明明顯示 200 OK，LINE Bot 為什麼不回話？一次真實的 Cloud Run 除錯紀錄

前三篇，我和 Codex 完成了 LINE Location Action、Vertex AI Google Maps Grounding、繁中轉譯、來源卡片與重複地點處理。

程式在本機已經能跑，我心裡想的其實很簡單：接下來應該只剩部署了吧？

沒想到，真正花時間的部分才正要開始。

> Cloud Run health 正常、LINE Webhook Verify 也是 200，但使用者傳送位置後，聊天室完全沒反應。

最麻煩的不是看到一個大大的紅色錯誤，而是每個地方看起來都正常，Bot 卻像已讀不回。

這篇想記錄的，就是我和 Codex 怎麼從 Google Cloud 登入、權限與部署一路查到程式本身，最後在一排漂亮的 `200 OK` 背後找到真正原因。

---

## 🔐 明明登入成功，為什麼還是沒有權限？

要讓本機程式呼叫 Vertex AI，Google Cloud 需要先知道「現在是誰在操作」。這裡用的是 Application Default Credentials，簡稱 ADC。可以先把它想成一張給本機程式使用的 Google Cloud 工作證。

```bash
gcloud auth application-default login
```

登入頁正常開啟、瀏覽器也顯示授權成功，credentials 確實寫進了電腦。照理說應該可以用了。

結果第一次真的呼叫 Vertex AI，就被擋下來：

```text
Permission 'aiplatform.endpoints.predict' denied
```

我第一個反應是：「是不是還少了某個 IAM role？」但 Codex 先沒有動權限，而是把目前登入狀態攤開來看：

```bash
gcloud auth list
gcloud config configurations list
gcloud projects list
```

一查才發現，現在登入的帳號根本看不到這個 GCP project。

所以問題不是「沒有登入」，而是「登入了錯的帳號」。那個 Google 帳號是真的，工作證也是真的，只是它沒有這個 project 的門禁權限。

重新指定正確 project owner 帳號並同步 ADC 後：

```bash
gcloud auth login <project-owner-account> --update-adc
```

換成正確帳號後，我們再確認 active account、project、IAM role 與 Vertex AI API 都正確，才繼續往下走。

這次我才真的分清楚兩個很容易混在一起的概念：

> Authentication 是警衛認得你；Authorization 是你的卡真的刷得進這一層樓。

---

## 🧩 有了工作證，還要說明帳要算在哪個專案

帳號正確後，還要設定 quota project。白話來說，就是告訴 Google：「接下來這些 API 用量與配額，要算在哪個 project 名下？」

接著執行：

```bash
gcloud auth application-default set-quota-project your-project-id
```

然後，它又失敗了。

錯誤訊息裡出現 `testIamPermissions`，第一眼又很像權限不夠。但 Codex 把完整 details 往下讀，真正重要的是這一行：

```text
Cloud Resource Manager API has not been used or is disabled
```

原來不是 role 不夠，而是 Cloud Resource Manager API 根本還沒啟用。真正需要做的是：

```bash
gcloud services enable cloudresourcemanager.googleapis.com
```

API 啟用後，我們重新設定 quota project，再用台北座標跑一次 Maps Grounding。這次終於成功拿到繁中摘要與 Google Maps 來源。

這段除錯讓我很有感。錯誤訊息第一行常常只是表面症狀，真正可採取行動的原因可能藏在後面的 details 裡。比起一次亂改很多設定，更有效的方式是找到結構化 reason、一次驗證一個假設，再立刻重跑最小測試。

---

## ☁️ 本機用我的帳號，Cloud Run 上線後要用誰的？

本機終於能呼叫 Vertex AI，接下來才輪到 Cloud Run。

在自己的電腦上，我可以用個人帳號登入；但服務部署到雲端後，總不能一直借用我的個人身分。Cloud Run 也需要一個自己的身分，這就是 service account。

如果把 Cloud Run 想成一位正式上工的機器員工，service account 就是它的員工證。它只拿工作需要的權限，不需要擁有整個 project。

我們替這個服務建立了專用帳號：

```text
line-map-grounding@<project-id>.iam.gserviceaccount.com
```

它只需要兩個角色：

- `roles/aiplatform.user`
- `roles/serviceusage.serviceUsageConsumer`

確認 Cloud Run、Cloud Build 與 Artifact Registry 等必要 API 都已啟用後，才正式部署：

```bash
gcloud run deploy line-map-grounding \
  --source . \
  --region asia-east1 \
  --allow-unauthenticated \
  --no-cpu-throttling \
  --service-account line-map-grounding@<project-id>.iam.gserviceaccount.com \
  --env-vars-file cloud-run-env.yaml
```

終端機最後顯示 `Done`，但「部署指令跑完」和「產品真的能用」是兩回事。我們接著確認 revision 已接收流量、runtime service account 正確、`/health` 回 200，LINE 官方的 Webhook Verify 也顯示成功。

看到這裡，我真的以為完成了。

---

## 😶 所有燈號都是綠的，Bot 卻像已讀不回

我拿起手機傳送位置。位置訊息成功送出，接著等了幾秒、十幾秒，聊天室仍然一片安靜。

沒有錯誤訊息，沒有推薦卡片，也沒有任何「正在處理」的提示。

Codex 先從 Cloud Run 的 request logs 看起，結果每一項都很正常：

- LINE 確實呼叫 `POST /webhook`
- request size 正常
- user agent 是 LINE webhook
- response status 是 200

如果只看 HTTP 狀態，這次請求甚至可以算是成功。這種問題反而比直接跳 500 更難查，因為系統很有禮貌地告訴你：「我都處理好了。」但使用者明明什麼都沒收到。

接著，我們把範圍縮小，只看這個 revision 的應用程式輸出。奇怪的是，裡面沒有 Maps Grounding 開始、完成，也沒有 LINE 回覆成功的紀錄。

這代表 LINE 的請求確實到了，但真正的搜尋流程可能根本沒有可靠地跑完。再回頭看 webhook route，終於找到可疑的順序：

```typescript
res.sendStatus(200);

const results = await Promise.allSettled(
  req.body.events.map(handleWebhookEvent)
);
```

程式一收到 webhook，就先送出 HTTP 200，然後才開始處理每個 event。

這就像櫃台一拿到申請單，還沒辦理就先蓋上「已完成」的章。對 LINE 來說，這次 webhook 已經成功；但真正要查地圖、翻譯和回傳訊息的工作，其實才剛開始。

在本機測試時，Node.js process 還在，response 結束後的 Promise 可能照樣跑完，所以這個問題不一定會立刻出現。但到了 Cloud Run，HTTP response 結束之後的背景工作不能被當成可靠保證。

那個漂亮的 200，只能證明 webhook endpoint 有回應，不能證明咖啡廳搜尋和 LINE 訊息都完成了。

---

## 💡 不要先說「完成」，而是讓整個工作真的做完

找到問題後，我們不再提早結束 request：先顯示 Loading Animation，等待 Maps Grounding 與翻譯完成，用 Push Message 傳回結果，最後才回 HTTP 200。

```typescript
await lineClient.showLoadingAnimation({
  chatId: targetId,
  loadingSeconds: 60
});

const result = await findNearbyCafes(latitude, longitude);

await lineClient.pushMessage({
  to: targetId,
  messages: createCafeResultMessages(result)
});

res.sendStatus(200);
```

這裡同時做了兩個重要調整。

第一個是把結果改用 Push Message 傳送。Maps Grounding 加上翻譯需要一點時間，將結果傳送與原始 reply token 解耦後，就不需要讓長時間任務一直綁著 reply token。

第二個是先顯示 Loading Animation。

這不會讓模型真的變快，但會讓等待變得可以理解。以前傳完位置後，使用者只看到一片安靜；現在至少會先知道 Bot 正在找，通常再等 20～40 秒，就會收到推薦卡片。

修改後，我們重新執行 build 與測試、部署到 Cloud Run，再拿手機走一次完整流程。看到 Loading Animation 出現、接著真的收到咖啡廳卡片時，才算修好。

---

## 🔍 只記錯誤還不夠，成功路徑也要留下腳印

這次會查這麼久，另一個原因是原本幾乎只有失敗時才寫 log。如果程式沒有丟出明顯錯誤，只是安靜地停在某個步驟，我們就很難知道它最後走到哪裡。

所以我們替 webhook 收到事件、搜尋開始、訊息送出與失敗等關鍵節點留下紀錄，並附上 `webhookEventId`、來源數量與處理時間。

```typescript
logger.info('Cafe search reply sent', {
  webhookEventId: event.webhookEventId,
  sourceCount: result.sources.length,
  elapsedMs: Date.now() - startedAt
});
```

這些 log 不是為了把 Cloud Logging 填滿，而是讓我們沿著同一個 `webhookEventId` 看出事件有沒有抵達、搜尋花了多久，以及 Push Message 是否成功。這比盯著一排 200 猜測有效太多。

---

## 🧪 到底怎樣才算「真的好了」？

Part 1 已經做過 typecheck、單元測試、本機啟動與 `/health` smoke test。部署後，我們也再次確認 Cloud Run 的 `/health` 正常。這些檢查很重要，但這次的經驗證明：它們只能告訴我們程式和服務「有活著」，不能證明 LINE 使用者真的收得到結果。

最後還有兩關。

### LINE 找不找得到 webhook？

先用 LINE 官方的 Webhook Verify，確認 endpoint、active status 與 test result 都正常。這可以證明 LINE 能把請求送到 Cloud Run，但仍然不能證明後面的搜尋與訊息發送有完成。

### 真正的使用者收不收得到結果？

最後拿手機走一次完整流程：

1. 傳送「開始」
2. 點擊「傳送目前位置」
3. 確認 Loading Animation
4. 等待繁中摘要與 Maps 卡片
5. 對照 Cloud Logging 的成功路徑

只有最後這一關通過，才代表 Bot 對使用者來說真的修好了。

---

## 🏆 第四篇實戰總結

這次從登入、權限、部署到 webhook 除錯，Codex 做的不只是提供幾條指令，而是跟著問題一路往下查：先確認眼前的錯誤究竟發生在哪一層，再修改、驗證，直到手機真的收到結果。

回頭看，真正花時間的不是某一行 TypeScript，而是在每個看似合理的地方繼續問：「這真的能證明下一步也成功嗎？」

登入成功，不代表帳號正確；帳號正確，不代表 API 已啟用；部署成功，不代表 webhook 後面的工作有完成；HTTP 200，更不代表使用者收到訊息。

**Codex 真正幫上忙的地方，不只是改 code，而是能讀取當下狀態、驗證假設，再陪我把問題一路追到使用者真的收到答案。**

如果你也在做需要 LLM、外部搜尋、圖片分析或其他長時間任務的 LINE Bot，請記住今天這個問題：

> 最難查的 Bug 往往不是 500，而是每個入口都顯示成功，使用者卻什麼都沒收到。

---

### 📂 專案開源與完整程式碼

👉 **GitHub：[https://github.com/zonawang/line-map-grounding](https://github.com/zonawang/line-map-grounding)**

👉 **更多 AI × LINE Bot 實作：[https://github.com/zonawang/zona-ai-learning-lab](https://github.com/zonawang/zona-ai-learning-lab)**
