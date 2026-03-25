# CLAUDE.md

此檔案為 Claude Code（claude.ai/code）在此專案（repository）中工作時的參考說明。

## 執行遊戲

無需建置工具（build tools）或套件管理器（package manager）。直接在瀏覽器（browser）開啟 `index.html`，或在本機（locally）啟動伺服器（server）：

```bash
python3 -m http.server 8080
# 然後開啟 http://localhost:8080
```

所有相依套件（dependencies）——Vue 3.4.21、Tailwind CSS、MQTT.js 5.10.1——均透過 CDN 載入。

## 架構（Architecture）

整個應用程式是單一檔案：`index.html`（約 1990 行），分為三個區段：

1. **HTML 模板（template）**（第 1–690 行）：使用 `v-if`／`v-for` 指令（directives）的 Vue 模板標記。遊戲階段（phases）——`lobby`、`waiting`、`playing`、`gameover`——控制哪個面板（panel）顯示。

2. **CSS**（第 41–160 行，位於 `<style>` 內）：低多邊形（low-poly）幾何風格的自訂類別（classes），如 `poly-tile`、`poly-shadow`、`poly-btn`。Tailwind 擴充了一套自訂暖色調色盤（color palette）。

3. **JavaScript**（第 691–1988 行，位於 `<script>` 內）：Vue 3 組合式 API（Composition API）的 `setup()` 函式，包含所有遊戲邏輯。

### JavaScript 各區段

| 行數 | 內容 |
|------|------|
| 695–933 | **驗證邏輯（Validation logic）** — `valSet`、`isGroupTwist`、`isRunTwist`、`isRunColorChange`、`isGroupMirror`、`isRunMirror`、`normalizeMirrorSet` |
| 937–996 | **響應式狀態（Reactive state）** — UI、網路（network）、遊戲、選取（selection）、計時器（timer）的 refs |
| 998–1029 | **計算屬性（Computed properties）** — `isMyTurn`、`opponents`、`displayedHand`、`timerWidth` |
| 1030–1068 | **MQTT 連線（connectivity）** — `connectMQTT`、`mqttSend`、`mqttSendDM` |
| 1069–1238 | **大廳動作（Lobby actions）** — `handleGo`、`startSoloGame`、`startGame`、`leaveRoom` |
| 1239–1440 | **主機權威模型（Authority model）** — `sanState`、`broadcastGS`、`updateLocalState`、`handleMQTTMessage` |
| 1440–1657 | **遊戲動作（Game actions）** — `doAction`、`processMeld`、`processEndTurn`、`processUndo`、`processDraw` |
| 1658–1705 | **AI 引擎（engine）** — `runAI`、`aiDraw`、`aiTryMeld` |
| 1706–1988 | **UI 互動（interactions）** — 手牌拖曳／捲動（drag/scroll）、牌塊選取（tile selection）、`sortHand`、`playSelected` |

### 主機權威模型（Authority Model）——多人模式（Multiplayer）

**主機（host）** 擁有權威性的 `engine` 物件（`{ phase, players, table, pool, currentTurn }`）。非主機的客戶端（non-host clients）絕不直接修改 `engine`。

- `sanState(pi)` — 產生針對特定玩家的狀態視圖（view），隱藏其他玩家的手牌內容
- `broadcastGS()` — 每次狀態異動（state mutation）後由主機呼叫；單人模式（solo mode）下直接更新本地狀態（local state）；多人模式下透過 MQTT 以私訊（DM）形式傳送 `gameState` 給各玩家
- 非主機客戶端發送 `{ type: "gameAction", ... }` 訊息；主機透過 `doAction()` 處理後再廣播（rebroadcast）

### 牌塊類型（Tile Types）——拉密變臉版（Rummikub Twist Variant）

一般牌塊格式為 `{ color: "red"|"blue"|"yellow"|"black", number: 1–13 }`。

特殊萬用牌（joker tiles）：
- `joker` — 萬用牌（wild card），得分 30
- `double` — 在順子（run）中佔兩個連續數字位置
- `colorchange` — 允許顏色在順子中途變換（change mid-run）
- `mirror` — 建立回文（palindrome）牌組，得分 0；必須置於奇數長度牌組的正中央

### 多人傳輸（Multiplayer Transport）

透過 WebSocket 使用 MQTT。依序嘗試兩個公開中繼伺服器（brokers）：
1. `wss://broker.hivemq.com:8884/mqtt`
2. `wss://test.mosquitto.org:8081/mqtt`

房間主題（Room topic）：`rummikub-game/{roomCode}`（4 位數字代碼）
私訊主題（DM topic）：`rummikub-game/{roomCode}/dm/{playerId}`

### AI 難度（Difficulty）

由 `aiDiff` ref 控制（`"easy"` ／ `"normal"` ／ `"hard"`）。`aiTryMeld` 內的成功門檻（threshold）分別為 `0.4 / 0.7 / 1.0`。`"hard"` 模式下，AI 會反覆執行 `aiTryMeld` 直到無法繼續出牌為止。
