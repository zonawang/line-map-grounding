# 明明顯示 200 OK，LINE Bot 為什麼不回話？一次真實的 Cloud Run 除錯紀錄

前三篇，我和 Codex 完成了 LINE Location Action、Vertex AI Google Maps Grounding、繁中轉譯、來源卡片與重複地點處理。

本機測試已經成功，我原本以為接下來只要按下部署，收工。

我原本以為這會是最簡單的一段，結果真正上線後，才遇到整個系列最值得記錄的問題：

> Cloud Run health 正常、LINE Webhook Verify 也是 200，但使用者傳送位置後，聊天室完全沒反應。

結果這段才是整個系列最像實戰的地方。這一篇會記錄 Codex 如何陪我登入 Google Cloud、辨認權限問題、部署服務，再從一堆「看起來都正常」的 logs 裡找到真正根因。

---

## 🔐 登入成功了，但拿錯了工作證

Vertex AI 本機開發使用 Application Default Credentials，簡稱 ADC。先不用記這個長名字，可以把它想成：**讓本機程式拿到一張 Google Cloud 工作證。**

```bash
gcloud auth application-default login
```

Codex 啟動登入流程，瀏覽器授權也成功，credentials 確實寫入本機。

但第一次真實呼叫 Vertex AI，收到：

```text
Permission 'aiplatform.endpoints.predict' denied
```

Codex 沒有立刻亂加 role，而是先查：

```bash
gcloud auth list
gcloud config configurations list
gcloud projects list
```

結果發現目前登入帳號根本看不到任何 GCP project。

問題不是「沒登入」，而是拿到另一個 Google 帳號的工作證。那個帳號雖然真實存在，卻沒有這個 project 的門禁權限。

重新指定正確 project owner 帳號並同步 ADC 後：

```bash
gcloud auth login <project-owner-account> --update-adc
```

Codex 再檢查一次，確認：

- active account 正確
- project 正確
- project 狀態為 ACTIVE
- 帳號具有 Owner role
- Vertex AI API 已啟用

這段讓我重新理解兩個常被混在一起的概念：

> Authentication 是警衛認得你；Authorization 是你的卡真的刷得進這一層樓。

---

## 🧩 有工作證之後，還要告訴 Google「這次算哪個專案的」

ADC 登入後，還要設定 quota project。白話來說，就是告訴 Google：這次 API 的用量與配額要記在哪個 project 名下。

接著執行：

```bash
gcloud auth application-default set-quota-project your-project-id
```

又失敗了。

錯誤裡出現 `testIamPermissions`，很像權限問題。但 Codex 往下讀完整 details，真正 reason 是：

```text
Cloud Resource Manager API has not been used or is disabled
```

所以解法不是繼續加 IAM，而是：

```bash
gcloud services enable cloudresourcemanager.googleapis.com
```

API 啟用後，Codex重新設定 quota project，再跑一次台北座標 Maps Grounding，成功取得繁中摘要與 Google Maps sources。

我很喜歡 Codex 這次的除錯方式：

1. 不只看錯誤第一行
2. 找到結構化 reason
3. 一次只修改一個假設
4. 修改後立刻重跑最小驗證

---

## ☁️ Cloud Run 也需要自己的「機器員工證」

本機 ADC 成功後，下一步是正式環境。

Cloud Run 上線後，不應該繼續借用我的個人登入。比較合理的做法，是替這個服務建立一位專用的「機器員工」，也就是 service account。

Codex 建立專用 runtime service account：

```text
line-map-grounding@<project-id>.iam.gserviceaccount.com
```

並授予：

- `roles/aiplatform.user`
- `roles/serviceusage.serviceUsageConsumer`

接著確認 Cloud Run、Cloud Build 與 Artifact Registry API，再進行部署：

```bash
gcloud run deploy line-map-grounding \
  --source . \
  --region asia-east1 \
  --allow-unauthenticated \
  --no-cpu-throttling \
  --service-account line-map-grounding@<project-id>.iam.gserviceaccount.com \
  --env-vars-file cloud-run-env.yaml
```

部署完成後，Codex 沒有只相信終端機的 `Done`，而是繼續驗證：

- revision 已 serving 100% traffic
- `/health` 回傳 200
- runtime service account 正確
- CPU throttling 設定正確
- LINE webhook endpoint 更新成功
- LINE 官方 webhook test 回傳 OK

到這裡，health 是綠的、LINE Verify 也是綠的，我們都以為終於完成了。

---

## 😶 所有燈號都是綠的，使用者卻只看到一片安靜

手機實測時，位置成功送出，但聊天室沒有任何回答。

Codex 先讀 Cloud Run request logs，看到：

- LINE 確實呼叫 `POST /webhook`
- request size 正常
- user agent 是 LINE webhook
- response status 是 200

