# Day 24：コスト・性能の継続改善を“定例化”する（レビュー観点＝指標の型）

---

Day19 で「性能はチューニングできる」、Day23 で「運用設計は SLO/アラート/Runbook で回る」と整理しました。
ただし現実は、リリースやトラフィック変動、依存サービス変更で **コストも性能も必ずズレます**。

そこで Day24 は結論から入ります。

* **コスト最適化は“イベント”ではなく“プロセス”**
* **最適化の成否は、定例レビューが“指標の型”を持っているかで決まる**

以降、Lambda の継続改善を回すための「定例の形」「見るべき指標」「改善の打ち手」をテンプレとしてまとめます。

---

## 1. まず押さえる：Lambda のコストはどこから出るか

Lambda の請求は大きく次で構成されます。

* **リクエスト数**
* **実行時間 × メモリ（GB-second）**
* **Provisioned Concurrency（使うなら別枠課金）**
* **/tmp の追加（エフェメラルストレージ 512MB 超過分）**
* （周辺コスト）CloudWatch Logs、X-Ray、データ転送、NAT Gateway、S3/DB 等

特に見落とされがちなのが **Provisioned Concurrency** と **/tmp 追加**と、そして **ログ量（CloudWatch Logs）** です。/tmp は 512MB までは無料で、超過分は GB-second 課金になります。 ([Amazon Web Services][1])
リクエスト課金や Provisioned Concurrency の料金体系も、まず “何が別枠か” だけ押さえておくとレビューが速くなります。 ([Amazon Web Services][2])

---

## 2. 定例化の設計：週次・月次・四半期で役割を分ける

「毎週、全部見る」は破綻します。定例は粒度を分けるのがコツです。

### 週次（30分）：異常検知と止血

目的：**先週からの急変（コスト/性能/エラー）を見つけて止血する**

* Cost Anomaly Detection の通知を確認（急増の根因を一旦特定） ([Amazon Web Services][3])
* p95/p99 Duration と Errors/Throttles の悪化確認（Day23 の SLO と突き合わせ）
* 「ログ爆増」「タイムアウト増」「並列詰まり」など、火種を潰す

アウトプット：**“今週やる改善チケット”を 1〜3 件に絞る**（やりすぎない）

### 月次（60分）：右肩上がりのムダを削る（最適化）

目的：**コスト効率を上げる（同じ品質で安く／同じコストで速く）**

* 上位関数（コスト/実行時間/エラー）の棚卸し
* メモリ設定の妥当性（後述）
* Provisioned Concurrency の効果検証（必要性・量・時間帯）
* CloudWatch Logs コスト（ログ量、保持期間、構造化）見直し

アウトプット：**翌月の改善バックログ**（合意した優先度つき）

### 四半期（90分）：設計変更を検討する（構造改善）

目的：**“設定変更では効かない領域”を、設計で取りに行く**

例：

* fan-out の形（SQS/Kinesis/EventBridge）の見直し
* バッチ化・集約・キャッシュ戦略
* VPC/NAT 依存の整理（Day21 の復習）
* 依存サービスのスループット設計（DB/外部API）

---

## 3. レビュー観点（指標）の「型」：この 10 個を毎回同じ順で見る

定例レビューは、毎回「同じ順序」で見ます。これが属人性を消します。

### A. 需要（Workload）

* Invocations（呼び出し数）
* 同時実行（ConcurrentExecutions）とスパイクの形

狙い：需要が増えたのか、処理が重くなったのかを分離する。

### B. 品質（SLO 直結）

* Duration（平均ではなく p95/p99）
* Errors（率で見る）
* Throttles（落ちている／詰まっている）

### C. リソース効率（ムダの温床）

* MaxMemoryUsed（メモリ余り/不足）
* Init Duration（コールドスタート影響：Day19〜Day23に接続）
* /tmp 使用量（大きいワークロードはコスト直撃）

