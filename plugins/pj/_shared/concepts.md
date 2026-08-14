# pj パイプライン — 共通の背骨（concepts）

> **5スキル共通の唯一の出典。** 起動したらまずこれを読む。**ここは規範だけを置く**——根拠と実例は
> `README.md`「設計の根拠」、レイヤー固有の振る舞いは各 SKILL.md、場面の限られる詳細は下記にある。
>
> | 読むのは | ファイル |
> |---|---|
> | package を作る・切り出す・監査するとき | `_shared/product-model.md`（割る基準・可搬・強制） |
> | UI を持つプロジェクト | `_shared/visual.md`（視覚の正 §V1 / intake §V2） |
> | audit を起動したとき | `agents/drift-auditor.md`（監査観点の正） |
> | 変更の高度を判定するとき | `skills/change/SKILL.md`（L0〜L4 の実行手順） |
> | CLAUDE.md に運用宣言を焼くとき | `_shared/templates/managed-block.md`（正準ブロック） |
>
> **言語: 応答も成果物（md・コード内コメント）も日本語**。ユーザーが他言語を明示したときだけそれに従う。

## 1. pj とは

spec → design → build の3フェーズで **1つのプロジェクトを WHAT から コード まで一貫して育てる**
仕様駆動パイプライン。**その3つに直交して `/pj:setup` が居る。**

```
spec     何を作るか決める          docs/specs/
design   どう作るか決める          docs/design/
build    feature を動くコードに     src/ ＋ tests
─────────────────────────────────────────────
setup    決めたことを実体にする     package.json / 依存 / 設定 / 器
```

**setup はフェーズではなく、決定が起きるたびに呼ばれる**（`package` = 器を作る／`stack` = 決まった技術を
入れて配線する／`sync` = docs と実体のズレを埋める）。**中身は作らない。** **docs が先に進み実体が後から
追いつくのは正常**で、ズレたままにしないために `/pj:setup sync` がある。

**setup はモデル（Claude）からも起動できる**（承認はモード別。正は `skills/setup/SKILL.md` のゲート）。
**それでも次は義務のまま残る——気づいた者が黙って回避しない**:

- **`/pj:spec`** — package の docs を触ったら**器の有無を確認して報告**する。無ければ手で埋めず案内する
- **`/pj:design`** — `stack.md` を書いたら**実体の有無を確認して報告**し、`/pj:setup stack` を案内する
- **`/pj:build`** — 土台が無ければ**警告して setup に戻す**。自分で FW を入れ始めない

## 2. product と feature（単位の定義 — 唯一の定義）

| 語 | 定義 | 持つもの |
|---|---|---|
| **product** | spec + design + build を1周持つ単位。**独立して作れる** | 自分の `docs/specs/` `docs/design/` `src/` ＋ tests |
| **feature** | product の中の「振る舞いの単位」 | 受入条件 `(<feature>-AC-NN)`。test に 1:1 で写る |

```
root product（app）   リポジトリのゴールそのもの。docs/ は repo 直下
  features            業務機能・画面・ユースケース
  ↓ install して使う
packages/<name>       それを作るために自分で作るライブラリ。app を知らない
  features            その package の振る舞い
```

**app も package も product**で、feature はその中身。**package は feature の兄弟ではなく app の兄弟。**

**product の発見（マニフェストを持たない）**: **`docs/specs/overview.md` を持つディレクトリが product。**
探索は `docs/specs/overview.md` ＋ `packages/*/docs/specs/overview.md` ＋
`packages/*/*/docs/specs/overview.md` の3箇所（**ファイルの存在が唯一の事実**）。

- **`packages/` 直下はフラットが既定。** 並びを揃えたいだけなら**ディレクトリ名の接頭辞**で束ねる
  （`adapter-slack/` `adapter-line/`）。**先回りしてグループを作らない。**
- 数が増えたら**グループ用の中間ディレクトリを1段だけ**挟んでよい（`packages/<group>/<name>/`）。
  **グループ自身は product ではない**（overview.md を持たない）。**深さは `packages/` 配下2段まで。**
- **product の中に product を置かない**（`packages/a/packages/b`）。

**依存（depend）と準拠（conform）は別物** —— product モデルの要:

