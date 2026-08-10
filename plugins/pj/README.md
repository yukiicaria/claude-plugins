# pj — 仕様駆動パイプライン

1つのプロジェクトを **WHAT → HOW → コード** まで一貫して育てる Claude Code プラグイン。
3フェーズが **同じ readiness 軸・共通モード・glossary・保守規律**を共有する。

```
/pj:setup   DAY 0  足回り + 器     git/lint/format/husky、pj-managed な CLAUDE.md、デザインシステム導入
                                   package <name> で「自分で作るライブラリ」の器を生やす
/pj:spec    WHAT   docs/specs/    要件を対話で詰める
/pj:design  HOW    docs/design/   技術設計（スタック/スキーマ/横断規約/ADR）に落とす
/pj:build   DOING  src/ + tests   受入条件を test に写してテスト先行で実装
/pj:change  保守   全レイヤー横断  「直したい」唯一の入口。層も product も意識せず使える
```

新規プロジェクトは **`/pj:setup` で立ち上げ → `/pj:spec` で WHAT を書き始める**。setup は中身を作らず、
足回り・pj-managed な CLAUDE.md・デザインシステム（**3スロット契約**: 実体／導線／正の宣言）を用意して引き渡す。
デザインシステムは特定ライブラリに依存せず、既存 DS（例: oyster-lib）でも preset でも同じ枠に収まる。

## 起動経路（コマンド／モデル起動）

`/pj:*` と打つ従来の起動はすべての skill で使える。加えて **`spec` / `design` / `change` の3つは
Claude 自身の判断でも起動できる**（`build` / `setup` は実コードを書く・依存を入れるためコマンド必須）。

モデル起動のときだけ、各 SKILL.md 冒頭の**「モデル起動時のゲート」**を通る:

| | 条件 |
|---|---|
| 前提 | ルート `CLAUDE.md` に `<!-- pj:managed start` が無ければ**起動せず即降りる**（pj 外の repo で誤爆しない） |
| 無確認で進む | `status` / `next` / `review` / `audit` — 成果物を編集しないモード |
| 承認が要る | 対話・軽量修正・`intake`・`glossary`・`change` の反映 — 成果物を書き換えるモード |

承認は「対象 product ／ 何を変えるか」を1文で提示して取る。断られたら何も書かずに降りる。

## 単位は product（app も、自分で作るライブラリも）

pj が回す単位は **product** — spec + design + build を1周持ち、**独立して作れる**もの。

```
1 リポジトリ = 実現したいこと 1 つ

  root product（app）      リポジトリのゴール本体。docs/ は repo 直下
    feature                 その product の中の「振る舞いの単位」（受入条件を持つ）
    ↓ install して使う
  packages/<name>           自分で作るライブラリ。これも product。app を知らない
    feature
```

**package は feature の兄弟ではなく app の兄弟。** 大きくなるプロジェクトで「ここはライブラリとして
切り出したい」と思ったら **`/pj:setup package <name>`** で器が生え、そこから同じパイプラインを回す。

肝は **依存（depend）と準拠（conform）の区別**:

- **依存** — `feature → package` は OK。**`package → app` は禁止**（import も spec 参照も app 固有語彙も）。
  package が自分の責務のドメイン語彙を持つのは構わない（判定は「別の app がそのまま使えるか」）。
  `package.json` の依存・import lint・pj audit の3枚で強制する。
- **準拠** — 全 product が root の `docs/design/conventions.md`（作法）と `design-language.md`（視覚の正）に従う。
  **これは依存ではない**（ESLint 設定に従うライブラリを「設定に依存している」とは言わない）。

判断に迷ったら「**別リポジトリに切り出したとき、そのまま動くか**」。定義は `_shared/concepts.md` §2。

## 開発者が覚えるのは1語だけ: `/pj:change`

使い続けるうちの修正は **`/pj:change <やりたいこと>`** に集約。開発者は spec/design/build の層を知らなくていい。
能動「○○を変えたい」と受動「もう code を直した・ズレてる」を**自動判別**し、正しいレイヤーで直して
トレースの鎖（受入条件↔test↔code / design↔実装）を伝播・整合させる。毎回「これは〈どの層〉の変更です」と
宣言するので、使ううちにモデルを受動的に学べる（覚える義務はない）。

## 整合性を保つ仕組み（肥大化してもブレない）