補足：MaxMemoryUsed や CPU/ディスク/ネットワークまで見たいなら **Lambda Insights**が有効です（CPU時間、メモリ、ディスク、ネットワーク等を収集）。 ([AWS ドキュメント][4])
ただし Insights はメトリクスとログが増えるため、目的のある関数から段階的に使うのが現実的です。 ([AWS ドキュメント][4])

### D. キャパシティ設計（詰まりの根因）

* Reserved Concurrency / Provisioned Concurrency の設定
* Provisioned Concurrency を入れているなら「利用率」と「スピル（溢れ）」が妥当か

### E. コスト（結果）

* コスト上位の関数（サービス別ではなく“関数別/タグ別”に寄せる）
* コストの内訳：実行（GB-s）／リクエスト／PC／/tmp 追加／ログ

---

## 4. “定例で回る”ようにするコツ：指標をダッシュボード化して固定する

定例のたびに画面を探し回ると、絶対に続きません。

* Lambda コンソールには CloudWatch の機能を活用した **組み込みのインサイト/ダッシュボード**が追加されており、上位関数や異常を素早く把握できます（まずはここからで十分）。 ([Amazon Web Services][5])
* Cost Explorer は「保存したレポート」「タグ別」「アカウント別」を定例用に固定
* 重要関数だけ CloudWatch Dashboard（p95/p99、Errors、Throttles、MaxMemoryUsed）

---

## 5. 改善の打ち手：定例で“よく出る結論”を先にテンプレ化する

### パターン1：Duration は悪化、MaxMemoryUsed は余り → **CPU不足の可能性**

Lambda はメモリを上げると CPU も増えます（Day19 の復習）。
このとき「メモリを下げて節約」は逆効果になりがちです。

対策：

* メモリを 1段階上げて Duration が縮むかを計測
* “コスト = GB-s”なので、**速くなってトータルコストが下がる**ケースがある

### パターン2：Throttles 増 → **同時実行設計の問題**

対策：

* Reserved Concurrency で被害範囲を封じる（課金事故対策にもなる：Day19/Day23接続）
* SQS のバッチサイズや最大同時実行など、イベントソース側を調整（Day20 の fan-out に接続）

### パターン3：コスト急増 → **“需要増”か“効率悪化”かを分ける**

対策：

* Invocations が増えたなら：ビジネス要因、バッチ化/キャッシュ、上流の制御
* Duration が伸びたなら：依存先遅延、I/O増、ライブラリ更新、VPC/NAT要因（Day21）

### パターン4：/tmp を増やしたら地味に高い → **設計変更の検討**

/tmp 512MB 超過は GB-second 課金です。 ([Amazon Web Services][1])
画像/PDF 系は Day20 の文脈で、以下が効きます。

* まとめて処理 → 分割（ページ単位 fan-out）
* 解像度や中間生成物のサイズ最適化
* /tmp を増やす前に “書き出し回数” を減らす

---

## 6. 定例を“自動化で補助”する：おすすめ2つ（少ない労力で効く）

### (1) Cost Anomaly Detection：急増はまず機械に拾わせる

異常な支出パターンを検知してアラートできます（根因特定も支援）。 ([Amazon Web Services][3])
定例以前に「気づけない」を潰すのが目的です。

### (2) Compute Optimizer：メモリ推奨を“月次”の入力にする

Compute Optimizer は Lambda の **メモリサイズ推奨**を出せます。 ([AWS ドキュメント][6])
ただし要件があり、例えば「直近14日で一定回数呼ばれている」等を満たした関数が対象です。 ([AWS ドキュメント][7])
月次レビューで「上位関数の推奨だけ拾う」運用が現実的です。

---

## 7. “継続改善”を成立させる運用ルール（ここが一番重要）

最後に、仕組みを回すルールを固定します。

* **レビューは“意思決定の場”**：観測だけで終わらせない（必ずチケット化）
* **改善は WIP 制限**：同時に 1〜3 件まで（やり散らかさない）
* **変更には Deploy Marker**：いつ/何を変えたかを追跡（Day14/Day18に接続）
* **SLO が主語**：コスト削減で SLO を壊したら本末転倒（Day23に接続）

