# Day 22：Well-Architected（Serverless Lens）で棚卸しする

---

この記事は [AWS Lambda 実践入門 Advent Calendar 2025] の Day22 です。
Day1〜21 で作った構成を、今日は **Well-Architected（Serverless Lens）で棚卸し** して「直すべき点」を洗い出し、改善チケットに落とすところまでを扱います。
 
---

## なぜ Well-Architected をやるのか（サーバレスは“速い”ほど壊れやすい）

サーバレスは「作る」スピードが速い反面、次が後追いになりがちです。

* 監視・アラートが弱い（気づけない）
* IAM が雑（権限過多）
* リトライや DLQ が曖昧（壊れ方が地獄）
* コストが見えない（いつの間にか増える）

そこで有効なのが AWS Well-Architected Framework。
**6つの柱（Operational Excellence / Security / Reliability / Performance Efficiency / Cost Optimization / Sustainability）**の観点で「質問に答える」だけで、抜け漏れを体系的に発見できます。 ([AWS ドキュメント][3])

さらに Serverless Applications Lens は、サーバレスの典型構成に即した観点が整理されています。 ([AWS ドキュメント][1])

---

## 使うもの：AWS Well-Architected Tool + Serverless Lens

AWS Well-Architected Tool は、Workload を定義し、Lens（質問セット）を適用してレビューする仕組みです。
Lens は **改善計画（Improvement plan）**まで繋がるので、「棚卸しで終わらない」のが実務的に強いです。 ([AWS ドキュメント][2])

---

## 進め方（最短で“棚卸し→改善チケット化”する手順）

### Step 0. Workload の境界を決める（ここが一番大事）

最初から全部はやりません。おすすめは次のどれかです。

* PROD 環境のみ（STAGING は後）
* 1つの業務機能（例：帳票取り込み）だけ
* 主要イベント経路だけ（API→Lambda→DB、S3→Lambda→SQS 等）

### Step 1. 60〜90分で一次回答（完璧主義を捨てる）

質問に対して「Yes/No/Not applicable」を付け、**根拠（証跡リンク）**をメモします。
証跡は例として次のようなリンクを貼ると、監査・引継ぎにも強くなります。

* CloudWatch Dashboard / Alarm
* CloudWatch Logs Insights クエリ
* IaC（SAM/CDK）該当ファイルの行
* Runbook（障害対応手順）
* CloudTrail/Config の確認結果（Day18 へ接続）

### Step 2. HRI を Backlog 化する（ここがゴール）

棚卸しで見つかった改善点は「気づいた」で終わりがちです。
**HRI（High risk issues）や重要改善を、必ずチケット化**します。

チケットテンプレ例：

* タイトル：`[WA][Security] Lambda 実行ロールの権限を最小化`
* 背景：Well-Architected Serverless Lens の回答で “No”
* 影響：権限過多により情報漏洩リスク
* 対応：IAM ポリシーを S3 prefix 単位に絞る、条件を付ける
* 受入条件：最小権限のポリシー + 回帰テスト + 監査ログ確認
* 優先度 / 期限 / Owner

---

## “まずはこれだけ” Serverless Lens 棚卸しチェックリスト（15項目）

### 運用（Operational Excellence）

1. 主要指標（Errors/Duration/Throttles/IteratorAge 等）をダッシュボード化しているか
2. アラートが「行動」に繋がる形（通知＋一次対応手順）になっているか
3. デプロイと変更履歴を追えるか（Deploy Marker / CloudTrail） ※Day14/18 へ接続

### セキュリティ（Security）

4. 実行ロールは最小権限か（Resource/Condition まで絞れているか）
5. シークレットは Secrets Manager / Parameter Store + KMS で管理しているか ※Day16 へ接続
6. 依存ライブラリの脆弱性を継続検知できているか（Inspector 等） ※Day17 へ接続

### 信頼性（Reliability）

