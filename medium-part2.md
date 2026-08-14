# 從 API Key 轉向 Vertex AI：讓 LINE Bot 用 Google Maps Grounding 找附近咖啡廳

上一篇，我和 Codex 從空 GitHub repo 建立了 LINE Bot 骨架，並用 Location Action 取得乾淨的 latitude 與 longitude。

這一篇終於要讓 Bot 開始「看地圖」了：

> 如何把位置交給 Vertex AI Google Maps Grounding，產生有來源的咖啡廳推薦？

這一段比我原本想像中更曲折。我們做到一半才發現 API 入口應該換成 Vertex AI，而且這不只是把 API key 刪掉而已，連 client、認證、參數位置與 response 格式都要重新確認。

---

## 🧭 寫 Code 之前，Codex 先幫我確認「到底走哪一條路」

第一版原本參考 Gemini API Maps Grounding 文件，採用 API key 與 Interactions API。

後來我指定另一份 Google Cloud 文件，並明確要求：

**要走 Vertex AI，不要使用 Gemini API key。**

這不是把 `API_KEY` 那一行刪掉就結束。Codex 重新比對了整條呼叫方式：

- client 初始化方式
- `interactions.create` 與 `models.generateContent` 的差異
- Maps tool 的參數位置
- 使用者經緯度欄位
- response 裡 Grounding metadata 的格式
- Google Cloud ADC 認證方式

接著它直接查看已安裝的 `@google/genai` TypeScript declarations，確認 SDK 真正支援：

```typescript
googleMaps?: GoogleMaps;
toolConfig?: ToolConfig;
retrievalConfig?: RetrievalConfig;
groundingChunks?: GroundingChunk[];
```

這次讓我很有感：Codex 不只看網頁文件，還會回頭檢查我電腦裡真正安裝的 SDK 版本。畢竟網頁可能是最新版，但專案裡的套件不一定完全一樣。

用白話說，文件像使用說明書，TypeScript declarations 則像手上這台機器真正有哪些按鈕。兩邊都對上，寫下去才安心。

---

## 🗺️ 不帶 API Key，改拿 Google Cloud 的工作證

最後 client 改成：

```typescript
import { GoogleGenAI } from '@google/genai';

const ai = new GoogleGenAI({
  enterprise: true,
  project: process.env.GOOGLE_CLOUD_PROJECT,
  location: process.env.GOOGLE_CLOUD_LOCATION || 'global',
  apiVersion: 'v1'
});
```

環境變數也不再需要 `GEMINI_API_KEY`：

```env
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=global
GEMINI_MAPS_MODEL=gemini-2.5-flash
GEMINI_TRANSLATION_MODEL=gemini-2.5-flash
```

Codex 同步修改了：

- `.env.example`
- 環境變數驗證
- 本機 `.env`
- README 的 ADC 說明
- Cloud Run service account 部署方式
- 單元測試中的測試環境

這就是整合型修改和只改一小段 code 的差別。

如果只改 client，README 還在叫人填 API key，下一次連我自己都可能被搞混。

---

## 📍 把 LINE 給的兩個數字，交給 Google Maps

位置查詢的核心如下：

```typescript
const response = await ai.models.generateContent({
  model: 'gemini-2.5-flash',
  contents: [
    'Find 3 to 5 good cafes near the supplied user location.',
    'Prioritize places practical for sitting down with a laptop.',
    'Do not invent outlet, Wi-Fi, time-limit, or noise information.',
    'Keep the answer concise and respond in English.'
  ].join(' '),
  config: {
    tools: [{ googleMaps: {} }],
    toolConfig: {
      retrievalConfig: {
        latLng: { latitude, longitude },
        languageCode: 'en_US'
      }
    }
  }
});
```

程式看起來不少，但核心其實很單純：把使用者的 latitude、longitude 放進 `latLng`，再告訴模型可以使用 Google Maps。

Prompt 裡我特別保留一句：

> 沒有資料時，不要自行宣稱有插座、Wi-Fi、不限時或安靜。

這些條件很適合咖啡廳推薦，但不一定存在於每間店的 Maps 資料裡。

Codex 在這裡沒有幫我把答案硬補得很漂亮，反而替 prompt 畫了一條線。對這種有地圖資料的產品來說，少說一點，通常比很有自信地猜更好。

---

## 🏆 第二篇實戰總結

這一篇先完成了 Maps Grounding 的核心呼叫：

- Codex 對照文件與本機 SDK 型別
- 從 Gemini API key 改成 Vertex AI client
- 將 LINE 經緯度放入 `retrievalConfig.latLng`
- 用英文完成有來源的 Maps Grounding
- 限制模型不要猜測 Wi-Fi、插座或不限時等資訊

此時 Bot 已經能把 LINE 傳來的經緯度交給 Vertex AI，並取得附近咖啡廳的英文推薦與 Grounding metadata。

但「模型有回答」還不等於「使用者拿到好用的答案」。下一篇，我們要把英文推薦安全地翻成繁中、把 Maps 來源做成 LINE 卡片，並處理一個只有真實呼叫才看得到的問題：五個來源，可能根本不是五間店。

👉 **下一篇：五個 Maps 來源不等於五間店——繁中轉譯、來源卡片與 `placeId` 去重**

---

### 📂 專案開源與完整程式碼

👉 **GitHub：[https://github.com/zonawang/line-map-grounding](https://github.com/zonawang/line-map-grounding)**

👉 **更多 AI × LINE Bot 實作：[https://github.com/zonawang/zona-ai-learning-lab](https://github.com/zonawang/zona-ai-learning-lab)**
