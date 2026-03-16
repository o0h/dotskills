---
name: worktree-drain
description: ワークツリーでチェックアウト済みのブランチをメインリポジトリディレクトリにチェックアウトする。「AIが実装を完了したのでIDEやメイン側で確認したい」「ワークツリーのブランチをメインに持ってきたい」「worktreeで使っているブランチをメインディレクトリにチェックアウトしたい」「実装が終わったのでレビューしたい」という場面で使用する。ワークツリー側のdetach HEAD・未コミット変更のstash→pop・worktreeの削除まで自動的に処理する。
---

# Worktree Drain

AIがワークツリー上で実装を完了した後、メインリポジトリディレクトリで同じブランチをチェックアウトするスキル。
ワークツリーの状態をすべてメイン側に"drain"し、ワークツリーを削除するまで一括で処理する。

**ブランチとそのコミットは削除されない。** ワークツリー（ディレクトリ）だけが削除される。

## フロー

### Step 1: ブランチ選択

直近のブランチ候補を取得する：

```bash
git branch --sort=-committerdate --format='%(refname:short)' | head -6
```

`AskUserQuestion` で選択肢を提示する：
- 直近6ブランチ（main/master を除いてよい）
- 「その他（自由入力）」

### Step 2: ワークツリーの状態確認

`git wt --json` でワークツリー一覧をJSON取得し、対象ブランチを持つエントリを探す：

```bash
git wt --json
```

JSONの `branch` フィールドが `refs/heads/<target-branch>` と一致するエントリが対象。
先頭エントリ（メインリポジトリ）は除外する。

### Step 3: 未コミット変更をstash

対象ワークツリーの変更状況を確認する：

```bash
git -C <worktree-path> status --porcelain
```

変更がある場合、確認なしで自動的にstashする。
stashはリポジトリ全体で共有されるため、ワークツリー削除後にメイン側でpopできる。

```bash
git -C <worktree-path> stash push -u -m "worktree-drain auto-stash: <branch>"
```

`-u` フラグで untracked ファイルも含めてstashする。

### Step 4: ワークツリー側で detach HEAD

対象ブランチがワークツリーにチェックアウト中の場合のみ実行：

```bash
git -C <worktree-path> switch --detach
```

### Step 5: メインリポジトリでチェックアウト

メインリポジトリのパスは `git wt --json` の先頭エントリの `path` を使う：

```bash
git -C <main-repo-path> checkout <target-branch>
```

### Step 6: ワークツリーを削除
**重要**
ワークツリーが、「元のレポジトリ名 + 文字列」という名前の場合は削除しない。
通常のfeture実装用のワークツリーではなく、特殊な用途に用いられるため。
例)
レポジトリ名: `o0h/my-works` / ワークツリー名: `my-works-2` => 削除しない
レポジトリ名: `o0h/my-works` / ワークツリー名: `010-feat-009-som-work` => 削除する

ワークツリーの削除を行う場合、ブランチは保持したまま、ワークツリーディレクトリだけを削除する：

```bash
git worktree remove <worktree-path>
```

`git wt -d` はブランチまで削除してしまうため使わない。

### Step 7: stash popでメイン側に変更を復元（Step 3でstashした場合のみ）

```bash
git -C <main-repo-path> stash pop
```

これで未コミットの変更がメイン側に移される。drain完了。

## 完了メッセージ

```
✅ feature/xxx をメインディレクトリにdrainしました
   メイン: /path/to/main-repo  →  feature/xxx
   ワークツリー: /path/to/.wt/feature-xxx  →  削除済み
   ブランチとコミットは保持されています

# stashした場合のみ追記：
💾 未コミットの変更もメイン側に復元しました（stash pop済み）
```

## エッジケース

- **対象ブランチがどのワークツリーにもない場合**: そのままメイン側でチェックアウトする（Step 3〜7はスキップ）
- **メインリポジトリ自体が対象ブランチをチェックアウト中の場合**: すでに目的の状態なので完了を伝える