- **受入条件の安定 ID** `(<feature>-AC-NN)` を test が参照 → 鎖を grep で機械検証可能（採番は `/pj:change` が自動）。
  ID 規約（AC-NN / DL-NN）の唯一の定義は `_shared/concepts.md` §4。
- **外から来た設計成果物は取り込んで正にする**（`docs/design/intake/`・visual.md §V2）。
  Artifact のエクスポート・ハンドオフ・スクリーンショット・API 設計書を受け取り、
  **忠実度（fidelity）を宣言**してから中身を `AC-NN` / `DL-NN` に振り分ける。
  塩梅は忠実度が決める — `sketch`（スクショ）は構造だけ、`hifi`（数値つき設計書）は寸法まで拘束する。
  **出典は正ではない。振り分けて ID になったものだけが拘束する。** 取り込みは `/pj:design intake <パス>`、
  受け口は `/pj:setup` が作る `design-inbox/`。
- **ルールが向こうから来る**: `docs/<layer>/CLAUDE.md`（そのディレクトリ作業時に自動ロード）＋
  project root の pj 入口ポインタ（spec 初回起動時に1行追記）。規律を知らない人でも着地できる。
- **ドリフト検出は `audit` が担う**: 「pj を通さず code を直接編集した」取りこぼしは、鎖
  （受入条件↔test↔code / DL・UX↔実装）を実際に突合する **`drift-auditor`** が拾う。
  **hook による毎回の警告は持たない**（後述）。

## 共通の背骨

`_shared/concepts.md` が **5スキル共通の唯一の出典**（readiness 軸 / モード / 安定 ID 規約 §4 /
トレースの鎖 / 保守規律 / 2軸ダッシュボード / glossary / sub-agent 委譲 / audit の起動仕様 §14 / 禁止事項）。
各 SKILL.md は起動時にこれを読み、**同じ規則を二重に書かずここを参照する**。

## 共通モード（全スキル同じ意味）

`status`（盤面）/ `next`（次の一手）/ `review`（sub-agent 委譲で判定）/ `audit`（ドリフト監査）/
軽量修正 / 対話（既定）。次フェーズへは handoff 成果物を作らず、各 skill が spec/design を直読みする。

## sub-agent

- `readiness-reviewer` — 「次フェーズが追加対話なしに進めるか」を判定（spec/design 両対応）
- `terminologist` — 用語の一貫性を全レイヤー横断で監査
- `drift-auditor` — 受入条件↔test↔code / design↔実装 の乖離を検出（保守の要）

build のコード品質レビューは専用 agent を作らず既存の `/code-review` `/security-review` `/verify` に委譲する。

## 中心契約: 受入条件 = test = 完了

spec の `受入条件` を build で test に 1:1 で写し、**全受入条件テストが緑になって初めて実装完了**とみなす。
WHAT を詰めるほどテスト仕様が太る。

## 保守（一度作って直す）

変更は**正しいレイヤーから入れ、トレースの鎖を下へ伝播**させる。実装中に仕様/設計の誤りに気づいたら
**`/pj:change` に投げる**（層を選び分けない）。不安なときは `audit` で乖離を検出する（`_shared/concepts.md` §8）。

## インストール

```
/plugin marketplace add ~/.claude/local-plugins
/plugin install pj@yuki-local
```

## hooks — 現在は持たない

かつて `hooks/nudge.py`（PostToolUse・warning-only）が「code を pj を通さず編集した」ときに警告を出していたが、
**撤去した**。理由:

- hook に渡るのは編集パスと作業ディレクトリだけで、**`/pj:build` による正規の実装中かどうかを区別できない**。
  結果、正しく実装している最中にも毎回鳴り、**警告が読まれなくなる**（本当に拾いたい場面でも無視される）。
- 目的（ドリフト検出）は **`audit` の `drift-auditor`** がより確実に果たす。あちらは鎖
  （受入条件↔test↔code / DL-NN↔実装）を**実際に突合**するので、憶測でなく根拠つきで検出できる。

**取りこぼしが不安なときは `audit` を叩く**（`/pj:change` の受動入口も同じ監査を通る。concepts §14）。

> **将来ブロック強制を入れるなら**、受入条件↔test カバレッジ検査や `updated`/変更履歴ガードのように
> **機械的に真偽が決まるもの**に限る。「pj を通したか」のような**判定できない条件で警告を出さない**
> ——それが今回の撤去で得た教訓。