7. 非同期・キュー処理に DLQ を用意し、再処理手順があるか
8. リトライ前提で **冪等性**（重複実行に耐える）設計か
9. 同時実行で下流（DB/API）が落ちないよう制御しているか（Reserved/Provisioned の使い分け）※Dayロードマップ Phase6 と接続

### 性能（Performance Efficiency）

10. メモリ設定を根拠をもって調整しているか（p95/p99 を見る） ※Day19
11. 重い処理（PDF/画像など）は /tmp・並列・分割（fan-out）で設計しているか ※Day20
12. コールドスタート対策（初期化の置き場所・接続再利用）ができているか ※Day2/ロードマップ Phase1

### コスト（Cost Optimization）

13. 失敗リトライや無限ループで“課金事故”が起きない設計か（タイムアウト、ガード、上限）
14. 不要なログ過多、過剰メモリ、過剰プロビジョニングになっていないか
15. コストの可視化（タグ、コスト配賦、機能別の概算）ができているか

※ Well-Architected の柱そのものは AWS 公式が定義しています。 ([AWS ドキュメント][3])

---

## 例：よく出る “改善ポイント” と、Day1〜21 との繋ぎ方

* **IAM が広い** → Day4/Day16 の最小権限へ戻る
* **脆弱性対応が属人化** → Day17（Inspector）＋Day18（監査）へ繋げる
* **性能は出たが落ち方が雑** → Day19（可視化）＋DLQ/冪等性で Reliability を固める
* **VPC 化したが運用が重い** → Day21（VPCEndpoint/NAT-less）を前提に、可観測性と運用設計を見直す

---

## まとめ：Well-Architected は “レビュー” ではなく “改善の起点”

Day22 のゴールは、レビューで満点を取ることではありません。
**「直すべきことがBacklogに積まれ、Ownerと期限が付いた」**状態を作ることが最大の成果です。

---

## 付録：棚卸しを回すおすすめ運用

* 頻度：四半期に1回 + 大きな変更前後
* 参加者：開発1名・運用1名・セキュリティ1名（兼任でOK）
* 時間：一次回答 90分、チケット化 30分、優先度決め 30分

---

[1]: https://docs.aws.amazon.com/ja_jp/wellarchitected/latest/serverless-applications-lens/welcome.html?utm_source=chatgpt.com "Serverless Applications Lens - AWS Well-Architected ..."
[2]: https://docs.aws.amazon.com/wellarchitected/latest/userguide/lenses.html?utm_source=chatgpt.com "Using lenses in AWS WA Tool - AWS Well-Architected Tool"
[3]: https://docs.aws.amazon.com/wellarchitected/latest/framework/the-pillars-of-the-framework.html?utm_source=chatgpt.com "The pillars of the framework - AWS Well-Architected ..."


## 付録A：Serverless Lens の回答例（Yes / No の具体例）

> 前提：あなたの連載（Day1〜21）で扱っている典型構成
>
> * SAM で IaC、CircleCI + OIDC でデプロイ（Day11〜15）
> * Layer 運用（Day8〜10）、Inspector/監査（Day17〜18）
> * 性能最適化・PDF/画像（Day19〜20）
> * VPC + VPCEndpoint（Day21）
>   を想定した「答え方の型」です。

### 回答の型（これだけでレビューが速くなる）

* **回答**：Yes / No / Not applicable
* **根拠（Evidence）**：URL もしくはファイルパス + 行番号（例：`template.yaml:L120-160`）
* **補足**：制約・例外（例：一部の関数は例外で許可、など）
* **改善案**：No の場合は「次アクション」を1行で書く

---

### Operational Excellence（運用上の優秀性）

**Q1. 主要メトリクスがダッシュボード化され、運用が“見る”形になっているか？**

* 回答例：**Yes**
* Evidence：CloudWatch Dashboard（`Errors/Duration/Throttles/IteratorAge`）URL
* 補足：p95/p99 を見る（Day19 の方針）
* 改善案：—

**Q2. 変更が起きた時に「いつ・何が・なぜ」が追跡できるか？**

