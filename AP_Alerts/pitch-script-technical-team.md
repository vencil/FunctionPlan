# Dynamic Alerting Platform — 技術團隊簡報大綱

> **對象**：Platform Engineers、SREs、DBA、DevOps、Tech Leads
> **目標**：理解架構原理、學會部署與整合、掌握工具鏈使用
> **時長建議**：30–40 分鐘（含 Demo + Q&A）

---

## 1. 問題陳述：規模化監控的工程挑戰（5 min）

**核心訊息**：傳統「一個租戶一條 rule」的作法，工程上不可持續。

### 1.1 複雜度爆炸
- N 租戶 × M 規則 = N×M 條 PromQL — 線性膨脹
- 效能實測：100 租戶 ~800ms+（每 evaluation cycle），Dynamic ~20ms（固定）
- 展示 benchmark 數據（5 輪 mean ± 1.9ms），出處見 `docs/architecture-and-design.md` §4.1–4.2

### 1.2 維運陷阱
- 巨型 ConfigMap merge conflicts
- 租戶寫 PromQL 寫錯 label → 靜默失敗（silent misfire）
- 規則修改 = PR → CI → reload → 上線，response time 拉長

### 1.3 遷移債
- 手動重寫數百條規則：label 不一致、聚合方式不明、`rate()` 嵌套深度不一
- 一次性切換 = 潛在監控盲區

## 2. 架構設計（10 min）

**核心訊息**：Config-driven + Vector Matching + Projected Volume = O(M)。

### 2.1 核心原理：Threshold 外部化
```
Tenant YAML → threshold-exporter → Prometheus gauge metrics
                                        ↓
Rule Packs → Recording Rules + Alert Rules ← group_left join
```
- threshold-exporter 將 YAML 轉為 `user_threshold` gauge
- PromQL 用 `> on(tenant) group_left` 一條 rule 匹配所有租戶
- 展示程式碼對比（傳統 vs 動態）

### 2.2 threshold-exporter 元件
- Go，port 8080，HA ×2
- **Directory Scanner**：`-config-dir conf.d/` 支援 per-tenant 檔案
- **Hot-Reload**：SHA-256 hash 偵測 → 零重啟
- **三態邏輯**：Custom / Default（省略）/ Disable（`"disable"`）
- **維度閾值**：Regex `=~` 運算子 → `_re` 後綴 label
- **排程式閾值**：YAML 時間窗口定義（如 `22:00-06:00`）
- **多層嚴重度**：`_critical` 後綴獨立閾值

### 2.3 Rule Pack 架構
- 9 個獨立 ConfigMap，透過 Kubernetes **Projected Volume** 掛載
- `optional: true` — 刪除 ConfigMap 不影響 Prometheus（零崩潰退出）
- 規則結構：Recording Rule（data normalization + threshold normalization）→ Alert Rule
- **Normalization 語義**：
  - Threshold：一律 `max by(tenant)` 防 HA 翻倍
  - Data：依語義選擇（connections 用 `max`，rate/ratio 用 `sum`）
- 共 85 Recording + 56 Alert = 141 條規則，27 Groups

### 2.4 Auto-Suppression 機制
- `migrate_rule.py` 自動偵測 warning ↔ critical 配對
- 為 warning alert 注入 `unless on(tenant)` 子句
- Critical 觸發 → 對應 Warning 自動靜默，零人工介入

## 3. 部署與整合（5 min）

**核心訊息**：一行 helm install，3 步 BYOP 整合。

### 3.1 全新部署
```bash
helm install threshold-exporter \
  oci://ghcr.io/vencil/charts/threshold-exporter --version 1.1.0 \
  -n monitoring --create-namespace \
  -f values-override.yaml
```
- chart 內已綁定 image 版本 — 無需手動管理 tag
- base chart `tenants: {}` + overlay 分離 — 符合 GitOps

### 3.2 整合現有 Prometheus（BYOP）
1. 確保 data 有 `tenant` label（relabeling `namespace` → `tenant`）
2. Scrape threshold-exporter endpoint
3. Mount Rule Pack ConfigMap（Projected Volume）

詳見 `docs/byo-prometheus-integration.md`，含 Prometheus Operator 附錄

### 3.3 CI/CD Pipeline
- GitHub Actions：`v*` tag → exporter image + Helm chart，`tools/v*` → da-tools image
- Chart.yaml version verification gate — 防版號不一致
- `make version-check` 本地驗證

