# OpenClaw LCM 安裝配置指南

> 基於實戰部署經驗的 [lossless-claw](https://github.com/Martian-Engineering/lossless-claw) (LCM) 安裝、混搭模型配置、排障完整指南。

## 這是什麼

[lossless-claw](https://github.com/Martian-Engineering/lossless-claw) 是 [OpenClaw](https://github.com/openclaw/openclaw) 的 DAG 式對話壓縮插件，用階層摘要取代傳統的滑動窗口截斷，讓 AI **記住所有對話**而不遺失上下文。

本倉庫是一份 **OpenClaw AgentSkill**，包含：

- ✅ 從零開始的安裝步驟
- ✅ 混搭模型配置（主模型 Claude + 摘要用 MiniMax，省 20 倍成本）
- ✅ macOS launchd 環境變數設定（踩坑重點）
- ✅ 已知問題與解法（假陽性 auth error、provider 401、compaction 不觸發）
- ✅ 新機器部署 Checklist
- ✅ 實測數據（200+ summaries，零漂移，零 errors）

## 快速開始

### 1. 安裝插件

```bash
openclaw plugins install lossless-claw
openclaw plugins enable lossless-claw
```

### 2. 設為上下文引擎

在 `openclaw.json` 中加入：

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw"
    }
  }
}
```

> ⚠️ **不設 `contextEngine` 是最常見的坑** — 插件載入了但不會接管壓縮。

### 3. 混搭模型（可選，省成本）

```json
{
  "plugins": {
    "entries": {
      "lossless-claw": {
        "config": {
          "summaryModel": "minimax/MiniMax-M2.7-highspeed",
          "summaryProvider": "minimax",
          "expansionModel": "minimax/MiniMax-M2.7-highspeed",
          "expansionProvider": "minimax",
          "contextThreshold": 0.75
        }
      }
    }
  }
}
```

### 4. macOS launchd 環境變數

```bash
# 在 ~/Library/LaunchAgents/ai.openclaw.gateway.plist 的
# EnvironmentVariables 區段加入：
# <key>MINIMAX_API_KEY</key>
# <string>你的 API Key</string>

# 重啟 Gateway
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

### 5. 驗證

```bash
# 查看最新摘要
sqlite3 ~/.openclaw/lcm.db \
  "SELECT summary_id, model, created_at FROM summaries ORDER BY created_at DESC LIMIT 5;"
```

## 部署 Checklist

- [ ] `openclaw plugins install lossless-claw` 完成
- [ ] `plugins.slots.contextEngine: "lossless-claw"` 已設
- [ ] launchd plist 有對應 provider 的 env var（macOS）
- [ ] Gateway 已重啟
- [ ] 對話 3-5 輪後 DB 有新 summary
- [ ] model 欄位顯示正確的摘要模型
- [ ] err log 無 401 auth error

## 已知問題

| 問題 | 根因 | 解法 | PR |
|------|------|------|----|
| 假陽性 auth error | `stripAuthErrors()` 誤判對話內容 | 改 `pickAuthInspectionValue()` 回傳 `{}` | [#178](https://github.com/Martian-Engineering/lossless-claw/pull/178) |
| 混搭 provider 401 | `getApiKey()` 缺 config fallback | launchd env var + v0.5.1 升級 | [#179](https://github.com/Martian-Engineering/lossless-claw/pull/179) |
| models.json env-marker 覆蓋 | Gateway 重啟覆寫 apiKey 為字面量 | launchd env var 或改用 Anthropic | [ops-lcm-fix](https://github.com/catgodtwno1/ops-lcm-fix) |
| Compaction 不觸發 | 缺 `contextEngine` slot | 設 `plugins.slots.contextEngine` | — |

## 實測數據

| 機器 | 主模型 | LCM 模型 | Summaries | 漂移 | Errors |
|------|--------|---------|-----------|------|--------|
| Scott#1 | Claude | Haiku 4.5 | 正常 | — | Bug 1 假陽性（已 patch） |
| Scott#4 | Claude Opus | M2.7 HS | 200+ | 零 | 零 |
| Scott#2 | Claude Opus | M2.7 HS | 40+ | 零 | 零 |

## 作為 AgentSkill 使用

將 `SKILL.md` 複製到 `~/.openclaw/skills/ops-lcm-setup/` 目錄下：

```bash
mkdir -p ~/.openclaw/skills/ops-lcm-setup
cp SKILL.md ~/.openclaw/skills/ops-lcm-setup/
```

Agent 會在需要部署 LCM 時自動載入此 Skill。

## 相關倉庫

| 倉庫 | 說明 |
|------|------|
| [Martian-Engineering/lossless-claw](https://github.com/Martian-Engineering/lossless-claw) | LCM 上游倉庫 |
| [catgodtwno4/lossless-claw](https://github.com/catgodtwno4/lossless-claw) | Fork（含修復分支） |
| [openclaw/openclaw](https://github.com/openclaw/openclaw) | OpenClaw 主專案 |
| [catgodtwno4/openclaw-five-layer-memory-nas](https://github.com/catgodtwno4/openclaw-five-layer-memory-nas) | 五層記憶棧 NAS 部署指南 |
| [catgodtwno4/openclaw-dashboard](https://github.com/catgodtwno4/openclaw-dashboard) | OpenClaw Dashboard |
| [catgodtwno4/openclaw-browser](https://github.com/catgodtwno4/openclaw-browser) | 瀏覽器操作 Skill |
| [catgodtwno4/openclaw-im-control](https://github.com/catgodtwno4/openclaw-im-control) | 媒體傳送 Skill |

## 許可證

MIT
