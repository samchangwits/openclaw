# OpenClaw 本機設定紀錄

> Branch: `samTestOpenClaw`  
> 最後更新: 2026-03-29

---

## 環境架構

```
Mac Studio (32GB RAM)
├── Ollama（本機 LLM）
│   ├── qwen3-coder:30b  ← 目前預設，約 21GB
│   ├── llama3.1:latest  （8b）
│   └── llama3.1:8b
├── Docker
│   └── openclaw-openclaw-gateway-1  (port 18789)
│       └── 透過 host.docker.internal:11434 連本機 Ollama
└── Tailscale IP: 100.117.179.68
```

---

## 重要路徑

| 用途 | 路徑 |
|------|------|
| 設定檔 | `/Users/zhangheli/.openclaw/openclaw.json` |
| Config 目錄 | `/Users/zhangheli/.openclaw/` |
| Workspace | `/Users/zhangheli/openclaw-workspace/` |
| Gateway 原始碼 | `/Volumes/D/Documents/OpenClaw/` |
| 環境變數 .env | `/Volumes/D/Documents/OpenClaw/.env` |
| Ollama 模型 | `~/.ollama/models/`（約 22GB） |
| Ollama 安裝位置 | `/Applications/Ollama.app/` |
| Ollama 執行檔 | `/usr/local/bin/ollama`（軟連結） |

---

## 常用指令

### Gateway 管理

```bash
# 重啟 gateway
cd /Volumes/D/Documents/OpenClaw && docker compose restart openclaw-gateway

# 查看 gateway 狀態
docker ps --format "table {{.Names}}\t{{.Status}}" | grep gateway

# 查看 gateway 日誌（最後 30 行）
docker logs openclaw-openclaw-gateway-1 --tail 30
```

### Ollama 管理

```bash
# 查看目前載入的模型
ollama ps

# 強制卸載模型（釋放記憶體）
ollama stop qwen3-coder:30b

# 列出所有已安裝模型
ollama list
```

### 切換模型（Telegram 指令）

```
/model ollama/qwen3-coder:30b   ← 本機 30B（預設）
/model ollama/llama3.1:8b       ← 本機 8B（輕量快速）
/model openai/gpt-4o            ← 雲端 GPT-4o（需 API key）
/model openai/gpt-4o-mini       ← 雲端 GPT-4o-mini
```

---

## 重要設定說明

### docker-compose.yml 修改項目

- 新增 `extra_hosts: host.docker.internal:host-gateway`
  → 讓 Docker 容器能連到本機 Ollama（11434 port）

### openclaw.json 重點設定

- `agents.defaults.model.primary`: `ollama/qwen3-coder:30b`（預設模型）
- `models.providers.ollama.baseUrl`: `http://host.docker.internal:11434`
- `channels.telegram.dmPolicy`: `pairing`（需配對才能使用）
- `gateway.bind`: `lan`（區網可連）
- `gateway.auth.token`: 見 .env 檔

### contextWindow 設定原則

OpenClaw 要求最低 16000 tokens，否則會自動 fallback 到 GPT-4o：

| 模型 | contextWindow | maxTokens |
|------|--------------|-----------|
| llama3.1:8b / latest | 16384 | 4096 |
| qwen3-coder:30b | 32768 | 8192 |

---

## 遠端存取

| 方式 | 說明 |
|------|------|
| **Telegram Bot** | 外出完全可用，不需額外設定 |
| **VNC（Tailscale）** | MacBook Air 連 `100.117.179.68`，需開 Tailscale |
| **Control UI（區網）** | `http://192.168.1.118:18789`，需輸入 token |
| **Control UI（外出）** | SSH tunnel: `ssh -N -L 18789:127.0.0.1:18789 zhangheli@100.117.179.68` |

> ⚠️ Control UI 需要先透過 Telegram 配對（`dmPolicy: pairing`）

---

## 注意事項

- Mac Studio 不要開睡眠，否則 Telegram bot 會斷線
  → 系統設定 → 省電 → 睡眠設為「永不」
- Ollama 閒置 5 分鐘後會自動卸載模型（釋放記憶體）
- 30B 模型第一次載入約需 15~30 秒
- 同時只能有一個模型在記憶體，切換時自動排擠
- D 碟剩餘空間約 49GB，系統碟剩餘約 134GB，新模型建議裝在系統碟