---

## 付録A：月次レビュー議事録テンプレ（コピペ用）

* 対象期間：YYYY/MM/DD〜YYYY/MM/DD
* 先月からの変更（Deploy Marker）：

  * （例）メモリ 1024→1536、SQS BatchSize 5→10、Layer 更新…など
* 先月の SLO 状況：達成/未達（未達の理由）
* Top3（コスト）：関数A / 関数B / 関数C
* Top3（性能劣化）：関数D / 関数E / 関数F
* アクション（最大3件）

  1. 何を：
     仮説：
     期待効果：
     計測方法：
     期限：
  2. …
  3. …

---


## 付録B：CloudWatch Logs Insights 定例クエリ（REPORT 行集計）

### 0) 共通：REPORT 行から主要項目を抽出するベース

以下の `parse` は、Lambda の REPORT 行（Duration / Billed Duration / Memory Size / Max Memory Used / Init Duration）を抜きます。

> Init Duration は「コールドスタートが発生したログにだけ出る」ため、出ない行もあります（= null になります）。

---

### 1) まず見る：p50/p95/p99 + MaxMemoryUsed + コールドスタート率（推奨：月次の1本目）

```sql
filter @type = "REPORT"
| parse @message /Duration: (?<duration_ms>[\d.]+) ms/ 
| parse @message /Billed Duration: (?<billed_ms>\d+) ms/
| parse @message /Memory Size: (?<memory_mb>\d+) MB/
| parse @message /Max Memory Used: (?<max_mem_mb>\d+) MB/
| parse @message /Init Duration: (?<init_ms>[\d.]+) ms/
| stats
    count(*) as invocations,
    percentile(duration_ms, 50) as p50_ms,
    percentile(duration_ms, 95) as p95_ms,
    percentile(duration_ms, 99) as p99_ms,
    avg(duration_ms) as avg_ms,
    max(duration_ms) as max_ms,
    max(max_mem_mb) as peak_max_mem_mb,
    avg(max_mem_mb) as avg_max_mem_mb,
    count(init_ms) as cold_starts,
    (count(init_ms) * 100.0 / count(*)) as cold_start_rate_pct
  by @log
| sort invocations desc
```

* `@log` はロググループ名（= 関数単位）です。複数関数を選ぶと「関数別の一覧」になります。
* コールドスタートは「Init Duration が出た REPORT 行の回数」で近似しています（実務上これで十分使えます）。

---

### 2) “メモリ余り/不足”を見抜く：MaxMemoryUsed の余裕（余裕率）チェック

```sql
filter @type = "REPORT"
| parse @message /Memory Size: (?<memory_mb>\d+) MB/
| parse @message /Max Memory Used: (?<max_mem_mb>\d+) MB/
| stats
    count(*) as invocations,
    avg(memory_mb - max_mem_mb) as avg_headroom_mb,
    min(memory_mb - max_mem_mb) as min_headroom_mb,
    percentile(max_mem_mb, 95) as p95_max_mem_mb,
    max(max_mem_mb) as peak_max_mem_mb,
    max(memory_mb) as configured_memory_mb
  by @log
| sort avg_headroom_mb asc
```

* `avg_headroom_mb` が大きすぎる → メモリ“盛りすぎ”の可能性
* `min_headroom_mb` が小さい（0に近い）→ メモリ不足（スワップは無いので、OOM→失敗に寄りやすい）

---

### 3) コスト目線：概算 GB-second を関数別に出す（Billed Duration ベース）

> これは厳密な請求額ではなく、**「コストに効く順（概算）」を並べる**用途に強いです。
> Lambda はメモリ課金が GB-second なので、`billed_ms × memory_mb` が効きます。

```sql
filter @type = "REPORT"
| parse @message /Billed Duration: (?<billed_ms>\d+) ms/
| parse @message /Memory Size: (?<memory_mb>\d+) MB/
| stats
    count(*) as invocations,
    sum(billed_ms) as sum_billed_ms,
    avg(billed_ms) as avg_billed_ms,
    max(billed_ms) as max_billed_ms,
    max(memory_mb) as configured_memory_mb,
    (sum(billed_ms) * max(memory_mb) / 1024.0 / 1000.0) as approx_gb_seconds
  by @log
| sort approx_gb_seconds desc
```

