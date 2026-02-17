這是一份經過潤色後的 **`README_MCP.md`**，特別強化了技術架構的描述與操作手冊的實用性，適合放在你的 GitHub 專案中作為開發文件。

---

# 🚀 Claude MCP 跨環境整合指南：從 Windows 宿主機到 Dev Container

本文件記錄了如何配置 **Model Context Protocol (MCP)**，使 Claude AI 能夠穿透 Windows 宿主機環境，直接對運行在 **Docker Dev Container** 內的 **Kubernetes (Kind)** 叢集進行動態診斷與操作。

---

## 🛠️ 技術架構

為了讓 Claude 具備動態測試能力，我們建立了一條自動化指令通道：
**Claude Desktop (Windows)** ➔ **MCP Server (uvx)** ➔ **Docker Exec** ➔ **Dev Container (kubectl)** ➔ **Kind Cluster**

---

## 📋 配置步驟

### 1. 宿主機環境準備

確保 Windows 已經安裝以下核心組件：

* **uv**: Python 的高速啟動器與包管理工具。
* **Docker Desktop**: 確保 Docker 守護程序正在運行。

### 2. 打通 K8s 認證隧道 (Critical)

由於 Kind 叢集運行在 Container 內部，必須將憑證（kubeconfig）同步至 Container 內的 root 路徑，否則 `kubectl` 會因找不到 API Server 而報錯（預設指向 `localhost:8080`）。

請在 **Windows PowerShell** 執行以下自動化指令：

```powershell
# 建立 Container 內的認證目錄
docker exec vibe-dev-container mkdir -p /root/.kube

# 從 Kind 提取配置並精準傳送至 Container 內部
docker exec vibe-dev-container kind get kubeconfig --name dynamic-alerting-cluster | Out-File -FilePath "$env:TEMP\k8s_config" -Encoding ascii
docker cp "$env:TEMP\k8s_config" vibe-dev-container:/root/.kube/config

# 驗證連通性 (若顯示 Pod 列表則代表成功)
docker exec vibe-dev-container kubectl get pods -A

```

### 3. Claude Desktop 設定檔

修改 `%APPDATA%\Claude\claude_desktop_config.json`，加入官方 K8s MCP 伺服器並指定 Container 映射：

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "uvx",
      "args": ["mcp-server-kubernetes"],
      "env": {
        "DOCKER_CONTAINER": "vibe-dev-container",
        "KUBECONFIG": "/root/.kube/config"
      }
    }
  }
}

```

---

## 🔍 常見問題排除 (Troubleshooting)

| 錯誤訊息 | 原因分析 | 解決方案 |
| --- | --- | --- |
| `spawn uv ENOENT` | Windows PATH 找不到 uv 執行檔 | 安裝 uv 或改用絕對路徑指定 `uv.exe` |
| `Request timed out` | 首次啟動時 uv 正在下載 100+ 套件 | 完全退出並重啟 Claude，讓快取生效 |
| `connection refused` | Container 內缺乏有效的 kubeconfig | 重新執行上述「打通 K8s 認證隧道」指令 |
| `No solution found` | MCP 套件名稱不精確 | 確保使用官方名稱 `mcp-server-kubernetes` |

---

## 🌟 整合後的運作能力

現在，你可以直接在 Claude 對話框中要求 AI 執行以下任務：

* **叢集巡檢**：「檢查 `monitoring` namespace 下的所有 Pod 是否正常。」
* **日誌追蹤**：「當 `db-a` 的 MariaDB 重啟時，幫我讀取它的日誌並分析 crash 原因。」
* **動態驗證**：「執行 `make status` 並根據結果驗證多租戶告警規則是否已載入。」

---