* 回答例：**Yes（ただし一部No）**
* Evidence：Deploy Marker（Day14）＋ CloudTrail（Day18）へのリンク
* 補足：手動変更が混ざると追跡不能になる
* 改善案：No の箇所は「手動変更禁止」「CI/CD ロールのみに統一」

---

### Security（セキュリティ）

**Q3. Lambda 実行ロールは最小権限になっているか？（Resource/Condition まで絞れているか）**

* 回答例：**No**
* Evidence：`template.yaml` の `Policies:` が `S3ReadPolicy BucketName=...` のみ（prefix 制限なし）
* 補足：初期は仕方ないが、運用に入ったら必ず削る
* 改善案：S3 は prefix 条件、KMS は key 条件、STS は必要最小のみ

**Q4. 依存ライブラリの脆弱性が継続検知できているか？**

* 回答例：**Yes**
* Evidence：Inspector 結果、通知（SNS/EventBridge）設定（Day17）
* 補足：Layer 更新の責任境界（誰がいつ更新したか）を明確に
* 改善案：—

---

### Reliability（信頼性）

**Q5. 非同期系（SQS/EventBridge/S3）で DLQ と再処理手順があるか？**

* 回答例：**No（DLQ はあるが手順がない）**
* Evidence：SAM の DLQ 設定はあるが Runbook が無い
* 補足：DLQ があっても“復旧できない”と意味がない
* 改善案：DLQ 再投入手順（CLI/手動/自動）を Runbook 化

**Q6. 冪等性（重複実行に耐える）が担保されているか？**

* 回答例：**No**
* Evidence：S3 イベントで同一オブジェクトが再通知された場合のガードが無い
* 補足：Lambda は「少なくとも1回」前提で設計する
* 改善案：DynamoDB 条件付き書き込み、Etag/VersionId をキーに重複排除

---

### Performance Efficiency（性能効率）

**Q7. コールドスタートの影響を理解し、初期化の置き場所が最適化されているか？**

* 回答例：**Yes（ただし重い初期化が一部残る）**
* Evidence：グローバル領域に client を置く（Day2）＋ Init Duration を可視化
* 補足：VPC 付与時は特に初期化が効く
* 改善案：重い import の遅延ロード／接続再利用

**Q8. PDF/画像など重い処理で /tmp・並列・分割の設計根拠があるか？**

* 回答例：**Yes**
* Evidence：Day20 の設計（/tmp 10GB、fan-out、バッチサイズ根拠）
* 補足：上流イベント量増加に備えて同時実行制御もセット
* 改善案：Reserved/Provisioned の設計（Dayロードマップ Phase6）

---

### Cost Optimization（コスト最適化）

**Q9. “課金事故” を防ぐガード（無限ループ、過剰リトライ、過剰ログ）があるか？**

* 回答例：**No**
* Evidence：タイムアウトはあるが、再試行回数・上限・異常検知が未整備
* 補足：DLQ/アラーム/上限制御は「保険」ではなく必須
* 改善案：最大試行回数・バッチサイズ・最大同時実行の上限を定義

**Q10. コストの“内訳”が機能単位で追えるか？**

* 回答例：**No**
* Evidence：タグ／コスト配賦タグが未設計
* 補足：最適化は「可視化」が先
* 改善案：Stack/Lambda に Cost Allocation Tag を付与し、機能別に集計

---

### Sustainability（持続可能性）

**Q11. 使っていないログや過剰なデータ保持がないか？**

* 回答例：**No**
* Evidence：LogGroup の retention が無期限
* 改善案：保持期間を定義（例：30/90/365日）、監査要件に合わせる

**Q12. リソースサイズを根拠なく最大化していないか？**

* 回答例：**No**
* Evidence：全関数が 1024MB 固定など
* 改善案：Day19 の計測に基づきメモリを最適化（p95/p99）

---

## 付録B：改善チケット 10本のサンプル（そのまま Backlog 化）

> 形式は「そのまま Redmine / GitHub Issues に貼れる」粒度にしています。
> まずは **Priority と Acceptance Criteria** だけ埋める運用でも回ります。

