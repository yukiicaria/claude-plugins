---
name: spec
description: pj パイプラインの WHAT レイヤー。ソフトウェア要件定義を対話で詰める。サマリ・じっくり対話・軽量修正・next・review（sub-agent委譲）・用語整理・audit を文脈で切り替える。既存物の修正は /pj:change が入口。pj 運用宣言（CLAUDE.md の pj:managed ブロック）を持つ repo でのみ使う。
argument-hint: "[status|next|review|glossary|audit|<修正依頼 or feature名>]"
---

仕様書を一緒に育てるコマンド。対象 product の `docs/specs/` 配下の md をユーザーと協働で編集する。

引数: $ARGUMENTS

## モデル起動時のゲート（`/pj:spec` と打たれた起動では読み飛ばす）

ユーザーが明示的に `/pj:spec ...` と入力したなら、この節は**丸ごと無視**する。**モデル（Claude）の判断で
起動したときだけ**上から順に通る。**`concepts.md` を読むのはゲート通過後**（降りる判断をトークンを使う前に終える）。

1. **pj 運用の repo か確かめる** — ルート `CLAUDE.md` に `<!-- pj:managed start` があるか grep する。
   **無ければ即座に降りる**（何も読まず書かず、「この repo は pj 運用ではないので通常どおり進めます」と一行）。
2. **status / next / review / audit なら承認は要らない**（編集しないモード。stamp もしない）。そのまま進む。
3. **編集系（対話・軽量修正・glossary）なら AskUserQuestion で一行確認** —
   「**対象 product** ／ 触るレイヤー（spec）／ **何を変えるか**」を1文で。
4. **承認が得られなければ何も書かずに降りる。** 代案を押し売りしない。

## 最初に必ず読む

**`../../_shared/concepts.md`**（背骨）。起動したらまず読む。ここには **spec レイヤー固有の振る舞い**だけを書く。

## ゴール（spec レイヤー）

**別の実装/設計 Agent がこの仕様だけを見て、追加の設計対話なしに着手できるレベル**まで詰める。常にこれを
判断軸にする。**実装の詳細（ライブラリ・コード設計・SQL スキーマ）は書かない**——それは `/pj:design` の仕事。

## 対象 product を決める（起動時に必ず）

**concepts §2 の3箇所を glob して product を列挙し、対象を決めて一行宣言してから動く。** 引数に product 名／
`packages/<name>` があればそれ、無ければ **root product** が既定。直近の文脈が特定 package の話なら一言だけ確認。

**package を扱うときの追加規律**（root には不要）:

- **app 固有の語彙を書かない。** ただし**その package の責務の語彙は書いてよい**（判定は「別の app がそのまま
  使えるか」。特定 app の制度でしか説明できないなら、それは feature 側の話）
- **app / 他 feature を `[[リンク]]` で参照しない**（向きが壊れる・concepts §2）
- 全 product 共通の作法は **root の `docs/design/conventions.md`** が正。ここに写さない（concepts §13）
- readiness は「**app を知らずにこの package を作れるか**」でも測る（concepts §5）

## ファイル構造

> パスは**対象 product のルートからの相対**。

```
docs/specs/
  overview.md              # 目的・機能リスト・2軸ダッシュボード・論点（root なら product 表も）
  features/<name>.md       # 個別機能仕様（kebab-case）。小さい product では作らない（concepts §6）
  glossary.md              # この product の用語の正（concepts §10）
```

初回起動で対象 product の `docs/specs/` が無ければ `../../_shared/templates/` から作る:

- **root product**: overview.md / feature.md / glossary.md / specs-CLAUDE.md（→ `docs/specs/CLAUDE.md`）
- **package product**: **package-overview.md** を `overview.md` として ／ glossary.md / specs-CLAUDE.md
  （package の器は本来 `/pj:setup package <name>` が作る。ここで作るのは取りこぼしの補完）

あわせて、編集系フローなら**ルート `CLAUDE.md` に運用宣言を stamp** する（concepts §17）。

> **器の不在を黙って埋めない。** このスキルが作れるのは `docs/` だけで、**`package.json` / workspace 登録 /
> 依存方向の lint は作れない**（`/pj:setup` の担当）。docs だけあって器が無いと、**spec が完成しているのに
> 実装できない product** が育つ。
>
> **package の docs を作った・触ったら、毎回「`packages/<name>/package.json` があるか」「root の
> workspaces glob（`packages/*` と `packages/*/*`）に載っているか」を確認する。** 無ければそのターンの報告に
> 明記し、`/pj:setup package <name>`（既存なら `/pj:setup sync`）を案内する。会話は止めなくてよいが**黙って進めない**。

## モード別の振る舞い（dispatch は concepts §7）

### 対話（既定）

1. **対象 product を決め**、その `docs/specs/` 配下の全 md を読む（無ければテンプレから作る）
2. サマリを出す:

```
[product] workflow (package)  ← app を知らない側

[現状]
- 目的: <一行 / 未定>
- コア機能: <件数> 件
- spec 進捗: ■■■□□ (平均 3.2/5)

[個別の状況]
- event    ■■■■□  残: 単純セクションの項目確定
- contacts ■■■□□  残: 名寄せ候補の粒度
- schedule ■□□□□  名前だけ

今日は何の話します? それとも event の続き?
```