* `approx_gb_seconds` 上位 = 最適化の優先候補（Day24 の「月次でTopを見る」運用に直結します）

---

### 4) 時系列（週次の異常検知に便利）：1時間ごとの p95 / エラー率っぽいもの（REPORT だけ）

REPORT だけだと「関数エラー（Unhandled）」の率は取りにくいので、ここは **p95 の急変検知**に寄せます。

```sql
filter @type = "REPORT"
| parse @message /Duration: (?<duration_ms>[\d.]+) ms/
| stats
    count(*) as invocations,
    percentile(duration_ms, 95) as p95_ms,
    percentile(duration_ms, 99) as p99_ms,
    avg(duration_ms) as avg_ms,
    max(duration_ms) as max_ms
  by bin(1h), @log
| sort bin(1h) asc
```

* 週次レビューで「先週から p95 が跳ねた時間帯」をすぐ掘れます。

---

### 5) コールドスタートの“重さ”を見る：Init Duration の p95/p99（コールドスタートが痛い関数の特定）

```sql
filter @type = "REPORT"
| parse @message /Init Duration: (?<init_ms>[\d.]+) ms/
| filter ispresent(init_ms)
| stats
    count(*) as cold_starts,
    percentile(init_ms, 50) as init_p50_ms,
    percentile(init_ms, 95) as init_p95_ms,
    percentile(init_ms, 99) as init_p99_ms,
    avg(init_ms) as init_avg_ms,
    max(init_ms) as init_max_ms
  by @log
| sort cold_starts desc
```

* Provisioned Concurrency を検討する前に「そもそも Init がどれくらい痛いか」を数値で把握できます。

---

## 付録C：運用のコツ（Day24 の“定例化”に刺さる最小の補足）

* **週次（30分）**：クエリ(1) と (4) を回して「急変」と「トップ」を見つけ、チケットを最大3件に絞る
* **月次（60分）**：クエリ(3) で “概算 GB-second 上位” を見て、Day19 のメモリ×CPU最適化や Day20 の fan-out 設計見直しに繋げる
* **四半期（90分）**：クエリ(5) の結果を材料に、Provisioned Concurrency や依存構造の手当て（設計変更）を議題に上げる

---

## 付録D：CloudWatch Logs Insights 定例クエリ集（REPORT 行だけで p95 / MaxMemoryUsed / Init Duration を出す）

Day24 の「定例レビュー観点（指標）の型」を、毎回ブレずに回すための **CloudWatch Logs Insights クエリ集**です。
対象期間を選び、ロググループ `/aws/lambda/<function-name>`（複数選択可）を選んで実行します。

> **ポイント**
>
> * ここでは Lambda の **REPORT 行**を対象にします（`@type="REPORT"`）。
> * `Init Duration` は **コールドスタートが発生した REPORT 行にだけ出る**ため、出ない行は `null` になります。
> * 関数別に集計したいので、複数ロググループ選択時は `by @log`（=ロググループ名）でまとめます。

---

### D-0. 共通：REPORT 行の主要項目を抽出する（ベース）

以下の `parse` を使うと、REPORT 行から主要項目を取り出せます（クエリ内で必要な項目だけ残して使ってください）。

* Duration（ms）
* Billed Duration（ms）
* Memory Size（MB）
* Max Memory Used（MB）
* Init Duration（ms）

---

### D-1. 月次レビューの「1本目」：p50/p95/p99 + MaxMemoryUsed + コールドスタート率

定例レビューでまず見るべき項目を 1 回で出します。
「性能（p95/p99）」「メモリ（最大/平均）」「コールドスタート率」を同じ画面で確認できます。

