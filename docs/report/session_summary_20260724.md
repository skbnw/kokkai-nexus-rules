# セッションサマリー — COMMON_RULES.md 策定・kokkai-nexus-rules 新設

**日付**: 2026年7月24日  
**参加**: Souichirou / Claude  
**成果物の格納**: `docs/report/session_summary_20260724.md`（本ファイル）、正本は `COMMON_RULES.md`

---

## 1. 本セッションの目的

これまで個別に存在していた設計思想文書（political_platform_concept_v2.docx）・利用規約（terms_v03.docx/terms_simple.docx）・グランドデザイン報告書（PoliData系）・開発セッションサマリー（cursor_session_summary.md）を統合し、Kokkai Nexus / PoliDATA 系全リポジトリが従う単一の上位規範を確立すること。あわせて、この規範をAIエージェント（Claude Code / Cursor）にどう実装レベルで守らせるかの具体的な配線を作ること。

---

## 2. 成果物

| ファイル | 内容 | バージョン |
|---------|------|-----------|
| `COMMON_RULES.md` | 完全版・人間向けの正本 | v1.5 |
| `CLAUDE_CORE.md` | Claude Code の `CLAUDE.md` から `@import` する100行程度のダイジェスト | v1.5に追従 |
| `CLAUDE.md.template` | 各リポジトリ配置用の雛形（policy-rag-poc / openclaw-news-pipeline 向け記入例つき） | — |
| `kokkai-nexus-rules.zip` | 上記＋schemas/*.json（trust_layer / evidence / event_type / event / event_attendee / entity_master）＋document_catalog.template.yaml＋README.md（セットアップ手順） | — |

---

## 3. 主要な決定事項（版ごとの要約）

`COMMON_RULES.md §12`（変更履歴）が正本。ここではインデックスとして章番号のみ示す。

- **v1.0**: 初版。不変原則P1〜P7、ID規約、trust_layer/trust_score/medallionの三軸分離、ストレージ責務分担、パイプライン規約を新設
- **v1.1**: §5-6「政府白書の体系化」、§6-1「GraphRAG標準パイプライン（Extraction/Resolution/Assembly/Querying）」を新設
- **v1.2**: §6-2「Dynamic Workflows（Claude Code公式機能）」を実行基盤として整理。第三者記事と公式一次情報の切り分けを明記
- **v1.3**: 未決事項A（kokkai-nexus-rules独立リポジトリ新設）・C（event_id/theme_id採番形式）を解決。B（L4データの二系統ブランド間流用）を§3-6として部分決定（要最終確認）
- **v1.4**: §4-4「物理バックアップ（3-2-1ルール）」、§5-7「YouTube記者会見動画」、§5-8「evidence/event_type統一登録簿」、§5-9「TV放送TSデータ」を新設
- **v1.5**: §9-4を全面改訂。COMMON_RULES.md全文の@importを禁止し、CLAUDE_CORE.mdとの二層構成を確立

---

## 4. 特に重要な論点と判断根拠

### 4-1. 三軸混同の禁止（§3-1）

`trust_layer`（L1〜L5）・`trust_score`（1〜5）・`medallion`（bronze/silver/gold）が同じ「1〜5」の数字を使うため、裸の数字表記を禁止し接頭辞必須とした。

### 4-2. event_id発番の一本化（§2-1）

Supabaseの`events`テーブルのみが発番権を持つ。GCS/Pineconeは参照側のみ。形式は`EV-<YYYYMMDD>-<4桁連番>`、ID文字列のパースをクエリ条件に使うことを禁止し、時系列フィルタは構造化された日付カラムを使う方針とした。

### 4-3. L4データの商用流用（§3-6・未決B）

Souichirouからの提案（デフォルト許諾＋アプリ層フィルタ）に対し、以下2点を修正して反映した。

1. 同意フラグのデフォルトを `true` ではなく `false`（未同意）に変更
2. アプリ層のクエリフィルタだけでなく、SupabaseのRLS（Row Level Security）でDB層に強制

**この2点は事業判断であり、Souichirouの最終確認が未了である。**

### 4-4. Dynamic Workflowsの出典切り分け（§6-2）

共有された資料のうち、Dynamic Workflows自体はAnthropic公式機能（2026年5月28日公開、検索で確認済み）だが、付随する「14-step roadmap」記事は第三者による技術解説（GPTZeroにより AI Detected 判定）であり、「Anthropicのシニアエンジニアが公開した」という言及は裏付けが取れない伝聞である。両者を区別して記録した。

### 4-5. テロップOCRによる話者同定（§5-9-c）

TV放送のTSデータについて、生字幕（CC）よりも画面上のテロップ（名前表示）のOCRの方が話者同定の信頼度が高いと判断し、`broadcast_telop`（高信頼）と`broadcast_cc`（低信頼）を区別するevidence種別を新設した。

### 4-6. CLAUDE.mdの二層構成（§9-4）

Claude Code公式ドキュメントで「CLAUDE.mdは200行を超えるとコンテキスト消費が増え指示追従性が下がる」という制約を確認した。600行超の`COMMON_RULES.md`をそのまま`@import`することを避け、100行程度のダイジェスト`CLAUDE_CORE.md`を新設して各リポの`CLAUDE.md`から`@import`させ、詳細は必要な章番号を`view`で読みに行かせる設計とした。

---

## 5. 未決事項一覧（COMMON_RULES.md §11 のインデックス）

| 記号 | 内容 | 状態 |
|------|------|------|
| A | 共通ルールの物理共有方法 | 解決済み（v1.3） |
| B | 二系統ブランドのL4流用範囲 | 部分決定・Souichirouの最終確認待ち |
| C | event_id/theme_id採番形式 | 解決済み（v1.3） |
| D | Human-in-the-Loopの昇格境界 | 未着手（次回優先候補） |
| E | GraphRAGとPineconeの検索ルーティング | 未着手 |
| F | 評価メトリクス（政治的公平性・時系列矛盾検知） | 未着手 |
| G | 先例集・年表の構造化精度 | 未着手 |
| H | 白書の年度差分検知の実装方式 | 未着手 |
| I | GraphRAG（Assembly段階）のグラフ保存先 | 未着手 |
| J | Dynamic Workflowsの適用範囲の実測 | 計測待ち |
| K | `.claude/rules/*.md`条件付きロードの導入可否 | 見送り中（Windows環境のシンボリックリンク問題） |

---

## 6. 保留・未完了タスク（2026-07-24時点）

- `policy-rag-poc` / `openclaw-news-pipeline` のCLAUDE.md整備・サブモジュール追加は未実施（2026-07-25セッションで完了予定）
- `kokkai-nexus-rules` のGitHub push・両リポジトリへの `git submodule add` は未実施（同上）
- 未決事項B（`commercial_use_allowed` のデフォルト値）はSouichirouの最終確認が必要

---

## 7. 次回セッションへの申し送り

- 未決事項D（Human-in-the-Loopの昇格境界）から着手すると、B（「AI自動処理と人間確定の境目」問題）と合わせて効率的に議論できる
- `kokkai-nexus-rules` のセットアップ結果（成功/エラー）を共有してもらい、`policy-rag-poc`・`openclaw-news-pipeline` の既存スキーマとの実際の差分を洗い出す
- 白書（§5-6）・YouTube（§5-7）・TS放送（§5-9）の各取込パイプラインは、まだ設計のみで実装コードは書かれていない

---

> 本ファイルは `COMMON_RULES.md` v1.5 時点のスナップショットである。以後の規約変更は `COMMON_RULES.md §12`（変更履歴）を正とする。
