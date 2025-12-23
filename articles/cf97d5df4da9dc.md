---
title: "ユースケース別に理解するuvを使ったPythonパッケージ バージョン管理"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [uv, python]
published: true
---

## はじめに

みなさん、uv 使ってますか？
mcp サーバー使おうとすると uvx が多いし、気づいたら uv がデファクトスタンダードな雰囲気あって、真面目に整理するかーという気持ちでこの記事を書いてます。
uv 以前は poetry とかあったけど、人によって好み違いそうだし、pyenv と venv 使うかーってなってましたが、最近は uv でいいかーという気持ちです。同じような気持ちの人に届け。

まずですが、そもそも Python で開発してると以下のような場面がそこそこある気がします。

- Python のバージョンが人によって違う
- 途中参加すると環境が再現できない
- requirements.txt が最新じゃなくて動かない

**uv** は、これらをなんとかしてくれます。
この記事では **ユースケース** に uv の使い方と考え方を整理します。

https://docs.astral.sh/uv/

---

## 前提：uv で管理する 3 点セット

uv 前提の Python プロジェクトは、次の 3 点で整理されます。覚えましょう。

```
.python-version   # 実際に使うPython
pyproject.toml    # 意図・制約・設定
uv.lock           # 依存関係の確定結果
```

ちなみに、`.venv/` は生成物なので Git 管理しない
`.python-version` と `uv.lock` は必ずコミット

---

# ユースケース編

## ユースケース ①：新規プロジェクトを作る（Python 一覧確認含む）

### やりたいこと

- Python バージョンを最初から固定したい
- チームで再現可能な環境にしたい

### 利用可能な Python を確認する

```
uv python list
```

インストール済みのみ表示：

```
uv python list --only-installed
```

### プロジェクト作成 & Python 固定

```
mkdir myproj && cd myproj
uv init
uv python pin 3.12
uv sync
```

#### 何が起きているか

- pin で`.python-version` が作られ、Python 3.12 が固定される
- 必要なら uv が Python を自動ダウンロード
- `pyproject.toml` が生成される
- `.venv` が作られ、プロジェクトの環境での参照先になる
- `uv.lock` に結果が保存される

👉 最初に pin するの大事！！

---

## ユースケース ②：既存プロジェクトに途中参加する

（Python が固定されていない場合を含む）

### ケース A：`.python-version` がある（理想）

```
uv sync
```

- `.python-version` を読む
- Python がなければ自動でダウンロード
- `.venv` 作成
- `uv.lock` 通りに依存をインストール

👉 README を読まなくても同じ環境が再現される

### ケース B：`.python-version` がない（よくある事故）

```
uv sync
```

- brew / pyenv / system python が使われる
- 人によって Python バージョンがズレる

| 人  | 使われる Python |
| --- | --------------- |
| A   | brew 3.12       |
| B   | pyenv 3.11      |
| C   | system 3.10     |

#### `.venv` の Python が brew 由来になることはある？

**YES（pin していない場合）**

👉 `.python-version` を置かないと事故る

#### 正しい対処（プロジェクト側）

```
uv python pin 3.12
uv sync
git add .python-version
git commit
```

---

## ユースケース ③：Python バージョンを変更したい

（アップグレード / ダウングレード共通）

```
uv python pin 3.12
uv sync
```

- `.python-version` 更新
- `.venv` 再生成
- `uv.lock` 再解決

👉 Python のロールバックが簡単

---

## ユースケース ④：複数プロジェクトを並行で触る

```
proj-a/.python-version → 3.11
proj-b/.python-version → 3.12
```

- プロジェクトごとに Python / venv / lock が完全分離
- pyenv の切り替え不要 (uv run python で実行するから)
- PATH を汚さない

---

## ユースケース ⑤：AI Agent が pyproject.toml を書いた

> AI は「提案」まで、確定は uv に任せる

### 推奨フロー

1. AI の変更内容をレビュー
2. 設定系（ruff / pyright / scripts 等）は反映 OK
3. 依存が変わったら：

```
uv sync
```

---

# その他（補助知識）

## .venv は activate すべき？

**しなくても OK**

```
uv run python app.py
uv run pytest
```

---

## VS Code ではどうなる？

- `.venv/` があると **自動検出**されることが多い
- 初回のみ選択を求められる場合あり

必要なら

```
  Cmd + Shift + P
  → Python: Select Interpreter
  → .venv/bin/python
```

---

## ruff / pyright / pytest は何をしている？

### ruff

- 超高速 linter / formatter（Rust 製）
- flake8 + isort + black 相当
- **コードの無駄・規約・整形**

### pyright

- 静的型チェッカー
- 実行前に型の破綻を検出

### pytest

- Python 標準的テストランナー

### プロジェクトに入れる場合

```
uv add ruff pyright pytest --dev
```

### 実行

```
uv run ruff check .
uv run ruff format .
uv run pyright
uv run pytest
```

---

## uvx とは？

```
uvx ruff check .
```

- Python 版 npx
- 一時的にツールを実行
- `.venv` や依存を汚さない

---

# まとめ

- `.python-version` は必須
- Python 変更は `uv python pin`
- 新規参加者は `uv sync`
- AI が触ったら `uv sync`
- 実行は `uv run`
- VS Code は `.venv` を自動検出

---

## 一言まとめ

結構シンプル。venv と互換性保つため`.venv`を使ってるの好感持てる。

---

## 参考文献

- uv 公式ドキュメント
  https://docs.astral.sh/uv/
- Turing Motors さんの記事
  https://zenn.dev/turing_motors/articles/594fbef42a36ee
- yareyare さんの記事
  https://zenn.dev/yareyare/articles/d67aa75b37537c
