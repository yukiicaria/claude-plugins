# pj — 仕様駆動パイプライン

1つのプロジェクトを **WHAT → HOW → コード** まで一貫して育てる Claude Code プラグイン。
5つのスキルが **同じ readiness 軸・共通モード・glossary・保守規律**を共有する。

```
/pj:setup   DAY 0  足回り + 器     git/lint/format/husky、pj-managed な CLAUDE.md、デザインシステム導入
                                   package <name> で「自分で作るライブラリ」の器を生やす
                                   stack で design の決定を実体化 / sync で docs と実体のズレを埋める
/pj:spec    WHAT   docs/specs/    要件を対話で詰める
/pj:design  HOW    docs/design/   技術設計（スタック/スキーマ/横断規約/ADR/デザイン言語）に落とす
/pj:build   DOING  src/ + tests   受入条件を test に写してテスト先行で実装
/pj:change  保守   全レイヤー横断  「直したい」唯一の入口。層も product も意識せず使える
```

新規プロジェクトは **`/pj:setup` で立ち上げ → `/pj:spec` で WHAT を書き始める**。setup は中身を作らず、
足回り・pj-managed な CLAUDE.md・デザインシステム（**3スロット契約**: 実体／導線／正の宣言）を用意して引き渡す。

## ドキュメントの構成（どこに何が書いてあるか）

| ファイル | 内容 | 読まれるとき |
|---|---|---|
| `_shared/concepts.md` | **5スキル共通の背骨**（product / 3レイヤー / 安定 ID §4 / readiness §5 / モード §7 / 保守 §8 / ダッシュボード §9 / glossary §10 / 3分類 §12 / audit §14 / 禁止 §15 / 高度 §16 / 運用宣言 §17） | **毎回** |
| `_shared/product-model.md` | package の実務（割る基準・可搬・強制の4枚・アダプタの判定） | package を作る/切り出す/監査するとき |
| `_shared/visual.md` | 視覚の正 §V1 と intake §V2 | UI を持つプロジェクト |
| `_shared/templates/managed-block.md` | ルート CLAUDE.md に焼く運用宣言の正準ブロック | stamp するとき |
| `skills/*/SKILL.md` | そのレイヤー固有の振る舞いだけ | そのスキルの起動時 |
| `agents/*.md` | 委譲先の判定基準（監査観点の正はここ） | sub-agent 起動時 |

**同じ規則を二重に書かない。** 各 SKILL.md は背骨を参照し、レイヤー固有のことだけを書く。

## 起動経路（コマンド／モデル起動）

`/pj:*` と打つ従来の起動はすべてのスキルで使える。加えて **`spec` / `design` / `change` / `setup` の4つは
Claude 自身の判断でも起動できる**（`build` はコマンド必須）。

モデル起動のときだけ、各 SKILL.md 冒頭の**「モデル起動時のゲート」**を通る:

| | 条件 |
|---|---|
| 前提 | ルート `CLAUDE.md` に `<!-- pj:managed start` が無ければ**起動せず即降りる**（pj 外の repo で誤爆しない） |
| 無確認で進む | `status` / `next` / `review` / `audit` — 成果物を編集しないモード ＋ **`/pj:setup package`**（器を生やすだけ） |
| 承認が要る | 対話・軽量修正・`intake`・`glossary`・`change` の反映 ＋ **`/pj:setup` の `stack` / `ds` / フル立ち上げ**（依存を install する） |

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

**package は feature の兄弟ではなく app の兄弟。**「ここはライブラリとして切り出したい」と思ったら
**`/pj:setup package <name>`** で器が生え、そこから同じパイプラインを回す。

肝は **依存（depend）と準拠（conform）の区別**:

- **依存** — `feature → package` は OK。**`package → app` は禁止**（import も spec 参照も app 固有語彙も）。
  package が自分の責務のドメイン語彙を持つのは構わない（判定は「別の app がそのまま使えるか」）。
