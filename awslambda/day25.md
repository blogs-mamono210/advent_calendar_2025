# Day 25：AWS Lambda 運用大全（25日間の総まとめ）


---

## 対応Dayの早見（このページの読み方）

* **基礎・イベント駆動の土台**：Day1〜Day8
* **Layer / IaC / CI/CD / リリース**：Day9〜Day13
* **変更の可視化（Deploy Marker）**：Day14
* **可観測性（構造化ログ・メトリクス等）**：Day15〜Day19（特に Day19）
* **セキュリティ / 監査 / 脆弱性**：Day16〜Day18（特に Day16 / Day17 / Day18）
* **性能・重い処理・スケール設計**：Day19〜Day21（特に Day19 / Day20）
* **棚卸し（Well-Architected）**：Day22
* **運用設計（SLO/アラート/Runbook）**：Day23
* **継続改善の定例化（コスト・性能）**：Day24
* **総まとめ**：Day25

---

## はじめに

この 25 日間で、Lambda の基礎から運用、CI/CD、セキュリティ、性能最適化、そして AIOps まで、実務に必要な要素を“点”ではなく“体系”として学んできました。

Lambda は「書いて終わり」ではありません。
本番の現場で重要なのは、次の問いにいつでも答えられることです。

* いま動いているのは **どのバージョン**か？（**Day13**）
* いつ・誰が・なぜ変えたのか？（監査・説明責任）（**Day14 / Day18**）
* 遅い／高い／落ちる、が起きた時に **どこから切り分ける**か？（**Day19 / Day23 / Day24**）
* 改善は一度きりではなく **継続できる仕組み**になっているか？（**Day24**）

最終回では、これらを ひとつの運用アーキテクチャとして統合し、「明日から使える運用の型」をまとめます。

---

## Lambda 運用の全体像（全25記事の統合アーキテクチャ）

ポイントは、Lambda 運用を「コード」ではなく **ライフサイクル**として捉えることです。

```mermaid
flowchart LR
  Code[コード/テスト<br/>(Day1-8)] --> IaC[SAM IaC<br/>(Day11-12)]
  IaC --> CICD[CI/CD（OIDC）<br/>(Day11)]
  CICD --> Release[Version/Alias/Canary<br/>(Day13)]
  Release --> Lambda[Lambda 本番]

  Lambda --> Obs[可観測性（メトリクス/ログ/トレース）<br/>(Day15-19)]
  Lambda --> Layer[Layer 管理<br/>(Day9-10)]
  Layer --> Inspector[Inspector（CVE 検出）<br/>(Day17)]

  CICD --> Marker[Deploy Marker<br/>(Day14)]
  Marker --> Timeline[変更履歴タイムライン<br/>(Day14/Day18)]
  Obs --> Timeline

  Timeline --> Improve[継続改善（コスト/性能/SLO）<br/>(Day23-24)]
  Improve --> Backlog[改善バックログ化<br/>(Day24)]
  Backlog --> IaC
```

* デプロイは“変更の入口”（OIDCで安全に、Version/Aliasで壊れない）（**Day11 / Day13**）
* 観測は“切り分けの入口”（構造化ログ＋メトリクスで見える化）（**Day15 / Day19**）
* 監査は“説明責任の入口”（CloudTrail/Config/Deploy Marker で追跡）（**Day14 / Day18**）
* 改善は“繰り返せる仕組み”（定例化・バックログ化で継続）（**Day24**）

ここまで揃うと、Lambda は「運用できるプロダクト」になります。

---

## 25日間を「運用ライフサイクル」に再配置する

Day1〜Day24 の内容は、テーマ別に整理すると理解が一段深くなります。
（読者が「いま自分はどこを学んだのか」を俯瞰できるようになります）

### 1) 設計（壊れにくさの源泉）【対応：Day1〜Day8】

* イベント駆動・冪等性・リトライ前提（**Day1-8**）
* 例外設計（DLQ / 再処理 / Poison Message）（**Day1-8**）
* メモリ＝CPU という特性を踏まえた設計（**Day19 に接続**）

狙い： “落ちても復旧できる”を設計段階で作る

### 2) デプロイ（安全な変更）【対応：Day11〜Day14】

* SAM による IaC（**Day11-12**）
* OIDC による短命クレデンシャル（**Day11**）
* Version / Alias / Canary / Rollback（**Day13**）
* Deploy Marker による「いつ・何が・なぜ」の可視化（**Day14**）

狙い： 変更が怖くなくなる（むしろ、変更が速くなる）

### 3) 可観測性（調査スピード）【対応：Day15 / Day19 / Day23】

* 構造化ログ（検索できる、集計できる）（**Day15**）
* メトリクス（p95/p99 を見て “平均の罠” を避ける）（**Day19**）
* 変更履歴（タイムラインと紐付く）（**Day14 / Day18**）

狙い： 障害対応を「人の勘」から「再現可能な手順」に寄せる

### 4) セキュリティ・監査（説明責任）【対応：Day16〜Day18】

* IAM 最小権限、Secrets/KMS（**Day16**）
* ランタイム EOL 管理（**Day16**）
* Inspector（依存関係の脆弱性を継続検知）（**Day17**）
* CloudTrail / Config（いつ・誰が・何を）（**Day18**）

狙い： “安全に運用できる”を、監査に耐える形で実装する

### 5) 性能・コスト（継続改善）【対応：Day19〜Day24】

