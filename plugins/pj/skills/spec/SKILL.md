---
name: spec
description: pj パイプラインの WHAT レイヤー。ソフトウェア要件定義を対話で詰める。サマリ・じっくり対話・軽量修正・next・review（sub-agent委譲）・用語整理・audit を文脈で切り替える。既存物の修正は /pj:change が入口。
disable-model-invocation: true
argument-hint: "[status|next|review|glossary|audit|<修正依頼 or feature名>]"
---

仕様書を一緒に育てるコマンド。プロジェクトの `docs/specs/` 配下の md をユーザーと協働で編集する。

引数: $ARGUMENTS

## 最初に必ず読む

このスキルの**共通の軸・モード・保守規律・禁止事項は `../../_shared/concepts.md`**（pj パイプラインの背骨）。
起動したらまずそれを読み、ここでは **spec レイヤー固有の振る舞い**だけを定義する。

## ゴール（spec レイヤー）

**別の実装/設計 Agent がこの仕様だけを見て、追加の設計対話なしに着手できるレベル**まで詰める。
常にこれを判断軸にする。**実装の詳細（ライブラリ・コード設計・SQL スキーマ）は書かない**——それは
design レイヤー（`/pj:design` → `docs/design/`）の仕事。「何を作るか」「どう振る舞うか」が完全に
決まっている状態がゴール。

## 対象 product を決める（起動時に必ず・concepts §2）

pj の単位は **product**（app も package も product）。**どの product の spec の話かを最初に決める。**

1. **product を列挙する**: `docs/specs/overview.md` と `packages/*/docs/specs/overview.md` を glob
   （これが product の定義。マニフェストは無い）。
2. **対象を決める**: 引数に product 名／`packages/<name>` があればそれ。無ければ **root product** を既定にする。
   ただし直近の文脈が特定 package の話なら、それを候補として**一言だけ確認**する。
3. **宣言する**: 「`<product>` の spec を見ます」と一行で言ってから動く。

**package product を扱うときの追加規律**（root product には不要）:

- **app 固有の語彙を書かない。** ただし**その package の責務の語彙は書いてよい**（concepts §2）。
  判定は「別の app がそのまま使えるか」。特定 app の制度でしか説明できないなら、それは feature
- **app / 他 feature を `[[リンク]]` で参照しない**（向きが壊れる・concepts §2）
- 全 product 共通の作法は **root の `docs/design/conventions.md`** が正。ここに写さない（concepts §12）
- readiness は「**app を知らずにこの package を作れるか**」でも測る（concepts §5）

## ファイル構造

> パスは**対象 product のルートからの相対**（root product なら repo 直下、package なら `packages/<name>/`）。

```
docs/specs/
  overview.md              # 目的・機能リスト・2軸ダッシュボード・論点（root なら product 表も）
  features/<name>.md       # 個別機能仕様（kebab-case）。小さい product では作らない（concepts §6）
  glossary.md              # この product の用語の正（concepts.md §10）
```

**深さは可変**（concepts §6）: overview 1枚から始め、feature が複数見えてきたら `features/` に割る。
**先回りして割らない。** 段階1でも AC ID は必ず振る（鎖は器の形ではなく ID で成立する）。

初回起動で対象 product の `docs/specs/` が無ければ、`../../_shared/templates/` から作る:

- **root product**: overview.md / feature.md / glossary.md / specs-CLAUDE.md
  （specs-CLAUDE.md は `docs/specs/CLAUDE.md` として配置）
- **package product**: **package-overview.md** を `overview.md` として ／ glossary.md / specs-CLAUDE.md
  （package の器は本来 `/pj:setup package <name>` が作る。ここで作るのは取りこぼしの補完）
あわせて**プロジェクト root の `CLAUDE.md` に運用宣言の管理ブロックを stamp** する（concepts.md §16。無ければ
作る・既にマーカーがあれば重複させない）。これにより規律を知らない人・新しいセッションが必ず入口（`/pj:change`）に
着地できる。対話・軽量修正の編集系フローでも、管理ブロックが無ければ同様に stamp する。

## モード別の振る舞い（dispatch は concepts.md §7）

### 対話（既定）
起動時に必須:
1. **対象 product を決める**（上記「対象 product を決める」）。その product の `docs/specs/` 配下の
   全 md を読む（無ければテンプレから作る）
2. 下記サマリを出す:
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
- テンプレ質問を上から順に投げない。文脈に応じて必要な質問だけ
- 自由発話を解釈して該当 md に書き込む／更新する。更新通知は1行で（diff は見せない）
- 構造・大枠・全体像を個別項目より先に詰める（個別フィールドの取捨は後回しでよい）
- feature を追加/更新したら**その product の** overview を同期（機能リストは**項目の増減のみ**、
  進捗は2軸ダッシュボードのみ）。**package を触ったら続けて root の product 表（上段）も引き直す**
  （下段が正・上段は導出。concepts §9）
- 「これは全 package 共通の話だ」と気づいたら spec に書かず **root の `docs/design/conventions.md`** へ
  送る（concepts §12）。ここで抱え込むとレイヤー違反になる
- 「これは app を知らずに作れる独立したライブラリだ」と見えてきたら、**`/pj:setup package <name>`** を
  案内する（器を作ってから中身を書く）
- **前提を壊して作り直すことを恐れない**（concepts.md §8）。作り直したら変更履歴と関連 feature を追って直す

