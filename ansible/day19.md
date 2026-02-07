# 「CI のログが読めない」を解決する：Ansible 実行ログを 人間用(YAML) と AI解析用(JSONL) に分離する最短実装

---

## はじめに

Ansible の Role/Playbook を CI で回すと、コンソールログは“人間が読む”には十分でも、**機械的に判定・集計・要約**するには扱いづらい場面が増えます。

ここで生成AIを使うと、**CI 失敗時のログを短時間で要約し、原因候補と次に確認すべきポイントを整理**できます。
ただし、AI に「正しい状態か」を推測させるのは危険です。**合否判定は Ansible に任せ、AI は説明と整理に使う**のがコツです。

この記事では、次の2本立てを最短で作ります。

* CI コンソールには **人間が読みやすいログ**（default + YAML 風）
* 同時に **JSONL（1行=1 JSON）** の実行ログを生成し、`jq` や生成AIで解析できる形にする

### 前提（重要）

* **ansible-core 2.13 以降**（`ANSIBLE_CALLBACK_RESULT_FORMAT` を使うため）

---

## 全体像

* **コンソール（人間用）**：`ansible.builtin.default` を使い、`result_format=yaml` で見やすく
* **JSONL（機械/AI用）**：自作 callback plugin（追加 callback）でファイル出力
* **監査（判定の根拠）**：Playbook に `VERIFY | ...` を入れて、期待値との差分をログに残す

---

## 1. まずは “見える” と “残る” を両立（tee）

最小構成はこれです。

```bash
ansible-playbook -i "localhost," -c local site.yml 2>&1 | tee -a ansible.console.log
```

---

## 2. stdout は default のまま、表示だけ YAML 風にする

`ANSIBLE_STDOUT_CALLBACK=yaml` は環境によっては別 plugin を参照して事故ることがあるため、stdout は **default** に固定し、**result の見え方だけ**整えます。

```bash
ANSIBLE_STDOUT_CALLBACK=default \
ANSIBLE_CALLBACK_RESULT_FORMAT=yaml \
ANSIBLE_CALLBACK_FORMAT_PRETTY=true \
ansible-playbook -i "localhost," -c local site.yml
```

---

## 3. 自作 callback plugin で JSONL を出す

### 3.1 配置

Playbook と同じリポジトリに置くのが簡単です。

```text
.
├── site.yml
└── callback_plugins/
    └── jsonfile.py
```

### 3.2 callback_plugins/jsonfile.py（最短ルート実装）

JSONL（1行=1 JSON）で追記します。

```python
# callback_plugins/jsonfile.py
from __future__ import annotations

import json
import os
import time
from ansible.plugins.callback import CallbackBase

class CallbackModule(CallbackBase):
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = "aggregate"
    CALLBACK_NAME = "jsonfile"
    CALLBACK_NEEDS_WHITELIST = True

    def __init__(self):
        super().__init__()
        self.path = os.getenv("ANSIBLE_JSON_LOG_PATH", "ansible.events.jsonl")

    def _write(self, payload: dict):
        payload["ts"] = time.time()
        # 最短実装：イベントごとに open/write/close（小〜中規模なら十分）
        # 大規模ではI/O増の可能性があるため、必要ならバッファリング等を検討
        with open(self.path, "a", encoding="utf-8") as f:
            f.write(json.dumps(payload, ensure_ascii=False) + "\n")

    def v2_runner_on_ok(self, result, **kwargs):
        self._write({
            "event": "ok",
            "task": result.task_name,
            "host": result._host.get_name(),
            "result": result._result,
        })

    def v2_runner_on_failed(self, result, ignore_errors=False):
        self._write({
            "event": "failed",
            "task": result.task_name,
            "host": result._host.get_name(),
            "ignore_errors": ignore_errors,
            "result": result._result,
        })
```

---

## 4. 実行（人間用ログ + JSONL を同時生成）

```bash
ANSIBLE_STDOUT_CALLBACK=default \
ANSIBLE_CALLBACK_RESULT_FORMAT=yaml \
ANSIBLE_CALLBACK_FORMAT_PRETTY=true \
ANSIBLE_CALLBACKS_ENABLED=jsonfile \
ANSIBLE_CALLBACK_PLUGINS=./callback_plugins \
ANSIBLE_JSON_LOG_PATH=./ansible.events.jsonl \
ansible-playbook -i "localhost," -c local site.yml 2>&1 | tee -a ansible.console.log
```

---

## 5. “監査できるログ” にする：VERIFY タスク設計（AIのための下準備）

生成AIは便利ですが、**ログを渡しただけで「正しい状態か」を正確に判定するのは苦手**です。
そこで方針をこう分けます。