* メモリ×CPU・並列数・コールドスタート（**Day19**）
* 重い処理（PDF/画像）を Lambda の強みで最適化（**Day20**）
* fan-out / Step Functions のスケール設計（**Day20**＋（シリーズ内の該当日））
* 定例レビューで継続改善（コスト・性能の仕組み化）（**Day24**）
* 運用設計のゴール（SLO・アラート・Runbook）（**Day23**）

狙い： 一度速くするのではなく、速さと安さを維持する

---

## 本シリーズで扱った主要テーマ（最終版：対応Dayつき）

* Lambda 実行モデル（コールド/ウォーム、制約と強み）【**Day1-8 / Day19**】
* イベントソース設計（S3 / SQS / EventBridge）【**Day1-8**】
* 冪等性・リトライ・DLQ（落ちても戻せる設計）【**Day1-8**】
* SAM による IaC（テンプレートが“仕様書”になる）【**Day11-12**】
* Layer 設計・バージョン管理（依存の整理と更新）【**Day9-10**】
* CI/CD（OIDC / 自動デプロイ / 変更の標準化）【**Day11-12**】
* リリース戦略（Version / Alias / Canary / Rollback）【**Day13**】
* 可観測性（構造化ログ、メトリクス、ダッシュボード）【**Day15 / Day19 / Day25 付録D〜F**】
* セキュリティ（IAM / Secrets / KMS / EOL）【**Day16**】
* 監査（CloudTrail / Config / 変更履歴）【**Day18 / Day14**】
* 脆弱性管理（Inspector）【**Day17**】
* 性能最適化（メモリ＝CPU、p95、ボトルネック分解）【**Day19**】
* 大規模処理（PDF/画像、fan-out、Step Functions）【**Day20**】
* VPC / 閉域設計（VPCEndpoint 等）【**Day21**】
* 棚卸し（Well-Architected Serverless Lens）【**Day22**】
* 運用設計（SLO・アラート・Runbook）【**Day23**】
* 継続改善（コスト・性能・定例化）【**Day24**】
* AIOps（変更×ログ×メトリクスの統合で“原因に近づく”）【**Day14 / Day15 / Day19 / Day24**】

---

## Lambda 運用の最重要ポイント（完全版チェックリスト：対応Dayつき）

### 設計【対応：Day1〜Day8（＋Day19に接続）】

* 冪等性：同じイベントが複数回届いても結果が壊れない【**Day1-8**】
  落とし穴：SQS の再配送、EventBridge の重複を想定していない【**Day1-8**】
* イベント駆動の理解：再試行・順序・重複の性質を把握【**Day1-8**】
  落とし穴：SQS FIFO と Standard を同じ感覚で扱う【**Day1-8**】
* メモリ＝CPU：遅い時は“メモリ増”が最短ルート【**Day19**】
  落とし穴：コード最適化に入る前にリソース調整を試していない【**Day19**】
* タイムアウトと分割：300秒に収める（分割・fan-out・Step Functions）【**Day20**】
  落とし穴：重い処理を 1 発でやり切ろうとしてリトライ地獄【**Day20**】

### セキュリティ【対応：Day16〜Day18】

* IAM 最小権限【**Day16**】
  落とし穴：`*` を許してしまい監査で止まる【**Day16 / Day18**】
* Secrets / KMS【**Day16**】
  落とし穴：ローカル検証の都合で平文化し、そのまま本番へ【**Day16**】
* ランタイム EOL 管理【**Day16**】
  落とし穴：EOL が迫ってから慌てて全関数同時アップグレード【**Day16**】

### IaC / CI/CD【対応：Day11〜Day14】

* SAM 管理【**Day11-12**】
  落とし穴：手動変更が混ざり、差分が追えなくなる【**Day11-12**】
* OIDC【**Day11**】
  落とし穴：権限が強すぎるロールを 1 本で使い回す【**Day11**】
* Version / Alias / Canary【**Day13**】
  落とし穴：ロールバック手段がなく“戻せない変更”になる【**Day13**】
* Deploy Marker【**Day14**】
  落とし穴：変更履歴が Slack や個人メモに散って追えない【**Day14**】

### 運用・可観測性【対応：Day15 / Day17 / Day18 / Day19 / Day23 / Day24】

* 構造化ログ【**Day15**】
  落とし穴：文字列ログだけで Insights 集計ができない【**Day15 / Day25付録D**】
* メトリクス観測（p95/p99・エラー率）【**Day19**】
  落とし穴：平均が良いのに“遅い”苦情が止まらない【**Day19**】
* Inspector【**Day17**】
  落とし穴：依存更新が属人化し、古い脆弱性が残る【**Day17**】
* CloudTrail / Config【**Day18**】
  落とし穴：障害時に「最後に触った人探し」になってしまう【**Day18**】
* AIOps（変更×指標×ログ）【**Day14 / Day15 / Day19 / Day24**】
  落とし穴：AI 導入が目的化し、入力データ（ログ設計）が未整備【**Day15**】

---

## Day25 で押さえる「運用の型」：定例化がすべてを救う【対応：Day23 / Day24】

* 毎週（30分）：エラー率・p95・コスト増の有無を確認【**Day19 / Day24 / Day25付録D**】
* 隔週〜月次（60分）：改善バックログの消化【**Day24**】
* 四半期：ランタイム更新計画、依存（Layer）更新、権限棚卸し【**Day16 / Day17 / Day18**】

