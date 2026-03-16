---
name: jog-ready
description: |
  jog MCP を使って Issue を調査し、Ready 状態にするスキル。
  Issue の特定・作成・仕様確認・Ready 化までを担う。
  Craft の作成・実装には進まない。

  以下の場合に使用する：
  - 「Issueを調べて」「仕様を固めて」「Ready化して」と言われた時
  - 「調査だけして」「実装はまだ」と明示された時
  - jog-start の前段として呼ばれる時（通常は直接呼ばない）
---

# jog-ready: Issue 調査・Ready 化スキル

ユーザーの指示を受けてから「仕様が固まった状態」まで整える準備作業を担う。
実装（コーディング）や Craft 作成は行わない。

---

## ステップ 0: jog ツールのロード

jog MCP ツールは遅延ロードなので、使用前に ToolSearch で検索しておく。

```
ToolSearch: "jog issue"
```

---

## Phase 1: Issue の特定

ユーザーの入力は 2 パターンある。

### パターン A: Issue 番号が指定されている場合

`issue_get(issue_id=N)` で内容を取得して Phase 2 へ。

### パターン B: 自然言語でタスクが説明された場合

1. `issue_list` で既存 Issue を確認し、重複・類似するものがないか確認する
   - 完全に重複するものがあればそれを採用して Phase 2 へ
   - 類似するものがあればユーザーに確認してから判断する
2. 重複がなければ `issue_create(title="...", body="...", status="draft")` で新規作成する
   - title は短く要点をついたもの（日本語 OK）
   - body には目的・背景・受け入れ条件を記載する（不明な部分は TODO で OK）

---

## Phase 2: Issue の Ready 化

**実装前に仕様を固めることが目的。** 仕様が曖昧なまま作業を進めると手戻りが発生する。

1. `issue_get` で現在の内容を確認する
2. 以下の点が不明・曖昧・複数案あるなら**ユーザーに確認する**:
   - 何を実装するか（スコープ）
   - どう動けば OK か（受け入れ条件）
   - 影響する既存の仕組みや懸念点
3. `issue_update(issue_id=N, title="...", body="...", labels=[...])` で内容を充実させる
   - もともとdraftで存在していたissueの場合、コメントとして「draft時の記載内容」を追加して残しておく
4. `issue_change_status(issue_id=N, status="ready")` で Ready に変更する

> 仕様が固まっていない Issue を Ready にするのは禁止。ユーザーとの対話で解消してから進む。

---

## 完了後の報告

呼び出し元（jog-start）またはユーザーに以下を伝える：

- Ready 化した **Issue** の番号・タイトル
- 次のステップとして `jog-impl` で実装を開始できること

---

## やってはいけないこと

- 仕様が曖昧なまま `status="ready"` にする
- Craft を作成する（Craft 作成は jog-impl の役割）
- Issue の番号を確認せずに作業を進める
