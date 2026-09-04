---
title: "Airflowのゾンビタスクはなぜ起きるのか - ハートビートの仕組みから原因の切り分けまで"
emoji: "👻"
type: "tech"
topics: ["airflow", "cloudcomposer", "googlecloud", "障害調査"]
published: false
---

Airflowを運用していると、Schedulerのログに `Detected zombie job` が出て、動いていたはずのタスクが失敗することがあります。これがゾンビタスクです。厄介なのは、このログを見ても「なぜそうなったのか」が分からないことです。ログが伝えるのは「Airflowがタスクの動作を確認できなくなった」という結果だけで、原因はワーカーのメトリクスやほかのログと突き合わせて絞り込むしかありません。

この記事では、ゾンビタスクが検出される仕組みを先に押さえてから、発生の確認と原因の切り分けの手順をまとめます。コマンド例はGoogle CloudのマネージドAirflow向けですが、考え方は自前で運用するAirflowでも同じです。対象はAirflow 2系で、Airflow 3で変わった点は最後にまとめます。

## ゾンビタスクが検出される仕組み

原因を探す前に、Airflowが何を根拠にゾンビと判定しているのかを確認しておきます。判定の根拠が分かれば、その根拠が崩れる状況がそのまま原因候補になるからです。

Airflowでは、タスクを実行しているプロセスが、既定では5秒おきにメタデータDBの「最終更新時刻」を現在時刻に書き換えます。これが「まだ動いている」という合図で、心拍のように一定の間隔で打ち続けることから**ハートビート**と呼ばれています。この記事でも、以降はこの合図をハートビートと書きます。

![ワーカー・メタデータDB・Schedulerの関係](/images/airflow-zombie-job/fig1-heartbeat.png)
*① ワーカーは5秒おきに現在時刻をDBに書き込む。② その時刻がタスクの最終更新時刻として残る。③ Schedulerはワーカーを直接見ず、この時刻を読んで、最近更新されていれば動いていると判断する*

