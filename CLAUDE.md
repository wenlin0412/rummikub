# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概述

Rummikub Twist 線上桌遊，支援單機 AI 對戰與多人連線（MQTT WebSocket）。完全靜態，無需建置步驟。

## 開發與執行

```bash
# 本機開發（任選其一）
python -m http.server 8000
npx http-server
# 或直接用瀏覽器開啟 index.html（file:// 協定）
```

無自動化測試；測試方式為在瀏覽器手動操作，多人模式可開多個分頁模擬。

## 部署

上傳 `index.html` 到任何靜態主機（GitHub Pages、Vercel 等）即可。無需後端或設定。

## 程式架構

### 技術堆疊（均透過 CDN 載入，無 package.json）

- **Vue 3** Composition API（`setup()` 單一元件）
- **Tailwind CSS 4**
- **mqtt.js v5.10.1**（MQTT over WebSocket）

### 唯一程式檔案：`index.html`（~2153 行）

所有 CSS、JavaScript、HTML 標記均嵌入單一檔案。邏輯按行號分布如下：

| 行號區段 | 功能 |
|----------|------|
| 912–985 | 核心牌組驗證（`valSet`, `isGroupTwist`, `isRunTwist`） |
| 1076 | Vue 入口點（`createApp()`，`setup()` 開始） |
| 1078–1134 | UI 響應式狀態（手牌、選牌、借牌追蹤、計時器） |
| 1134 | 遊戲引擎物件（`engine`：phase, players, table, pool） |
| 1172 | MQTT 網路層（連線、訂閱、訊息處理） |
| 1264 | 牌池初始化（`createPool`，112 張牌） |
| 1615 | 動作處理（`doAction`，meld/draw/endTurn/undo） |
| 1811 | AI 邏輯（`runAI`，三難度） |
| 1968–2020 | UI 互動函式（`playSelected`, `addToExistingSet`, `splitFromSet`） |

### 關鍵架構決策

**Host Authority 模型（房主權威）**
- 僅房主執行授權遊戲引擎，驗證所有動作後廣播完整 `gameState`
- 其他玩家只發送 `gameAction`，不自行驗證
- MQTT 主題：廣播用 `rummikub-game/{roomCode}`，私訊用 `rummikub-game/{roomCode}/dm/{playerId}`
- Broker：HiveMQ（主）→ Mosquitto（備援）

**Tile ID 系統**
- 每張牌有唯一 `id`（0–111），動作以 ID 而非陣列索引傳遞
- 允許玩家本地端排列手牌（`localHandOrder`）而不影響遊戲狀態

**回合備份（Turn Backup）**
- 每回合開始前快照 `turnBackup`（完整桌面＋所有手牌）
- Undo 或逾時懲罰皆回滾至備份

**四種特殊 Joker 牌**
- `joker`（紅）：可代替任意牌，可被換走
- `double`（藍）：代表同組兩張相鄰數字
- `colorchange`（黑白）：僅用於順子，換色邊界
- `mirror`（黃）：對稱牌組，計 0 分

### 遊戲流程

```
lobby → waiting → playing → finished
```

AI 回合透過 `advanceTurn()` 在 800ms 延遲後呼叫 `runAI()`。
首次出牌需累積 ≥30 分（break ice）。
回合計時 120 秒，逾時自動 undo＋懲罰抽牌。
