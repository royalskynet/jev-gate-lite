# Jev Gate Lite

**繁體中文** · [English](README.en.md)

Jev Gate 的精簡版：只用雲端 Jev 判斷，不含本機 Laya 模型。

它給 Claude Code 與 Codex 這些東西：

| 元件 | 做什麼 |
|---|---|
| 混合壓縮 | 預設 context 到 60% 或 200K token 時自動發動。Jev 逐筆判斷哪些工具呼叫與輸出可以刪，留下的內容一字不改；省不到 30% 就交回內建壓縮做摘要 |
| 工具輸出裁剪 | 長日誌、搜尋結果只留和眼前工作相關的段落 |
| MCP server | `jev_evaluate`、`jev_apply_policy` 兩個工具，給分類與審查建議（advisory，不是授權） |
| subagent 分級 hook | 預留介面，目前不改模型（見下文） |

## 混合壓縮怎麼運作

### 什麼時候發動

每輪對話結束時檢查一次，下面兩個條件**先到的那個**觸發，不用手動下指令：

| 條件 | 預設 | 變數 |
|---|---|---|
| context window 用量百分比 | 60% | `JEV_COMPACT_AT_PERCENT` |
| 絕對 token 數 | 200000（200K） | `JEV_COMPACT_AT_TOKENS`，設 0 關掉 |

- 200K window：60%＝120K，百分比先到。
- 1M window：60%＝600K，200K 先到。長 context 品質在 600K 之前就下滑，所以多一道絕對上限。

Claude Code 自己撞牆觸發的壓縮、手動 `/compact` 也走同一套流程。

### 一次壓縮做什麼

1. 每個工具呼叫配一題 Jev System One 判斷（約 350 in／35 out token），整批一次送出。
2. 只刪工具呼叫與工具輸出；使用者與助手的文字不刪、不截斷。呼叫和它的結果一起刪，不拆開。
3. 第一輪和最近 6 輪固定保留。
4. 留下的內容逐字保留，沒有改寫或摘要。

### 什麼時候交給內建壓縮

下面任一情況，原訊息原樣交回，Claude Code 內建壓縮接手做摘要：

| 情況 | 預設 | 變數 |
|---|---|---|
| 省下的比例太少 | 不到 30% | `JEV_COMPACT_MIN_REDUCTION_PCT` |
| 省下的字數太少 | 不到 4000 字元 | `JEV_COMPACT_MIN_REDUCTION` |
| 上次 Jev 壓縮後留下太多 | 超過 60000 token | `JEV_COMPACT_RESET_ABOVE` |
| Jev 判斷失敗、逾時、金鑰錯 | — | — |

第三條的原因：Jev 只刪工具輸出，文字會一輪輪累積。殘量過線時先用內建摘要清一次，之後 Jev 再接手。

失敗方向是「保留 context」，不會因為 Jev 出錯而丟內容。

### 設計依據