| | 意味 | 許される向き |
|---|---|---|
| **依存** | 相手のコード・型・spec・語彙を参照する | `feature → package` は OK ／ **`package → app` は禁止** ／ package 間は一方向のみ・循環禁止 |
| **準拠** | repo の建て方のルール（作法）に従う | 全 product → 作法。**これは依存ではない**（§13） |

**「独立して作れる」は2つで測る。両方を満たして初めて独立している**:

> **1. 語彙** — 別の app がこの package をそのまま使えるか。**人が判断する**（terminologist・§14）。
> **package は自分の責務のドメイン語彙を持ってよい。禁じられるのは app 固有の語**（その app の制度・
> 他 feature の概念に依存した語）。
> **2. 可搬** — 別リポジトリにコピーして `bun install` し、型検査とテストが通るか。**実行して確かめる。**
> **最初から満たす**（hoisting が不足を隠すので、壊れていても静かに壊れる）。

> **チェックリスト・分割の基準・強制の4枚は `_shared/product-model.md`。**

## 3. 1つの真実の3レイヤー（トレース可能）

**product ごとに `docs/` を持ち、それがその product の単一の真実。下は上から導出され、必ず上へ辿れる。**

> **以下すべてのパスは「対象 product のルートからの相対」**（root なら repo 直下、package なら
> `packages/<name>/` 直下）。**pj のどの文書もパスを絶対で書かない。**

| レイヤー | 問い | スキル | 成果物 |
|---|---|---|---|
| **spec**   | 何を・どう振る舞うか | `/pj:spec`   | `docs/specs/`（overview / features / glossary） |
| **design** | どの技術で・どんな構造で | `/pj:design` | `docs/design/`（stack / data-model / adr / intake ／ **root のみ**: conventions・design-language） |
| **build**  | 実際に作る（テスト先行） | `/pj:build`  | `src/` + テスト（状態は自 product の overview ダッシュボードに反映） |

**root product だけが持つもの**（全 product が**準拠**する）: `docs/design/conventions.md`（作法）・
`docs/design/design-language.md`（視覚の正・DL-NN）。**package は参照するが所有しない。**
**`docs/design/intake/` は product ごと**（visual.md §V2）。

**トレースの鎖（保守の基盤）**: `受入条件 ↔ test ↔ code` ／ `design 決定 ↔ コードモジュール / スキーマ` ／
`DL-NN ↔ コンポーネント`。各成果物は `[[リンク]]` と file:line で互いを指し、**鎖が切れた状態を
「ドリフト」と呼ぶ**（§8）。鎖は**安定 ID** で機械検証可能にする（形式・採番者・不変ルールは §4 が唯一の定義）。

## 4. 安定 ID 規約（唯一の定義 — 他所はここを参照する）

| ID | 何に振るか | 置き場 | 参照する側 | 採番者 |
|---|---|---|---|---|
| `(<feature>-AC-NN)` | 受入条件 | 対象 product の `docs/specs/features/<feature>.md`（spec が1ファイルのうちは `overview.md`・§6） | test 名 / コメント | **新規は `/pj:spec`・増減は `/pj:change`** |
| `(DL-NN)` | デザイン原則・トークン・コンポーネント語彙（**横断**） | **root product の** `docs/design/design-language.md` | コンポーネント先頭の `@satisfies DL-NN`（**どの product のコードでも可**） | **`/pj:design`・`/pj:change`** |

**全 ID 共通の不変ルール**:

- **一度振った ID は変えない。** 廃止したら**欠番**にし、**再利用しない**（欠番はその旨と理由を1行残す）。
- **NN は2桁ゼロ埋めの連番**。次番は既存の最大値 +1（`design-language.md` は末尾に**採番台帳**を持つ）。
- **採番は上表の採番者だけが行う。** 他スキルの軽量修正モードで ID を増減しない（§7 モード5）。
- **受入条件だけ採番者が分かれる。** 判定は「**この feature に ID 付きの受入条件が1つでもあるか**」——
  **無ければ `/pj:spec`**（書いた直後に ID 未採番のドラフトを残さない）、**あれば `/pj:change`**。
- 鎖の検証は**どの ID も同じ形**: 「全 ID に参照側が存在するか」を grep で突合する（未参照 = 未実装）。
- **ID を1つ増やしたら、この表に1行足すだけで済むようにする。** 他所は種類を列挙せず「§4 の安定 ID」と書く。
- **feature 名は product をまたいで repo 内で一意にする**（AC ID に product 名を含めないため。衝突は audit が検出）。