これを「個人の頑張り」ではなく **仕組み（チェックリスト＋議事録テンプレ）**に落とすと、チーム運用に耐える体系になります。
（Day24 の「継続改善定例化」との接続点もここで明確になります）

---

## 25日間の学習を終えたあなたへ（次にやること）

次のステップ（おすすめ順）

1. 自分の関数にチェックリストを当てて“穴”を見つける【**Day25（本記事）**】
2. Version/Alias/Canary を導入して “戻せる変更” にする【**Day13**】
3. 構造化ログ＋Deploy Marker を入れて、調査時間を短縮する【**Day15 / Day14**】
4. 定例レビュー（コスト・性能）を小さく始めて継続する【**Day24 / Day25付録C〜E**】
5. 余力が出たら Step Functions / fan-out で重い処理や大規模処理を安定化する【**Day20**】

---

## おわりに

Lambda 運用で本当に難しいのは、技術そのものではなく 「続けられる形」にすることです。
25 日間で積み上げたのは、まさにその“続けられる運用設計”の部品一式です。

あとは、あなたのプロジェクトに合わせてテンプレートを調整し、**実運用のループ（観測→改善→標準化）**へ落とし込んでください。
このシリーズが、あなたの Lambda 運用の助けになれば幸いです。

---



[1]: https://qiita.com/mamono210/items/d5d03411fb7fd812233e?utm_source=chatgpt.com "AWS Lambda 超入門：最初の10分で仕組みと概要を理解する ..."
[2]: https://qiita.com/mamono210/items/001addc0a9b7f16afbf6?utm_source=chatgpt.com "Lambda 実行環境の正しい理解（Cold Start / CPU / tmp / 同時 ..."
[3]: https://qiita.com/mamono210/items/dd0b0a4791a57f6785d8?utm_source=chatgpt.com "Day 3：イベント駆動の基本（S3 / SQS / EventBridge）"
[4]: https://qiita.com/mamono210/items/87cfc14b4f7d94db91d0?utm_source=chatgpt.com "Day 4：IAM が分からないと Lambda は詰む（権限設計入門）"
[5]: https://qiita.com/mamono210/items/f762698e629b122d4e2b?utm_source=chatgpt.com "Day 5：S3 → Lambda が動かない理由トップ10（完全保存版）"
[6]: https://qiita.com/mamono210/items/a43e533bf5ad8facb21a?utm_source=chatgpt.com "Day 6：AWS SAM 入門（IaCでLambda管理）"
[7]: https://qiita.com/mamono210/items/627a69eb4ceed2252c48?utm_source=chatgpt.com "Day 7：template.yaml の設計原則（Globals / Parameters ..."
[8]: https://qiita.com/mamono210/items/8e93a53489891294b270?utm_source=chatgpt.com "Day 8：Lambda Layer 完全ガイド（依存管理とディレクトリ構造）"


## 付録：最小の成果物セット（これだけで運用が回り始める）

このシリーズの内容を“知識”で終わらせないために、最後に **最小限の成果物（テンプレ）**を置きます。
まずはこの 3 点だけ整備してください。

* A. 構造化ログのフィールド一覧（ログ設計の標準）
* B. Deploy Marker の項目（変更履歴の標準）
* C. 定例議事録テンプレ（継続改善の標準）

---

## 付録A：構造化ログの最小フィールド一覧（JSON）

### 目的

* 障害調査を「勘」ではなく **検索・集計**で進める
* 変更（Deploy Marker）とログを紐付け、原因に近づける
* CloudWatch Logs Insights で **p95 / エラー率 / 影響範囲**が即座に出せる

### 最小セット（推奨）

以下のフィールドが揃っていれば、運用は十分に回せます。

#### 1) 必須（全ログ共通）

| フィールド        | 例                               | 意味                                    |
| ------------ | ------------------------------- | ------------------------------------- |
| `ts`         | `2025-12-20T12:34:56.789+09:00` | タイムスタンプ（ISO8601）                      |
| `level`      | `INFO` / `WARN` / `ERROR`       | ログレベル                                 |
| `msg`        | `"completed"`                   | 人間が読むメッセージ                            |
| `service`    | `"receipt-api"`                 | サービス名（横断検索の軸）                         |
| `env`        | `stg` / `prod`                  | 環境                                    |
| `function`   | `"shitadan-reciept-pdftojpeg"`  | Lambda 関数名                            |
| `request_id` | `"…"`                           | AWS RequestId（調査の最小単位）                |
| `trace_id`   | `"…"`                           | X-Ray / OpenTelemetry の trace（任意だが強い） |

#### 2) 変更追跡（デプロイと紐付ける）

| フィールド       | 例                           | 意味                      |
| ----------- | --------------------------- | ----------------------- |
| `git_sha`   | `"a1b2c3d"`                 | デプロイしたコミット              |
| `version`   | `"42"`                      | Lambda Version          |
| `alias`     | `"prod"`                    | Alias（本番経路）             |
| `deploy_id` | `"2025-12-20T030001Z-1234"` | Deploy Marker と紐付ける一意ID |

#### 3) 運用で効く（最小の“業務コンテキスト”）

| フィールド             | 例                            | 意味              |
| ----------------- | ---------------------------- | --------------- |
| `event_source`    | `sqs` / `s3` / `eventbridge` | イベント種別          |
| `entity_id`       | `"user-123"` / `"order-456"` | 影響範囲を切るキー（任意）   |
| `idempotency_key` | `"…"`                        | 冪等性キー（採用している場合） |