```sql
filter @type = "REPORT"
| parse @message /Duration: (?<duration_ms>[\d.]+) ms/ 
| parse @message /Billed Duration: (?<billed_ms>\d+) ms/
| parse @message /Memory Size: (?<memory_mb>\d+) MB/
| parse @message /Max Memory Used: (?<max_mem_mb>\d+) MB/
| parse @message /Init Duration: (?<init_ms>[\d.]+) ms/
| stats
    count(*) as invocations,
    percentile(duration_ms, 50) as p50_ms,
    percentile(duration_ms, 95) as p95_ms,
    percentile(duration_ms, 99) as p99_ms,
    avg(duration_ms) as avg_ms,
    max(duration_ms) as max_ms,
    max(max_mem_mb) as peak_max_mem_mb,
    avg(max_mem_mb) as avg_max_mem_mb,
    count(init_ms) as cold_starts,
    (count(init_ms) * 100.0 / count(*)) as cold_start_rate_pct
  by @log
| sort invocations desc
```

**読み方（定例の判断基準の例）**

* `p95_ms / p99_ms` が先月より悪化 → 依存先遅延、I/O増、同時実行設計（Day19/Day20）を疑う
* `peak_max_mem_mb` が `memory_mb` に近い → OOM リスク（メモリ不足）
* `cold_start_rate_pct` が高い & Init が大きい → コールドスタート対策（初期化/依存削減/必要ならPC検討）

---

### D-2. メモリ余り/不足の棚卸し：ヘッドルーム（余裕）を見る

「メモリを盛りすぎている関数」「ギリギリで危ない関数」を短時間で見つけます。

```sql
filter @type = "REPORT"
| parse @message /Memory Size: (?<memory_mb>\d+) MB/
| parse @message /Max Memory Used: (?<max_mem_mb>\d+) MB/
| stats
    count(*) as invocations,
    avg(memory_mb - max_mem_mb) as avg_headroom_mb,
    min(memory_mb - max_mem_mb) as min_headroom_mb,
    percentile(max_mem_mb, 95) as p95_max_mem_mb,
    max(max_mem_mb) as peak_max_mem_mb,
    max(memory_mb) as configured_memory_mb
  by @log
| sort avg_headroom_mb asc
```

**読み方**

* `avg_headroom_mb` が極端に大きい → メモリ過剰（ただし CPU も増えるので Day19 の文脈で「性能とのトレードオフ」を評価）
* `min_headroom_mb` が 0 に近い → メモリ不足（失敗や不安定化の温床）

---

### D-3. コスト最適化の入口：概算 GB-second（“効く順”）で上位関数を出す

請求額の厳密再現ではなく、**月次レビューで「どれから触るべきか」**を出す目的のクエリです。
（メモリ課金は GB-second なので、`billed_ms × memory_mb` の寄与が大きい）

```sql
filter @type = "REPORT"
| parse @message /Billed Duration: (?<billed_ms>\d+) ms/
| parse @message /Memory Size: (?<memory_mb>\d+) MB/
| stats
    count(*) as invocations,
    sum(billed_ms) as sum_billed_ms,
    avg(billed_ms) as avg_billed_ms,
    max(billed_ms) as max_billed_ms,
    max(memory_mb) as configured_memory_mb,
    (sum(billed_ms) * max(memory_mb) / 1024.0 / 1000.0) as approx_gb_seconds
  by @log
| sort approx_gb_seconds desc
```

**読み方**

* `approx_gb_seconds` 上位 = 最適化優先候補

  * Day19：メモリ×CPU の最適点を探す
  * Day20：fan-out / バッチ化 / 中間生成物削減で Duration を削る

---

### D-4. 週次の異常検知に強い：1時間ごとの p95/p99 を時系列で出す

「いつ悪化したか」を切り出して、該当時間帯の変更（Deploy Marker）や依存先イベントに紐づけます。

```sql
filter @type = "REPORT"
| parse @message /Duration: (?<duration_ms>[\d.]+) ms/
| stats
    count(*) as invocations,
    percentile(duration_ms, 95) as p95_ms,
    percentile(duration_ms, 99) as p99_ms,
    avg(duration_ms) as avg_ms,
    max(duration_ms) as max_ms
  by bin(1h), @log
| sort bin(1h) asc
```