## 5. readiness 軸（0〜5）— 全レイヤー共通

**唯一の評価基準**: 「**この成果物を次フェーズにそのまま渡したら、追加対話なしに進めるか**」。
項目数や文章量では測らない。**残っている意思決定の量**で測る。

| レイヤー | readiness が測るもの | 単位 | 格納先（frontmatter） |
|---|---|---|---|
| spec | 実装/設計 Agent が**追加の設計対話なし**に着手できるか。**package なら「app を知らずに作れるか」も含む**（§2） | feature | `docs/specs/features/<name>.md` の `progress` |
| design | 実装 Agent が**追加の技術判断なし**にコードを書けるか。**UI があれば「追加の視覚判断なし」に画面を組めるかも含む** | 成果物ごと | `stack.md` / `data-model.md` /（root のみ）`conventions.md`・`design-language.md` の `progress` |
| build | その feature の**全受入条件のテストが緑**か | feature | `docs/specs/features/<name>.md` の `build_progress` |

| 値 | バー | 意味 |
|---|---|---|
| 0 | □□□□□ | 何もない（ファイルだけ） |
| 1 | ■□□□□ | 概要だけ |
| 2 | ■■□□□ | 主要な方向性が見えてきた |
| 3 | ■■■□□ | 主要フロー・構造は固まった。各部の中身／エッジは未定（＝まだ対話が要る） |
| 4 | ■■■■□ | 細部・エラーケースまで概ね詰まり、対話はほぼ不要 |
| 5 | ■■■■■ | 追加対話ゼロで次フェーズにそのまま渡せる |

**運用ルール（全レイヤー厳守）**:

- **4-5 は「追加対話が不要」と言い切れるときだけ。** これから「詰める／深掘りする」と話している成果物は
  最大 3。**枠・構造が決まっただけでは 3 どまり。**
- **ラチェットではない。毎回つけ直す。** 論点が増えた・前提が覆ったら**遠慮なく下げる**（後退は変更履歴に残す）。
- **必ず frontmatter に保存し、表示はその保存値から出す**（都度推定し直すと値が揺れて後退に気づけない）。
- **product ごとに閉じて測る。** 他 product の未決を持ち込まない。
- **design 全体の確定度 = 上表の各ファイルの `progress` の平均**（UI が無ければ design-language を除く。
  **package は root 所有の conventions・design-language を含めない**）。
- バー表記（■と□）を正とする。数値は補助。

## 6. spec の深さは可変（product の大小に合わせる）

```
段階1   docs/specs/overview.md だけ           受入条件も overview に直接書く（AC ID は振る）
段階2   docs/specs/features/<name>.md へ割る   feature が複数見えてきたら切り出す
```

**全 product に `features/` を強制しない。段階1でも AC ID は必ず振り**（鎖は器の形ではなく ID で成立する）、
**切り出しても ID は変えない**（移動するだけ）。判断基準は「1ファイルが読みにくくなったか」。
**先回りして割らない。** root product も同じ。

## 7. 共通モード（全スキル同じ意味）

引数と文脈から **1つの振る舞いを選ぶ**。判断順は上から:

1. **status** — 「今どんな感じ?」「進捗」系、または `status` → 盤面表示
2. **next** — 「次どこ?」「何から?」系、または `next` → 次の一手提案
3. **review** — 「これでいい?」「見て」系、または `review` → sub-agent 委譲で客観判定（§14）
4. **audit** — 「ズレてない?」「乖離」系、または `audit` → ドリフト監査（§8・§14）
5. **軽量修正** — 既存成果物への一点修正のうち、**§4 の安定 ID を増減しないもの** → 即時反映（§8）。
   **ID の増減を伴うものは `/pj:change` に送る。ID の種類をここで列挙しない**（正は §4 の表）
6. **対話（既定）** — 上のどれでもない → サマリ → 会話で詰める

**共通の作法**:

