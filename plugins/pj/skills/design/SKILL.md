---
name: design
description: pj パイプラインの HOW レイヤー。spec（WHAT）を技術設計に落とす。スタック選定・具体データモデル・横断規約・ADR・デザイン言語（DL-NN）を対話で確定する。intake モードで外部の設計成果物（Artifact・ハンドオフ・スクショ）を取り込み、忠実度を宣言して AC / DL に振り分ける。status・next・review・audit を文脈で切り替える。既存物の修正は /pj:change が入口。
disable-model-invocation: true
argument-hint: "[status|next|review|audit|intake <パス>|<決定事項 or 論点>]"
---

spec を実装可能な技術設計に落とすコマンド。`docs/design/` をユーザーと協働で編集する。

引数: $ARGUMENTS

## 最初に必ず読む

共通の軸・モード・保守規律・禁止事項は **`../../_shared/concepts.md`**（pj パイプラインの背骨）。
起動したらまずそれを読む。design は spec の**下流**なので、起動時に **`docs/specs/` も読む**こと
（spec が WHAT の正。design はそれを変えない）。

**UI を持つプロジェクトなら `../../_shared/visual.md` も読む**（§V1 視覚の正 / §V2 intake）。
design-language と取り込みを扱うのはこの skill なので、実質いつも必要。CLI 等 UI 無しなら読まなくてよい。

## ゴール（design レイヤー）

**実装 Agent が追加の技術判断なしにコードを書けるレベル**まで HOW を確定する。「どの技術で・どんな
構造で作るか」が一意に決まっている状態がゴール。振る舞い（WHAT）を変えたくなったら、ここでこね回さず
**`/pj:change`** に投げる（正レイヤーの判定と伝播はそちらが引き受ける。concepts §8）。

## 対象 product を決める（起動時に必ず・concepts §2）

spec と同じ手順で**どの product の design の話か**を先に決める:
`docs/specs/overview.md` ＋ `packages/*/docs/specs/overview.md` ＋ `packages/*/*/docs/specs/overview.md` を glob して product を列挙し、
引数・文脈から対象を決め（既定は root product）、**一行宣言してから動く**。

## ファイル構造

> パスは**対象 product のルートからの相対**。★ の3つは **root product だけが持ち、全 product が準拠する**
> （concepts §3・§13）。package では**作らない**。

```
docs/design/
  stack.md          # 言語/FW/DB/ORM/UI/状態管理/テストランナー（＋採用理由）
  data-model.md     # spec の概念骨格 → 具体エンティティ/関連/不変条件
  adr/NNNN-*.md     # 大きな技術決定の記録（理由が辿れる）

  ★ conventions.md      # 作法。フォルダ構成・命名・横断方針・テスト方針・product 構成
  ★ design-language.md  # 視覚の正（UI を持つ場合）。デザイン原則/トークン/コンポーネント語彙＋DL-NN
  intake/               # 外から取り込んだ設計成果物（visual.md §V2）。**product ごと**
      README.md         #   索引
      <intake-id>/intake.md + 実体
```

**package product の design で扱うのは `stack.md` / `data-model.md` / `adr/` だけ。**
作法と視覚の正は root を**参照する**（所有しない）。package ごとに視覚の正を持つと DS が割れる。
UI を持つ package（`<lib>-ui` 等）も **root の design-language に準拠**し、コードに `@satisfies DL-NN` を付ける。

初回起動で対象 product の `docs/design/` が無ければ `../../_shared/templates/` から作る:

- **root product**: stack.md / data-model.md / conventions.md / design-language.md / adr.md / design-CLAUDE.md
  （design-CLAUDE.md は `docs/design/CLAUDE.md` へ。design-language.md は UI を持つ場合のみ。
  CLI 等 UI 無しは N/A 宣言で省略可）
- **package product**: stack.md / data-model.md / adr.md / design-CLAUDE.md のみ。
  `docs/design/CLAUDE.md` に「作法と視覚の正は root が正。ここでは所有しない」を1行書く。

> **必要になってから作る。** データを持たない package に data-model.md を先回りで置かない。

