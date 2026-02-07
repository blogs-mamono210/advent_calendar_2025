# Day 11：CircleCI × OIDC × SAM で壊れないデプロイを作る

---

## はじめに
AWS Lambda における CI/CD は、これまで AccessKey/SecretKey を環境変数に入れて実行する方式が主流でした。

しかしこの方式は、運用が続くほど次のリスクが顕在化します。

- 誤ってキーが漏洩すると即アウト（影響範囲が読めない）
- ローテーションが「いつかやる」で止まりがち
- 権限が広すぎる設定が温存されやすい
- 監査で「長期クレデンシャルの管理」を問われやすい

そこで AWS と CI/CD の最新トレンドは  **「OIDC（OpenID Connect）で安全に AssumeRole する方式」** です。

この記事では、CircleCI × OIDC × SAM を組み合わせて **“壊れないデプロイ（安全・再現性・監査性が揃う）”** を実現する方法を紹介します。

また Day10 の「最新 Layer ARN を自動取得」と接続し、 **“最新 Layer を参照しつつ、鍵レスでデプロイ”** までを一気通貫で作ります。

---

## OIDC とは？（最短 30 秒で理解）
CI/CD が AWS に対して「短時間だけ有効な一時クレデンシャル」を発行してもらう仕組みです。

ポイントは次の 3 つです。

- CircleCI はジョブ実行時に OIDC トークン（ID Token）を発行する  
  （環境変数 `$CIRCLE_OIDC_TOKEN` / `$CIRCLE_OIDC_TOKEN_V2` で参照できる）
- AWS 側は「この発行元（CircleCI）を信頼する」設定を行う
- STS が `AssumeRoleWithWebIdentity` で一時クレデンシャルを払い出す

```mermaid
sequenceDiagram
  participant CI as CircleCI
  participant IdP as OIDC Provider
  participant AWS as AWS STS

  CI->>IdP: OIDC Token を取得
  IdP-->>CI: Signed Token
  CI->>AWS: AssumeRoleWithWebIdentity
  AWS-->>CI: Temporary Credentials
  CI->>AWS: sam deploy
````

---

## 何が “壊れない” のか（定義を先に置く）

この記事で言う「壊れないデプロイ」は、次の 4 要素を満たす状態です。

1. **秘密情報（長期キー）を持たない**
   → 漏洩事故のクラスを消す
2. **最小権限で実行できる**
   → “できてしまう事故” を防ぐ
3. **誰が・いつ・どの経路でデプロイしたか追跡できる**
   → 監査・調査が壊れない
4. **デプロイ手順が再現可能（環境差分が出にくい）**
   → 人間の手作業・属人性で壊れない

OIDC は 1 と 3 を強力に満たします。
2 と 4 は IAM 設計と CI 設計で補強して完成します。

---

## 手順1：AWS 側（OIDC Provider と IAM Role）を作る

ここが記事に無いと、読者が実装で止まります。

### 1) IAM Identity Provider（OIDC Provider）を作成

CircleCI の AWS OIDC では、プロバイダ URL は次の形式です。

* `https://oidc.circleci.com/org/<organization_id>`

Audience（`aud`）はデフォルトで **Organization ID** です。
（カスタム `aud` を使う場合は CircleCI 側でカスタムクレームを発行します）

※ Organization ID は CircleCI の Organization Settings から確認できます。

### 2) Assume 用 IAM Role を作る（Trust Policy が肝）

最初は「Organization だけ許可」でも動きますが、実務では **Project / Branch まで絞る**のが推奨です。

#### Project で絞る例（まずはここまでで十分）

`sub` クレームを StringLike で絞ります。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/oidc.circleci.com/org/<organization_id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "oidc.circleci.com/org/<organization_id>:sub": "org/<organization_id>/project/<project_id>/user/*"
        }
      }
    }
  ]
}
```

#### Branch（main）で絞る例（より堅牢）

`$CIRCLE_OIDC_TOKEN_V2` の `sub` 形式を前提に、main ブランチのみ許可します。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/oidc.circleci.com/org/<organization_id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "oidc.circleci.com/org/<organization_id>:sub": "org/<organization_id>/project/*/user/*/vcs-origin/github.com/<your-org>/*/vcs-ref/refs/heads/main"
        }
      }
    }
  ]
}
```