混合法依據 JetBrains Research 的論文 [The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management](https://arxiv.org/abs/2508.21433)（Lindenbauer 等，arXiv 2508.21433）。論文在 SWE-agent／SWE-bench Verified 上比較五種模型設定：

- 只遮蔽舊的工具輸出（observation masking），成本比不處理少一半，解題率和 LLM 摘要持平，有時略高。
- 以遮蔽為主、較晚才做摘要的混合法，成本比只遮蔽再少 7%，比只摘要少 11%。

Jev 逐筆刪工具輸出就是這裡的「遮蔽」，只是改成由 Jev 判斷哪些可以刪，不照固定輪數刪。內建壓縮負責「晚期摘要」。

殘量門檻 60000 token 來自 Claude Code transcript 實測（200K 視窗、壓縮前 140K–200K 的樣本，下一輪 context 中位數）：

| 同一 session 第幾次壓縮 | 只用 Jev | 只用內建 |
|---|---|---|
| 1 | 47K | 39K |
| 2 | 58K | 44K |
| 3 | 68K | 48K |
| 4 次以上 | 75K | 51K |

Jev 每壓一次殘量就多一截，因為對話文字一直累積。所以殘量過 60K 時插一次內建摘要清掉，之後再回到 Jev。我們也試過「Jev 之後只摘要中段」，結果反而多出 21%，因為內建摘要會把最近讀過的檔案帶回來，所以沒有採用。

## 工具輸出裁剪怎麼運作

工具回傳之後、進入模型 context 之前裁剪：

- 長輸出切成每塊 16000 字元，每次最多判斷 8 塊，其餘原樣通過（不截斷）。
- Jev 給「刪」的機率 ≥ 0.7 才刪。這個門檻用實際工具輸出校準過：0.6 會刪到埋在雜訊裡的錯誤行。
- 每次判斷逾時 2000 ms（`JEV_TIMEOUT_MS`），逾時就保留原文。
- `JEV_PRUNE_ENABLED=0` 關掉，工具輸出完全不送 Jev。

## subagent 分級 hook

`agent.spawn` hook 的設計是依任務難度挑 subagent 模型（`JEV_TRIAGE_MODEL_CHEAP`／`MEDIUM`／`HARD` 可改）。**目前 hook 裡沒有接 Jev 連線，判斷一律 fail-open 成「維持原模型」，所以實際上不改任何 subagent 的模型。** 高風險、低信心、大 context 的情況設計上也不降級。`JEV_TRIAGE_ENABLED=0` 可整個關掉。

## 和完整版差在哪

完整版 [jev-gate-mcp](https://github.com/royalskynet/jev-gate-mcp) 另有本機 Laya 小模型、cascade server，以及 OpenMausBot／Hermes／OpenClaw 整合。精簡版拿掉這些，所以：

- 不用下載約 540 MB 的本機模型，也沒有 `@receptron/laya` 依賴。
- 每次判斷都打雲端 Jev，可能計費。
- `JEV_PROVIDER=laya` 會直接報錯 `provider laya is not included in this build`。

## 安裝

需要 Node.js 22 或 24，以及一把 Jev 金鑰（OpenCode Zen 免費層或 OpenRouter）。

```sh
git clone https://github.com/royalskynet/jev-gate-lite.git
cd jev-gate-lite
npm install
npm test
```

### Claude Code 插件（壓縮＋裁剪）

在 `~/.claude/settings.json` 註冊成 local marketplace：

```json
"extraKnownMarketplaces": {
  "jev-gate": { "source": { "source": "directory", "path": "/path/to/jev-gate-lite" } }
},
"enabledPlugins": { "jev-gate@jev-gate": true }
```

或單一 session 試用：`claude --plugin-dir /path/to/jev-gate-lite`。下一個 session 載入才生效。

### MCP server（判斷工具）

```sh
claude mcp add jev_gate -s user \
  -e JEV_PROVIDER=openrouter -e JEV_MODEL=typesafe/jev-1.13 \
  -e JEV_API_KEY_FILE=/path/to/OPENROUTER_API_KEY.txt \
  -- node /path/to/jev-gate-lite/gateway/src/stdio.js
```

server 名稱要放在 `-e` 前面。驗證：`claude mcp list` 看到 `jev_gate` ✓ connected。

Codex：repo 內的 `.codex/config.toml` 已設好 MCP，把 repo 標成 trusted 即可。

## 金鑰與 provider

| 設定 | 預設 |
|---|---|
| 金鑰檔 | `~/.creds/kv/OPENCODE_ZEN_API_KEY.txt`（`JEV_API_KEY_FILE` 可改） |
| endpoint | `https://opencode.ai/zen/v1/systemone`（Zen 免費層） |
| 改走 OpenRouter | `JEV_ENDPOINT=https://openrouter.ai/api/v1/systemone`、`JEV_MODEL=typesafe/jev-1.13` |

Zen 免費層有額度，也可能被上游限流。失敗時會冷卻並改問 OpenRouter 備援（`~/.creds/kv/OPENROUTER_SECOND_API_KEY.txt`，`JEV_ZEN_BACKUP_KEY_FILE` 可改）；沒有備援金鑰就 fail closed。

金鑰只從檔案讀，不經工具參數、不寫進日誌。送出前的 state 會先遮蔽已知格式的金鑰。

## 會寫到本機的檔案

| 檔案 | 內容 | 關掉／改路徑 |
|---|---|---|
| `~/.jevgate/usage*.jsonl` | 每次 Jev 呼叫的 token 數與狀態，不含內容 | `JEV_USAGE_LOG=off` |
| `~/.jevgate/teacher-corpus.jsonl` | MCP server 每次判斷的 state、題目與答案（金鑰已遮蔽，檔案權限 0600） | `JEV_TEACHER_CORPUS=/dev/null` |
| `~/.jevgate/zen-free-status.json` | Zen 冷卻狀態 | — |
| `~/.openclaw/state/jev-prune.log` | 裁剪紀錄 | `JEV_LEDGER` |
| `~/.openclaw/state/jev-compact.log` | 壓縮紀錄 | — |

teacher corpus 會存對話內容。不想留就設 `JEV_TEACHER_CORPUS=/dev/null`。

## 常用開關

| 變數 | 作用 |
|---|---|
| `JEV_COMPACT_ENABLED=0` | 關掉混合壓縮，只用內建壓縮 |
| `JEV_COMPACT_PIN_LAST` | 固定保留最近幾輪，預設 6 |
| `JEV_COMPACT_BACKEND` | `auto`（預設，依 transcript 格式選）、`anthropic`、`openai` |
| `JEV_PRUNE_ENABLED=0` | 關掉工具輸出裁剪 |
| `JEV_TRIAGE_ENABLED=0` | 關掉 subagent 分級 hook |
| `JEV_KILL_SWITCH=1` | MCP server 的判斷工具全部停用（插件 hook 不受影響） |
| `JEV_UI_LOG=1` | 在 Claude Code 介面顯示 hook 紀錄 |

觸發與交棒門檻見上面「混合壓縮怎麼運作」。

不要同時啟用這個插件和獨立的 fast-jev-compaction 插件：兩個都會回應同一個壓縮事件。偵測到另一個已啟用時，這個會自動讓位。

## 授權

MIT，見 [LICENSE](LICENSE)。
