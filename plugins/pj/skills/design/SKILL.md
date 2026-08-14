---
name: design
description: pj パイプラインの HOW レイヤー。spec（WHAT）を技術設計に落とす。スタック選定・具体データモデル・横断規約・ADR・デザイン言語（DL-NN）を対話で確定する。intake モードで外部の設計成果物（Artifact・ハンドオフ・スクショ）を取り込み、忠実度を宣言して AC / DL に振り分ける。status・next・review・audit を文脈で切り替える。既存物の修正は /pj:change が入口。pj 運用宣言（CLAUDE.md の pj:managed ブロック）を持つ repo でのみ使う。
argument-hint: "[status|next|review|audit|intake <パス>|<決定事項 or 論点>]"
---

spec を実装可能な技術設計に落とすコマンド。対象 product の `docs/design/` をユーザーと協働で編集する。

引数: $ARGUMENTS

## モデル起動時のゲート（`/pj:design` と打たれた起動では読み飛ばす）

ユーザーが明示的に `/pj:design ...` と入力したなら、この節は**丸ごと無視**する。**モデル（Claude）の判断で
起動したときだけ**上から順に通る。**`concepts.md` / `visual.md` を読むのはゲート通過後。**

1. **pj 運用の repo か確かめる** — ルート `CLAUDE.md` に `<!-- pj:managed start` があるか grep する。
   **無ければ即座に降りる**（何も読まず書かず、「この repo は pj 運用ではないので通常どおり進めます」と一行）。
2. **status / next / review / audit なら承認は要らない**（編集しないモード）。そのまま進む。
3. **編集系（決定事項の対話・軽量修正・intake）なら AskUserQuestion で一行確認** —
   「**対象 product** ／ 触るレイヤー（design）／ **何を決める・取り込むか**」を1文で。
4. **承認が得られなければ何も書かずに降りる。** 代案を押し売りしない。

## 最初に必ず読む

- **`../../_shared/concepts.md`**（背骨）。
- **対象 product の `docs/specs/`** — design は spec の**下流**（spec が WHAT の正。design はそれを変えない）。
- **UI を持つなら `../../_shared/visual.md`**（§V1 視覚の正 / §V2 intake）。design-language と取り込みを
  扱うのはこのスキルなので実質いつも必要。CLI 等 UI 無しなら不要。

## ゴール（design レイヤー）

**実装 Agent が追加の技術判断なしにコードを書けるレベル**まで HOW を確定する。「どの技術で・どんな構造で
作るか」が一意に決まっている状態がゴール。**振る舞い（WHAT）を変えたくなったらここでこね回さず `/pj:change`**
（正レイヤーの判定と伝播はそちらが引き受ける・concepts §8）。

## 対象 product を決める（起動時に必ず）

**concepts §2 の3箇所を glob して product を列挙し、対象を決めて一行宣言してから動く**（既定は root product）。

## ファイル構造

> パスは**対象 product のルートからの相対**。★ の2つは **root product だけが持ち、全 product が準拠する**
> （concepts §3・§13）。package では**作らない**。

```
docs/design/
  stack.md          # 言語/FW/DB/ORM/UI/状態管理/テストランナー（＋採用理由）
  data-model.md     # spec の概念骨格 → 具体エンティティ/関連/不変条件
  adr/NNNN-*.md     # 大きな技術決定の記録（理由が辿れる）

  ★ conventions.md      # 作法。フォルダ構成・命名・横断方針・テスト方針・配置の表（concepts §12）
  ★ design-language.md  # 視覚の正（UI を持つ場合）。デザイン原則/トークン/コンポーネント語彙＋DL-NN
  intake/               # 外から取り込んだ設計成果物（visual.md §V2）。**product ごと**
      README.md         #   索引
      <intake-id>/intake.md + 実体
```

**package product の design で扱うのは `stack.md` / `data-model.md` / `adr/` だけ。** 作法と視覚の正は root を
**参照する**（所有しない）。UI を持つ package も **root の design-language に準拠**し、コードに
`@satisfies DL-NN` を付ける。

初回起動で対象 product の `docs/design/` が無ければ `../../_shared/templates/` から作る:

- **root**: stack.md / data-model.md / conventions.md / design-language.md / adr.md / design-CLAUDE.md
  （design-CLAUDE.md は `docs/design/CLAUDE.md` へ。design-language.md は UI を持つ場合のみ）
- **package**: stack.md / data-model.md / adr.md / design-CLAUDE.md のみ。`docs/design/CLAUDE.md` に
  「作法と視覚の正は root が正。ここでは所有しない」を1行書く。