- **準拠** — 全 product が root の `conventions.md`（作法）と `design-language.md`（視覚の正）に従う。
  **これは依存ではない**（ESLint 設定に従うライブラリを「設定に依存している」とは言わない）。

判断に迷ったら「**別リポジトリに切り出したとき、そのまま動くか**」。定義は `_shared/concepts.md` §2、
実務（割る基準・可搬・強制）は `_shared/product-model.md`。

## 開発者が覚えるのは1語だけ: `/pj:change`

使い続けるうちの修正は **`/pj:change <やりたいこと>`** に集約。能動「○○を変えたい」と受動「もう code を
直した・ズレてる」を**自動判別**し、正しいレイヤーで直してトレースの鎖（受入条件↔test↔code / design↔実装）を
伝播・整合させる。毎回「これは〈どの層〉の変更です」と宣言するので、使ううちにモデルを受動的に学べる。

## 整合性を保つ仕組み（肥大化してもブレない）

- **受入条件の安定 ID** `(<feature>-AC-NN)` を test が参照 → 鎖を grep で機械検証できる。
  ID 規約（AC-NN / DL-NN）の唯一の定義は `_shared/concepts.md` §4。
- **外から来た設計成果物は取り込んで正にする**（`docs/design/intake/`・visual.md §V2）。**忠実度（fidelity）を
  宣言**してから中身を `AC-NN` / `DL-NN` に振り分ける。塩梅は忠実度が決める — `sketch` は構造だけ、`hifi` は
  寸法まで拘束する。**出典は正ではない。振り分けて ID になったものだけが拘束する。**
- **ルールが向こうから来る**: ルート `CLAUDE.md` の **pj:managed ブロック**（毎セッション常時ロード）＋
  `docs/<layer>/CLAUDE.md`・`src/components/CLAUDE.md`（そのディレクトリ作業時に自動ロード）。
  規律を知らない人・新しいセッションでも必ず入口（`/pj:change`）に着地する。
- **ドリフト検出は `audit`**: 「pj を通さず code を直接編集した」取りこぼしは、鎖を実際に突合する
  **`drift-auditor`** が拾う（起動仕様は concepts §14、監査観点は agent 自身が持つ）。

## sub-agent

- `readiness-reviewer` — 「次フェーズが追加対話なしに進めるか」を判定（spec/design 両対応）
- `terminologist` — 用語の一貫性を全レイヤー横断で監査
- `drift-auditor` — 鎖の乖離と product 境界の違反を検出（保守の要）
- `design-reviewer` — UI が design-language と取り込んだ出典に従うかを監査

build のコード品質レビューは専用 agent を作らず既存の `/code-review` `/security-review` `/verify` に委譲する。

## 中心契約: 受入条件 = test = 完了

spec の `受入条件` を build で test に 1:1 で写し、**全受入条件テストが緑になって初めて実装完了**とみなす。
WHAT を詰めるほどテスト仕様が太る。

## インストール

```
/plugin marketplace add ~/.claude/local-plugins
/plugin install pj@yuki-local
```

## hooks — 現在は持たない

かつて `hooks/nudge.py`（PostToolUse・warning-only）が「code を pj を通さず編集した」ときに警告していたが、
**撤去した**。hook に渡るのは編集パスと作業ディレクトリだけで、**`/pj:build` による正規の実装中かどうかを
区別できない**——結果、正しく実装している最中にも毎回鳴り、**警告が読まれなくなる**。目的（ドリフト検出）は
`audit` の `drift-auditor` が鎖を実際に突合してより確実に果たす。

> **将来ブロック強制を入れるなら**、受入条件↔test カバレッジ検査や `updated`/変更履歴ガードのように
> **機械的に真偽が決まるもの**に限る。「pj を通したか」のような**判定できない条件で警告を出さない**
> ——それが今回の撤去で得た教訓。
