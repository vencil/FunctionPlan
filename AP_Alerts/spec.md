# Multi-Tenant Dynamic Alerting Architecture
---

## 1. Executive Summary

本提案旨在解決當前監控系統在面對「高客製化需求」與「大規模使用者（數萬級別）」時的擴展性瓶頸。

引入 **「Configuration as Metrics** 模式，將告警規則從「靜態設定檔」轉型為「動態數據流」。透過此架構，我們能在**不重啟 Prometheus、不增加 Rule File** 的前提下，同時支援：

1. 使用者自訂 CPU/Memory 數值閾值。
2. 針對 Pod 內特定 Container 的精準偵測。
3. Kubernetes 狀態與字串類型的邏輯比對。
4. 具備「特定情境優先，預設值兜底」的複合邏輯。

---

## 2. Problem Statement

若採用傳統的「一使用者一規則（One-Rule-Per-User）」模式，將導致現有架構面臨以下挑戰：

1. **維運成本呈線性增長**：目前若要支援使用者自訂告警，需為每位使用者生成獨立的 Rule File。當使用者達到萬級規模，Rule 管理將成為災難。
2. **Performance**：數十萬條 Rule 的 Evaluation Loop 會耗盡 Prometheus 的 CPU 資源，且頻繁的 Config Reload 會造成服務不穩定。
3. **Inflexibility**：難以支援複雜邏輯（例如：「在 Test 環境 80% 告警，但在 Prod 環境 60% 告警，其餘 Default 50%」），導致 Rule Template 過於複雜且難以維護。

---

## 3. Proposed Solution

我們將監控邏輯拆解為三個層次，實現完全的解耦：

1. **數據層 (Data Layer)**：
* **Raw Metrics**: 來自基礎設施的原始數據。
* **Config Metrics**: 由 Exporter 將使用者設定轉換為標準 Metrics，包含 `user_id`, `threshold`, `condition` 等標籤。


2. **聚合層 (Normalization Layer)**：
* 使用 Recording Rules 將原始數據標準化（例如統一轉為百分比），降低基數 (Cardinality)。


3. **邏輯層 (Logic Layer)**：
* 使用 PromQL 的 `group_left` (Many-to-One Join) 動態結合數據與設定。
* 利用 `*` (乘法) 進行交集比對。
* 利用 `unless` / `or` 實現優先權邏輯。

---

## 4. Benefits & ROI

| 構面 | 改善前 (Before) | 改善後 (After) | 效益評估 |
| --- | --- | --- | --- |
| **擴展性** | 規則數量隨使用者線性增加 () | 規則數量恆定 () | **支援無限使用者** |
| **即時性** | 修改設定需等待 Reload (數秒至數分) | 修改設定即時生效 (下一次 Scrape) | **Zero Downtime** |
| **維運複雜度** | 需維護龐大的 Rule Template 生成器 | 僅需維護簡單的 Exporter 與 SQL | **開發與除錯更直觀** |
| **覆蓋率** | 容易忽略 Sidecar 或特殊狀態 | 透過通用邏輯全覆蓋 (Weakest Link) | **提升系統可靠度 (Reliability)** |

---

## 5. Context Diagram

### 5.1 轉型前：靜態檔案生成模式 (Legacy)

* **痛點**：Rule Generator 是單點瓶頸，Prometheus 負載過重。

```mermaid
C4Context
    title Context - Legacy: Static Rule Generation

    Person(user, "User", "設定複雜規則")

    System_Boundary(legacy, "Legacy System") {
        System(api, "API / Dashboard", "接收設定")
        System(gen, "Rule Generator", "邏輯判斷與檔案寫入")
        System(prom, "Prometheus", "載入巨量 YAML Files")
    }

    Rel(user, api, "設定: Test環境>80%, 其他>60%")
    Rel(api, gen, "傳遞參數")
    Rel(gen, prom, "1. 生成 user_123.yml\n2. 觸發 Reload")

    Note(prom, "Reload 時期造成效能抖動\n數萬條 Rules 導致 Evaluation 延遲")

```