#### 4) エラー時に必須（例外の標準化）

| フィールド           | 例                     | 意味              |
| --------------- | --------------------- | --------------- |
| `error_type`    | `"TimeoutError"`      | 例外種別            |
| `error_message` | `"S3 read timed out"` | 例外メッセージ（長すぎ注意）  |
| `stacktrace`    | `"…"`                 | スタックトレース（必要時のみ） |

### ログ設計の運用ルール（最小）

* **INFO は「開始」「完了」「主要分岐」**だけに絞る（多すぎると調査性が落ちる）
* `deploy_id` / `git_sha` / `alias` / `version` は **必ず全ログに載せる**（後から効く）
* 例外は `ERROR` に統一し、`error_type` を必ず付ける（Insights 集計が楽）

---

## 付録B：Deploy Marker の最小項目（変更履歴の標準）

### 目的

* 「いつ・誰が・何を・なぜ変えたか」を **機械的に追跡**できるようにする
* 障害が起きた瞬間に「直近変更」を 1発で引けるようにする
* AIOps（分析）に渡せる形にする

### 最小セット（イベント1件のスキーマ）

| フィールド            | 例                                            | 意味             |
| ---------------- | -------------------------------------------- | -------------- |
| `ts`             | `2025-12-20T03:00:01Z`                       | デプロイ時刻         |
| `deploy_id`      | `2025-12-20T030001Z-1234`                    | 一意ID（ログとも共通）   |
| `env`            | `prod`                                       | 環境             |
| `service`        | `receipt-api`                                | 対象サービス         |
| `stack_name`     | `receipt-api-prod`                           | スタック名          |
| `actor`          | `circleci@oidc`                              | 実行主体（人 or ロール） |
| `git_sha`        | `a1b2c3d`                                    | 変更元            |
| `ref`            | `main`                                       | ブランチ/タグ        |
| `artifact`       | `lambda:function:shitadan-reciept-pdftojpeg` | 何を変えたか（対象）     |
| `change_summary` | `timeout 180->300, memory 1024->1536`        | 変更要約（短文）       |
| `reason`         | `p95改善 / OOM回避`                              | 変更理由（必須）       |
| `risk`           | `low/medium/high`                            | リスク（運用判断）      |
| `rollback`       | `alias rollback to version 41`               | ロールバック手順（短く）   |
| `links`          | `runbook / PR / ticket`                      | 関連リンク（台帳化）     |

### 運用ルール（最小）

* `change_summary` と `reason` は **必須**（ここが空だと調査が破綻）
* 変更の単位は「関数」ではなく、**サービス/スタック単位**でまとめても良い（現場に合わせる）
* ロールバックは「誰でも 5 分でできる」粒度で 1 行にする

---

## 付録C：定例議事録テンプレ（30〜60分で回す）

### 目的

* コスト・性能・信頼性の改善を **属人化させず、継続できる形**にする
* “見た”で終わらせず、必ず **アクション（バックログ）**に落とす
* Day24 の「継続改善定例化」を、Day25 の成果物として完結させる

以下をそのままコピペして使ってください（Markdown）。

---

### 運用定例（Lambda）議事録テンプレ

* 日時：
* 参加者：
* 対象環境：`prod / stg`
* 対象サービス（スタック名）：
* 期間：`YYYY-MM-DD 〜 YYYY-MM-DD`

#### 1. 先週の結論（3行）

* 良かった点：
* 悪かった点（兆候含む）：
* 今週の最優先：

---

#### 2. SLO/信頼性（Error / Throttle / DLQ）

* エラー率（全体 / 主要関数）：
* Throttles / Concurrency：
* DLQ 件数 / 再処理状況：
* 重大インシデント：有 / 無

  * 有の場合：概要／影響／暫定対応／恒久対応（リンク）

---

#### 3. 性能（p95/p99 / Cold Start）

* p95 / p99 Duration（主要関数）：
* Init Duration（コールドスタートの増減）：
* タイムアウト発生：有 / 無
* ボトルネック仮説（3つまで）：
  1)
  2)
  3)

---

#### 4. コスト（増分と要因）

* 先週比の増減（概算でOK）：
* 上振れ要因（例：リトライ増 / fan-out 増 / メモリ増）：
* すぐ効く手当（例：メモリ調整 / BatchSize / 再試行制御）：

---

#### 5. 変更履歴（Deploy Marker 起点）

* 期間内 Deploy Marker 一覧（リンク）：
* 影響が大きかった変更（上位3つ）：

  1. deploy_id / change_summary / reason
  2.
  3.
* 変更と指標変動の相関（あれば）：

---

#### 6. セキュリティ（Inspector / EOL / 権限）

* Inspector Findings（新規 / 未対応）：
* ランタイム / 依存更新（予定・期限）：
* IAM / Secrets / KMS の棚卸し要否：

---

#### 7. 決定事項（Decision）

* 決定1：
* 決定2：

---

#### 8. アクション（Backlog化）

| 優先度 | 期限 | 担当 | 内容 | 期待効果 | 参照（ログ/メトリクス/Deploy） |
| --- | -- | -- | -- | ---- | ------------------- |
| P0  |    |    |    |      |                     |
| P1  |    |    |    |      |                     |
| P2  |    |    |    |      |                     |

---