> product が1つ（package を持たない repo）なら `[product]` 行は出さない（ノイズになる）。

3. ユーザーの答えを待つ。会話の進め方:

- 自由発話を解釈して該当 md に書き込む／更新する（更新通知は1行・diff は見せない）
- **構造・大枠・全体像を個別項目より先に**詰める（個別フィールドの取捨は後回しでよい）
- feature を追加/更新したら**その product の** overview を同期（機能リストは**項目の増減のみ**、進捗は
  2軸ダッシュボードのみ）。**package を触ったら続けて root の product 表も引き直す**（concepts §9）
- 「これは全 package 共通の話だ」と気づいたら spec に書かず **root の `conventions.md`** へ送る（concepts §13）
- 「これは app を知らずに作れる独立したライブラリだ」と見えてきたら **`/pj:setup package <name>`** を案内する
- **前提を壊して作り直すことを恐れない**（concepts §8）

### 軽量修正

concepts §8 の手順に従う。**受入条件（AC-NN）を増やす/廃止する修正はここで処理せず `/pj:change` に渡す**
（concepts §7 モード5）。文言の明確化など ID を動かさない修正は即時反映してよい。

> **ただし「新規に書き起こす」は別**（concepts §4）。**まだ ID 付きの受入条件が1つも無い feature に
> 初めて書き出すときは、このスキルが採番する。** `/pj:change` に回すと書いた直後の spec に ID の無い受入条件が
> 残り、鎖が切れる。判定は「**その feature に ID 付きの受入条件が1つでもあるか**」。

### status / next

読むだけ。status は2軸ダッシュボード＋未確定の全体論点。next は次に詰めるとよい feature を 2〜3 個
（優先: progress 3 の一押し / 多数から参照される基盤 / overview の全体論点 / コアなのに手薄）。

### review（readiness-reviewer に委譲）

`Agent` で **`pj:readiness-reviewer`** を起動し、「対象 = 対象 product の `docs/specs/`（または指定 feature）、
レイヤー = spec、product 種別 = app / package」を渡す。返った判定を**要点整理して伝える**（重大な指摘を先頭に）。
編集しない。**package では判定軸が1本増える**（「app を一切知らずに作れるか」・concepts §2）。

### glossary（用語整理 — terminologist で監査 ＋ 適用）

「用語を整えて」「用語バラバラ」「命名を統一」系、または `glossary`/`terms`/`用語`。仕様づくりは試行錯誤で
用語がバラける。それを一意に整え **対象 product の** `docs/specs/glossary.md`（concepts §10）に固定するのは必須工程。

1. **監査（委譲）**: `Agent` で **`pj:terminologist`** を起動（対象 = 対象 product の `docs/specs/`、
   レイヤー = spec、product 種別）。衝突・ゆれ・恣意的な名前・曖昧語・旧称残骸を file:line 付きで洗い出させ、
   canonical 用語案・glossary ドラフト・改名候補を受け取る。**package では「app 固有の語彙の混入」も報告させる**
   （自分の責務の語彙は違反ではない）。
2. **整理して提示**: 丸投げせず要点を整理。**衝突（同一語が複数概念）と恣意的な名前を先頭に**。改名は
   「ただ揃える」でなく**悪い名前は良い名前に変える**前提で提案する。
3. **主観の入る選択だけ確認**: 明らかなものは確定として進め、新しい呼称の選定など好みが割れるものだけ簡潔に確認。
   「保留」は glossary の open 扱いで先へ。
4. **適用（ここで初めて編集する）**:
   - `glossary.md` を構築/更新（無ければ `../../_shared/templates/glossary.md` から）。形式は分野ごとの
     **「用語 / 定義」2列の表**。旧称・衝突語の「避ける列」は作らない。
   - 確定した改名を**全 feature・overview に適用**。一括置換は特定の複合語のみ安全に行い、**一般語を
     巻き込まない**よう文脈付きで。衝突語は文脈ごとに分けて置換。適用後は **grep で「旧称の本文残存ゼロ」を検証**
     （変更履歴の中に旧称が残るのは可）。
   - overview 冒頭に「**用語は glossary.md が正**」の一文が無ければ追加する。
   - 触れた各 feature の変更履歴に「用語統一（◯◯→△△）」を1行、`updated` を今日に。

### audit（ドリフト監査）

**concepts §14「audit の起動仕様」に従う**（全スキル共通の1つの監査）。spec から起動した場合は、報告の
並び順だけ**受入条件まわりの乖離を先頭**にする（**product 境界の違反はさらにその前**）。編集しない。

> spec が固まったら **`/pj:design` で技術を決めてから `/pj:build`** が標準導線。

## 進捗管理

frontmatter は `name / progress / build_progress / updated / open_questions`。評価基準・運用ルール・バー表記は
concepts §5。`build_progress` は `/pj:build` が更新するので spec では基本さわらない。`updated` は必ず今日の日付。
2軸ダッシュボードは feature 更新のたび同期し、**package を触ったら root の product 表も引き直す**（concepts §9）。