### 1. [WA][Security] Lambda 実行ロールの S3 権限を prefix 条件で最小化

* Priority：P0
* Background：Serverless Lens Q3 が No（権限過多）
* Scope：S3 Read/Write を `bucket/*` → `bucket/prefix/*` へ
* Tasks：ポリシー修正、回帰テスト、CloudTrail で権限不足が出ないこと確認
* Acceptance Criteria：

  * 必要な prefix のみ許可
  * 既存ユースケースが動作
  * 監査用に Evidence をリンク

### 2. [WA][Reliability] SQS トリガーに DLQ + 再処理 Runbook を整備

* Priority：P0
* Background：DLQ はあるが運用手順が無い
* Tasks：Runbook（再投入方法/調査手順/責任者）作成、手動再投入コマンド例追加
* Acceptance Criteria：障害時に 15 分以内に再処理が開始できる

### 3. [WA][Reliability] 冪等性キー設計（重複イベント対策）を導入

* Priority：P0
* Background：S3/SQS は重複があり得る
* Tasks：DynamoDB に冪等性テーブル、条件付き書き込み、TTL で自動掃除
* Acceptance Criteria：同一入力が複数回届いても副作用が 1 回に収束

### 4. [WA][Operational] Deploy Marker を全関数・全スタックに統一適用

* Priority：P1
* Background：変更追跡の抜け（手動変更混入）が残る
* Tasks：CircleCI の共通ジョブ化、marker schema 固定、通知先統一
* Acceptance Criteria：障害時に “直近変更” を 1 分で辿れる

### 5. [WA][Security] Secrets をコード/環境変数直書きから Secrets Manager 参照へ移行

* Priority：P0
* Background：漏洩リスク・ローテ不可
* Tasks：Secrets Manager + KMS、参照権限最小化、ローテ手順
* Acceptance Criteria：シークレットがリポジトリ/平文 env に存在しない

### 6. [WA][Security] Layer 更新の責任境界を明確化（Owner/互換性/更新手順）

* Priority：P1
* Background：依存の破壊的変更が起きやすい
* Tasks：Layer README の必須項目、互換性ポリシー、更新フロー（Day9〜10）
* Acceptance Criteria：Layer 更新で本番が壊れた場合の原因追跡が可能

### 7. [WA][Performance] Init Duration/Duration の p95/p99 を定期レポート化

* Priority：P1
* Background：平均だけだと遅延が見えない
* Tasks：Logs Insights 集計クエリ、週次レポート、しきい値定義
* Acceptance Criteria：遅延劣化をリリース前に検知できる

### 8. [WA][Cost] 同時実行の上限設計（Reserved / Provisioned）を機能単位で定義

* Priority：P0
* Background：下流（DB/API）を落とす・課金事故
* Tasks：上限値決定、緊急時の絞り方（Runbook）、必要箇所だけ Provisioned
* Acceptance Criteria：負荷試験で下流が落ちない、Throttles を許容範囲に制御

### 9. [WA][Cost] ログ保持期間（Retention）とログ量削減（サンプリング/構造化）を整備

* Priority：P2
* Background：LogGroup 無期限、ログ過多
* Tasks：Retention ポリシー、不要ログ削減、必要ログは JSON 構造化に統一
* Acceptance Criteria：保持要件を満たしつつ、月次ログコストが可視化される

### 10. [WA][Network] VPC 構成の NAT-less 化を完成（必要な Interface Endpoint を追加）

* Priority：P1
* Background：VPC Lambda で外向き通信がボトルネック/コスト要因
* Tasks：必要な VPCE（例：Logs, STS, ECR, KMS 等）追加、Route53/DNS 方針整理
* Acceptance Criteria：NAT 経由が不要になり、コストと障害点が減る（Day21 と接続）

---

## 付録C：Evidence（証跡）テンプレ

