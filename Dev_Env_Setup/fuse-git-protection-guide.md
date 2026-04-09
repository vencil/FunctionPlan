---
title: "FUSE 掛載環境 Git Phantom Lock 防治指南"
tags: [documentation, cross-platform]
audience: [developer, ai-agent]
version: v1.0.0
lang: zh
---
# FUSE 掛載環境 Git Phantom Lock 防治指南

> Windows + VM/Container 跨層掛載開發環境下，消除 Git phantom lock 的完整方案。
> 適用於任何使用 Dev Container、Docker bind mount、VirtioFS/9P 掛載的專案。

## 問題本質

當開發環境架構為：

```
Windows NTFS  ──VirtioFS/FUSE──▶  Linux VM  ──bind mount──▶  Docker Container
```

Git 的 `.git/index.lock` 與 `HEAD.lock` 機制會在三個層級的交界處產生衝突：

1. **權限不對稱**：某一層建立的 lock file，其他層可能因 FUSE 權限語義不同而無法刪除（`Operation not permitted`）
2. **背景存取競爭**：Windows 端的 VS Code Git 整合、Defender 即時掃描、Search Indexer 會在背景讀寫 `.git/`，與 VM/Container 裡的 git 操作搶 lock
3. **FUSE inode 不同步**：FUSE 的 `ctime` 經常延遲同步，導致 Git 誤判大量檔案被修改，觸發不必要的 index 更新和 lock

## 防治架構總覽

```
┌─────────────────────────────────────────────────────┐
│ 預防層（降低 Lock 發生機率）                          │
│                                                     │
│  ① VS Code Git 開關    ─ 專案級，不影響其他專案       │
│  ② Git Config 調校     ─ includeIf 路徑條件式        │
│  ③ .gitattributes      ─ 統一 LF，減少 diff 雜訊     │
│  ④ Windows 端降噪      ─ Defender + Search 排除      │
│  ⑤ Repo-local config   ─ fsmonitor/trustctime 關閉   │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│ 診斷層（遇到 Lock 時安全處理）                        │
│                                                     │
│  ⑥ Lock 診斷腳本       ─ 檢查年齡 + 活躍程序再決定   │
│  ⑦ Windows 端 fallback ─ Remove-Item 強制清理        │
└─────────────────────────────────────────────────────┘
```

---

## 第一部分：套用步驟（人工操作）

### 步驟 1：建立 `.gitattributes`（repo 內，一次性）

在專案根目錄建立 `.gitattributes`，確保 repo 內統一使用 LF 換行。這消除了 CRLF/LF 混用在 FUSE 上造成的額外 index 壓力。

```gitattributes
* text=auto eol=lf

# Source code
*.sh    text eol=lf
*.py    text eol=lf
*.go    text eol=lf
*.js    text eol=lf
*.jsx   text eol=lf
*.ts    text eol=lf
*.html  text eol=lf
*.css   text eol=lf

# Config / Data
*.yaml  text eol=lf
*.yml   text eol=lf
*.json  text eol=lf
*.toml  text eol=lf

# Documentation
*.md    text eol=lf
*.txt   text eol=lf

# Build
Dockerfile* text eol=lf
Makefile    text eol=lf

# Binary
*.png   binary
*.jpg   binary
*.ico   binary
*.gz    binary
*.zip   binary
```

根據專案實際使用的檔案類型增減項目。

### 步驟 2：設定 Repo-local Git Config（repo 內，一次性）

在 repo 目錄內執行：

```bash
git config core.fsmonitor false
git config core.untrackedCache false
git config core.trustctime false
git config core.filesRefLockTimeout 1500
```

這些寫入 `.git/config`（不會進 repo），從任何層級存取這個 repo 時都會生效。

各設定的作用：

| 設定 | 預設值 | 改為 | 原因 |
|------|--------|------|------|
| `core.fsmonitor` | true | **false** | FUSE 上的 file watcher 是 phantom lock 最大來源 |
| `core.untrackedCache` | true | **false** | 減少對 FUSE 的 `stat()` 呼叫量 |
| `core.trustctime` | true | **false** | FUSE 的 inode ctime 經常不同步，Git 會誤判大量檔案被修改 |
| `core.filesRefLockTimeout` | 0 | **1500** | Lock 被短暫佔用時等 1.5 秒重試（需 Git >= 2.39，低版本忽略此設定） |