## この skill のキモ: spec から技術決定バックログを収穫する

起動時、`docs/specs/` 全体を読んで **HOW で決めるべき論点を自動で集める**:
- 各 feature の「実装裁量」「未定」「要検討」マーカー
- overview の非機能要件・制約・**横断論点**（例: 同時編集の競合制御のような全 feature の宿題）
- spec の概念骨格のうち、まだ具体スキーマに落ちていないもの

これを技術決定バックログとして提示し、優先度順に詰める。**横断項目は conventions.md で1回だけ決めて
全 feature に効かせる**（各実装にゆだねるとバラつく事故を防ぐ）。

**「全 package 共通の◯◯」は必ず root の `conventions.md` に置く**（concepts §13）。
package の design にも app の spec にも書かない。前者は他 package に効かず、後者はレイヤー違反かつ
`package → app` の向きを作る。**app の spec にこの手の作法を見つけたら、それは移設対象**として報告する。

## デザイン言語を spec から導出する（UI を持つプロジェクト）

視覚の HOW＝`design-language.md`（視覚の正・visual.md §V1）も design フェーズで決める。技術決定バックログの収穫と
対をなす作業:
- spec（利用者・データ量・頻度・デバイス・トーン・既存色資産）から **UX 意図を収穫**し、テンプレ内の
  **導出インタビュー**を 1〜2 問ずつ詰めて原則を確定、各決定に **DL-NN を自動採番**する。
- 「視覚階層・強調・色の意味論・コンポーネント語彙」は**横断決定として 1 回決めて全 feature に効かせる**
  （conventions.md「コンポーネント語彙」と接続。各画面の作り込みにゆだねるとブレる）。
- **greenfield**: コードより先に design-language を作る（実装 Agent が**追加の視覚判断なし**に画面を組める水準＝readiness）。
- **brownfield**: 既存 UI を `pj:design-reviewer` で棚卸しし、実態と理想の差分を踏まえて reconcile する。

## intake（外部の設計成果物を取り込む — `/pj:design intake <パス or URL> [<intake-id>]`）

外から来た設計成果物（Artifact のエクスポート・ハンドオフ・スクリーンショット・API 設計書）を
`<product>/docs/design/intake/` に取り込み、**中身を `AC-NN` / `DL-NN` に振り分ける**（方法論は visual.md §V2）。

**この skill は成果物を作らない。取り込んで振り分けるだけ。**
受け口は `design-inbox/`（`/pj:setup` が作る gitignore 済みの置き場）。

**まず `<product>/docs/design/intake/<intake-id>/` が既にあるかで経路が分かれる**:
- **無い → 下の「新規取り込み」**
- **ある → さらに下の「差し替え」**（作り直して当て直すケース。AC・DL・テスト・実装が既にある）

### 新規取り込み（`intake/<intake-id>/` が無い）

1. **対象 product を決める**（concepts §2）。**画面なら root、部品やライブラリならその package。**
   まだ product が無いなら `/pj:setup package <name>` を先に案内する（器を作ってから中身を入れる）。
2. **忠実度を宣言する**（visual.md §V2）。出典側の申告があれば尊重し、無ければ入力から判断する:
   スクショ = `sketch` / 動くプロトタイプ = `structure` / 数値つき設計書 = `hifi`。
   **迷ったら低いほうを宣言する**（守れない約束は高くつく）。
3. **実体を取り込む**: `docs/design/intake/<intake-id>/` に保存。**同梱アセットは剥がす**
   （DS バンドル・フォント・画像。目安 100KB / 件）。開いて読める状態を保つ。
4. **`intake.md` を作る**: `../../_shared/templates/intake.md` から。**出典・取り込み日・忠実度**を必ず残す。
   索引が無ければ `../../_shared/templates/intake-README.md` を `intake/README.md` として置き、1行追加する。
5. **正 / 出典 / 参考を仕分ける**（visual.md §V2）。同梱の参考実装・型は**参考**であって正ではない。
   **写経しない**——ルーティング・状態管理・命名は実装先の作法に合わせる。
