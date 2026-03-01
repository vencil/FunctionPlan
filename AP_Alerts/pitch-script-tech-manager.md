# Dynamic Alerting Platform — 高階技術主管簡報大綱

> **對象**：CTO、VP of Engineering、IT Director、技術決策者
> **目標**：說明為什麼需要這個平台、帶來什麼商業效益、如何降低風險
> **時長建議**：15–20 分鐘（含 Q&A）

---

## 1. 開場：我們正在解決什麼問題（3 min）

**核心訊息**：隨著租戶數量增長，傳統監控架構的成本與風險呈線性爆炸。

- **規模痛點量化**
  - 100 租戶 × 50 規則 = 5,000 條獨立 PromQL — 每 15 秒全部重新評估
  - 規則膨脹 → Prometheus CPU 飆升 → 評估延遲 → SLA 風險
- **人力瓶頸**
  - 租戶必須學 PromQL → 平台團隊代寫/除錯 → 團隊成為瓶頸
  - 手動遷移數百條規則 → 數週工時 + 切換失敗風險（監控盲區）
- **維運債**
  - 一個巨型 ConfigMap → merge conflicts → 部署風險
  - 閾值修改 = PR → CI/CD → reload → 上線週期長

## 2. 解決方案全景（5 min）

**核心訊息**：一套 config-driven 的多租戶監控治理平台，從部署到退場全流程覆蓋。

### 2.1 架構一句話
> M 條規則服務 N 個租戶。複雜度 O(N×M) → O(M)。

- 展示概念對比圖（Traditional vs Dynamic）
- 關鍵技術選擇：`group_left` 向量匹配、SHA-256 熱重載、Projected Volume

### 2.2 租戶體驗
- 零 PromQL — 租戶只寫 YAML（`mysql_connections: "80"`）
- 互動式 `scaffold` → 幾分鐘內產生合規配置
- `da-tools` 容器——`docker pull` 即用，不需 clone repo、不需裝 Python

### 2.3 企業級能力亮點
- **一行部署**：Helm OCI registry，`helm install oci://...` 完成安裝
- **Config 分離**：base chart `tenants: {}` + overlay，符合 GitOps 慣例
- **三層治理模型**：Platform Team → Domain Experts → Tenant Tech Leads
- **Auto-Suppression**：Critical 觸發自動靜默 Warning，減少告警疲勞
- **排程式閾值**：夜間/假日自動放寬，降低非工作時段誤報
- **Multi-DB 生態系**：9 個 Rule Pack（7 種 DB + K8s + Platform），可選載入

## 3. 商業效益與 ROI（4 min）

**核心訊息**：降成本、縮時程、控風險。

| 維度 | 傳統方案 | Dynamic Alerting | 改善幅度 |
|------|---------|------------------|---------|
| **規則維護成本** | O(N×M) 線性成長 | O(M) 固定 | 100 租戶 = ~100x 規則減少 |
| **租戶導入時間** | 學 PromQL + 手動配置（天→週） | `scaffold` 互動式產生（分鐘） | 數量級下降 |
| **遷移風險** | 一次性切換，失敗=盲區 | Shadow Monitoring 雙軌 + 漸進切換 | 零停機風險 |
| **部署複雜度** | clone repo + 手動管理 chart/image | `helm install oci://...` 一行 | 消除部署依賴 |
| **告警雜訊** | Warning + Critical 同時響 | Auto-Suppression + 排程式 + 三態 | 顯著降低 MTTR |
| **評估效能** | ~800ms+ @ 100 tenants | ~20ms（固定） | ~40x |

### 治理與合規
- Per-tenant YAML in Git = 天然稽核軌跡
- 檔案級 RBAC + CI deny-list linting
- 全生命週期覆蓋：導入 → 營運 → 下架

## 4. 風險控制與退出策略（2 min）

**核心訊息**：平台設計上確保進退自如，不會被鎖死。

- **零崩潰退出**：Projected Volume `optional: true` — 刪掉 Rule Pack 不影響 Prometheus
- **AST 遷移引擎**：Rust PyO3 精準解析 + `custom_` prefix 隔離 — 新舊規則並存
- **Shadow Monitoring**：遷移前後數值 diff ≤ 5% 容差 — 有數據證明才切換
- **Dry-run 全覆蓋**：每個工具都有 `--dry-run` 或 Pre-check，操作前可預覽影響

## 5. 即時驗證（可選 Live Demo）（3 min）

**核心訊息**：不是 Slide Deck 承諾，是即時可驗證的工程成果。

- `make demo-full`：端對端展演（負載注入 → alert 觸發 → 清除 → 自動恢復）< 5 分鐘
- 每個價值主張都附帶可執行命令（見 README 企業價值表「可驗證性」欄）

## 6. 建議下一步（1 min）

- **評估**：在現有 Prometheus 環境執行 `da-tools validate` 對比當前規則
- **Pilot**：選一個租戶跑 `scaffold` → `migrate` → Shadow Monitoring 雙軌
- **擴展**：驗證通過後，透過 OCI chart + `values-override.yaml` 逐步 rollout

---

## 附錄：預期 Q&A 準備

| 問題方向 | 回應重點 |
|---------|---------|
| 跟現有 Prometheus 怎麼整合？ | BYOP 整合指南：3 步（tenant label + scrape + rule mount） |
| 誰負責維護 Rule Pack？ | 三層治理模型：Platform / Domain / Tenant 分層權責 |
| 遷移要停機嗎？ | Shadow Monitoring 雙軌並行，零停機 |
| 如果不適合，退出成本？ | `optional: true` + prefix 隔離，移除零風險 |
| 支援哪些資料庫？ | 7 種 DB + K8s + Platform，9 個 Rule Pack |