### 步驟 3：Windows 端 Git includeIf（Windows 全域，一次性）

如果你有多個專案、只有部分走 FUSE 掛載，用 `includeIf` 讓調校只對特定 repo 生效。

**3a.** 建立調校檔 `%USERPROFILE%\gitconfig-fuse-tuning`：

```ini
[core]
    fsmonitor = false
    untrackedCache = false
    trustctime = false
    filesRefLockTimeout = 1500
```

**3b.** 在 `%USERPROFILE%\.gitconfig` 末尾加入：

```ini
[includeIf "gitdir:C:/Users/<USERNAME>/<REPO_NAME>/"]
    path = ~/gitconfig-fuse-tuning
```

> 將 `<USERNAME>` 替換為你的 Windows 使用者名稱，`<REPO_NAME>` 替換為 repo 目錄名。
> 路徑用正斜線 `/`、結尾需有 `/`。
> 多個 FUSE 掛載的 repo 可重複此區塊，指向同一個 tuning 檔案。

**驗證**：

```powershell
cd C:\Users\<USERNAME>\<REPO_NAME>
git config core.fsmonitor
# 應輸出：false

cd C:\Users\<USERNAME>\其他專案
git config core.fsmonitor
# 應輸出空白（代表使用系統預設）
```

> **步驟 2 和步驟 3 的關係**：步驟 2（repo-local）保護從 VM/Container 端的存取；步驟 3（includeIf）保護從 Windows 端的存取。兩者互為保險，建議都設。如果你只在一端做 git 操作，只設該端即可。

### 步驟 4：Windows Defender 排除（Windows，一次性）

開啟**系統管理員** PowerShell：

```powershell
Add-MpPreference -ExclusionPath "C:\Users\<USERNAME>\<REPO_NAME>\.git"
```

這讓 Defender 即時掃描不再碰 `.git/` 目錄。只排除 `.git/`，原始碼仍受保護。

**驗證**：

```powershell
Get-MpPreference | Select-Object -ExpandProperty ExclusionPath
```

### 步驟 5：Windows Search Indexer 排除（Windows，可選）

「設定」→「索引選項」→「修改」→ 取消勾選 `.git` 子目錄。

影響較小，但也是一個背景存取源。

### 步驟 6：VS Code 專案級 Git 隔離

在 repo 的 `.gitignore` 中確認 `.vscode/*` 已排除（保留 `!.vscode/extensions.json` 等需要共享的設定）。

建立切換腳本（完整範本見附錄 A），提供以下介面：

```bash
python scripts/ops/vscode_git_toggle.py off   # 關閉 VS Code Git（Agent 模式）
python scripts/ops/vscode_git_toggle.py on    # 開啟 VS Code Git（手動模式）
python scripts/ops/vscode_git_toggle.py       # 查看目前狀態
```

原理：腳本讀寫 `.vscode/settings.json` 中的 `git.enabled`、`git.autoRepositoryDetection`、`git.autofetch`。VS Code 會即時 hot-reload 此檔案，不需重啟。

建議搭配 Makefile 快捷指令：

```makefile
.PHONY: vscode-git-off
vscode-git-off: ## 關閉 VS Code Git（Agent session 用）
	@python3 scripts/ops/vscode_git_toggle.py off

.PHONY: vscode-git-on
vscode-git-on: ## 開啟 VS Code Git（手動開發用）
	@python3 scripts/ops/vscode_git_toggle.py on
```

### 步驟 7：建立 Lock 診斷腳本

完整範本見附錄 B。腳本應具備以下行為：

1. 搜尋 `.git/` 下所有 `*.lock` 檔案
2. 顯示每個 lock 的年齡（秒數）
3. 檢查是否有活躍的 git 程序
4. 帶 `--clean` 參數時，僅清理 **>30 秒且無活躍 git 程序** 的 stale lock
5. 若 VM 無法刪除，輸出 Windows 端的 `Remove-Item` 指令提示

搭配 Makefile：

```makefile
.PHONY: git-lock
git-lock: ## 診斷 .git lock 殘留 (加 ARGS="--clean" 安全清理)
	@bash scripts/ops/git_check_lock.sh $(ARGS)
```

---

## 第二部分：日常工作流

### 場景 A：使用 AI Agent (Cowork / Claude Code) 開發