6. **振り分ける（ここが仕事の本体）**: 出典の記述を1つずつ見て行き先を決める。判定は**「テストできるか」1つ**。
   - **テストできる**（寸法・閾値・時間・状態遷移）→ **AC**。その product の spec に落とす。
     **採番は `/pj:spec`（新規）か `/pj:change`（増減）**——ここでは振らず、どちらに送るかを告げる（concepts §4）
   - **横断の原則**（影を使わない・トークンの出どころ）→ **DL-NN** を採番して design-language に追記
   - **見比べれば分かるもの**（余白・配置・列順）→ **ID を振らない**。実体との突き合わせで足りる
   - `intake.md` の**振り分け表**に、出典の記述と行き先を1行ずつ書く
7. **拾わなかったものを残す**: `intake.md` に理由つきで書く。書かないと、次に読む人が
   「見落とし」か「意図的に外した」かを区別できない。
8. **WHAT の気づきは spec へ**: 出典を読んで振る舞いの誤り・不足に気づいたら、ここで辻褄を合わせず
   `/pj:change` に投げる（visual.md §V2 末尾）。反映先だけ `intake.md` に残す。

### 差し替え（`intake/<intake-id>/` が既にある）

実装を見て「これは違う」となり作り直したケース。**既定 L3**（HOW が変わる・WHAT は不変）。

0. **足場の確認**: 対象の **AC テストが緑か**確認する。赤いまま差し替えると、あとで赤くなったのが
   差し替えのせいか元からかを切り分けられない。
1. **現状を控える**: `intake.md` の振り分け表（どの AC / DL がこの出典から来たか）を控える。
   これが無いと何が消えたか分からない。
2. **取り込む**: 実体を差し替える（同梱アセットは剥がす）。旧版は git 履歴に残るので別名保存はしない。
   `intake.md` の「出典」に**新しい世代を1行追加**する（**旧行は消さない**）。
3. **3方向の差分を取る**: ①新旧の出典（何を変えたかったか）②新出典 vs 現実装（実装をどう変えるか）
   ③**新出典 vs 現 AC（仕様が壊れないか）**。③を飛ばさない。
4. **AC を1件ずつ照合する（最重要）**: 現 AC のうち、新出典が**明示的に否定している**ものだけを抽出する。
   **「描かれていないだけ」は否定ではない**（出典はハッピーパスのダミーデータで作られ、エラー状態・空状態・
   権限による非表示・バリデーションは描かれないのが普通）。否定と判断したものは**必ずユーザーに確認**し、
   合意できたら `/pj:change`（L2）で spec から廃止する。**この skill で AC を消さない。**
5. **忠実度を見直す**: 入力の質が変われば宣言も変わる（スクショ → プロトタイプなら `sketch` → `structure`）。
6. **DL との衝突を判定**: 新出典が design-language に反する見た目を持つなら、「**DL を変える**」のか
   「**出典を DS に寄せる**」のかを決める。DL を変えるなら `/pj:change` で採番し直す（黙って折衷しない）。
7. **実装へ伝播**: `/pj:build <feature>` か `/pj:change` で実装を合わせる。最後に **AC テストが緑のまま**
   であることを確認する——**赤くなったら振る舞いを変えてしまった証拠**なので手順4に戻る。
8. **記録**: `intake.md` の変更履歴・`updated` を更新。未決が開いたなら `design-language.md` の
   `progress` を**下げる**（concepts §5）。

## モード別の振る舞い（dispatch は concepts.md §7）

### 対話（既定）
1. `docs/specs/`（WHAT）と `docs/design/`（現在の HOW）を読む
2. サマリを出す:
```
[design 現状]
- スタック: <主要な確定/未定>
- データモデル: <エンティティ数 / 未確定の関連>
- 設計確定度: ■■■□□

[spec から拾った未決の技術論点]
1. 同時編集の競合制御（overextends 全 feature・未着手）
2. リッチエディタ選定（event のプログラム）
3. <...>

どれから決めます?
```
3. 決定したら該当ファイルに反映。大きな選定は adr/ に1件記録（理由付き）。
- エンティティ名・識別子は **glossary に従う**（spec と双方向トレース可能に）
- 1つの技術決定が複数 feature に効くなら、conventions.md に横断方針として書く

