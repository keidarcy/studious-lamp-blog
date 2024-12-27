+++
title = 'タイムスライスSLOでDatadogのSLO管理を効率化する'
date = 2024-12-27T11:52:46+09:00
description = 'Streamlining Datadog SLO management with time slice SLOS'
+++

### はじめに

DatadogのSLOには3種類のオプションがあります：モニターベース SLO ([Monitor-based SLOs](https://docs.datadoghq.com/service_management/service_level_objectives/monitor/))、メトリクスベース SLO ([Metric-based SLOs](https://docs.datadoghq.com/service_management/service_level_objectives/metric/))、そして新しく追加されたタイムスライス SLO ([Time Slice SLOs](https://docs.datadoghq.com/service_management/service_level_objectives/time_slice/))です。

今回は、特にタイムスライス SLOの利便性とメリットについて、具体例を挙げて説明します。

---

### 従来の方法: モニターベース SLO

タイムスライス SLOが登場する以前、たとえばNode.jsのExpress APIにおけるレイテンシーSLOを作成する場合、**モニターベース SLO**を使用して次のように設定していました。以下はTerraformを使った例です。

```hcl
resource "datadog_monitor" "request_get_latency" {
  name    = "📝 [SLO] Resource Get API has an abnormal latency"
  type    = "query alert"
  message = "Resource Get API latency is above 1 second"
  query   = <<EOT
percentile(last_5m):p95:trace.express.request{resource_name:get_${local.path}} > 1
EOT

  monitor_thresholds {
    critical = 1.0 # 1秒
  }

  evaluation_delay    = 300 # 5分
  require_full_window = false
}

resource "datadog_service_level_objective" "request_get_latency" {
  name        = "🏝️ Get API p95 latency *monitor* SLO"
  type        = "monitor"
  description = "99.9%の時間、Get APIのp95レイテンシーが1秒以下であること。"

  thresholds {
    timeframe = "30d"
    target    = 99.9
  }

  monitor_ids = [
    datadog_monitor.request_get_latency.id
  ]
}

```

モニターベース SLOでは、SLOを作成する前にモニターの定義が必要です。この手間を軽減するために、タイムスライス SLOが登場しました。

---

### 新しい選択肢: タイムスライス SLO

タイムスライス SLOでは、**モニターを別途作成する必要がなくなります**。Terraformでの記述もシンプルで、以下のように1つのリソースで完結します。

```hcl
resource "datadog_service_level_objective" "request_get_latency_time_slice" {
  name        = "🏝️ Get API p95 latency *time slice* SLO"
  type        = "time_slice"
  description = "99.0%の時間、Get APIのp95レイテンシーが1秒以下であること。"

  sli_specification {
    time_slice {
      query {
        formula {
          formula_expression = "query1"
        }
        query {
          metric_query {
            name  = "query1"
            query = "p95:trace.express.request{env:production,resource_name:get_${local.path}}"
          }
        }
      }
      comparator             = "<="
      threshold              = 1.0
      query_interval_seconds = 300
    }
  }

  thresholds {
    timeframe = "30d"
    target    = 99.0
  }
}

```

このように、タイムスライス SLOでは、モニターを介さずに直接SLOを作成できます。コードの簡潔さからも、運用の負担が軽減されることがわかります。

---

### 運用画面の比較

タイムスライス SLOの詳細ページを見てみると、Error BudgetやBurn Rateの遷移、Timelineなどの情報が豊富で、運用がさらに効率的になっています。

![time slice slo](https://storage.googleapis.com/zenn-user-upload/df217aae9b8c-20241227.png)

一方、モニターベース SLOの詳細ページには、右上に「**Export to Time Slice SLO**」というオプションがあります。このボタンをクリックすると、既存のモニターベース SLOからタイムスライス SLOへの移行が簡単に行えます。

![monitor based slo](https://storage.googleapis.com/zenn-user-upload/cddc7164a341-20241227.png)

![export to time slice slo](https://storage.googleapis.com/zenn-user-upload/fd6fc0b4f233-20241227.png)

---

### 終わりに

今回は、レイテンシーSLOを例に、**モニターベース SLO**と**タイムスライス SLO**の作成と運用を比較しました。その結果、タイムスライス SLOは設定の簡潔さ、豊富なデータ表示、そして運用効率の観点から非常に優れています。

今後、SLOを作成する際には、タイムスライス SLOを優先的に検討することをおすすめします。