表面上沒有任何錯誤。這種問題比直接跳 500 更讓人困惑，因為系統像是在很有禮貌地告訴你：「我一切正常。」

接著 Codex 把查詢縮小，只看該 revision 的 stdout / stderr，發現應用程式沒有 Grounding 開始、完成或 LINE reply 的 logs。

再回頭讀 route，根因出現了：

```typescript
res.sendStatus(200);

const results = await Promise.allSettled(
  req.body.events.map(handleWebhookEvent)
);
```

Webhook 一進來就先回覆「收到」，等於櫃台先把案件蓋上「已完成」章，才轉身開始處理真正的工作。

本機 Node.js 可能看起來還會跑，但在 Cloud Run 上，不能把 response 結束後的背景 Promise 當成可靠保證。

這就是為什麼監控顯示漂亮的 200，使用者體感卻像 Bot 壞掉。

---

## 💡 先讓使用者知道我們正在找，再把結果主動送回去

Codex 將流程改成：

1. 收到 location event
2. 顯示 LINE Loading Animation
3. 保持 Cloud Run request
4. 等待 Vertex Maps Grounding 與翻譯
5. 使用 Push Message 傳送結果
6. 所有 event 完成後才回 HTTP 200

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

為什麼 Codex 改用 Push Message？

因為 Grounding 與翻譯需要時間。結果傳送與原始 reply token 解耦後，不需要把長時間任務綁死在 reply token 上。

Loading Animation 也改善了使用者體驗。

以前傳位置後畫面完全靜止；現在會先看到 Bot 正在處理，20～40 秒後再收到推薦卡片。

Codex 修改後立刻執行：

- TypeScript build
- 單元測試
- Cloud Run redeploy
- health check
- revision 檢查
- Git commit 與 push

修復不是停在本機，而是一路更新到線上版本。

---

## 🔍 Logs 就像沿路留下腳印，不然只能猜 Bot 走到哪裡

這次難查，是因為原本只有失敗會寫 logs。只要程式安靜地停在中間，我們就不知道它走到哪一步。

Codex 加入：

- `Webhook event received`
- `Cafe search started`
- `Cafe search reply sent`
- `Cafe search failed`
- `webhookEventId`
- `sourceCount`
- `elapsedMs`

```typescript
logger.info('Cafe search reply sent', {
  webhookEventId: event.webhookEventId,
  sourceCount: result.sources.length,
  elapsedMs: Date.now() - startedAt
});
```

現在如果又有人說「Bot 沒反應」，我們可以直接判斷：

- LINE event 有沒有到
- Maps Grounding 有沒有開始
- 模型花了多久
- 回了幾個來源
- Push Message 是否成功

這比盯著一排 200 猜測有效太多。

---

## 🧪 最後整理出的五層測試

### 1. 程式層

```bash
npm run typecheck
npm test
```

### 2. 本機服務

```bash
curl http://localhost:3000/health
```

### 3. Cloud Run

```bash
curl https://<service-url>/health
```

### 4. LINE Webhook Verify

確認 endpoint、active status 與 test result。

### 5. 手機 End-to-End

1. 傳送「開始」
2. 點擊「傳送目前位置」
3. 確認 Loading Animation
4. 等待繁中摘要與 Maps 卡片
5. 對照 Cloud Logging 成功路徑

Codex 這次最重要的提醒是：前四層全部成功，仍然不能取代最後真的拿手機用一次。

---

## 🏆 第四篇實戰總結

這次從部署到修復，我不只是叫 Codex 提供指令，而是讓它直接參與整個閉環：

- 啟動 Google 登入
- 發現錯誤帳號
- 驗證 project IAM
- 啟用缺少的 API
- 設定 ADC quota project
- 建立 runtime service account
- 部署 Cloud Run
- 更新 LINE webhook
- 讀 request logs 與 application logs
- 從 200 response 找到非同步生命週期問題
- 修改 Loading Animation 與 Push Message
- 重新測試、部署、提交

這次 Codex 的存在感不是只出現在文章最後的心得，而是散在每一個真實決定裡：看完整錯誤、切換帳號、驗證權限、部署、縮小 log 範圍、修改流程，再請我重新傳一次位置。

**它真正有價值的地方，不只是會寫 code，而是能使用終端機、讀現況、驗證假設，陪我把線上問題一路查到使用者真的收到答案。**

如果你也在做需要 LLM、外部搜尋、圖片分析或其他長時間任務的 LINE Bot，請記住今天這個問題：

> 最難查的 Bug 往往不是 500，而是每個入口都顯示成功，使用者卻什麼都沒收到。

---

### 📂 專案開源與完整程式碼

👉 **GitHub：[https://github.com/zonawang/line-map-grounding](https://github.com/zonawang/line-map-grounding)**

👉 **更多 AI × LINE Bot 實作：[https://github.com/zonawang/zona-ai-learning-lab](https://github.com/zonawang/zona-ai-learning-lab)**