### 5.2 轉型後：動態向量匹配模式 (Proposed)

* **優勢**：Prometheus 僅需維護 < 10 條通用規則，所有邏輯在查詢時動態計算。

```mermaid
C4Context
    title Context - Proposed: Configuration as Metrics

    Person(user, "User", "設定複雜規則")

    System_Boundary(new, "Modern Architecture") {
        System(api, "API / DB", "儲存設定 (Source of Truth)")
        System(exporter, "Config Exporter", "將設定轉為 Metrics (Pull/Push)")
        System(prom, "Prometheus", "執行通用 PromQL (Join/Unless)")
    }

    Container(pod, "App Pods", "Raw Metrics Source")

    Rel(user, api, "更新設定 (SQL Update)")
    Rel(prom, exporter, "Scrape Config Metrics")
    Rel(prom, pod, "Scrape Raw Metrics")

    BiRel(prom, prom, "動態運算:\n1. 數值比對 (>)\n2. 字串交集 (*)\n3. 優先權 (unless)")

    Note(prom, "Zero Reload\nO(1) Rule Complexity")

```

---

## 6. 核心場景實作設計 (Core Use Cases Implementation)

以下展示本架構如何以一套統一的邏輯，同時滿足四種截然不同的商業需求。

### 場景 A：基礎動態閾值 (Dynamic Thresholds)

* **關鍵技術**：標準 `group_left`。

#### A.1 資料源層 (Exporter Design)

使用者的設定不再寫入設定檔，而是由 Exporter 吐出 Metrics。
***範例 Metrics:***

```text
user_cpu_threshold{user_name="User_A", target_component="B", severity="info"} 50
user_cpu_threshold{user_name="User_B", target_component="B", severity="warning"} 70

```

#### A.2 聚合層 (Recording Rules)

為了避免高基數 (High Cardinality) 的 Join 運算，我們引入 Recording Rule 進行預計算降維。

***檔案***: `recording_rules.yml`

```yaml
groups:
  - name: aggregation_rules
    interval: 1m
    rules:
      # 將底層 Pod/Container 數據聚合為 Component 層級的百分比
      - record: job:component_cpu_usage:percent
        expr: >
          sum by (component) (rate(container_cpu_usage_seconds_total[5m])) * 100

```

#### A.3 邏輯層 (Alerting Rules)

這是核心邏輯，使用 PromQL 的 `Many-to-One` Join 技術。

***檔案***: `alerting_rules.yml`

```yaml
groups:
  - name: dynamic_alerting
    interval: 1m
    rules:
      - alert: UserDefinedHighCPU
        annotations:
          summary: "CPU Usage High: {{ $value }}%"
          description: "User {{ $labels.user_name }} threshold: {{ $labels.threshold_value }}%"
        expr: |
          # 1. 取出聚合後的實際值
          job:component_cpu_usage:percent

          # 2. 動態比對：實際值 > 使用者設定值
          > on(component) group_left(user_name, severity)

          # 3. 取出設定值 (並將數值複製到 label 供顯示用)
          label_replace(user_cpu_threshold, "threshold_value", "$1", "__name__", "(.+)")

```

### 場景 B：容器級短板偵測 (Weakest Link Detection)

* **需求**：監控 Pod 內部的 Sidecar (e.g., Istio)。只要**任何一個** Container OOM，即使 Pod 總體記憶體充足，也要告警。
* **關鍵技術**：保留左側 `container` 標籤，不進行 Sum 聚合。

```yaml
- alert: UserDefinedContainerMemory
  expr: |
    # 左側：保留 container 維度 (Many)
    job:container_memory_usage:percent

    # 比對：忽略 container 標籤差異，與 Component 級別設定比對
    > on(component) group_left(user_name, severity)

    # 右側：Component 級別設定 (One)
    user_memory_config

```

### 場景 C：狀態與字串比對 (State/String Matching)