```
開始 session
  └→ Agent 自動執行 vscode_git_toggle.py off
       └→ VS Code Git 立即靜音
       └→ 所有 git 操作由 Agent 在 VM 內單一入口執行
       └→ 如遇 lock → make git-lock ARGS="--clean"

結束 session
  └→ 你手動執行 make vscode-git-on（或下次在 VS Code 發現 Git 面板沒反應時再開）
```

### 場景 B：不用 Agent，純手動在 VS Code 開發

```
VS Code 開啟專案
  └→ vscode_git_toggle.py on（如果上次 session 留在 off 狀態）
  └→ 正常使用 VS Code Git 面板
  └→ includeIf + repo-local config 仍在背景保護（fsmonitor off 等）
```

### 場景 C：遇到 Lock 錯誤

```
git commit 失敗："Unable to create '.git/index.lock': File exists"
  └→ make git-lock                     # 先診斷
  └→ make git-lock ARGS="--clean"      # 安全清理 stale lock
  └→ 若 VM 無法刪除 → 腳本輸出 Windows 端指令，用 PowerShell 執行
```

---

## 第三部分：AI Agent 指引

> 以下內容供 AI Agent 的 system prompt 或 CLAUDE.md 引用。

### Agent 起手式

每次 session 開始，在進行任何 git 操作之前：

```bash
python scripts/ops/vscode_git_toggle.py off
```

### Git 操作原則

1. **單一入口**：所有 git 寫入操作（add / commit / push / rebase）在同一個環境執行，不要跨層。讀取操作（status / log / diff）不受此限
2. **遇到 lock 不盲刪**：執行 `bash scripts/ops/git_check_lock.sh` 先診斷，確認是 stale lock 後再用 `--clean` 清理
3. **VM 無法刪除時**：使用 Windows MCP 的 `Remove-Item` 清理，腳本會輸出具體指令
4. **Release / Tag 操作**：如果環境頻繁出現 lock，考慮 `git clone --depth 1` 到本地 `/tmp/` 做 tag 操作，完全避開 FUSE 層
5. **Pre-commit hooks 失敗**：如果 hook 中斷留下 lock，同樣走診斷流程，不要跳過 hooks（`--no-verify`）

### Agent 結束時

理想情況下恢復 VS Code Git：

```bash
python scripts/ops/vscode_git_toggle.py on
```

但如果 session 非正常結束（斷線、超時），使用者會自行透過 `make vscode-git-on` 恢復。

---

## 第四部分：套用到新專案的 Checklist

當你有另一個使用 Dev Container + FUSE 掛載的專案，按以下清單逐項執行：

```
□  1. 建立 .gitattributes（從本指南步驟 1 複製，依專案調整檔案類型）
□  2. 設定 repo-local git config（步驟 2 的四行指令）
□  3. Windows 端 includeIf 加入新 repo 路徑（步驟 3，指向同一個 tuning 檔）
□  4. Defender 排除新 repo 的 .git/（步驟 4）
□  5. 複製 vscode_git_toggle.py 到新專案的 scripts/ 下
□  6. 複製 git_check_lock.sh 到新專案的 scripts/ 下
□  7. Makefile 加入 vscode-git-off / vscode-git-on / git-lock 三個 target
□  8. .gitignore 確認 .vscode/* 已排除
□  9. 如果有 CLAUDE.md 或 AI Agent 設定，加入 Agent 起手式指引
```

> 步驟 3 的 `gitconfig-fuse-tuning` 檔案是 Windows 全域共用的，只需建立一次。新專案只需在 `.gitconfig` 多加一個 `includeIf` 區塊。

---

## 附錄 A：vscode_git_toggle.py 完整範本