---

## 常見問題 Q&A

### Q: 外出時能用 Telegram 連回本機 Ollama 嗎？
**可以。** Telegram Bot 是主動連出到 Telegram 伺服器，不需要 Mac Studio 有公網 IP。
只要 Mac Studio 保持開機不睡眠，Telegram Bot 不管你在哪裡都能正常使用 Ollama。

### Q: 同時用 Telegram（8b）和 Mac app（30b），Mac Studio 會吃不消嗎？
**不會。** Ollama 不會同時在記憶體裡放兩個模型，切換時會自動排擠：
- 8b 約佔 5GB，30b 約佔 21GB，32GB RAM 不夠同時放兩個
- Ollama 會自動卸載舊模型、載入新模型（需幾秒切換時間）
- 不會當機，只是切換時要稍等

### Q: Mac app 操作會比 Telegram 快嗎？
**不會。** 速度跟介面無關，瓶頸在模型本身的推理速度：

| 模型 | 速度（tok/s） | 回應 "hi" 約需 |
|------|--------------|---------------|
| llama3.1:8b | 30~50 tok/s | 3~5 秒 |
| qwen3-coder:30b | 5~10 tok/s | 20~60 秒 |
| GPT-4o（雲端） | 串流輸出 | 2~5 秒 |

**建議使用策略：**
- 日常聊天、快速問答 → `llama3.1:8b`（預設，秒回）
- 寫程式、複雜分析 → `/model ollama/qwen3-coder:30b`（慢但準）
- 需要最強效果 → `/model openai/gpt-4o`（雲端，快又強）

### Q: Ollama 模型路徑在哪？可以搬到 D 碟嗎？
- 目前位置：`~/.ollama/models/`（系統碟，約 22GB）
- 系統碟剩 134GB，D 碟剩 49GB，**建議留在系統碟**，D 碟反而更快滿
- 若要搬移需設定環境變數 `OLLAMA_MODELS`，但現在不需要

### Q: Ollama 常駐佔用 20GB+ 記憶體正常嗎？
**正常。** 例如 qwen3-coder:30b 量化後約 21GB，`100% GPU` 表示用 Apple Silicon 統一記憶體跑，這是正常現象。閒置 5 分鐘後會自動卸載釋放記憶體。

### Q: contextWindow 設多少合適？
OpenClaw 要求最低 16000 tokens，設太小會自動 fallback 到 GPT-4o：
- 8b 模型設 `16384` 即可（再大會讓模型佔用更多記憶體且變慢）
- 30b 模型設 `32768`（有足夠記憶體支撐）

### Q: 出現 "Compacting context..." 是什麼意思？
**正常現象，不需擔心。** 這是 OpenClaw 的自動 context 壓縮機制：
- 每次對話累積的 tokens 接近 `contextWindow` 上限時自動觸發
- 以 8b（contextWindow: 16384）為例，約累積 12,000+ tokens 時觸發
- 壓縮過程：把舊對話「摘要」後刪除，騰出空間給新對話

**影響：**
- 很久以前說的內容可能被摘要掉，bot 可能「忘記」早期細節
- 當前問題不受影響，照常回答
- 壓縮本身需要幾秒

**如何減少觸發：**
- 定期 `/reset` 清掉對話（推薦）
- 或把 contextWindow 調大（但會讓 8b 變慢）

### Q: `/reset` 之後預期看到什麼？怎麼確認真的清掉了？
在 Telegram 發送 `/reset` 後，bot 會回覆類似：
```
🔄 Session reset.
```
或是一段確認訊息表示已重置。

**確認清掉的方法：**
1. 發送 `/status`，看 `Tokens: X in / X out` 數字有沒有歸零或變很小
2. 問 bot「你記得我剛才說了什麼嗎？」→ 如果真的清掉，它會說不記得
3. 看 `Context: X/128k (X%)` 百分比有沒有降回很低

**什麼時候需要 `/reset`：**
- 出現 "Compacting context..." 時
- bot 開始回答奇怪或前後矛盾時
- 想開始全新對話、切換話題時