---

### D-5. コールドスタートの“重さ”を定量化：Init Duration の p95/p99 を出す

Provisioned Concurrency を検討する前に、「Init がどれくらい痛いか」を数値で把握します。

```sql
filter @type = "REPORT"
| parse @message /Init Duration: (?<init_ms>[\d.]+) ms/
| filter ispresent(init_ms)
| stats
    count(*) as cold_starts,
    percentile(init_ms, 50) as init_p50_ms,
    percentile(init_ms, 95) as init_p95_ms,
    percentile(init_ms, 99) as init_p99_ms,
    avg(init_ms) as init_avg_ms,
    max(init_ms) as init_max_ms
  by @log
| sort cold_starts desc
```

---

### D-6. 定例での使い分け（テンプレ）

* **週次（30分）**

  * A-1（全体の健康診断）
  * A-4（p95急変の時間帯を特定）
  * → “今週やる改善チケット”を最大 1〜3 件に絞る

* **月次（60分）**

  * A-3（概算 GB-second 上位を確定）
  * A-2（メモリ余り/不足を棚卸し）
  * → Day19/Day20 の打ち手へ落とし込む

* **四半期（90分）**

  * A-5（Init の痛さを定量化）
  * → Provisioned Concurrency や設計変更（構造改善）の議題化

---


## 付録E：定例レビュー議事録テンプレ（CloudWatch / Cost Explorer / Deploy Marker）

このテンプレは、Day24 の「定例化」を **“誰がやっても同じ粒度で回る”** 状態にするためのものです。
週次・月次・四半期で共通フォーマットにしておくと、比較が効き、改善が継続します。

> 使い方
>
> * **週次（30分）**：異常検知と止血（アクション最大3件）
> * **月次（60分）**：Top（コスト/性能）の棚卸しと最適化
> * **四半期（90分）**：構造改善（設計変更・PC検討・依存整理）

---

### E-1. 基本情報

* 定例種別：週次 / 月次 / 四半期
* 対象期間：YYYY/MM/DD 〜 YYYY/MM/DD
* 参加者：
* 対象環境：PROD / STAGING / 両方
* 対象サービス/関数範囲：例）`receipt-*` 系、`pdf-*` 系 など

---

### E-2. 先月（先週）からの変更一覧（Deploy Marker）

> ここが **比較の起点**。必ず埋める（“変えたのに記録が無い”が一番危険）

* Deploy Marker（貼り付け欄）

  * 例：`2025-12-XX | component=lambda | app=receipt | version=v1.12.0 | change=memory 1024→1536`
  * 例：`2025-12-XX | component=layer | name=lib-pillow | 12→13 | reason=CVE fix`
* 変更タイプ（チェック）

  * [ ] コード更新（Function）
  * [ ] Layer 更新
  * [ ] メモリ/タイムアウト/環境変数
  * [ ] Concurrency（Reserved/Provisioned）
  * [ ] イベントソース（SQS BatchSize 等）
  * [ ] VPC/NAT/Endpoint
  * [ ] 依存先（DB/API）
* 変更に紐づくチケット（URL / ID）：

---

### E-3. SLO/品質の状況（Day23 との接続点）

* 対象SLO（例：p95 < 500ms、Error rate < 0.1% など）：
* 達成状況：達成 / 未達
* 未達の根因（暫定でOK）：

  * 例：外部API遅延、コールドスタート増、スロットリング、リトライ増 など
* 今回の方針：**SLO を守るために優先する改善**（1行で）：

---

### E-4. CloudWatch（定例の“数字”）貼り付け欄

#### E-4-1. Logs Insights（付録Aの結果を貼る）

* 実行したクエリ：A-1 / A-2 / A-3 / A-4 / A-5
* 結果サマリ（貼る・または数値を転記）：

  * invocations 上位：
  * p95/p99 悪化関数：
  * cold_start_rate 上位：
  * approx_gb_seconds 上位（A-3）：