Schedulerはワーカーを直接監視しているわけではなく、DBに残った最終更新時刻だけを見ています。定期的にDBを確認し、状態が `running` のタスクのうち、次のどちらかに当てはまるものをゾンビと判定します（公式ドキュメント: [Zombie/Undead Tasks](https://airflow.apache.org/docs/apache-airflow/2.10.5/core-concepts/tasks.html#zombie-undead-tasks)）。

- タスクを実行しているジョブが `running` ではなくなっている
- 最終更新時刻から一定時間（既定で300秒）が過ぎている

判定されたタスクは失敗になります。再試行の設定があれば再試行されます。

![ハートビートが途絶えてからゾンビと判定されるまで](/images/airflow-zombie-job/fig2-timeline.png)
*Airflow上の状態は running のまま、ハートビートだけが止まる。最終更新時刻から300秒過ぎるとゾンビと判定される*

関係する設定は次の3つで、いずれも `[scheduler]` セクションにあります。

| 設定 | 既定値 | 意味 |
|---|---|---|
| `job_heartbeat_sec` | 5秒 | ハートビートを打つ間隔（最終更新時刻を書き換える間隔） |
| `scheduler_zombie_task_threshold` | 300秒 | 最終更新時刻からこの時間が過ぎたらゾンビとみなす |
| `zombie_detection_interval` | 10秒 | Schedulerがゾンビを探す間隔 |

ここで押さえておきたいのは、ゾンビタスクは原因の名前ではなく「ハートビートが途絶えた（最終更新時刻が書き換えられなくなった）」という結果だということです。途絶えた理由は複数考えられ、`Detected zombie job` のログにはどれなのか書かれていません。

## 発生を確認する

### Schedulerログを検索する

Google Cloudでは、Schedulerのログを `Detected zombie job` で検索します。[公式のトラブルシューティング](https://docs.cloud.google.com/composer/docs/composer-3/troubleshooting-dags#zombie-tasks)にも同じ検索条件が載っています。時刻はUTCで指定します。

```shell
gcloud logging read 'resource.type="cloud_composer_environment"
AND resource.labels.environment_name="YOUR_COMPOSER_ENVIRONMENT"
AND log_id("airflow-scheduler")
AND textPayload:"Detected zombie job"
AND timestamp>="2026-08-01T00:00:00Z"
AND timestamp<="2026-08-01T12:00:00Z"' \
  --project=YOUR_PROJECT_ID \
  --order=asc \
  --limit=1000 \
  --format='value(timestamp,textPayload)'
```

1件も見つからないときは、「発生していない」のか「ログの保持期間を過ぎている」のかを区別してください。保持日数は次のコマンドで確認できます。

```shell
gcloud logging buckets describe _Default \
  --location=global \
  --project=YOUR_PROJECT_ID \
  --format='value(retentionDays)'
```

期間全体の傾向を見たいときは、ゾンビとして停止したタスク数のメトリクス `composer.googleapis.com/environment/zombie_task_killed_count` をMetrics Explorerで表示すると早いです。メトリクスの一覧は[公式の監視ドキュメント](https://docs.cloud.google.com/composer/docs/composer-3/monitor-environments)にあります。

### ログから対象を読み取る

ログ本文の `msg` に、どのタスクがどのワーカーで動いていたかがそのまま書かれています。

```text
Detected zombie job: {'full_filepath': '/home/airflow/gcs/dags/sample_dag.py', ..., 'msg': "{'DAG Id': 'sample_dag', 'Task Id': 'load', 'Run Id': 'scheduled__2026-08-01T00:00:00+00:00', 'Hostname': 'airflow-worker-xxxxx'}", ...} (See https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#zombie-undead-tasks)
```

調査に使うのは次の4つです。件数が多いときは、ログの時刻とこの4つを表計算ソフトなどに書き出しておくと、あとの手順で見比べやすくなります。

| 項目 | 使いみち |
|---|---|
| ログの時刻 | メトリクスやほかのログと同じ時間帯で突き合わせる |
| DAG Id・Task Id | 影響した処理を特定する |
| Run Id | 同じDAGの別の実行と区別する |
| Hostname | 実行していたワーカー。複数のタスクが同じワーカーに集中していないかを見る |

## ハートビートが途絶える原因

![正常時と、ハートビートが途絶える2つの場合の比較](/images/airflow-zombie-job/fig3-causes.png)
*上:合図が届き続けるのでDBの最終更新時刻が更新され、正常と判断される。中:プロセスが消えて合図が送られなくなる。下:プロセスは動いているが合図がDBに届かない。中と下はどちらも最終更新時刻が古いままになり、ゾンビと判定される*

ハートビートが途絶える経路は大きく2つあります。プロセス自体が消えた場合と、プロセスは動いているのに合図がDBに届かない場合です。原因候補を経路ごとに整理すると次のようになります。

| 経路 | 原因 | 主な手がかり |
|---|---|---|
| プロセスが消えた | メモリ不足による強制終了 | ワーカーログの `Negsignal.SIGKILL`、メモリ使用率 |
| プロセスが消えた | ワーカーの再起動 | ワーカーの再起動回数 |
| プロセスが消えた | ワーカーの退避・縮小・環境更新 | Podの退避回数、ワーカー数の推移、監査ログ |
| 合図が届かない | ワーカーの高負荷 | CPU・メモリ使用率、同時実行数 |
| 合図が届かない | メタデータDBの高負荷・停止 | DBの正常性、CPU・メモリ使用率 |
| 合図が届かない | ネットワーク障害 | 接続エラー、ワーカーログの `Heartbeat time limit exceeded` |
| 設定 | 判定までの時間が短すぎる | 300秒を超える一時的な遅れが繰り返し起きている |

Google Cloudの[公式ドキュメント](https://docs.cloud.google.com/composer/docs/composer-3/monitor-key-metrics)では、最も多い原因はワーカーのCPU・メモリ不足とされています。そのため、次の手順もワーカーから見ていきます。

## 原因を切り分ける

原因候補が複数あるので、検出時刻の前後10〜15分に絞って、ワーカー → メタデータDB → 環境の変更 → 通信の順に確認します。

![検出時刻を軸に各情報を重ねて見る](/images/airflow-zombie-job/fig4-window.png)
*検出時刻を軸に、同じ時間帯の情報を重ねて見る。この例ではメモリの上昇と再起動が検出直前に重なり、環境の変更は時間帯の外にある*

以降のコマンドの `YOUR_PROJECT_ID`、`YOUR_COMPOSER_ENVIRONMENT`、`YOUR_WORKER_NAME` は調査対象に置き換えてください。メトリクスはすべてMetrics Explorer（`https://console.cloud.google.com/monitoring/metrics-explorer?project=YOUR_PROJECT_ID`）で表示できます。

### 1. 同じワーカーに集中していないか

最初に、先ほど読み取ったログの時刻と Hostname を並べて見比べます。

![同じワーカーに集中している場合とばらばらの場合](/images/airflow-zombie-job/fig5-same-worker.png)
*左:複数のDAGのゾンビが同じワーカー・同じ時間帯に集中している。右:ワーカーも時間帯もばらばら。並べ方が違うだけで、疑う対象が変わる*

複数のDAGのタスクが同じワーカーでほぼ同時にゾンビになっていれば、個々のタスクではなく、そのワーカー自体の再起動やリソース不足を疑います。ワーカーも時刻もばらばらなら、タスク固有の問題か、DBやネットワークなど環境全体の問題に目を向けます。

ただし、ここで分かるのは「同じワーカーに割り当てられていた」ことまでです。どのタスクが負荷をかけたのか、あるタスクが原因でほかのタスクまで止まったのかは、次の手順で確かめます。

### 2. ワーカーの再起動・退避

| 確認対象 | メトリクス・ログ |
|---|---|
| ワーカーの再起動回数 | `composer.googleapis.com/workload/restart_count` |
| Podの退避回数 | `composer.googleapis.com/environment/worker/pod_eviction_count` |
| 強制終了・停止の記録 | ワーカーログの `Negsignal.SIGKILL`、`Received SIGTERM` |

ワーカーログは次のコマンドで検索します。手順6で使う `Heartbeat time limit exceeded` も一緒に探しています。`Negsignal.SIGKILL` と `Received SIGTERM` の意味は、Google Cloudの[トラブルシューティング](https://docs.cloud.google.com/composer/docs/composer-3/troubleshooting-dags#sigterm)に説明があります。

```shell
gcloud logging read 'resource.type="cloud_composer_environment"
AND resource.labels.environment_name="YOUR_COMPOSER_ENVIRONMENT"
AND log_id("airflow-worker")
AND (textPayload:"Negsignal.SIGKILL"
     OR textPayload:"Received SIGTERM"
     OR textPayload:"Heartbeat time limit exceeded")
AND timestamp>="2026-08-01T00:00:00Z"
AND timestamp<="2026-08-01T12:00:00Z"' \
  --project=YOUR_PROJECT_ID \
  --order=asc \
  --limit=1000 \
  --format='value(timestamp,textPayload)'
```

検出の直前に同じワーカーの再起動や退避があれば、プロセスが失われたことが直接の要因です。ただし再起動の回数だけでは「なぜ再起動したのか」までは分からないので、次のメモリ・CPUと合わせて見ます。

### 3. ワーカーのメモリ・CPU

| 確認対象 | メトリクス・設定 |
|---|---|
| メモリ使用量 | `composer.googleapis.com/workload/memory/bytes_used` |
| メモリ上限 | `composer.googleapis.com/workload/memory/quota` |
| CPU使用時間 | `composer.googleapis.com/workload/cpu/usage_time` |
| 同時実行数の上限 | Airflow設定の `[celery] worker_concurrency` |

メモリは使用量そのものではなく、上限に対する割合で見ます。Metrics Explorerのクエリ言語に **PromQL** を選び、次のクエリを実行すると使用率（%）がそのまま表示されます。メトリクス名は[PromQLでの書き方](https://docs.cloud.google.com/monitoring/promql/promql-mapping)に従って、`.` と `/` を `_` に置き換えています。

```promql
100 *
composer_googleapis_com:workload_memory_bytes_used{
  monitored_resource="cloud_composer_workload",
  environment_name="YOUR_COMPOSER_ENVIRONMENT",
  workload_name="YOUR_WORKER_NAME"
}
/
composer_googleapis_com:workload_memory_quota{
  monitored_resource="cloud_composer_workload",
  environment_name="YOUR_COMPOSER_ENVIRONMENT",
  workload_name="YOUR_WORKER_NAME"
}
```

見るときの注意点は次のとおりです。

- 全ワーカーの合計ではなく、ゾンビになったタスクを実行していたワーカー（ログの Hostname）を指定する
- メトリクスは60秒間隔で記録されるため、短時間の急上昇は取りこぼす。検出前後の最大値を見る
- 80%を超える状態が続き、同じ時間帯に再起動や `Negsignal.SIGKILL` があれば、メモリ不足を有力な候補とする

`worker_concurrency` は1台のワーカーが同時に実行できるタスク数の上限であって、その数を安定して処理できるという保証ではありません。同じ時間帯に動いていたタスク数と、1タスクあたりの負荷を合わせて見ます。

### 4. メタデータDB

ワーカー側に異常が見つからなければ、ハートビートの書き込み先であるDBを疑います。

| 確認対象 | メトリクス |
|---|---|
| 正常性 | `composer.googleapis.com/environment/database_health` |
| CPU使用率 | `composer.googleapis.com/environment/database/cpu/utilization` |
| メモリ使用率 | `composer.googleapis.com/environment/database/memory/utilization` |

検出時刻にDBが正常で使用率も低ければ、DBが原因である可能性は下がります。

### 5. 環境の変更・メンテナンス

環境の更新やパッケージのインストール中はワーカーが入れ替わります。実行中のタスクが猶予時間内に終わらないと中断され、ゾンビとして検出されます。環境の変更は監査ログで確認できます。`log_id` の引数はURLエンコードせずに書きます（エンコードすると一致しなくなることが[クエリ言語の仕様](https://docs.cloud.google.com/logging/docs/view/logging-query-language#log_id)に書かれています）。

```shell
gcloud logging read 'resource.type="cloud_composer_environment"
AND resource.labels.environment_name="YOUR_COMPOSER_ENVIRONMENT"
AND log_id("cloudaudit.googleapis.com/activity")
AND timestamp>="2026-08-01T00:00:00Z"
AND timestamp<="2026-08-01T12:00:00Z"' \
  --project=YOUR_PROJECT_ID \
  --order=asc \
  --limit=1000 \
  --format='value(timestamp,protoPayload.methodName)'
```

自動スケールでワーカーが減っていないかは、ワーカー数のメトリクス `composer.googleapis.com/environment/num_celery_workers` の推移で分かります。

### 6. 通信エラーと設定値

ワーカーがDBに書き込めない状態が判定までの時間を超えると、ワーカー側のログにも `Heartbeat time limit exceeded` が残ります。手順2の検索でこれが見つかっていれば、ワーカーとDBの間の通信か、DB側の応答の遅れを疑います。DBが正常なら、ネットワークの接続エラーやタイムアウトを同じ時間帯のログから探します。

一時的な遅れが原因で、待てば回復していたと判断できる場合は、`scheduler_zombie_task_threshold` を伸ばす方法もあります。ただし伸ばした分だけ、本当に止まったタスクの検出も遅れます。

## 事実と推測を分ける

観測できた情報から言えることと言えないことを分けて記録します。ここを混同すると、再発防止策を誤ります。

| 観測結果 | 言えること | 言えないこと |
|---|---|---|
| `Detected zombie job` | ハートビートが途絶えた | 途絶えた理由 |
| 検出直前のワーカー再起動 | プロセスが失われた直接の要因 | 再起動の理由 |
| 高いメモリ使用率 | メモリ不足の可能性 | `Negsignal.SIGKILL` などの記録がない状態での強制終了の断定 |
| 複数DAGが同じワーカーで同時に検出 | ワーカー共通の問題の疑い | どのタスクが負荷をかけたか |
| エラーログがない | 該当するログを確認できなかった | エラーが起きなかったこと |

## Airflow 3 での変更点

Airflow 3では「ゾンビ」という呼び方がなくなり、「ハートビートのないタスクインスタンス」と表現されるようになりました（[公式ドキュメント](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#task-instance-heartbeat-timeout)）。仕組みは同じですが、ログの文言と設定名が変わっているため、検索条件は読み替えてください。Google CloudのマネージドAirflowでもAirflow 3のイメージが選べます。

| Airflow 2 | Airflow 3 |
|---|---|
| ログ `Detected zombie job` | ログ `Detected a task instance without a heartbeat` |
| `scheduler_zombie_task_threshold` | `task_instance_heartbeat_timeout` |
| `zombie_detection_interval` | `task_instance_heartbeat_timeout_detection_interval` |