- **1〜4 は読むだけ。成果物を編集しない。**
- 編集する系（5・6・`/pj:change` の反映）は**毎ターン応答前に関連ファイルを読み直す**
- 編集 **diff は見せない**。「○○.md を更新しました（要点1行）」程度に留める
- 気付いた論点は溜めず**その場で 1〜2 個ずつ**出す。終わり際にまとめて出さない
- 次フェーズへ**専用の受け渡し成果物（handoff）を作らない**。各スキルが spec/design を直読みする

**起動経路**: `/pj:*` と打つ起動に加え、**spec / design / change / setup の4つはモデル（Claude）判断でも
起動できる**（`build` はコマンド必須）。**前提チェックと承認の取り方の正は各 SKILL.md 冒頭の
「モデル起動時のゲート」**——この文書を読むより前に通る節なので、ここには置かない。

## 8. メンテナンス規律（パイプラインの第一級市民）

原則: **変更は正しいレイヤーから入れ、トレースの鎖を下へ伝播させる。**
**開発者の入口は `/pj:change` 一本**（能動「○○を変えたい」／受動「もう code を直した・ズレてる」を自動判別）。

**典型シナリオ**:

1. **実装中に仕様の誤りに気づく** → build で辻褄を合わせず **`/pj:change <気づいたこと>`** に投げる。
2. **仕様変更要求** → spec→design→build を**増分**で流す（greenfield ではなく差分として扱う）。
3. **定期 or 不安なとき** → `audit` で **drift-auditor** が鎖の食い違いを検出・報告する（§14）。

**軽量修正モードの手順（全レイヤー共通）**:

0. **スコープ判定**: **§4 の安定 ID を増やす/廃止する**なら、ここで処理せず「ID の増減を伴うので
   `/pj:change` で入れます」と告げて渡す（§7 モード5）。
1. 該当ファイルを特定（不明なら一言で確認）→ 2. 該当箇所を更新
3. ファイル末尾の `## 変更履歴` に `YYYY-MM-DD: <変更内容>` を追記 → 4. frontmatter の `updated` を今日に
5. 影響しそうな**下位レイヤー / 関連成果物を 1 個だけ**提示（push しすぎない）
6. 変更で論点が再び開いたら readiness を**下げる**（§5）

**不変条件**: すべての成果物は `updated` と `## 変更履歴` を持つ。前提を覆して書き直したら
「○○の前提を破棄し△△に作り直し」と履歴に残し、古い前提を参照していた**他の記述も追って直す**。

## 9. ダッシュボード（2段 × 2軸）

**下段（正）= 各 product の `docs/specs/overview.md`** —— その product の **feature × 2軸**。**ここが唯一の正。**
**上段（導出）= root の `docs/specs/overview.md` の product 表** —— 全 product の一覧。
**値は下段からの roll-up であって独立した事実ではない。**

| 下段（feature 表） | spec | build | 残課題 |   | 上段（product 表） | 種別 | spec | build | 依存 |
|---|---|---|---|---|---|---|---|---|---|
| contacts | ■■■■□ | ■■□□□ | 名寄せ候補の粒度 |   | （repo 名） | app | ■■□□□ | □□□□□ | workflow |
| event | ■■■■□ | □□□□□ | 未着手 |   | workflow | package | ■■■■□ | □□□□□ | temporal |

- **spec 軸は `/pj:spec`、build 軸は `/pj:build` が更新する**（design の確定度は各 product の `docs/design/` 側）。
- feature を更新するたびに**下段を更新し、続けて上段の該当行を引き直す**（**上段だけを手で動かさない**）。
- overview の「コア機能」「product 一覧」は**何があるか**だけを持ち、**完了状態は持たない**。
- 上段の「依存」列は §2 の依存グラフ。**`package → app` の行が現れたら違反**（audit が拾う・§14）。

## 10. glossary — product ごとの用語辞書（全レイヤー共通）

**`docs/specs/glossary.md` がその product の用語の正。** spec の本文だけでなく、**design のスキーマ名・
コードの識別子も同じ語に従う**。監査は terminologist（§14）。

- **root** = その app 固有の語（制度・業務の運用に固有のもの）／**package** = その package の責務の語彙
- **package が自分の glossary に定義していない概念語を spec 本文で使っていたら要調査**——root の glossary から
  借りているなら app の語彙を使っている疑いが濃い。**自分の責務の語なら定義すればよい**（§2）。
- 同じ語が複数 product に現れること自体は問題ない（各 product 内で一意なら可）。
  ただし**意味が違う同語**は事故のもとなので terminologist が指摘する。