* 気づき（3行以内）：

  * 例：p95悪化は特定時間帯に集中、Deploy Marker と一致、など

#### E-4-2. ダッシュボード（ある場合）

* ダッシュボード名：
* スクショ/リンク貼り付け欄（任意）：
* 気づき（1〜2行）：

---

### E-5. Cost Explorer（定例の“金額”）貼り付け欄

> ここは「請求の事実」。CloudWatch は原因、Cost Explorer は結果。

* 対象期間の総額：
* 先月比（増減率）：+X% / -X%
* 増減要因（暫定でOK）：

  * 例：呼び出し増、GB-second 増、ログ増、PC増、NAT増、データ転送増
* 上位サービス（必要なら）：
  1.
  2.
  3.
* 関数別に寄せられるなら（タグ/コスト配賦がある場合）：

  * Top3 関数：A / B / C
* コスト異常（Cost Anomaly）が出たか：

  * 出た / 出てない
  * 出た場合の調査状況：

---

### E-6. トップ課題（優先順位づけ）

#### E-6-1. Top3（コスト）

1. 対象（関数名/ロググループ）：

   * 事実（数字）：approx_gb_seconds / Cost Explorer の増分
   * 仮説：需要増 / 効率悪化 / ログ爆増 / /tmp増 / PC過剰 など
   * 方針：まず何をするか（1行）

2.

3.

#### E-6-2. Top3（性能/信頼性）

1. 対象：

   * 事実（p95/p99、Errors、Throttles、Init 等）：
   * 仮説：依存先遅延 / 同時実行詰まり / コールドスタート / I/O など
   * 方針：

2.

3.

---

### E-7. 今回決めるアクション（最大3件：WIP制限）

> “観測で終わらせない”ための欄。**最大3件**に絞るのが継続のコツ。

#### Action 1

* 対象：
* 何をする：
* 仮説（なぜ効く）：
* 期待効果（KPI）：例）p95 -30%、approx_gb_seconds -20%
* 計測方法：付録Aのどのクエリで追うか（A-1/A-3/A-4/A-5）
* 実施期限：YYYY/MM/DD
* オーナー：
* 変更時の注意（SLO/リスク）：

#### Action 2

（同上）

#### Action 3

（同上）

---

### E-8. 変更後の検証（次回定例で確認する項目）

* 検証タイミング：次回週次 / 次回月次 / 臨時
* 観測する指標（付録A対応）：

  * [ ] p95/p99（A-1/A-4）
  * [ ] MaxMemoryUsed / headroom（A-2）
  * [ ] approx_gb_seconds（A-3）
  * [ ] Init Duration（A-5）
* 成功条件（合否ライン）：
* ロールバック条件（Day23 Runbook へリンク）：

---

### E-9. 参考（必要なら貼る）

* 関連Runbook：
* 関連チケット：
* 関連PR：
* 依存先の変更情報（RDS/外部API 等）：


[1]: https://aws.amazon.com/lambda/pricing/ "AWS Lambda Pricing"
[2]: https://aws.amazon.com/jp/lambda/pricing/ "AWS Lambda の料金"
[3]: https://aws.amazon.com/aws-cost-management/aws-cost-anomaly-detection/ "AWS Cost Anomaly Detection - Amazon Web Services"
[4]: https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/monitoring-insights.html "Amazon CloudWatch Lambda Insights を使用した関数 ..."
[5]: https://aws.amazon.com/jp/about-aws/whats-new/2024/10/aws-lambda-console-key-function-insights-amazon-cloudwatch-dashboard "AWS Lambda コンソールが組み込みの Amazon CloudWatch ..."
[6]: https://docs.aws.amazon.com/compute-optimizer/latest/ug/view-lambda-recommendations.html "Viewing Lambda function recommendations"
[7]: https://docs.aws.amazon.com/compute-optimizer/latest/ug/requirements.html "Resource requirements - AWS Compute Optimizer"
