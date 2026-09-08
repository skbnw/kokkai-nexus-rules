# ADR-0003 — `event_type` に `election` / `gazette` / `cabinet_decision` を追加する

| 項目 | 内容 |
|---|---|
| 状態 | **承認**（2026-09-08 / Souichirou） |
| 起票日 | 2026-09-08 |
| 影響する規約 | `COMMON_RULES.md` §5-8-b（event_type 統一登録簿）／`schemas/event_type.enum.json` |
| 関連 | `polidata_development/99-plan/20260907_unified_platform_design_v1.1.md` §5 ／ ADR-0004 |

---

## 1. 文脈

統合プラットフォームの背骨（Supabase `events`）に、選挙・官報・政策日程を載せる作業を始める。
しかし現行の `event_type` 列挙は**国会・党内会合の会議体しか想定していない**。

現行値（§5-8-b）:
`plenary` / `committee` / `caucus` / `party_meeting` / `study_group` / `press` / `broadcast_program` / `other`

**「選挙」も「官報公示」も「閣議決定」も表現できない。**

§5-8 は「各リポジトリが個別に列挙値を持つことを禁止する（MUST）」と定めているため、
各プロジェクトが勝手に値を足すことはできない。本 ADR で登録簿そのものを拡張する。

### 1-1. 実データによる裏付け

| 追加候補 | 対象データ | 実測 |
|---|---|---|
| `cabinet_decision` | `polidata_schedule` SQLite `kakugi` | **5,887案件 / 338回の閣議**（2024-01-05〜2026-08-25） |
| `election` | Supabase `eln_elections` | 17件（ただし `genre` は `HR`/`HRC` のみ。知事選・地方選は未投入） |
| `gazette` | 官報 | **未実装**。ただし `kakugi.category` に `公布(法律)` 247件 ＋ `公布(条約)` 41件 ＝ **288件の公布記録が既にある** |

`gazette` を先に定義しておくことで、官報コレクタ（設計書 §6-1 / §4a 適用）を作る前に
**既存の閣議データ288件を照合対象（ground truth）として使える**。

---

## 2. 決定

### 2-1. 3値を追加する

| 値 | ラベル | 定義 | 構造 |
|---|---|---|---|
| `election` | 選挙 | 選挙の**投開票**という単一事象 | 候補者・選挙区・得票が結果としてぶら下がる |
| `gazette` | 官報 | 官報における**公布・公示** | 1つの官報号に複数の公布事項がぶら下がる |
| `cabinet_decision` | 閣議 | **閣議1回**（定例・臨時・持ち回り・繰上げ・初閣議） | 1回の閣議に複数の案件がぶら下がる |

### 2-2. 粒度の線引き（重要）

**`events` は「会議体・事象の1回」を表し、その中身は `event_links` にぶら下げる。**
背骨を案件単位で肥大させない。

| データ | events 行数 | event_links 行数 |
|---|---:|---:|
| 閣議 | **338**（`date` + `kakugi_type`） | **5,887**（案件。`link_role='agenda'`） |
| 選挙 | 1選挙 = 1行 | 候補者・市町村別得票 |
| 官報 | 1号 = 1行 | 公布事項 |

- `election` は**投開票日**の事象。**公示・告示は `gazette` 側**で表現する（官報に載るため）。
  同一選挙の公示と投開票は別イベントとし、同じ `theme_id` で束ねる。
- `cabinet_decision` は**閣議という会議体の開催**を指す。個々の「◯◯を決定した」は案件＝`event_links`。
- `kakugi.category='公布(法律)'` の案件は、官報コレクタ実装後に
  対応する `gazette` イベントへ `event_links` で結び直す（閣議決定 → 公布 のリードタイム分析が目的）。

### 2-3. `pm_activities.event_type` は統合しない

Supabase `pm_activities`（168,323行）は独自の `event_type` を持つ:

```
面会 43,569 / 会議 24,051 / 職務・公務 22,828 / 移動 14,552
メディア 3,417 / その他 2,963 / null 56,943
```

これは**首相の行動分類**であり、§5-8-b の `event_type`（**会議体・事象の構造分類**）とは目的が違う。

**統合しない。** `pm_activities.event_type` はドメイン固有列として現状のまま維持し、
背骨に載せる際は `events.event_type` を別途与える（例: 記者会見なら `press`）。

> 混ぜた場合の弊害: `面会` を登録簿に入れると「会議体の構造」と「人物の行動」が同一軸に並び、
> `plenary` と `移動` が同じ列挙に共存する。どちらの分類としても集計できなくなる。

---

## 3. 却下した代替案

| 案 | 内容 | 却下理由 |
|---|---|---|
| **A** | `other` で代用する | 3つの PoC すべてが `other` になる。`event_type` による集計・絞り込みが機能しなくなり、登録簿を持つ意味が消える |
| **B** | `pm_activities` の日本語分類（面会・会議…）に統一する | 2-3 の通り、会議体分類と行動分類の混同。既存8値との整合も取れない |
| **C** | `event_type` を自由文字列にする | §5-8「各リポジトリが個別に列挙値を持つことを禁止する（MUST）」に正面から反する。表記揺れが並立する |
| **D** | 閣議を**案件単位**でイベント化する（5,887件） | 背骨が案件で埋まり、「1回の閣議」という事象が表現できなくなる。`source_count` 等の集計単位も壊れる |
| **E** | 公示・告示も `election` に含める | 公示は官報に載る公的行為であり、投開票とは日付も性質も異なる。同一 `theme_id` で束ねれば十分 |

---

## 4. 影響範囲

| 対象 | 変更内容 |
|---|---|
| `schemas/event_type.enum.json` | `enum` に3値を追加。`meta` にラベルと定義を追記 |
| `COMMON_RULES.md` §5-8-b | 列挙の記述を更新。粒度の線引き（2-2）を追記 |
| `COMMON_RULES.md` §12 | 変更履歴に v1.7 を追記（§0-3 で MUST） |
| `CLAUDE_CORE.md` | 版参照を v1.7 へ更新 |
| Supabase `events.event_type` | CHECK 制約 or 参照テーブルの値域（Phase 2 の `0012_event_spine.sql`） |
| `Kokkai_Nexus_app/shared/schema/events.ts` | Drizzle 側の型 |
| 各リポの submodule | `kokkai-nexus-rules` の tag を v1.7 へ更新（§0-1 のピン留め運用） |

**後方互換性**: 既存8値は変更しない。追加のみ。Supabase `events` は現在0行のため、既存データの再分類は発生しない。

---

## 5. 検証方法

1. `event_type.enum.json` が JSON Schema として妥当（既存の CI 検証に乗る）
2. 閣議338件を投入したとき、`events` が 338行・`event_links` が 5,887行になる
3. `select event_type, count(*) from events group by 1` に `other` が偏って出ない
4. `pm_activities.event_type` の値が `events.event_type` に混入していないこと

---

## 6. 未解決の論点（本 ADR の範囲外）

- `eln_elections.genre` の値域拡張（現在 `HR`/`HRC` のみ。知事選・地方選の追加）は選挙クラスタ側の課題。
  `event_type='election'` の定義自体はジャンルに依存しないため、本 ADR では扱わない
- 官報の号・部（本紙／号外／政府調達等）をどこまで構造化するかは Phase 6 の `kanpo_notices` 設計で扱う
