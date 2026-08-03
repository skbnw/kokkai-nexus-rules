# CLAUDE_CORE — Kokkai Nexus / PoliDATA 系 AIエージェント向け中核規約

> このファイルは Claude Code の `@import` で全リポジトリの `CLAUDE.md` から読み込まれる**ダイジェスト**である。完全な規範は `COMMON_RULES.md`（このファイルと同じ `kokkai-nexus-rules` リポジトリ内）に定める。ここに書ききれない詳細は、該当章番号を参照し `view` で読みに行くこと。**このファイル自体を200行以内に保つこと（MUST）。**

## 絶対に破ってはならない7原則（COMMON_RULES §1）

1. **記録者の独立性** — 記録対象・広告主は記録内容に介入できない
2. **非削除** — 物理削除 MUST NOT。除去は `is_hidden`/`missing`→`archived`＋監査ログで表現する
3. **完全な検証可能性** — LLM回答は一次ソース（`source_id`/`snapshot_id`）へ100%逆引き可能でなければならない。逆引きできない情報は出力しない
4. **事実と表現の分離** — 報道本文・SNS原文の転載 MUST NOT。要約＋出典リンクのみ
5. **LLMは入力補助、確定は記名の人間** — AI生成値は必ず `is_ai_generated`/`confidence` を伴う。Layer4/5への昇格は人間確認を経る
6. **影響力行使支援の忌避** — 「訪問ルート」等ロビイング支援機能は実装しない（プロンプト制約だけに頼らず、機能自体を存在させない）
7. **後世への責任** — 採否基準は「今の権力に都合がよいか」ではなく「後世の検証に必要か」

## ID規約（COMMON_RULES §2-1）

| ID | 形式 | 発番元（唯一） |
|---|---|---|
| `person_id` | `P000001` | Supabase `polidata-db` |
| `event_id` | `EV-<YYYYMMDD>-<4桁連番>` | Supabase `events` |
| `theme_id` | `TH-<SLUG>-<3桁連番>` | Supabase |
| `source_id` / `snapshot_id` | UUID | 各コレクタ |

**MUST**: 他リポ・他システムはこれらのIDを独自発番しない。ID文字列のパースをクエリ条件に使わない（日付フィルタは構造化カラムを使う）。

## データレイヤー（COMMON_RULES §3）— 3つの軸を混同しない

裸の数字を書かない。必ず接頭辞を付ける。

- `trust_layer`: L1(公開静的)〜L5(本人証言)
- `trust_score`: 1〜5（ソースの信頼性ガードレール）
- `medallion`: bronze / silver / gold（加工段階）

L4/L5 の掲載基準（全項充足・§3-4）: 記名／独立した複数証言／情報の性質と確認状況の明記／公的活動に関する事項／数値スコア化しない。匿名情報は自動破棄。

## パイプラインの基本形（COMMON_RULES §5, §6-1）

- 原本と抽出結果を分離。取得値と人手補正を分離。候補（`pending`）と確定を分離
- セマンティック・チャンキングは発言者単位。文字数分割 MUST NOT
- GraphRAG標準形: Extraction(Haikuクラス) → Resolution(Sonnetクラス) → Assembly → Querying。詳細は `COMMON_RULES.md §6-1`

## Bronze取得契約（COMMON_RULES §4a 必読）

**外部データを取得するコードを書く前に、必ず `COMMON_RULES.md §4a` を `view` すること。**
- 取得1回ごとに `fetch_id`（UUIDv7）を発番。意味を埋め込んだIDを主キーにしない
- デコード前のバイト列を無加工で保存し、`sha256` を `content_hash` に記録する
- URL / status / 全レスポンスヘッダ / リダイレクト履歴 / TZ付き取得時刻 / 収集器バージョンを記録
- `robots.txt` と利用規約を対象データと同時に取得・保存する（`rights_snapshot`）
- 失敗と非巡回期間も記録する。レコードを作らないことは禁止
- 全レコードに `bronze_contract_version` を記録する（遡及補完は捏造であり禁止）

本契約に適合しない収集器は、本番パイプラインに追加してはならない。

## 実装前に必ず確認すること

新機能・新パイプラインを実装する前に、以下のいずれかに該当する場合は `COMMON_RULES.md` の該当章を `view` で読んでから着手すること。

| やろうとしていること | 参照章 |
|---|---|
| 外部からデータを取得するコードを書く（収集器の新規実装） | **§4a** |
| 白書・先例集など固定資料の取込 | §5-6 |
| YouTube動画のトランスクリプト取込 | §5-7 |
| TV放送TSデータの取込 | §5-9 |
| evidence種別・event_typeの追加 | §5-8 |
| L4データを商用（PoliData）側で使う設計 | §3-6 |
| LLM出力をDBへ書き込む処理 | §6-3, §6-4 |
| Dynamic Workflows / サブエージェント並列化 | §6-2 |

規約に反する実装が必要と判断した場合は、実装せず ADR（`docs/decisions/ADR-NNNN-*.md`）の起票を提案すること。

---
**版**: この `CLAUDE_CORE.md` は `COMMON_RULES.md` の版に追従する。参照している版: v1.6
