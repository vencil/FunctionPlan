這份文件專門針對 **Windows 使用者**，解決了三大核心矛盾：

1. **資安 (Security)**：限制 AI Agent (Claude) 只能存取特定專案，防止其掃描全機敏感資料。
2. **效能 (Performance)**：解決 Windows 檔案系統 (NTFS) 與 Linux 核心之間的 I/O 效能瓶頸。
3. **相容性 (Compatibility)**：在 Windows 上完美運行原生 Linux (Ubuntu) 工具鏈 (K8s, Helm, Docker)。
---

# Vibe Coding：Windows 高效能安全 K8s 開發環境建置指南

**版本**：v1.0.0
**適用環境**：Windows 10/11 Pro (或是支援 WSL2 的 Home 版)
**硬體建議**：RAM 32GB 以上 (K8s 模擬需求)

## 1. 架構設計理念 (Architecture)

本配置採用**「混合掛載策略 (Hybrid Mount Strategy)」**，將檔案儲存與運算執行分離，以達到資安與效能的最佳平衡。

* **儲存層 (Storage)**：專案原始碼存放於 **Windows 使用者家目錄** (`C:\Users\%USERNAME%\Projects`)。
* *目的*：滿足 Claude Desktop 的「Home Directory Lock」安全機制，讓 AI 能讀寫代碼。


* **運算層 (Compute)**：開發環境運行於 **WSL2 Docker Dev Container**。
* *目的*：提供 100% 相容的 Ubuntu Linux 環境，執行 K8s (Kind)、Helm 等工具。


* **資料層 (Data)**：資料庫 (MariaDB/Prometheus) 儲存於 **Docker Named Volume**。
* *目的*：避開 NTFS 的 I/O 瓶頸，確保資料庫讀寫達到原生 Linux 速度。



---

## 2. 前置準備 (Prerequisites)

### 2.1 優化 WSL2 資源配置

為了避免 Docker 吃光 Windows 記憶體導致系統卡頓，需限制 WSL2 的資源上限。

1. 在 Windows 使用者目錄 (`C:\Users\%USERNAME%`) 建立檔案 `.wslconfig`。
2. 寫入以下內容 (針對 32GB RAM 機種建議)：
```ini
[wsl2]
memory=24GB
processors=8
swap=8GB

```


3. 開啟 PowerShell 執行 `wsl --shutdown` 使設定生效。

### 2.2 安裝必要軟體

1. **Docker Desktop**：
* Settings -> Resources -> WSL Integration -> 勾選你的 Ubuntu 發行版。


2. **VS Code**：安裝 `Dev Containers` 擴充套件。
3. **Node.js (LTS)**：Windows 本機需安裝 (供 Claude MCP Server 使用)。
4. **Claude Desktop App**。

---

## 3. 專案目錄配置 (Project Setup)