* **需求**：使用者想監控 `CrashLoopBackOff` 或 `ImagePullBackOff` 狀態。
* **關鍵技術**：**乘法交集 (Intersection)**。利用 `Value * 1` 的特性，只有標籤匹配時才會有值。

```yaml
- alert: UserDefinedPodState
  expr: |
    # 左側：真實發生的狀態 (e.g., phase="CrashLoopBackOff")
    kube_pod_status_phase

    # 運算：乘法 (AND Logic)。只有當 phase 相同時，結果才存在
    * on(component, phase) group_left(user_name, severity)

    # 右側：使用者關注的狀態 (e.g., phase="CrashLoopBackOff")
    user_state_config

```

### 場景 D：複合優先權邏輯 (Composite Priority Logic)

* **關鍵技術**：`unless` (排除例外) + `or` (聯集)。

#### D.1. 資料設計 (Data Design)

為了支援這種複合 Case，你的 Exporter 吐出的 Config Metrics 需要稍微規劃一下標籤。假設關鍵的區分條件標籤叫做 `condition`。

***Raw Metrics (原始數據):***
會有各種 `condition` (test1, test2, prod, staging...)

```text
# 這是 test1 環境，目前 CPU 75%
job:cpu_usage:percent{component="B", condition="test1"} 75

# 這是 test2 環境，目前 CPU 75%
job:cpu_usage:percent{component="B", condition="test2"} 75

# 這是其他環境 (other)，目前 CPU 65%
job:cpu_usage:percent{component="B", condition="other"} 65

```

**Config Metrics (使用者設定):**
我們約定一個特殊的字串（例如 `default`）代表預設值。

```text
# User A 針對 test1 設定 70% (Specific)
user_cpu_threshold{user="User_A", component="B", condition="test1"} 70

# User A 針對 test2 設定 80% (Specific)
user_cpu_threshold{user="User_A", component="B", condition="test2"} 80

# User A 的預設值是 60% (Default / Fallback)
# 注意：這裡雖然寫 condition="default"，但邏輯上它是給那些 "非 test1 且 非 test2" 用的
user_cpu_threshold{user="User_A", component="B", condition="default"} 60

```

---

#### D.2. 實作 PromQL (The Magic Query)

這條 Rule 看起來稍長，但邏輯非常清晰。

```yaml
groups:
  - name: composite_condition_alerts
    rules:
      - alert: UserDefinedCompositeCPU
        expr: |
          (
            # === 第一部分：命中特定規則 (Hit Specific) ===
            # 邏輯：直接拿 Raw Data 撞 Config，標籤 condition 必須完全一樣
            job:cpu_usage:percent
            > on(component, condition) group_left(user)
            user_cpu_threshold{condition!="default"}
          )
          or
          (
            # === 第二部分：命中預設規則 (Hit Default) ===
            # 1. 先用 unless 排除掉「已經有特定規則」的數據
            (
              job:cpu_usage:percent
              unless on(component, condition)
              user_cpu_threshold{condition!="default"}
            )

            # 2. 剩下的數據，拿去跟 "default" 設定比對
            # 注意：這裡要忽略左邊的 condition (因為它是 other)，去撞右邊的 condition="default"
            # 所以我們只 on(component)
            > on(component) group_left(user)
            user_cpu_threshold{condition="default"}
          )
        annotations:
          summary: "High CPU on {{ $labels.condition }}"
          description: "User {{ $labels.user }} alert. Value {{ $value }}% exceeds threshold."
```
---



## 7. 下一步計畫 (Next Steps)

為確保架構順利落地，建議分階段執行：

1. **POC 階段 (本週)**：
* 使用 **Pushgateway** 模擬 Exporter。
* 驗證上述四種 PromQL 邏輯在小規模數據下的正確性。


2. **MVP 開發 (下週)**：
* 開發 `config-exporter` (Go/Python)，對接使用者設定資料庫。
* 實作 Recording Rules 進行 Raw Data 的標準化 (Normalization)。


3. **生產環境部署**：
* 部署至 Prometheus Staging 環境。
* 進行壓力測試 (模擬 10k+ Config Metrics 的 Join 效能)。



---