#### 9. 次回までの宿題（任意）

* 宿題1：
* 宿題2：

---

## 付録を入れたことで「運用大全」になる理由

* 付録Aで **ログ設計が標準化**される（調査が速くなる）
* 付録Bで **変更履歴が標準化**される（説明責任と相関分析ができる）
* 付録Cで **改善が定例運用として回り始める**（継続できる）

この 3 点は、最小構成でありながら、チーム運用の背骨になります。

---

## 付録D：CloudWatch Logs Insights の最小定例クエリ集（週次レビュー用）

この付録では、Lambda の運用レビューを最短で回すために、CloudWatch Logs Insights の **定例クエリ（コピペ用）**をまとめます。

### 前提（ログ要件）

この付録のクエリは、以下のどちらでも動きます。

* **A) Lambda の `REPORT` 行**（デフォルトで出る）を集計する
* **B) 付録Aの構造化ログ**（JSON）を集計する（`deploy_id` を含む）

`deploy_id` は B) の構造化ログに入っている想定です。
（もし未導入なら、まずは A) の REPORT 集計だけでも“改善定例”は開始できます）

---

### D-0. まず期間とロググループを決める（定例のやり方）

* 期間：直近 7 日（週次）または直近 14 日（隔週）
* 対象：サービス単位で 1〜N 個の log group（例：`/aws/lambda/<function>` を束ねる）

---

## D-1：p95 / p99 / 平均 / 最大 Duration（REPORT 行から）

**目的**：平均に騙されず、尾の遅さ（p95/p99）で性能劣化を検出する。

```sql
fields @timestamp, @message
| filter @type = "REPORT"
| parse @message /Duration: (?<duration_ms>[\d.]+) ms/ 
| stats
    count() as invocations,
    avg(duration_ms) as avg_ms,
    pct(duration_ms, 95) as p95_ms,
    pct(duration_ms, 99) as p99_ms,
    max(duration_ms) as max_ms
  by bin(1d)
| sort bin(1d) desc
```

**読み方（定例の判断）**

* p95 が上がる：外れ値が増えている（依存先遅延／コールドスタート増／I/O詰まりの可能性）
* max だけ跳ねる：タイムアウト予備軍。例外ログと突合する

---

## D-2：MaxMemoryUsed の p95 と最大（REPORT 行から）

**目的**：OOM 予兆と、メモリ過剰割当（コスト）の両方を検出する。

```sql
fields @timestamp, @message
| filter @type = "REPORT"
| parse @message /Max Memory Used: (?<max_mem_mb>\d+) MB/
| stats
    count() as invocations,
    pct(max_mem_mb, 95) as p95_max_mem_mb,
    max(max_mem_mb) as max_max_mem_mb
  by bin(1d)
| sort bin(1d) desc
```

**読み方（定例の判断）**

* `max_max_mem_mb` が設定メモリに張り付く：OOM 直前（メモリ増 or 処理分割を検討）
* p95 が低いのに設定メモリが高い：下げ余地（コスト最適化の即効薬）

---

## D-3：Init Duration（コールドスタート）の傾向（REPORT 行から）

**目的**：コールドスタート増加をトレンドで検知する（Day19/Day20/Day21 接続）。

```sql
fields @timestamp, @message
| filter @type = "REPORT"
| parse @message /Init Duration: (?<init_ms>[\d.]+) ms/ 
| filter ispresent(init_ms)
| stats
    count() as cold_starts,
    avg(init_ms) as avg_init_ms,
    pct(init_ms, 95) as p95_init_ms,
    max(init_ms) as max_init_ms
  by bin(1d)
| sort bin(1d) desc
```

**読み方（定例の判断）**

* cold_starts が増える：トラフィック変動・スケールパターン・VPC・依存初期化の影響を疑う
* init が長い：初期化処理（ライブラリ・フォント・PDF等）や VPC/ENI の影響を疑う

---

## D-4：エラー率（構造化ログの level=ERROR から）

**目的**：失敗の増加を“率”で捉える（件数だけだとトラフィック増で誤判定する）。

> 前提：アプリログが JSON で `level` を持つ（付録A）。

```sql
fields @timestamp, level
| filter level in ["ERROR", "WARN", "INFO"]
| stats
    count() as total,
    sum(case(level="ERROR", 1, 0)) as errors,
    (100.0 * sum(case(level="ERROR", 1, 0)) / count()) as error_rate_pct
  by bin(1h)
| sort bin(1h) desc
```

**読み方（定例の判断）**

* error_rate がじわ増え：依存先／入力品質／リトライ地獄（SQS 再配送）を疑う
* スパイク：直近の Deploy Marker と突合（D-6 へ）

---

## D-5：エラーの内訳トップ（error_type / msg）

**目的**：どの種類の失敗が増えているかを即断する（バックログ化しやすい）。

```sql
fields @timestamp, level, error_type, msg
| filter level = "ERROR"
| stats count() as error_count by error_type, msg
| sort error_count desc
| limit 30
```

**読み方（定例の判断）**

* `Timeout` 系：外部I/O、並列、BatchSize、Step Functions 分割を検討
* `AccessDenied` 系：IAM変更・環境差分・KMS/Secrets の権限不足
* `Throttling` 系：依存先（DynamoDB等）や Lambda 同時実行の制御を検討

---

## D-6：deploy_id 別の p95 とエラー（“変更×影響”を一撃で見る）

