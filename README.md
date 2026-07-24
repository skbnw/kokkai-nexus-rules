# kokkai-nexus-rules

Kokkai Nexus / PoliDATA 系全リポジトリの共通規範を保持する軽量リポジトリ。
`COMMON_RULES.md` §0-1 の決定に基づき新設。**このリポジトリ単体をアプリケーションとして実行することはない。**

## 収録物

| ファイル/ディレクトリ | 内容 |
|---|---|
| `COMMON_RULES.md` | 完全版・人間向けの正本 |
| `CLAUDE_CORE.md` | Claude Code の `CLAUDE.md` から `@import` するダイジェスト（§9-4） |
| `CLAUDE.md.template` | 各リポジトリに配置する `CLAUDE.md` の雛形（policy-rag-poc / openclaw-news-pipeline の記入例つき） |
| `schemas/*.json` | ID・evidence・event_type・trust_layer 等のJSON Schema |
| `document_catalog.template.yaml` | 中央資料目録の雛形 |

## セットアップ手順（Windows / PowerShell）

### 1. このリポジトリ自体を初期化する

```powershell
cd C:\Users\user\Documents\Github\kokkai-nexus-rules
git init
git add .
git commit -m "Initial commit: COMMON_RULES v1.5"
git tag v1.5
# GitHub側でプライベートリポジトリを作成してから:
git remote add origin https://github.com/skbnw/kokkai-nexus-rules.git
git push -u origin main --tags
```

### 2. 既存リポジトリ（例: policy-rag-poc）へ Submodule として追加する

```powershell
cd C:\Users\user\Documents\Github\policy-rag-poc
git submodule add https://github.com/skbnw/kokkai-nexus-rules.git .rules
cd .rules
git checkout v1.5
cd ..
git add .rules .gitmodules
git commit -m "kokkai-nexus-rulesをSubmoduleとして追加（v1.5固定）"
```

`openclaw-news-pipeline` にも同様の手順を適用する。

### 3. 各リポジトリの `CLAUDE.md` を作成する

`CLAUDE.md.template` をコピーし、リポジトリ直下に `CLAUDE.md` として配置。コメントアウトされた記入例（policy-rag-poc用・openclaw-news-pipeline用）を参考に、「本リポジトリの担当範囲」「固有の差分」を埋める。

```powershell
Copy-Item .rules\CLAUDE.md.template CLAUDE.md
```

### 4. バージョン更新時の運用

`COMMON_RULES.md` が改訂されたら、このリポジトリ側で新しい `git tag`（例: `v1.6`）を打つ。各利用リポジトリは、動作確認のうえで

```powershell
cd .rules
git fetch --tags
git checkout v1.6
cd ..
git add .rules
git commit -m ".rules を v1.6 へ更新"
```

を実行して**明示的に**追従する。`.rules` が自動で最新化されることはない（§0-1: HEADへの浮動参照はMUST NOT）。

## 未確定事項

- `.claude/rules/*.md` によるファイル種別ごとの条件付きロードは、Submodule越しのシンボリックリンクがWindows環境で脆弱なため現時点では未導入（COMMON_RULES §11 未決事項K）
- Assembly段階（GraphRAG）で生成するグラフの永続化先は未定（未決事項I）。決まり次第 `schemas/` に追加する