**⚠️ 關鍵步驟：** 為了符合 Claude 的資安限制，**不要**將專案放在 WSL (`\\wsl$`) 或非系統碟 (`D:\`)。

1. 在 Windows 檔案總管，前往：`C:\Users\<你的使用者名稱>\`。
2. 建立專案資料夾，例如：`vibe-k8s-lab`。
3. (選用) 若有舊專案，請將代碼複製到此資料夾中。

---

## 4. AI Agent 安全存取設定 (Claude Configuration)

配置 MCP (Model Context Protocol) 讓 Claude 獲得對上述資料夾的「讀寫權限」，但**鎖定**其無法存取其他目錄。

1. 開啟 `%APPDATA%\Claude\claude_desktop_config.json`。
2. 填入以下內容 (請替換 `<你的使用者名稱>`):

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "cmd.exe",
      "args": [
        "/c",
        "npx",
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "C:\\Users\\<你的使用者名稱>\\vibe-k8s-lab"
      ]
    }
  },
  "preferences": {
    "coworkScheduledTasksEnabled": false,
    "sidebarMode": "task",
    "localAgentModeTrustedFolders": [
      "C:\\Users\\<你的使用者名稱>\\vibe-k8s-lab"
    ]
  }
}

```

* **技術細節**：使用 `cmd.exe /c npx` 是為了繞過 Windows 環境變數路徑解析的 Bug。
* **驗證**：重啟 Claude，輸入「請在當前目錄建立 `test.txt`」，確認檔案有出現在 Windows 資料夾中。

---

## 5. Dev Container 環境配置 (The Engine)

這是讓 Windows 擁有 K8s 能力的核心。我們使用 **Docker-in-Docker (DinD)** 技術來在容器內運行 Kind Cluster。

在專案根目錄建立 `.devcontainer/devcontainer.json`：

```json
{
  "name": "Vibe K8s Environment",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-22.04",
  "features": {
    // 啟用 Docker-in-Docker (執行 Kind 的必要條件)
    "ghcr.io/devcontainers/features/docker-in-docker:2": {
      "version": "latest",
      "enableNonRootDocker": "true",
      "moby": "true"
    },
    // 預裝 K8s 工具鏈
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {
      "version": "latest",
      "minikube": "none",
      "kubectl": "latest",
      "helm": "latest"
    }
  },
  // 自動初始化指令
  "postCreateCommand": "curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64 && chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind && kind create cluster --name vibe-cluster",
  
  // VS Code 擴充套件預裝
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "ms-azuretools.vscode-docker",
        "redhat.vscode-yaml"
      ]
    }
  },
  // 給予特權模式 (執行 K8s 必要)
  "runArgs": ["--privileged"]
}

```

---

## 6. 資料庫效能優化策略 (Database Performance)

**⚠️ 效能地雷警示：**
絕對**不要**把 MariaDB/PostgreSQL 的資料目錄 (`/var/lib/mysql`) 掛載到 Windows 的檔案系統 (`./data`)，這會導致資料庫慢 10 倍以上。

**正確做法**：使用 Docker Named Volume。

在 Claude 幫你生成 `deployment.yaml` 或 `docker-compose.yml` 時，請強制要求：

> "請配置 Database，但務必使用 Docker 內部的 Named Volume 來存放資料，不要 Bind Mount 到當前目錄。"

**YAML 範例：**

```yaml
volumes:
  - name: mysql-data
    emptyDir: {} # 測試用 (重啟消失)
    # 或者使用 persistentVolumeClaim (生產模擬)

```

---

## 7. 啟動與監控流程 (Workflow)

### 7.1 啟動環境

1. 開啟 VS Code -> Open Folder -> 選擇 `C:\Users\...\vibe-k8s-lab`。
2. 右下角彈出提示 -> 點擊 **"Reopen in Container"**。
3. 等待環境建置完成 (Terminal 出現 `vscode@...` 提示字元)。

### 7.2 監控服務 (Port Forwarding)

由於服務運行在容器內的容器 (Nested Container)，需要透過 VS Code 建立通道才能在 Windows 瀏覽器存取。

在 VS Code Terminal 執行：

```bash
# 範例：轉發 Grafana 與 Prometheus
kubectl port-forward -n monitoring svc/grafana 3000:3000 --address 0.0.0.0 &
kubectl port-forward -n monitoring svc/prometheus 9090:9090 --address 0.0.0.0 &

```

* **Grafana**: 開啟 Windows 瀏覽器存取 `http://localhost:3000`
* **Prometheus**: 開啟 Windows 瀏覽器存取 `http://localhost:9090`

---

## 8. 常見問題排除 (Troubleshooting)

* **Q: Claude 顯示 "Failed to start workspace"？**
* A: 檢查 `claude_desktop_config.json` 的 JSON 格式，反斜線必須是雙重 (`\\`)。確認 `npx` 路徑是否正確。


* **Q: K8s Pod 一直 Pending？**
* A: 檢查 `.wslconfig` 是否給予足夠記憶體 (建議至少 8GB)。執行 `kubectl describe pod <pod-name>` 查看原因。


* **Q: 資料庫寫入極慢？**
* A: 檢查是否誤用了 Windows 目錄作為 Volume。請確保資料路徑是在 Linux 原生 Volume 內。


* **Q: 瀏覽器打不開 Grafana？**
* A: 確認 `kubectl port-forward` 指令是否有加 `--address 0.0.0.0`，且 VS Code 的 Ports 分頁有顯示該連接埠。



---