**目的**：AIOps の最小形。**どのデプロイが指標を悪化させたか**を即座に当てる。
（Day14 Deploy Marker → Day19 指標 → Day24 定例化、の一本線）

> 前提：構造化ログに `deploy_id` と `duration_ms`（または処理時間）が入っている。
> もし `duration_ms` が無い場合でも、まずはエラーだけ集計すれば十分です。

### D-6A：deploy_id 別 p95（アプリが duration_ms を出している場合）

```sql
fields @timestamp, deploy_id, duration_ms
| filter ispresent(deploy_id) and ispresent(duration_ms)
| stats
    count() as n,
    pct(duration_ms, 95) as p95_ms,
    pct(duration_ms, 99) as p99_ms,
    avg(duration_ms) as avg_ms,
    max(duration_ms) as max_ms
  by deploy_id
| sort p95_ms desc
| limit 50
```

### D-6B：deploy_id 別 エラー率（最重要）

```sql
fields @timestamp, deploy_id, level
| filter ispresent(deploy_id) and level in ["ERROR","INFO","WARN"]
| stats
    count() as total,
    sum(case(level="ERROR", 1, 0)) as errors,
    (100.0 * sum(case(level="ERROR", 1, 0)) / count()) as error_rate_pct
  by deploy_id
| sort error_rate_pct desc, errors desc
| limit 50
```

**読み方（定例の判断）**

* 特定 deploy_id の error_rate が突出：その変更を最優先で疑う（ロールバック判断が速くなる）
* p95 が上がり、error_rate も上がる：ほぼその変更が原因の可能性が高い

---

## D-7（任意）：特定 deploy_id のログを掘る（調査の入口）

**目的**：D-6 で怪しい deploy_id を見つけたら、即掘る。

```sql
fields @timestamp, level, msg, error_type, request_id, deploy_id
| filter deploy_id = "ここに deploy_id を貼る"
| sort @timestamp desc
| limit 200
```

---

## 定例での “最小運用フロー”（これだけで回る）

1. **D-1〜D-3** で p95 / メモリ / コールドスタートのトレンドを見る
2. **D-4〜D-5** でエラー率と内訳を把握する
3. 悪化があれば **D-6** で deploy_id に紐付け、変更と相関を見る
4. 怪しい deploy_id を **D-7** で掘り、バックログ（付録C）に落とす

この流れが回り始めると、Lambda 運用は「頑張り」ではなく「仕組み」になります。

---

## 付録E：CloudWatch Dashboard の最小ウィジェット構成（週次定例の入口）

CloudWatch Logs Insights（付録D）は「深掘り」に強い一方、週次定例ではまず **全体の変化を一目で掴む入口**が必要です。
その入口として、CloudWatch Dashboard を **最小5ウィジェット**で構成します。

### 目的（このダッシュボードで達成すること）

* 週次で「変化が起きたか」を 5 分で判断できる
* 変化があれば、その場で Insights（付録D）に飛べる
* 初心者でも「どこを見ればいいか」が固定化される

---

## E-1：最小構成（5ウィジェット）

> 対象は “主要関数（重要度上位3〜5個）” から始めるのがコツです。
> いきなり全関数を載せると、見たい情報が埋もれます。

### 1) Duration p95（最重要）

* **見るもの**：p95 Duration（関数別、またはサービス別）
* **判断**：p95 が上がった日／時間帯があるか
* **深掘り**：付録D-1（p95/p99集計）→ 付録D-6（deploy_id別）

### 2) Errors（エラー件数）

* **見るもの**：Errors（関数別）
* **判断**：いつもより増えていないか
* **深掘り**：付録D-4（エラー率）→ 付録D-5（内訳）

### 3) Throttles（抑止）

* **見るもの**：Throttles
* **判断**：スパイクがあるか（依存先/同時実行/予約同時実行の問題）
* **深掘り**：同時実行数設定、イベントソース（SQS等）を点検

### 4) MaxMemoryUsed（メモリ使用量）

* **見るもの**：MaxMemoryUsed（可能なら p95 の目線で見る）
* **判断**：設定メモリに張り付いていないか／余らせすぎていないか
* **深掘り**：付録D-2（p95/最大の集計）

### 5) Init Duration（コールドスタート）

* **見るもの**：Init Duration（増加傾向）
* **判断**：cold start が増えていないか、init が伸びていないか
* **深掘り**：付録D-3（Init集計）→ 依存初期化や VPC を疑う

---

## E-2：最小ダッシュボード JSON（コピペ用）

以下は **CloudWatch Dashboard の body** として貼れる JSON です。
`YOUR_REGION` / `FUNCTION_NAME_1..3` を置換してください。

* Region 例：`ap-northeast-1`
* 関数名は “主要関数3つ” を例にしています（増やす場合はメトリクス配列を追加）

> 注意：CloudWatch ダッシュボードの JSON は環境により微調整が必要な場合があります（関数名・リージョン・期間）。ただし、この最小形はほとんどのケースでそのまま動きます。

