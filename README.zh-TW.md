# 🌸 LINE Bot 主動懸念破冰 ＆ CacheService 6輪滾動短期記憶 (GAS)

> 讓你的 LINE AI 助理不再只是「冷冰冰的一問一答」，而是擁有「開場主動惦記你的心頭懸念」與「90分鐘超省成本的短期對話記憶」！

## ✨ 核心亮點

1. **🎭 心頭懸念主動破冰 (Active Intention)**：
   - 剛開場對話時，AI 大腦會主動帶出惦記的話題破冰（例如：昨天的待辦、休假關心、昨晚聊到的話題）。
   - 當使用者一接話，進入連續對話後，**懸念自動閉嘴銷帳**，絕不跳痛插嘴打斷節奏！
2. **🧠 0 元 CacheService 滾動記憶**：
   - 使用 Google Apps Script 原生 `CacheService.getUserCache()`。
   - 記錄最近 6 輪（12 句）純文字對話，90 分鐘（5400 秒）未對話自動洗白成新的一輪。
   - **完全不花資料庫費用**、不存沉重的 Flex 卡片或工具 JSON，防止 Thought Signature 暴走與 Token 膨脹！

## 🚀 快速安裝與使用

### 1. 放入短期快取記憶模組
```javascript
/**
 * 🧠 短期對話快取記憶 (CacheService 90分鐘滾動記憶)
 */
function getShortChatHistory_(userId) {
  if (!userId) return [];
  try {
    const raw = CacheService.getUserCache().get('CHAT_HIST_' + userId);
    const list = raw ? JSON.parse(raw) : [];
    return Array.isArray(list) ? list : [];
  } catch (e) { return []; }
}

function appendShortChatHistory_(userId, userText, modelText) {
  if (!userId || !userText || !modelText) return;
  try {
    const hist = getShortChatHistory_(userId);
    hist.push({ role: 'user', parts: [{ text: String(userText).substring(0, 400) }] });
    hist.push({ role: 'model', parts: [{ text: String(modelText).substring(0, 600) }] });
    // 只留最近 6 輪（12 則訊息），90 分鐘未對話自動過期清空
    const trimmed = hist.slice(-12);
    CacheService.getUserCache().put('CHAT_HIST_' + userId, JSON.stringify(trimmed), 5400);
  } catch (e) {}
}
```

### 2. 接進你的 Gemini / LLM 呼叫鏈
```javascript
// 1. 區分真人互動與背景腳本
const isInteractive = Boolean(userId && userQuery);
const shortHistory = isInteractive ? getShortChatHistory_(userId) : [];

// 2. 只有【真人互動】且【剛開場（短期記憶是空的）】才注入懸念！
let intentionInjected = false;
if (isInteractive && shortHistory.length === 0 && typeof getIntentionPromptInjection === 'function') {
  const inj = getIntentionPromptInjection(userQuery);
  if (inj) {
    systemPrompt += inj;
    intentionInjected = true;
  }
}

// 3. 把過去幾輪對話接在前面，最新這句放最後
const payloadContents = shortHistory.concat([
  { role: 'user', parts: [{ text: '使用者輸入：' + promptText }] }
]);

// 4. 拿到模型 output 後安全寫入記憶
if (isInteractive) {
  const cleanOut = output.replace(/<!--[\s\S]*?-->/g, '').trim();
  if (cleanOut && !cleanOut.startsWith('⚠️')) {
    appendShortChatHistory_(userId, userQuery, cleanOut);
    if (intentionInjected && typeof consumeIntention === 'function') {
      consumeIntention();
    }
  }
}
```

## 📄 License
MIT License