### 軽量修正
concepts.md §8 の手順に従う（スコープ判定 → 該当 feature 特定 → 更新 → 変更履歴 → updated → 影響1個提示）。
**受入条件（AC-NN）を増やす/廃止する修正はこのモードで処理しない**。採番規律を一箇所に保つため
`/pj:change` に渡す（concepts.md §7 モード5 / §4）。文言の明確化など ID を動かさない修正はここで即時反映してよい。

### status / next
読むだけ。status は2軸ダッシュボード＋未確定の全体論点。next は次に詰めるとよい feature を 2〜3 個
提案（優先: progress 3 の一押し / 多数から参照される基盤 / overview の全体論点 / コアなのに手薄）。

### review（readiness-reviewer に委譲）
`Agent` ツールで **`pj:readiness-reviewer`** を起動。プロンプトに「対象 = 対象 product の `docs/specs/`
（または指定 feature）、レイヤー = spec、**product 種別 = app / package**」を渡す。返った判定（総合・
feature 別実装可能性・重大な指摘・抜け漏れ・スコープ整合・過剰・次の優先順位）を**要点整理してユーザーに
伝える**（重大な指摘を先頭に）。レビュー中は編集しない。

**package を見るときは判定軸を1本足す**: 「**この spec だけ読んで、app を一切知らずに作れるか**」
（concepts §2）。app の存在を前提にした記述が残っていたら、それは readiness を上げられない理由になる。

### glossary（用語整理 — terminologist で監査 ＋ 適用）
「用語を整えて」「用語バラバラ」「用語集つくって」「命名を統一」系、または `glossary`/`terms`/`用語`。
仕様づくりは試行錯誤で用語がバラける。それを一意に整え **対象 product の** `docs/specs/glossary.md`
（concepts.md §10）に固定するのは必須工程。「監査 → canonical 提案 → 適用 → glossary 構築/更新」を一括で行う。

> **glossary は product ごと**（concepts §10）。root には app 固有の語、package にはその責務の語彙。
> **package が自分の glossary に定義していない概念語を使っていたら要調査**——app の語を借りているなら
> 境界の破れで、言い換えでは直らない（spec の切り分けから直す）。**自分の責務の語なら定義すればよい**。

1. **監査（委譲）**: `Agent` で **`pj:terminologist`** を起動（対象 = 対象 product の `docs/specs/`、
   レイヤー = spec、product 種別 = app / package）。衝突・ゆれ・恣意的な名前・曖昧語・旧称残骸を
   file:line 付きで洗い出させ、canonical 用語案・glossary ドラフト・改名候補を受け取る。
   **package を見るときは「app 固有の語彙の混入」も必ず報告させる**（自分の責務の語彙は違反ではない）。
2. **整理して提示**: 丸投げせず要点を整理。**衝突（同一語が複数概念）と恣意的な名前を先頭に**。
   改名は「ただ揃える」でなく**悪い名前は良い名前に変える**前提で提案する。
3. **主観の入る選択だけ確認**: 明らかなものは確定として進め、新しい呼称の選定など好みが割れるものだけ
   簡潔に確認（数が多ければまとめて）。「保留」は glossary の open 扱いで先へ。
4. **適用（編集する）**:
   - `docs/specs/glossary.md` を構築/更新（無ければ `../../_shared/templates/glossary.md` から作る）。
     形式は分野ごとの**「用語 / 定義」2列の表**。旧称・衝突語の「避ける列」は作らない。
   - 確定した改名を**全 feature・overview に適用**。一括置換は特定の複合語のみ安全に行い、**一般語
     （普通の日本語）を巻き込まない**よう文脈付きで。衝突語は文脈ごとに分けて置換。適用後は
     **grep で「旧称の本文残存ゼロ」を検証**（変更履歴の中に旧称が残るのは可）。
   - overview 冒頭に「**用語は glossary.md が正**」の一文が無ければ追加し参照を張る。
   - 触れた各 feature の変更履歴に「用語統一（◯◯→△△）」を1行、`updated` を今日に。
5. 監査フェーズ（編集前）では編集しない。適用フェーズで初めて編集する。

### audit（ドリフト監査）
**concepts.md §13「audit の起動仕様」に従う**（全 skill 共通の1つの監査。対象は**対象 product の
トライアド** `docs/specs/` ＋ `docs/design/` ＋ `src/` ＋ root の作法）。product が明示されなければ
**全 product を順に**見る。spec から起動した場合は、報告の並び順だけ**受入条件まわりの乖離を先頭**にする
（ただし **product 境界の違反はさらにその前**・concepts §13 観点B）。編集しない。

> spec が固まったら **`/pj:design` で技術を決めてから `/pj:build`** が標準導線（受け渡しの作法は concepts §7）。

## 進捗管理

frontmatter は `name / progress / build_progress / updated / open_questions`。`progress`（spec readiness）の
評価基準・運用ルール・バー表記は concepts.md §5。`build_progress` は `/pj:build` が更新するので spec では
基本さわらない。`updated` は必ず今日の日付。**その product の**2軸ダッシュボードは feature 更新のたび同期し、
**package を触ったら root の product 表（上段）も引き直す**（concepts §9）。

**readiness は product ごとに閉じて測る**（concepts §5）。他 product の未決を持ち込まない——
持ち込むと「この package は独立して作れるか」という問いが測れなくなる。