```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 24,
      "height": 6,
      "properties": {
        "view": "timeSeries",
        "stacked": false,
        "region": "YOUR_REGION",
        "title": "Duration p95 (ms) - Top Functions",
        "period": 60,
        "stat": "p95",
        "metrics": [
          [ "AWS/Lambda", "Duration", "FunctionName", "FUNCTION_NAME_1" ],
          [ ".", "Duration", "FunctionName", "FUNCTION_NAME_2" ],
          [ ".", "Duration", "FunctionName", "FUNCTION_NAME_3" ]
        ]
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 6,
      "width": 12,
      "height": 6,
      "properties": {
        "view": "timeSeries",
        "stacked": false,
        "region": "YOUR_REGION",
        "title": "Errors - Top Functions",
        "period": 60,
        "stat": "Sum",
        "metrics": [
          [ "AWS/Lambda", "Errors", "FunctionName", "FUNCTION_NAME_1" ],
          [ ".", "Errors", "FunctionName", "FUNCTION_NAME_2" ],
          [ ".", "Errors", "FunctionName", "FUNCTION_NAME_3" ]
        ]
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 6,
      "width": 12,
      "height": 6,
      "properties": {
        "view": "timeSeries",
        "stacked": false,
        "region": "YOUR_REGION",
        "title": "Throttles - Top Functions",
        "period": 60,
        "stat": "Sum",
        "metrics": [
          [ "AWS/Lambda", "Throttles", "FunctionName", "FUNCTION_NAME_1" ],
          [ ".", "Throttles", "FunctionName", "FUNCTION_NAME_2" ],
          [ ".", "Throttles", "FunctionName", "FUNCTION_NAME_3" ]
        ]
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 12,
      "width": 12,
      "height": 6,
      "properties": {
        "view": "timeSeries",
        "stacked": false,
        "region": "YOUR_REGION",
        "title": "MaxMemoryUsed (MB) - Top Functions",
        "period": 60,
        "stat": "Maximum",
        "metrics": [
          [ "AWS/Lambda", "MaxMemoryUsed", "FunctionName", "FUNCTION_NAME_1" ],
          [ ".", "MaxMemoryUsed", "FunctionName", "FUNCTION_NAME_2" ],
          [ ".", "MaxMemoryUsed", "FunctionName", "FUNCTION_NAME_3" ]
        ]
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 12,
      "width": 12,
      "height": 6,
      "properties": {
        "view": "timeSeries",
        "stacked": false,
        "region": "YOUR_REGION",
        "title": "InitDuration (ms) - Cold Start Trend",
        "period": 60,
        "stat": "Average",
        "metrics": [
          [ "AWS/Lambda", "InitDuration", "FunctionName", "FUNCTION_NAME_1" ],
          [ ".", "InitDuration", "FunctionName", "FUNCTION_NAME_2" ],
          [ ".", "InitDuration", "FunctionName", "FUNCTION_NAME_3" ]
        ]
      }
    }
  ]
}
```

---

## E-3：運用のコツ（最小構成を崩さない）

### 1) まず “3〜5関数だけ” に絞る

* 週次定例で見るのは「重要度上位」だけで十分です。
* 重要度の目安：呼び出し回数、ビジネス影響、過去に障害が多い、コストが高い。

### 2) period/stat を固定して比較可能にする

* Duration は **p95**（平均は補助）
* Errors/Throttles は **Sum**
* MaxMemoryUsed は **Maximum**（まずは張り付き検知）
* InitDuration は **Average**（傾向を見る）

### 3) 変化があったら Insights に降りる

* p95 上昇 → 付録D-1 / D-6
* Errors 上昇 → 付録D-4 / D-5 / D-6
* MaxMemoryUsed 張り付き → 付録D-2
* InitDuration 増加 → 付録D-3

---

## E-4：この付録が Day25 に効く理由

* ダッシュボードは **定例の入口（スクリーニング）**
* Logs Insights は **根拠の深掘り（集計・相関）**
* Deploy Marker は **変更と紐付け（説明責任と因果推定）**

この 3 点が揃うと、「見て終わり」ではなく **運用改善ループ** が回り始めます。

---

## 付録F：CloudWatch Dashboard を IaC（SAM/CloudFormation）で管理する最小例

### 目的

* ダッシュボードを手作業で作らず、**IaC で再現可能**にする
* `stg/prod` で **ダッシュボード名と対象関数**を切り替える
* CI/CD から `sam deploy --parameter-overrides` で **主要関数を注入**して運用に乗せる

---

## F-1：SAM/CloudFormation テンプレ（最小）

以下をあなたの `template.yaml` に追加（または別スタックとして分離）してください。
CloudWatch Dashboard は `AWS::CloudWatch::Dashboard` で作れます。

> ポイント
>
> * `DashboardName` は `ServiceName` と `EnvName` から合成
> * 主要関数名（3つ）を Parameters で受け取り、widgets のメトリクスに差し込む
> * DashboardBody は JSON を **`!Sub`** で組み立てます（Parameter 置換ができる）

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Day25 Appendix F - CloudWatch Dashboard IaC minimal example

Parameters:
  ServiceName:
    Type: String
    Default: receipt-api
    Description: Logical service name used for dashboard naming.

  EnvName:
    Type: String
    AllowedValues: [stg, prod]
    Default: stg
    Description: Environment name for dashboard naming.

  DashboardRegion:
    Type: String
    Default: ap-northeast-1
    Description: Region used inside dashboard widgets.

  # 主要関数（まずは3つで開始。必要なら増やす）
  FunctionName1:
    Type: String
    Description: Primary Lambda function name (top 1).
  FunctionName2:
    Type: String
    Description: Primary Lambda function name (top 2).
  FunctionName3:
    Type: String
    Description: Primary Lambda function name (top 3).

