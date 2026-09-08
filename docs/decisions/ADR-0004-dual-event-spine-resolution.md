# ADR-0004 — 二重の `events` 背骨を解消する（Supabase ↔ SQLite）

| 項目 | 内容 |
|---|---|
| 状態 | **承認**（2026-09-08 / Souichirou） |
| 起票日 | 2026-09-08 |
| 影響する規約 | `COMMON_RULES.md` §2-1（発番の単一化・MUST）／§4-2（サービング層の役割分担） |
| 関連 | `polidata_development/99-plan/20260907_unified_platform_design_v1.1.md` §2 ／ ADR-0003 |

---

## 1. 文脈 — 規約違反がすでに発生している

§2-1 は「同一種のIDを複数システムが独立に発番してはならない（MUST）」と定め、
`event_id` の発番元を **Supabase `events` テーブルのみ**に固定している。

しかし 2026-09-07 の実測で、**`events` という名のテーブルが2つ、別々のID体系で存在する**ことが判明した。

| | Supabase `polidata-db` | SQLite `polidata_schedule/db/polidata_schedule.db` |
|---|---|---|
| 行数 | **0** | **2,963** |
| ID | `event_id text` = `EV-<YYYYMMDD>-<4桁>` | `event_id INTEGER PRIMARY KEY`（1からの連番） |
| 従属表 | `event_sources`（0行） | `event_sources`（**3,042行**） |
| 状態 | 規約準拠の器だけがある | **稼働中**（日次バッチで再構築） |

つまり **規約上の正本が空で、実際に動いている実装が規約の外にある**。
このまま Supabase 側に `EV-` を発番し始めると、同じ予定に2つの ID が付き、
どちらが正本か判別できない状態が固定化する。

### 1-1. ただし SQLite 側の実装は設計として優れている

`polidata_schedule/02-db/build_events.py` は、時事通信と policynews.jp の予定を
**名寄せキーで束ね、各社の原文を保持したまま1イベントに統合**している。

- `match_key = f"{date}|{signature(subject)}"` による名寄せ（`signature()` は NFKC正規化・括弧内除去・記号除去）
- `event_sources` が `(event_id, source, role, original_event, original_time, url)` を保持
  → **多対多・出典保持・原文保持**。これは設計書がいう `event_links` の要件をすでに満たしている
- `status`（確定／未定）＋ `confirmed_at` により「時刻未定 → 確定」の遷移を1レコードで追跡

**捨てるべき実装ではなく、昇格させるべき実装である。**

---

## 2. 決定 — 案B（対応表による橋渡し）

**SQLite は収集・名寄せ層に留め、`event_id` の発番は Supabase に一本化する。**
両者は `event_external_ids` 対応表で結ぶ。

```
SQLite（収集・名寄せ層）                Supabase（背骨・唯一の発番元）
──────────────────────                  ──────────────────────────────
jiji_calendar / policynews_calendar
        ↓ build_events.py（毎日再構築）
events(event_id INTEGER, match_key)  ──┐
event_sources(event_id, source, ...)   │  match_key で照合
                                       ▼
                        event_external_ids(system, external_id=match_key)
                                       ↓
                        events(event_id='EV-YYYYMMDD-NNNN')  ← next_event_id() で発番
                        event_links(event_id, entity_table, entity_pk, link_role)
```

### 2-1. 🔴 対応表のキーは `match_key`。`event_id` を使ってはならない（MUST NOT）

`build_events.py` は冪等設計のため**毎回テーブルを作り直す**:

```python
SCHEMA_SQL = """
DROP TABLE IF EXISTS event_sources;
DROP TABLE IF EXISTS events;     # ← 毎回まるごと再構築
...
event_id = 0
for key, members in groups.items():
    event_id += 1                # ← dict の反復順に 1 から振り直す
```

したがって **SQLite の `event_id INTEGER` は安定IDではない**。
raw テーブル（`jiji_calendar` / `policynews_calendar`）に行が増減すると `groups` の順序が動き、
同じ予定に別の番号が付きうる。

一方 `match_key` は `signature()` という**決定論的関数の出力**であり、
再構築しても同じ入力からは必ず同じ値が得られる。

> **これを取り違えると、日次バッチが走るたびに Supabase 側のリンクが別の予定を指す。**
> 例外もエラーも出ず、データだけが静かにずれる。発見が極めて遅れる種類の不具合である。

### 2-2. 対応表の定義

```sql
create table event_external_ids (
  event_id      text not null references events(event_id),
  system        text not null,          -- 'polidata_schedule_sqlite'
  external_id   text not null,          -- ← match_key を入れる（2-1）
  external_pk   text,                   -- 取り込み時点の SQLite event_id（監査用）
  synced_at     timestamptz not null default now(),
  primary key (system, external_id)
);
```

