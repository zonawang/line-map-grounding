# 我用 Codex 做了一個會看地圖的 LINE Bot：四篇實戰系列

原本的完整實作已重新整理成四篇可以獨立閱讀與發布的文章。

## 第一篇：LINE 定位與專案骨架

👉 [`medium-part1.md`](./medium-part1.md)

**我用 Codex 從空 Repo 做 LINE 定位 Bot：先讓 Bot 正確收到「我在哪裡」**

- 從空 GitHub repo 開始
- Codex 如何規劃 TypeScript / Express 架構
- LINE Location Action 與 Quick Reply
- Webhook、handler、message、service 分層
- 第一階段測試與部署準備

## 第二篇：接通 Vertex Maps Grounding

👉 [`medium-part2.md`](./medium-part2.md)

**從 API Key 轉向 Vertex AI：讓 LINE Bot 用 Google Maps Grounding 找附近咖啡廳**

- Codex 如何對照兩份 Google 文件
- 從 Gemini API key 改成 Vertex AI ADC
- 經緯度與 `googleMaps` tool
- 用英文取得有來源的附近咖啡廳推薦

## 第三篇：把 Grounding response 變成 LINE 訊息

👉 [`medium-part3.md`](./medium-part3.md)

**五個 Maps 來源不等於五間店：繁中轉譯、來源卡片與 `placeId` 去重**

- 英文 Grounding、繁中轉譯與失敗備案
- Grounding metadata 與原始 Maps URL
- Google Maps attribution 與 Flex Message
- 真實 response 中的 review URL 重複問題
- 使用 `placeId` 與正規化店名去重

## 第四篇：Loading Animation 與線上除錯

👉 [`medium-part4.md`](./medium-part4.md)

**LINE Bot 查地圖要等 30 秒：Loading Animation 背後，還藏著 Webhook 200 的陷阱**

- 為什麼「沒有反應」比「慢」更糟
- `showLoadingAnimation` 與不同聊天室 target ID
- 動畫失敗不影響主要搜尋
- Webhook 200 但背景 Promise 沒有可靠完成
- Push Message、結構化 logs 與 webhook 重試
- ADC、quota project 與 Cloud Run service account
