# 🌸 LINE Bot Active Intention & 6-Turn Rolling Memory (GAS)

> English | [繁體中文](./README.zh-TW.md)

Give your LINE AI assistant more than cold, stateless Q&A. This lightweight pattern adds **active intention ice-breaking** at the start of a conversation and **zero-cost 90-minute rolling short-term memory** powered by Google Apps Script's native `CacheService`.

## ✨ Highlights

1. **🎭 Active Intention Ice-Breaking**:
   - When a new conversation starts, the AI naturally brings up a pending topic or follow-up you care about (e.g., yesterday's unfinished task, a check-in, or an open thread).
   - Once the user replies and enters a continuous back-and-forth, **the intention is automatically consumed and silenced** so the AI never interrupts the flow awkwardly.
2. **🧠 Zero-Cost `CacheService` Rolling Memory**:
   - Uses Google Apps Script's built-in `CacheService.getUserCache()`.
   - Retains the last **6 turns (12 messages)** of plain text and automatically expires after **90 minutes (5,400 seconds)** of inactivity.
   - **No database fees**, no heavy Flex cards or raw tool JSON stored — preventing Thought Signature bloat and runaway token costs.

## 🚀 Quick Start

### 1. Add the Short-Term Cache Memory Module

```javascript
/**
 * 🧠 Short-Term Chat Cache (CacheService 90-min Rolling Memory)
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
    // Keep only the last 6 turns (12 messages); expires after 90 min of inactivity
    const trimmed = hist.slice(-12);
    CacheService.getUserCache().put('CHAT_HIST_' + userId, JSON.stringify(trimmed), 5400);
  } catch (e) {}
}
```

### 2. Wire It into Your Gemini / LLM Call Chain

```javascript
// 1. Distinguish interactive human chat from background triggers
const isInteractive = Boolean(userId && userQuery);
const shortHistory = isInteractive ? getShortChatHistory_(userId) : [];

// 2. Inject active intention ONLY when interactive AND opening a fresh session (empty short-term memory)
let intentionInjected = false;
if (isInteractive && shortHistory.length === 0 && typeof getIntentionPromptInjection === 'function') {
  const inj = getIntentionPromptInjection(userQuery);
  if (inj) {
    systemPrompt += inj;
    intentionInjected = true;
  }
}

// 3. Prepend rolling history before the latest user message
const payloadContents = shortHistory.concat([
  { role: 'user', parts: [{ text: 'User input: ' + promptText }] }
]);

// 4. Safely persist memory and consume intention after a valid model response
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
[MIT License](./LICENSE)