Resources:
  LambdaOpsDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: !Sub '${ServiceName}-${EnvName}-lambda-ops'
      DashboardBody: !Sub |
        {
          "widgets": [
            {
              "type": "metric",
              "x": 0,
              "y": 0,
              "width": 24,
              "height": 6,
              "properties": {
                "view": "timeSeries",
                "stacked": false,
                "region": "${DashboardRegion}",
                "title": "Duration p95 (ms) - Top Functions",
                "period": 60,
                "stat": "p95",
                "metrics": [
                  [ "AWS/Lambda", "Duration", "FunctionName", "${FunctionName1}" ],
                  [ ".", "Duration", "FunctionName", "${FunctionName2}" ],
                  [ ".", "Duration", "FunctionName", "${FunctionName3}" ]
                ]
              }
            },
            {
              "type": "metric",
              "x": 0,
              "y": 6,
              "width": 12,
              "height": 6,
              "properties": {
                "view": "timeSeries",
                "stacked": false,
                "region": "${DashboardRegion}",
                "title": "Errors - Top Functions",
                "period": 60,
                "stat": "Sum",
                "metrics": [
                  [ "AWS/Lambda", "Errors", "FunctionName", "${FunctionName1}" ],
                  [ ".", "Errors", "FunctionName", "${FunctionName2}" ],
                  [ ".", "Errors", "FunctionName", "${FunctionName3}" ]
                ]
              }
            },
            {
              "type": "metric",
              "x": 12,
              "y": 6,
              "width": 12,
              "height": 6,
              "properties": {
                "view": "timeSeries",
                "stacked": false,
                "region": "${DashboardRegion}",
                "title": "Throttles - Top Functions",
                "period": 60,
                "stat": "Sum",
                "metrics": [
                  [ "AWS/Lambda", "Throttles", "FunctionName", "${FunctionName1}" ],
                  [ ".", "Throttles", "FunctionName", "${FunctionName2}" ],
                  [ ".", "Throttles", "FunctionName", "${FunctionName3}" ]
                ]
              }
            },
            {
              "type": "metric",
              "x": 0,
              "y": 12,
              "width": 12,
              "height": 6,
              "properties": {
                "view": "timeSeries",
                "stacked": false,
                "region": "${DashboardRegion}",
                "title": "MaxMemoryUsed (MB) - Top Functions",
                "period": 60,
                "stat": "Maximum",
                "metrics": [
                  [ "AWS/Lambda", "MaxMemoryUsed", "FunctionName", "${FunctionName1}" ],
                  [ ".", "MaxMemoryUsed", "FunctionName", "${FunctionName2}" ],
                  [ ".", "MaxMemoryUsed", "FunctionName", "${FunctionName3}" ]
                ]
              }
            },
            {
              "type": "metric",
              "x": 12,
              "y": 12,
              "width": 12,
              "height": 6,
              "properties": {
                "view": "timeSeries",
                "stacked": false,
                "region": "${DashboardRegion}",
                "title": "InitDuration (ms) - Cold Start Trend",
                "period": 60,
                "stat": "Average",
                "metrics": [
                  [ "AWS/Lambda", "InitDuration", "FunctionName", "${FunctionName1}" ],
                  [ ".", "InitDuration", "FunctionName", "${FunctionName2}" ],
                  [ ".", "InitDuration", "FunctionName", "${FunctionName3}" ]
                ]
              }
            }
          ]
        }

Outputs:
  DashboardName:
    Description: CloudWatch Dashboard name
    Value: !Sub '${ServiceName}-${EnvName}-lambda-ops'
```

---

## F-2：CI/CD からのデプロイ例（環境別切替）

Day11〜Day13 の流れに合わせ、**stg/prod で Parameter を切り替える**最小例です。

### STAGING

```bash
sam deploy \
  --stack-name receipt-api-stg-ops \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    ServiceName=receipt-api \
    EnvName=stg \
    DashboardRegion=ap-northeast-1 \
    FunctionName1=receipt-api-ingest-stg \
    FunctionName2=receipt-api-transform-stg \
    FunctionName3=receipt-api-export-stg
```

### PROD

```bash
sam deploy \
  --stack-name receipt-api-prod-ops \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    ServiceName=receipt-api \
    EnvName=prod \
    DashboardRegion=ap-northeast-1 \
    FunctionName1=receipt-api-ingest-prod \
    FunctionName2=receipt-api-transform-prod \
    FunctionName3=receipt-api-export-prod
```

> 実務では「主要関数名」は固定のことが多いので、最初は手で入れてOKです。
> 余力が出たら「SAM Outputs から関数名を拾って注入」「タグで主要関数を決める」などに発展できます。

---

## F-3：運用設計の観点（なぜ IaC 管理が効くのか）

* 手作業の Dashboard は、いつの間にか **本番と検証で差分**が出る
* IaC 化すると、環境差分は Parameter に閉じ、**レビュー可能**になる
* 「監視もコード」として扱えるため、Day11 の OIDC / Day12 の環境切替 / Day13 のリリース戦略と同じ思想で運用できる

---

## F-4：最小の拡張案（必要になったら）

* 主要関数を 3 → 5 / 10 に増やす（Parameters を増設）
* サービス単位の “合算” を見たい場合は、関数別ではなく **Metric Math** で集約
* ダッシュボードに「Errors のアラーム状態」「SQS DLQ メッセージ数」などを追加し、SLO 運用へ寄せる