### 軽量修正
concepts.md §8 の手順（スコープ判定 → 該当ファイル更新 → 変更履歴 → updated → 影響1個提示）。
**デザイン原則（DL-NN）を増やす/廃止する修正はこのモードで処理しない**。`/pj:change` に渡す
（concepts.md §7 モード5 / §4）。既存 DL の言い回し調整など ID を動かさない修正はここで即時反映してよい。

### status / next
status は設計確定度＋未決の技術論点一覧。next は次に決めるとよい論点を 2〜3 個
（優先: 多数の feature に効く横断決定 / build の前提になる data-model / 未着手のコア）。

### review（readiness-reviewer に委譲）
`Agent` で **`pj:readiness-reviewer`** を起動（対象 = `docs/design/`＋`docs/specs/`、レイヤー = design）。
「実装 Agent が追加の技術判断なしにコードを書けるか」を判定させ、要点整理して伝える。編集しない。
UI/視覚の整合を見るときは加えて `Agent` で **`pj:design-reviewer`** を起動する
（対象 = `docs/design/design-language.md` ＋ **`docs/design/intake/`** ＋ `src/`）。違反 DL と、取り込みがある成果物は
**構成の乖離・振り分けの落下**を要点整理して伝える。

### audit（ドリフト監査）
**concepts.md §14「audit の起動仕様」に従う**（全 skill 共通の1つの監査。対象は**対象 product の
トライアド** `docs/specs/` ＋ `docs/design/` ＋ `src/` ＋ root の作法）。product が明示されなければ
**全 product を順に**見る。design から起動した場合は、報告の並び順だけ**design 決定 ↔ 実装/スキーマの
乖離を先頭**にする（ただし **product 境界の違反はさらにその前**・concepts §14 観点B）。
UI の DL 違反を深掘りするときは **`pj:design-reviewer`** を併用してよい。編集しない。

### build へ渡す
design が固まったら **`/pj:build <feature>` で受入条件を test に写してテスト先行で実装**、が標準導線
（受け渡しの作法は concepts §7）。大物 feature の事前 orientation が欲しいときは
`/pj:build plan <feature>`（コードを書かず計画だけ出す）を使う。

> **その前に `/pj:setup stack` が要ることがある。** **このスキルはコードを書かない**ので、
> `stack.md` で FW・DB・ORM・テストランナーを決めても**実体は生まれていない**。
> `/pj:build` は「動くプロジェクトがある」前提で受入条件を test に写すので、土台が無いと始まらない。
>
> **stack.md を書いた・更新したら、実体があるか確認して案内する:**
>
> ```
> stack.md の採用技術が package.json の依存に入っているか
> テストランナーが起動するか
> ```
>
> 入っていなければ **`/pj:setup stack`** を案内する（このスキルは入れない。決めるのが仕事）。
> **「決めたのに入っていない」状態を黙って build に渡さない。**

## 設計の確定度（格納先は concepts.md §5 の表）

**成果物ごとに frontmatter の `progress` を持つ**: `stack.md` / `data-model.md` ／ **root なら**
`conventions.md`・（UI があれば）`design-language.md`。評価軸は concepts.md §5（「実装 Agent が追加の
技術判断なしに書けるか」。UI があれば「追加の視覚判断なしに画面を組めるか」も含む）。

- **design 全体の確定度 = これらの `progress` の平均**（UI が無ければ design-language を除く。
  **package product は root 所有の conventions・design-language を平均に含めない**——自分で動かせない値を
  自分の確定度に混ぜると、他 product の都合で自分の数字が動く。concepts §5）。
  status はこの**保存値**から出す。**その場で推定し直さない**（都度推定だと値が揺れて後退に気づけない）。
- ファイルを直したら `progress` を**つけ直し**、`updated` を今日にし、変更履歴に1行残す。
  論点が再び開いたら遠慮なく**下げる**（§5）。
- **取り込んだ出典が AC / DL に振り分け終わるまで `design-language.md` の `progress` を 4-5 にしない**
  （実装 Agent が追加の視覚判断なしに組めないため）。