## 4. 工具鏈 Deep Dive（8 min）

**核心訊息**：全生命週期 CLI 工具鏈，容器化交付。

### 4.1 da-tools 容器
- `ghcr.io/vencil/da-tools` — 封裝 9 個 CLI 工具
- `docker pull` 即用，不需 clone、不需裝 Python
- `entrypoint.py` 子命令分派：`scaffold`、`migrate`、`validate`、`check-alert`...

### 4.2 租戶導入流程
```
scaffold → 產生 tenant YAML
    ↓
migrate  → 舊規則轉換（AST 引擎 + Triage CSV + custom_ prefix）
    ↓
validate → Shadow Monitoring 數值 diff（≤ 5% 容差）
    ↓
上線 → patch_config / helm upgrade
```

### 4.3 關鍵工具說明
| 工具 | 功能 | 關鍵 Flag |
|------|------|-----------|
| `scaffold` | 互動式租戶配置產生器 | `--tenant NAME --db TYPE` / `--catalog` |
| `migrate` | AST 遷移引擎（Rust PyO3 promql-parser） | `--triage` / `--dry-run` / `--no-prefix` / `--no-ast` |
| `validate` | Shadow Monitoring diff | `--mapping FILE` / `--old Q --new Q` |
| `baseline_discovery` | 負載觀測 + 閾值建議（p95/p99） | `--duration S --interval S --metrics LIST` |
| `offboard` | 租戶安全下架 | `--execute`（無則 dry-run） |
| `deprecate` | Rule/Metric 平滑下架 | `--execute`（三步自動化） |
| `lint_custom_rules` | CI deny-list linter | `--policy FILE` / `--ci` |

### 4.4 治理模型
- **三層**：Platform Team（全域預設）→ Domain Experts（黃金標準）→ Tenant Tech Leads（業務閾值）
- `_defaults.yaml` 由 Platform Team 管控 → 邊界規則防覆蓋
- `lint_custom_rules.py` CI gate — deny-list 確保合規

## 5. Live Demo（5 min）

**核心訊息**：每一個功能都可以即時驗證。

### 5.1 快速展演路徑
```bash
# scaffold → migrate → diagnose → baseline_discovery（不含負載）
make demo
```

### 5.2 完整展演路徑（含真實負載）
```bash
# Composite Load → alert 觸發 → 清除 → 自動恢復 < 5 min
make demo-full
```

### 5.3 展演重點
1. `scaffold` 互動式產生 → 看到 YAML 輸出
2. Prometheus UI 看 `user_threshold` gauge 即時出現
3. 修改閾值 → exporter hot-reload → 幾秒內生效
4. `make load-composite` → alert 觸發 → Alertmanager routing by tenant
5. 停止負載 → pending → resolved 完整循環

## 6. 技術決策 FAQ（2 min）

| 問題 | 回應 |
|------|------|
| 為什麼不用 Thanos Rules / Mimir Ruler？ | 不要求 Cortex/Mimir 基礎設施，純 Prometheus 原生 |
| HA 翻倍問題怎麼處理？ | Threshold normalization 用 `max by(tenant)` |
| AST 引擎用 Rust 為什麼？ | `promql-parser` PyO3 binding — 精準 parse，不依賴 regex heuristic |
| optional: true 有什麼陷阱？ | Prometheus 啟動時不存在不報錯；API 刪除後下個 reload 自動卸載 |
| 跟 Alertmanager inhibition 的差異？ | Auto-Suppression 在 PromQL 層面注入 `unless`，不依賴 Alertmanager 配置 |

## 7. 相關文件索引

| 文件 | 用途 |
|------|------|
| [架構與設計](../architecture-and-design.md) | O(M) 推導、HA、效能 benchmark |
| [BYOP 整合指南](../byo-prometheus-integration.md) | 3 步整合 + Operator 附錄 |
| [遷移指南](../migration-guide.md) | 5 個實戰場景 + da-tools 操作流程 |
| [客製化規則治理](../custom-rule-governance.md) | 三層模型 + SLA + CI Linting |
| [Shadow Monitoring SOP](../shadow-monitoring-sop.md) | 雙軌並行完整 Runbook |
| [da-tools CLI](../../components/da-tools/README.md) | 容器工具鏈完整文件 |
| [Rule Packs](../../rule-packs/README.md) | 9 個 Rule Pack 規格 |
| [Testing Playbook](testing-playbook.md) | K8s 環境測試手冊 |