> **必要になってから作る。** データを持たない package に data-model.md を先回りで置かない。

## この skill のキモ: spec から技術決定バックログを収穫する

起動時、`docs/specs/` 全体を読んで **HOW で決めるべき論点を自動で集める**:

- 各 feature の「実装裁量」「未定」「要検討」マーカー
- overview の非機能要件・制約・**横断論点**（例: 同時編集の競合制御のような全 feature の宿題）
- spec の概念骨格のうち、まだ具体スキーマに落ちていないもの

これを技術決定バックログとして提示し、優先度順に詰める。**横断項目は conventions.md で1回だけ決めて全 feature に
効かせる**（各実装にゆだねるとバラつく）。**「全 package 共通の◯◯」は必ず root の `conventions.md`**——
package の design にも app の spec にも書かない。**app の spec にこの手の作法を見つけたら移設対象として報告する。**

## デザイン言語を spec から導出する（UI を持つプロジェクト）

視覚の HOW＝`design-language.md`（visual.md §V1）も design フェーズで決める。技術決定バックログと対をなす作業:

- spec（利用者・データ量・頻度・デバイス・トーン・既存色資産）から **UX 意図を収穫**し、テンプレ内の
  **導出インタビュー**を 1〜2 問ずつ詰めて原則を確定、各決定に **DL-NN を採番**する。
- 「視覚階層・強調・色の意味論・コンポーネント語彙」は**横断決定として1回決めて全 feature に効かせる**。
- **greenfield**: コードより先に design-language を作る。
  **brownfield**: 既存 UI を `pj:design-reviewer` で棚卸しし、実態と理想の差分を踏まえて reconcile する。

## intake（外部の設計成果物を取り込む — `/pj:design intake <パス or URL> [<intake-id>]`）

外から来た設計成果物を `<product>/docs/design/intake/` に取り込み、**中身を `AC-NN` / `DL-NN` に振り分ける**。
**方法論の正は visual.md §V2**（3つの立場・忠実度・置き場・振り分けの判定・未描画≠否定）。
**このスキルは成果物を作らない。取り込んで振り分けるだけ。** 受け口は `design-inbox/`（`/pj:setup` が作る）。

**`<product>/docs/design/intake/<intake-id>/` が既にあるかで経路が分かれる。**

### 新規取り込み（`intake/<intake-id>/` が無い）

1. **対象 product を決める。** **画面なら root、部品やライブラリならその package。** product がまだ無いなら
   `/pj:setup package <name>` を先に案内する（器を作ってから中身を入れる）。
2. **忠実度を宣言する**（visual.md §V2）。出典側の申告を尊重し、無ければ入力から判断する:
   スクショ = `sketch` / 動くプロトタイプ = `structure` / 数値つき設計書 = `hifi`。**迷ったら低いほうを宣言する。**
3. **実体を取り込む**: `intake/<intake-id>/` に保存。**同梱アセットは剥がす**（目安 100KB / 件）。
4. **`intake.md` を作る**: `../../_shared/templates/intake.md` から。**出典・取り込み日・忠実度**を必ず残す。
   索引が無ければ `../../_shared/templates/intake-README.md` を `intake/README.md` として置き、1行追加する。
5. **正 / 出典 / 参考を仕分ける**（visual.md §V2）。**参考実装は写経しない**——ルーティング・状態管理・命名は
   実装先の作法に合わせる。
6. **振り分ける（仕事の本体）**: 出典の記述を1つずつ見て行き先を決める。**判定は「テストできるか」1つ**:
   テストできる（寸法・閾値・時間・状態遷移）→ **AC**（**採番は `/pj:spec` か `/pj:change`**。ここでは振らず、
   どちらに送るかを告げる・concepts §4）／横断の原則 → **DL-NN** を採番して design-language へ／
   見比べれば分かるもの（余白・配置・列順）→ **ID を振らない**。`intake.md` の**振り分け表**に1行ずつ書く。
7. **拾わなかったものを理由つきで残す**（書かないと次に読む人が「見落とし」と「意図的に外した」を区別できない）。
8. **WHAT の気づきは spec へ**: 振る舞いの誤り・不足に気づいたらここで辻褄を合わせず `/pj:change` に投げ、
   反映先だけ `intake.md` に残す。

### 差し替え（`intake/<intake-id>/` が既にある）

実装を見て「これは違う」となり作り直したケース。**既定 L3**（HOW が変わる・WHAT は不変・concepts §16）。
**既に AC・DL・テスト・実装がある状態への上書き**なので、順序が安全装置になっている:

0. **対象の AC テストが緑か確認する**（赤いまま差し替えると、後で赤くなった原因を切り分けられない）。
1. **`intake.md` の振り分け表を控える**（どの AC / DL がこの出典から来たか。無いと何が消えたか分からない）。
2. **実体を差し替える**（同梱アセットは剥がす）。旧版は git 履歴に残るので別名保存しない。`intake.md` の
   「出典」に**新しい世代を1行追加**する（**旧行は消さない**）。
3. **3方向の差分を取る**: ①新旧の出典 ②新出典 vs 現実装 ③**新出典 vs 現 AC**。**③を飛ばさない。**
4. **AC を1件ずつ照合する（最重要）**: 現 AC のうち新出典が**明示的に否定している**ものだけを抽出する。
   **「描かれていないだけ」は否定ではない**（visual.md §V2「未描画 ≠ 否定」）。否定と判断したものは
   **必ずユーザーに確認**し、合意できたら `/pj:change`（L2）で spec から廃止する。**このスキルで AC を消さない。**
5. **忠実度を見直す**（入力の質が変われば宣言も変わる）。**DL との衝突は折衷せず決める**——「DL を変える」なら
   `/pj:change` で採番し直す。
6. **実装へ伝播**し、最後に **AC テストが緑のまま**であることを確認する（**赤くなったら振る舞いを変えてしまった
   証拠**なので手順4に戻る）。`intake.md` の変更履歴・`updated` を更新し、未決が開いたら `progress` を**下げる**。

## モード別の振る舞い（dispatch は concepts §7）

### 対話（既定）

1. `docs/specs/`（WHAT）と `docs/design/`（現在の HOW）を読む
2. サマリを出す:

```
[design 現状]
- スタック: <主要な確定/未定>
- データモデル: <エンティティ数 / 未確定の関連>
- 設計確定度: ■■■□□

[spec から拾った未決の技術論点]
1. 同時編集の競合制御（全 feature・未着手）
2. リッチエディタ選定（event のプログラム）

どれから決めます?
```

3. 決定したら該当ファイルに反映。**大きな選定は adr/ に1件記録**（理由付き）。エンティティ名・識別子は
   **glossary に従う**。1つの技術決定が複数 feature に効くなら conventions.md に横断方針として書く。

### 軽量修正

concepts §8 の手順。**デザイン原則（DL-NN）を増やす/廃止する修正はここで処理せず `/pj:change` に渡す**
（concepts §7 モード5）。既存 DL の言い回し調整など ID を動かさない修正は即時反映してよい。

### status / next

status は設計確定度＋未決の技術論点一覧。next は次に決めるとよい論点を 2〜3 個
（優先: 多数の feature に効く横断決定 / build の前提になる data-model / 未着手のコア）。

### review（readiness-reviewer に委譲）

`Agent` で **`pj:readiness-reviewer`** を起動（対象 = `docs/design/`＋`docs/specs/`、レイヤー = design）。
「実装 Agent が追加の技術判断なしにコードを書けるか」を判定させ、要点整理して伝える。編集しない。
UI/視覚の整合を見るときは加えて **`pj:design-reviewer`**（対象 = `design-language.md` ＋ **`docs/design/intake/`**
＋ `src/`）。取り込みがある成果物は**構成の乖離・振り分けの落下**を要点整理して伝える。

### audit（ドリフト監査）

**concepts §14「audit の起動仕様」に従う**。design から起動した場合は、報告の並び順だけ**design 決定 ↔
実装/スキーマの乖離を先頭**にする（**product 境界の違反はさらにその前**）。編集しない。

### build へ渡す

design が固まったら **`/pj:build <feature>`** が標準導線。大物 feature の事前 orientation が欲しいときは
`/pj:build plan <feature>`。

> **その前に `/pj:setup stack` が要ることがある。** **このスキルはコードを書かない**ので、`stack.md` で
> FW・DB・ORM・テストランナーを決めても**実体は生まれていない**。`stack.md` を書いた・更新したら
> **「採用技術が package.json の依存に入っているか」「テストランナーが起動するか」を確認**し、
> 入っていなければ **`/pj:setup stack`** を案内する（このスキルは入れない。決めるのが仕事）。
> **「決めたのに入っていない」状態を黙って build に渡さない。**

## 設計の確定度

**成果物ごとに frontmatter の `progress` を持つ**（格納先・評価軸・平均の取り方・保存値から出すルールは
**concepts §5**）。ファイルを直したら `progress` をつけ直し、`updated` を今日にし、変更履歴に1行残す。

- **取り込んだ出典が AC / DL に振り分け終わるまで `design-language.md` の `progress` を 4-5 にしない**
  （実装 Agent が追加の視覚判断なしに組めないため）。
