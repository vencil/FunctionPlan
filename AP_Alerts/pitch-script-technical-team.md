# 講稿 A：面對技術團隊（SRE / Platform Engineer / DBA）

> 預計時長：5–7 分鐘。語氣：同行對同行，講問題講機制，不講形容詞。

---

## 開場（30 秒）

大家應該都碰過這個狀況：監控規則越寫越多，Prometheus 越跑越慢，然後每次有新租戶要上線，就是一輪「複製貼上、改 namespace、改閾值、發 PR、等 CI/CD」的循環。

今天我想跟大家分享我們怎麼用一套 config-driven 的架構，把這件事情從 O(N×M) 壓到 O(M)。

---

## 第一段：你現在的問題（1 分鐘）

先看數字。假設你有 100 個租戶、每個租戶 35 條警報規則。傳統做法就是 3,500 條 PromQL，每 15 秒全部跑一遍。我們實測過，這大概吃掉 850 毫秒以上的評估時間，而且是線性增長 — 租戶翻倍，時間翻倍。

更痛的是日常維護。所有規則擠在同一個 ConfigMap 裡，三個團隊同時改就是 merge conflict。有人想調個閾值，流程是 PR → review → merge → CI → reload，最快也要半小時。

---

## 第二段：我們怎麼解的（2 分鐘）

核心思路是把「閾值」從規則裡抽出來，變成 Prometheus metric。

具體來說，我們有一個 threshold-exporter，它讀取每個租戶的 YAML 配置，輸出成 `user_threshold` gauge metric。然後 Prometheus 那邊只需要一條 recording rule，用 `group_left` 做向量匹配，一次評估就涵蓋所有租戶。

結果就是：不管你有 10 個還是 1,000 個租戶，規則數固定在 85 條，評估時間固定在 20 毫秒左右。

租戶那邊的配置長這樣：

```yaml
tenants:
  db-a:
    mysql_connections: "100"
```

就是一行 YAML。不需要寫 PromQL，不需要知道 `rate`、`sum by`、`group_left` 是什麼。

---

## 第三段：遷移怎麼辦（1.5 分鐘）

我知道大家最擔心的是：「現在已經有幾千條舊規則了，怎麼遷移？」

我們設計了一個三步流程：

**第一步：Triage。** 跑 `migrate_rule.py --triage`，它會把你的舊規則自動分成四桶 — 可以自動轉換的、需要人工 review 的、建議直接用黃金標準替代的、以及可以跳過的。輸出是一份 CSV，可以在 Excel 裡批次決策。

**第二步：Shadow Monitoring。** 轉換完的新規則不會直接上線。它們帶有 `custom_` 前綴，跟黃金標準在命名空間層面完全隔離。然後用 `validate_migration.py` 去比對新舊 recording rule 的數值輸出，逐一確認結果一致。

**第三步：切換。** 確認數值吻合後再切換。如果出問題，隨時可以退回去 — 所有規則包在 Projected Volume 中設定了 `optional: true`，直接 `kubectl delete cm` 就能卸載，Prometheus 不會崩潰。

---

## 第四段：Live Demo（1 分鐘，搭配操作）

最後我直接跑給大家看。

```bash
make demo-full
```

這會做三件事：注入真實負載（stress-ng + 95 個 MariaDB 連線）、等待 alert 觸發到 FIRING 狀態、然後清除負載看 alert 自動恢復。整個循環大概 3-5 分鐘。

不改閾值、不 mock 數據、不做假。你現場看到的 alert 就是生產環境下會觸發的 alert。

---

## 收尾（30 秒）

整理一下。這套系統解決三個問題：規則不隨租戶膨脹、租戶不需要學 PromQL、遷移有完整的安全網。

所有工具都帶 `--dry-run`，所有操作都可以回退。有興趣的話可以直接在 Dev Container 裡跑起來，五分鐘內就能看到完整流程。

---

*Q&A 備用彈藥：*

- **「HA 怎麼處理？」** → 2 replicas + PDB + anti-affinity + `max by(tenant)` 避免閾值翻倍
- **「未使用的 rule pack 有成本嗎？」** → 空向量評估近乎零成本，實測 < 0.01ms
- **「支援哪些 DB？」** → 目前 MariaDB / Redis / MongoDB / ES / K8s，Oracle/DB2 在 roadmap
- **「怎麼做維護窗口？」** → 設定 `_state_maintenance: enable`，用 `unless` 自動抑制