* CloudWatch Dashboard：`<URL>`
* Alarm：`<URL>`
* Logs Insights：`<クエリ>` / `Saved query URL`
* SAM：`template.yaml:Lxx-Lyy`
* CI/CD：`.circleci/config.yml:Lxx-Lyy`
* Runbook：`docs/runbook/<name>.md`
* CloudTrail/Config：`<該当イベント>` / `Config rule`

---

以下を **Day22 の締め（復習導線）**として、そのまま貼ってください。付録Bの「改善チケット10本」を **連載Day番号**に紐付け、読者が“戻る場所”を明確にしたリンク集（導線）です。

---

## 付録D：改善チケット ⇄ 連載Day番号 対応表（復習導線）

> Well-Architected の棚卸しは「課題を見つけた」だけでは終わりません。
> 本当に価値が出るのは、課題を **“どの記事に戻って直せばいいか”**が明確になり、手が動く状態になった時です。
> ここでは、付録Bのチケット10本を、連載の該当 Day に紐付けました。

### 1. [WA][Security] Lambda 実行ロールの S3 権限を prefix 条件で最小化

* 関連Day：**Day4（IAM 基礎） / Day16（セキュリティ実務：IAM/KMS/Secrets）**
* 戻る観点：最小権限、Resource/Condition の設計、KMS 条件の考え方

### 2. [WA][Reliability] SQS トリガーに DLQ + 再処理 Runbook を整備

* 関連Day：**Day3（イベント駆動の選定） / Day15（運用：構造化ログ） / Day19（監視指標）**
* 戻る観点：失敗時の設計（DLQ/再試行）、運用手順を“Runbook化”する型

### 3. [WA][Reliability] 冪等性キー設計（重複イベント対策）を導入

* 関連Day：**Day3（イベント特性） / Day12〜13（デプロイ戦略）**
* 戻る観点：「少なくとも1回」前提での設計、修正を安全に出す（Version/Alias）

### 4. [WA][Operational] Deploy Marker を全関数・全スタックに統一適用

* 関連Day：**Day14（Deploy Marker） / Day11（CircleCI×OIDC×SAM）**
* 戻る観点：変更履歴の一元化、CI/CDでの強制、環境（STAGING/PROD）横断の整合

### 5. [WA][Security] Secrets を平文 env から Secrets Manager 参照へ移行

* 関連Day：**Day16（Secrets/KMS） / Day11（CI/CD と権限）**
* 戻る観点：秘密情報の扱い、参照権限の最小化、ローテーション運用

### 6. [WA][Security] Layer 更新の責任境界（Owner/互換性/更新手順）を明確化

* 関連Day：**Day8（Layer 基礎） / Day9（Layer 設計整理） / Day10（Layer を CI/CD で自動取得） / Day17（Inspector）**
* 戻る観点：Layer を“資産”として運用する、脆弱性対応と更新フローの統合

### 7. [WA][Performance] Init/Duration の p95/p99 を定期レポート化

* 関連Day：**Day19（性能最適化：メモリ×CPU×並列） / Day15（構造化ログ） / Day2（ライフサイクル）**
* 戻る観点：平均ではなく分位で見る、REPORT 行/構造化ログの活用、Init Duration の扱い

### 8. [WA][Cost] 同時実行の上限設計（Reserved / Provisioned）を機能単位で定義

* 関連Day：**Day19（並列数の扱い） + ロードマップ Phase6（Concurrency 制御）**
* 戻る観点：課金事故防止、下流保護、Reserved と Provisioned の使い分け

### 9. [WA][Cost] ログ保持（Retention）とログ量削減（サンプリング/構造化）を整備

* 関連Day：**Day15（構造化ログ） / Day19（監視） / Day18（監査：証跡）**
* 戻る観点：ログは“情報資産”、保持要件とコストのバランス、証跡としての設計

### 10. [WA][Network] VPC 構成の NAT-less 化を完成（VPCE 追加）

* 関連Day：**Day21（VPC + VPCEndpoint） / Day16（KMS/Secrets とネットワーク）**
* 戻る観点：VPC Lambda の通信経路、必要 Endpoint の棚卸し、DNS/Resolver との整合