- `primary key (system, external_id)` により、同じ `match_key` が二重にイベントを作らない
- `external_pk` は**監査用にのみ**保持し、**結合条件に使わない**
  （§2-1a「ID はラベル、フィルタ条件は別カラム」と同じ精神）

### 2-3. 同期の手順

日次バッチ（現行 00:05 JST）の末尾に同期ステップを追加する:

1. SQLite 側で `build_events.py` を実行（現行どおり）
2. SQLite `events` を全件読み、`match_key` で `event_external_ids` を引く
3. **在れば** 対応する Supabase `events` を更新（`status` / `confirmed_at` / `title` 等）
4. **無ければ** `next_event_id(occurred_on)` で発番し、`events` と `event_external_ids` に挿入
5. `event_sources` の各行を `event_links`（`entity_table='jiji_calendar'` 等）へ upsert

**SQLite 側の `event_id` は、以後 SQLite 内部の実装詳細**として扱い、外部へ公開しない。

---

## 3. 却下した代替案

| 案 | 内容 | 却下理由 |
|---|---|---|
| **A** | SQLite を廃し、収集も名寄せも Supabase へ完全移行する | 規約 §2-1 に最も忠実だが、稼働中の日次バッチ・Datasette 公開UI（`polidata-schedule.fly.dev`）・`build_events.py` の全面書き直しが必要。移行中に収集が止まるリスクがある。**将来の選択肢としては残す**（案Bの上位互換であり、案Bはその前提を壊さない） |
| **C** | 2系統を許容し、`event_links` で相互参照するだけ | §2-1 違反が固定化する。「どちらが正本か」を人間が都度判断し続けることになり、§1-P3（完全な検証可能性）が実務上守れない |
| **D** | SQLite 側を `EV-` 形式に書き換え、SQLite で発番する | 発番元が Supabase から移るだけで §2-1 の「発番元は Supabase」に反する。加えて `build_events.py` の冪等な再構築と、一度発番したIDの永続性が両立しない |
| **E** | `external_id` に `event_id INTEGER` を使う（案Bの変種） | 2-1 の通り、再構築で意味が変わる。**静かに壊れる**ため最も危険 |

---

## 4. 影響範囲

| 対象 | 変更内容 |
|---|---|
| Supabase | `event_external_ids` を新設（Phase 2 の `0012_event_spine.sql`） |
| `polidata_schedule` | 日次バッチに同期ステップを追加（2-3）。`build_events.py` 自体は**変更しない** |
| `polidata_schedule` Datasette | 影響なし（SQLite を直接配信しているため） |
| `COMMON_RULES.md` §2-1 | 「外部システムの安定キーは `<system>_external_ids` 対応表で受ける。対応表のキーには決定論的に再現できる値を用いる（MUST）」を追記 |
| `COMMON_RULES.md` §4-2 | 収集・名寄せ層としての SQLite の位置づけを追記 |
| `COMMON_RULES.md` §12 | 変更履歴に v1.7 を追記 |

**Bronze 契約（§4a）との関係**: 本 ADR は Silver/Gold 層の ID 対応の話であり、
Bronze 取得記録には影響しない。SQLite `events` は Bronze ではなく名寄せ済み（Silver 相当）である。

---

## 5. 検証方法

1. **冪等性**: `build_events.py` を2回連続実行し、同期後の Supabase `events` の行数と `event_id` が変わらないこと
2. **再構築耐性**: `jiji_calendar` に新規行を1行足してから再構築 → 既存イベントの `event_id`（`EV-`）が保持されること
3. **重複防止**: 同期を2回流しても `event_external_ids` の行数が増えないこと
4. **逆引き**: 任意の `EV-` から `event_links` 経由で `original_event`（各社原文）と `url` に到達できること（§1-P3）
5. **未定→確定**: SQLite 側で `status` が「未定」→「確定」に変わったとき、Supabase 側にも `confirmed_at` が伝播すること

---

## 6. 未解決の論点

- `komei_conference` / `ldp_conference` / `ldp_media` / `shugiin_kaigiroku` / `sangiin_koho` など、
  名寄せ対象外のテーブル（`build_events.py` の設計 §13.1 で対象外と明記）をいつ背骨へ載せるか
- Supabase `schedule_items`（5,128行・自民党系のみ）と SQLite `ldp_conference`（9,841行）の関係整理。
  同一ソースの二重投入である可能性が高く、別途照合が要る
- 案A（Supabase 完全移行）へ移行する判断基準。設計書 §10 のホスティング判断と同時に検討する