```python
#!/usr/bin/env python3
"""VS Code Git 整合開關 — 切換 .vscode/settings.json 中的 Git 設定。

用法：
  python vscode_git_toggle.py off   # 關閉 Git（Agent 模式）
  python vscode_git_toggle.py on    # 打開 Git（手動模式）
  python vscode_git_toggle.py       # 顯示目前狀態
"""

import json
import sys
from pathlib import Path

GIT_KEYS_OFF = {
    "git.enabled": False,
    "git.autoRepositoryDetection": False,
    "git.autofetch": False,
}
GIT_KEYS_ON = {
    "git.enabled": True,
    "git.autoRepositoryDetection": True,
    "git.autofetch": True,
}


def find_repo_root() -> Path:
    current = Path.cwd()
    for parent in [current, *current.parents]:
        if (parent / ".git").exists():
            return parent
    return Path(__file__).resolve().parent.parent.parent


def load_settings(path: Path) -> dict:
    if path.exists():
        try:
            with open(path, encoding="utf-8") as f:
                return json.load(f)
        except (json.JSONDecodeError, OSError):
            return {}
    return {}


def save_settings(path: Path, data: dict) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=4, ensure_ascii=False)
        f.write("\n")


def main() -> None:
    settings_path = find_repo_root() / ".vscode" / "settings.json"
    action = sys.argv[1].lower() if len(sys.argv) > 1 else "status"
    settings = load_settings(settings_path)

    if action == "off":
        settings.update(GIT_KEYS_OFF)
        save_settings(settings_path, settings)
        print("VS Code Git disabled (Agent mode)")
    elif action == "on":
        settings.update(GIT_KEYS_ON)
        save_settings(settings_path, settings)
        print("VS Code Git enabled (manual mode)")
    elif action == "status":
        enabled = settings.get("git.enabled", True)
        print(f"Git: {'on' if enabled else 'off'}")
    else:
        print(f"Unknown: {action}. Use: on / off", file=sys.stderr)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

## 附錄 B：git_check_lock.sh 完整範本

```bash
#!/bin/bash
# git_check_lock.sh — 安全診斷 .git lock 殘留
# 用法：
#   bash git_check_lock.sh           # 診斷
#   bash git_check_lock.sh --clean   # 診斷 + 清理 stale locks

set -euo pipefail

REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || echo ".")"
CLEAN_MODE="${1:-}"

mapfile -t LOCK_FILES < <(find "$REPO_ROOT/.git" -name "*.lock" 2>/dev/null)

if [ ${#LOCK_FILES[@]} -eq 0 ]; then
    echo "No lock files found."
    exit 0
fi

echo "Found ${#LOCK_FILES[@]} lock file(s):"
echo ""

NOW=$(date +%s)
HAS_STALE=false

for f in "${LOCK_FILES[@]}"; do
    if MTIME=$(stat -c %Y "$f" 2>/dev/null); then
        AGE=$(( NOW - MTIME ))
    else
        AGE=0
    fi
    REL_PATH="${f#"$REPO_ROOT"/}"

    if [ "$AGE" -gt 30 ]; then
        echo "  STALE: $REL_PATH (${AGE}s ago)"
        HAS_STALE=true
    else
        echo "  ACTIVE?: $REL_PATH (${AGE}s ago)"
    fi
done

echo ""
echo "--- Active git processes ---"
if pgrep -af "git" 2>/dev/null | grep -v "$0" | grep -v "pgrep" | head -5; then
    HAS_ACTIVE_GIT=true
else
    echo "(none)"
    HAS_ACTIVE_GIT=false
fi
echo ""

if [ "$CLEAN_MODE" = "--clean" ] && [ "$HAS_STALE" = true ] && [ "$HAS_ACTIVE_GIT" = false ]; then
    for f in "${LOCK_FILES[@]}"; do
        MTIME=$(stat -c %Y "$f" 2>/dev/null || echo "$NOW")
        AGE=$(( NOW - MTIME ))
        if [ "$AGE" -gt 30 ]; then
            if rm -f "$f" 2>/dev/null; then
                echo "Removed: ${f#"$REPO_ROOT"/}"
            else
                REL="${f#"$REPO_ROOT"/}"
                REL_WIN="${REL//\//\\}"
                echo "Cannot remove: $REL"
                echo "  Windows fallback: Remove-Item \"<REPO_WIN_PATH>\\$REL_WIN\" -Force"
            fi
        fi
    done
elif [ "$CLEAN_MODE" = "--clean" ] && [ "$HAS_ACTIVE_GIT" = true ]; then
    echo "Active git processes detected. Skipping cleanup."
elif [ "$HAS_STALE" = true ]; then
    echo "Run with --clean to remove stale locks."
fi
```

## 附錄 C：gitconfig-fuse-tuning 範本

```ini
[core]
    fsmonitor = false
    untrackedCache = false
    trustctime = false
    filesRefLockTimeout = 1500
```

## 延伸閱讀

- [Git `core.*` 設定文件](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corefsmonitor)
- [VirtioFS 技術文件](https://virtio-fs.gitlab.io/)
- [Git `includeIf` 條件式載入](https://git-scm.com/docs/git-config#_conditional_includes)