---

## 手順2：CircleCI 側（aws-cli orb で OIDC セットアップ）

CircleCI では `circleci/aws-cli` orb の `aws-cli/setup` を使うのが最短です。
このコマンドが OIDC トークンを使った認証セットアップを自動で行います。

### CircleCI の設定例（実用形式）

```yaml
- aws-cli/setup:
    role_arn: ${AWS_ROLE_ARN}
    region: ${AWS_REGION}
    profile_name: OIDC-PROFILE
    role_session_name: deploy-session
    session_duration: "900"
```

補足：

* `session_duration` は短くするほど安全です（例：15分）
* CircleCI の OIDC トークンは `exp` が発行から 1 時間のため、長時間ジョブは避ける設計が無難です

---

## 手順3：SAM デプロイに統合する（validate → build → deploy）

「壊れない」に寄せるなら、最低でも validate を先に入れます。

```bash
sam validate
sam build
sam deploy \
  --region "${AWS_REGION}" \
  --no-fail-on-empty-changeset \
  --no-confirm-changeset
```

---

## Day10 と合体：最新 Layer ARN を安全に注入する

Day10 で「最新 Layer ARN を取得してファイルに保存」している前提で、
それを workspace 経由で deploy に渡します。

### 例：get-layer → deploy の 2 ジョブ構成（workspace で受け渡し）

```yaml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@5

jobs:
  get-layer-arn:
    docker:
      - image: cimg/aws:2025.01
    steps:
      - checkout
      - run:
          name: "Get latest Layer ARN (Day10)"
          command: |
            ./scripts/get_layer_arn.sh
            cat scripts/layer_arn.txt
      - persist_to_workspace:
          root: .
          paths:
            - scripts/layer_arn.txt

  deploy:
    docker:
      - image: cimg/aws:2025.01
    environment:
      AWS_REGION: ap-northeast-1
    steps:
      - checkout
      - attach_workspace:
          at: .
      - aws-cli/setup:
          role_arn: ${AWS_ROLE_ARN}
          region: ${AWS_REGION}
          profile_name: OIDC-PROFILE
          role_session_name: deploy-session
          session_duration: "900"
      - run:
          name: "SAM Deploy"
          command: |
            LAYER_ARN="$(cat scripts/layer_arn.txt)"
            echo "Using Layer ARN: ${LAYER_ARN}"

            sam build
            sam deploy \
              --region "${AWS_REGION}" \
              --no-fail-on-empty-changeset \
              --no-confirm-changeset \
              --parameter-overrides LayerArn="${LAYER_ARN}"

workflows:
  deploy-workflow:
    jobs:
      - get-layer-arn
      - deploy:
          context: aws
          requires:
            - get-layer-arn
```

ポイント：

* Day10 の成果（Layer ARN 自動取得）を **workspace で確実に引き継ぐ**
* Day11 の成果（OIDC）で **AWS 側に鍵を置かない**
* “安全に最新へ追従” が CI/CD の標準形になります

---

## よくあるハマりどころ（短く潰す）

* **Trust Policy がゆるい**：最初は動いても、後で事故る。Project / Branch で絞る
* **Role の権限が強すぎる**：最小権限に寄せる（少なくとも “デプロイに必要な範囲” だけ）
* **トークンの有効期限**：長いジョブは分割（build と deploy を分けると安定する）
* **Parameter 名の不一致**：`LayerArn` は template.yaml の Parameters 名に合わせる

---

## まとめ

* **OIDC + SAM は現代の Lambda デプロイ標準**
* AccessKey/SecretKey を CI/CD から排除できる
* Trust Policy を Project / Branch で絞ると “壊れない” に近づく
* Day10（Layer ARN 自動取得）と組み合わせると「安全に最新へ追従」できる


[1]: https://circleci.com/docs/guides/permissions-authentication/openid-connect-tokens/ "Using OpenID Connect tokens in jobs :: CircleCI Documentation"