## 11. 視覚と体験の正 → `visual.md`（UI を持つプロジェクトのみ）

UI の正は **`_shared/visual.md`**（**§V1** design-language ＝ 視覚の正と DS の3スロット契約 ／ **§V2** intake
＝ 外部の設計成果物の取り込み）。**読むのは UI に触れるスキルだけ**——`/pj:setup`（DS 導入）・`/pj:design`・
`/pj:build`（UI 実装時）・`/pj:change`（UI に触れる変更時）。**`/pj:spec` は読まない。**
CLI 等 UI の無いプロジェクトでは誰も読まない。ID 規約（DL-NN）の正は §4。

## 12. 成果物の3分類と置き場（product の中の話）

**§2 が「どの product か」を決め、ここが「その product の中のどこか」を決める。**

| 分類 | 何を決めるか | 正のレイヤー | ID | 判定 |
|---|---|---|---|---|
| **挙動** | ルール・状態・不変条件・数値。**見た目なしでテストが書ける** | spec | `AC-NN` | テストできるか |
| **見た目** | トークン・原則・部品の語彙 | design | `DL-NN` | 全画面に効くか |
| **まとめる** | 配線・注入・ページ組み立て | design（作法） | **無し** | 上2つを繋ぐだけか |

- **判定は「テストできるか」1つ。** 数値だから見た目、ではない（「行高 39px」は検証できるので挙動、
  「影を使わない」は全画面に効くので見た目）。**アダプタの当て方は product-model.md §P4。**
- **新しい判断が生まれたら、それは挙動か見た目のどちらかに属するはず**——「まとめる」に ID が無いのが検査になる。

**置き場は conventions が持ち、build は引くだけ**（具体的なディレクトリの表は root の `conventions.md`・§13。
pj はディレクトリ名を規定しない）。規定するのは3つ:

1. **conventions に「配置の表」が必ずある**（3分類それぞれの置き場が書かれている）
2. **build は表を引くだけ。新しいトップレベルの箱を作らない。** 表に無いものが要ると気づいたら、
   その場で作らず **`/pj:change`** に投げる
3. **package の中身も同じ分類で組む**（検証は「別リポジトリに install してそのまま動くか」・§2）

## 13. 作法（conventions）— repo 全体にかかるルール

**作法は product ではなく repo 全体にかかるルール**（§2 の「準拠」）。置き場は
**root product の `docs/design/conventions.md` ただ1つ**。全 product がこれに従う。

入るもの（例）: 時間の扱い・永続と注入の作法・スキーマとマイグレーションの流し方・トランザクションの
握り方・エラーの表し方・テストの方針・配置の表（§12）。

- **package が従うルールを app の spec に書かない**（レイヤー違反であり、`package → app` の向きも作る）。
  **「全 package 共通の◯◯」と書きたくなったら conventions.md 行きのサイン。**
- package を切り出すときは、baked-in の設計判断としてその package の `docs/design/` に写して連れて行く。

## 14. sub-agent 委譲（トークン重い読み込みはメインを汚さない）

| agent | 役割 | 呼ばれるモード |
|---|---|---|
| `pj:readiness-reviewer` | 「次フェーズが追加対話なしに進める水準か」を批判的に判定（spec/design 両対応） | review |
| `pj:terminologist` | 用語・命名の一貫性を全レイヤーで監査し canonical 案を返す | spec の用語整理 / audit |
| `pj:drift-auditor` | 鎖の切れ目＋**product 境界の違反**を検出 | audit |
| `pj:design-reviewer` | UI が design-language と取り込んだ出典に従うかを DL-NN・file:line つきで監査 | review / audit（UI 対象時） |

**対象（docs パス / 特定 feature）とレイヤー・product 種別**を渡して起動し、返った結果は**丸投げせず要点を
整理**して伝える（重大な指摘を先頭に）。review/audit 自体では**編集しない**。build のコード品質レビューは
専用 agent を作らず既存の `/code-review` `/security-review` `/verify` に委譲する。

**audit の起動仕様（全スキル共通・ここが唯一の定義）** —— ドリフトは**レイヤーをまたいで**起きるので、
`/pj:spec audit` `/pj:design audit` `/pj:build audit` `/pj:change`（受動）は**どれも同じ監査**を起動する:

- agent は **`pj:drift-auditor`**（**監査観点の正はその agent が持つ**）。
- 対象は**対象 product のトライアド**（`docs/specs/` ＋ `docs/design/` ＋ `src/`）＋ root の
  `conventions.md`・`design-language.md`。**呼び出し元の層に関わらず常にこの3点セット。**
- **product が明示されなければ全 product を順に。** 特定 feature に絞るときも対象はトライアドのまま、
  **スコープとして feature 名を渡す**。
- 報告は**product 境界の違反を先頭**に（その次の並び順だけ呼び出し元の層に応じて変えてよい）。
  **編集しない**（直すのは `/pj:change`）。UI の深掘りが要るときだけ `pj:design-reviewer` を併用する。

## 15. やってはいけないこと（全レイヤー共通）

- 決まったテンプレ質問を機械的に上から順に投げる（文脈に応じて必要な質問だけ出す）
- ユーザーが「保留」と言った論点を蒸し返す（`open_questions` に積んで先に進む）
- 編集 diff を毎回見せる／気付いた論点を溜めて終わり際にまとめて出す（§7）
- status / next / review / audit モードで成果物を編集する（§7）
- レイヤーをまたいで辻褄合わせをする（誤りは正しいレイヤーに戻して直す・§8）
- 過去に決めた前提に固執し、覆すべき場面で小さな差分修正に逃げる
- 見栄えのために readiness を高いまま維持する（§5）
- **package から app / 他 feature を参照する**（import・`[[リンク]]`・spec からの言及・§2）
- **package の spec / glossary に app 固有の語彙を持ち込む**（自分の責務の語彙は違反ではない・§10）
- **「全 package 共通の◯◯」を app の spec に書く**（§13）
- **上段ダッシュボードを手で動かす**（§9）／**先回りして `features/` に割る**（§6）
- **package を切り出しやすくするためだけに AC ID を振り直す**（§4）

## 16. 変更の高度（トリアージの語彙 — 手順の正は `/pj:change`）

| 高度 | 何が変わるか | 例 | 入口レイヤー / 伝播 | docs |
|---|---|---|---|---|
| **L0** 局所/コスメ | 振る舞いも設計も不変 | ボタンの色・余白・文言・無害なリファクタ | code 直接 → test 緑 | 触らない |
| **L1** 実装バグ | 受入条件は正・code が違う | 仕様どおり動いていない | build: test↔code を辿って修正 | 触らない |
| **L2** 振る舞い変更 | WHAT が変わる | ルール追加・項目追加・フロー変更 | **spec → design 影響確認 → build** | specs（＋波及あれば design） |
| **L3** 技術/構造変更 | WHAT 不変・HOW が変わる | 実装方式・スキーマ・ライブラリ変更 | **design → build** | design |
| **L4** 根本 | spec も design も | コンセプト転換 | **spec → design → build を増分で** | 両方 |

- **L0/L1 は即実行。L2〜L4 は「どの層をどう変え・どこまで伝播するか」を一行確認してから動く。**
- **迷ったら高度を1つ上げて確認する。** **L0 コスメであっても design-language（DL-NN）に準拠する。**
- 判定手順・product の特定・採番の実務は **`skills/change/SKILL.md`** が持つ。

## 17. プロジェクトへの運用宣言（managed block）

pj で運用するプロジェクトは、**ルート `CLAUDE.md`（毎セッション常時ロード）に運用宣言を持つ**。
**正準のブロック本文は `_shared/templates/managed-block.md`**（stamp するときだけ読む）。

**stamp 規律（冪等・全スキル共通）**:

- **編集系フロー**（spec の init / 対話・軽量修正、`/pj:change` の反映、`/pj:setup`）に入ったら、ルート
  `CLAUDE.md` に `<!-- pj:managed start -->`〜`end` が**無ければ stamp** する（末尾近く or Overview 直後）。
- 既にマーカーがあれば**重複させない**。正準ブロックが更新されたら**マーカー区間だけ差し替え**
  （マーカー外の手書きルールは触らない）。
- **read-only モード（status / next / review / audit）では stamp しない**（§7）。
- プロジェクト固有の運用注記は**マーカー外**に書く（次回の再生成で消えない）。