* **合否判定は Ansible が行う**（`assert` で期待値と比較して OK/NG を確定）
* **説明や整理は AI に任せる**（何がNGかを要約、原因候補を列挙、次に見るべき点を提案）

### 運用ルール（シンプル）

* 監査は `changed_when: false`（監査で構成変更しない）
* 判定は `assert`（NGなら failed と理由が残る）
* タスク名は `VERIFY | ...` で統一（後で jq で抽出しやすい）

### sshd の例（PasswordAuthentication）

```yaml
- name: VERIFY | sshd PasswordAuthentication is no
  ansible.builtin.command: >-
    sshd -T | awk 'tolower($1)=="passwordauthentication" {print $2}'
  register: sshd_passwordauth
  changed_when: false
  tags: [verify]

- name: VERIFY | assert sshd PasswordAuthentication is no
  ansible.builtin.assert:
    that:
      - (sshd_passwordauth.stdout | trim) == "no"
    fail_msg: "PasswordAuthentication is NOT 'no' (actual={{ sshd_passwordauth.stdout | trim }})"
    success_msg: "PasswordAuthentication is 'no'"
  tags: [verify]
```

verify だけ実行したい場合：

```bash
ansible-playbook -i "localhost," -c local site.yml --tags verify
```

---

## 6. jq で “AI に渡す材料” を作る（VERIFY だけ抽出する）

JSONL 全体をそのまま AI に渡すと、情報が多すぎて逆に要点が伝わりません。
まずは **VERIFY だけ取り出して「監査結果の抜粋」を作る**のがコツです。

ここで、タスク名を `VERIFY |` に統一した運用ルールが効いてきます。

### (1) VERIFY だけ抽出（まずはこれで十分）

```bash
jq -c 'select(.task | startswith("VERIFY |"))' ansible.events.jsonl > verify_only.jsonl
```

### (2) failed だけ抽出（AI に“困ってるところだけ”見せたい場合）

```bash
jq -c '
  select(.task | startswith("VERIFY |")) |
  select((.result.failed // false)==true)
' ansible.events.jsonl > verify_failed.jsonl
```

### (3) CIで落とす（exit code 制御）

```bash
jq -e '
  any(select(.task | startswith("VERIFY |")) | select((.result.failed // false)==true))
' ansible.events.jsonl
```

---

## 7. 生成AIへの渡し方（初心者向けテンプレ：コピペで使える）

### 7.1 生成AIに渡すのは “verify_only / verify_failed” のどちら？

* **まずは `verify_only.jsonl`**：監査の全体像を要約してほしいとき
* **トラブル時は `verify_failed.jsonl`**：失敗原因に集中してほしいとき

### 7.2 プロンプト例（そのまま使える）

以下を生成AIに貼り、最後に `verify_failed.jsonl` の中身を貼り付けます。

```
あなたは Ansible の運用監査アシスタントです。
以下は Ansible 実行ログ（JSONL）で、VERIFY タスクの結果だけを抽出したものです。

やってほしいこと：
1) failed の項目があれば、ホスト別に短く要約してください
2) それぞれについて「よくある原因候補」を2〜3個に絞って挙げてください
3) 次に確認すべきコマンドやファイル（例: sshd_config, firewall-cmd）を提案してください
4) 追加すると良い VERIFY 項目があれば提案してください

ログ：
（ここに verify_failed.jsonl を貼る）
```

### 7.3 ログが長いときの対処（初心者向けTips）

stdout が巨大な場合は、先頭だけ残して AI に渡します。

```bash
jq -c '
  select(.task | startswith("VERIFY |")) |
  .result.stdout = ((.result.stdout // "") | tostring | .[0:2000])
' verify_only.jsonl > verify_trimmed.jsonl
```

### 7.4 セキュリティ注意（初心者ほど見落としがち）

商用ログには IP・ホスト名・ユーザー名・パス・環境変数、場合によってはシークレットが混入します。
生成AIに渡す前に、**マスキング・no_log の徹底・送信範囲の最小化**を必ず検討してください。

---

## まとめ

* CI コンソールは **default + YAML 風**で人間が読める形にする
* 同時に **JSONL を自作 callback plugin で生成**し、jq/AI で解析できる形にする
* “意図どおりか” は **VERIFY + assert**で確定し、根拠をログに残す
* AI は「判定」ではなく **要約・原因候補・次アクション整理**に使うと強い

---

## 付録：YAML でハマったときのチェック

* `- name:` の下のキーは **同じインデント**
* `tasks:` 配下の `- name:` は **同じ段**
* シェル文字列は `>-'` などで **エスケープ地獄を回避**
