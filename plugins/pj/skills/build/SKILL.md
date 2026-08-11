---
name: build
description: pj パイプラインの DOING レイヤー。spec（WHAT）と design（HOW）から、受入条件をテストに写してテスト先行で実装する。対象は1つでも複数でもよく、複数なら依存層ごとに並列で回す（承認は最初の1回・詰まった product は blocked にして次へ）。status・next・plan・review・audit を文脈で切り替える。完了は「全受入条件のテストが緑」で測る。
disable-model-invocation: true
argument-hint: "[status|next|plan <対象…>|review|audit|<feature名>|<product名…>|packages]"
---

spec と design を実コードに落とすコマンド。テストファーストで feature を実装し、状態を overview に反映する。

引数: $ARGUMENTS

## 最初に必ず読む

共通の軸・モード・保守規律・禁止事項は **`../../_shared/concepts.md`**（pj パイプラインの背骨）。

**まず対象 product を決める**（concepts §2）。`docs/specs/overview.md` と
`packages/*/docs/specs/overview.md` ＋ `packages/*/*/docs/specs/overview.md` を glob して product を列挙し、引数の feature 名がどの product の
ものかで決める（同名 feature は repo 内で禁止なので一意に定まる・concepts §4）。**一行宣言してから動く。**

**対象は1つとは限らない。** 引数に product 名が複数並ぶ・`packages` のような範囲キーワードが来たら
「まとめて build」（下記）に入る。そのときも product ごとの中身は既定 build と同じで、
**下の「読む」対象も対象 product ごとに読み直す**（1本目の design を2本目に流用しない）。

build は spec・design の**下流**。実装に入る前に必ず両方を読む（パスは**対象 product 相対**）:
- `docs/specs/`（WHAT — 特に対象 feature の受入条件）
- `docs/design/`（HOW — stack / data-model / 関連 ADR。これが拘束条件）
- **root の `docs/design/conventions.md`**（全 product が準拠する作法・concepts §13）と、
  UI を実装するなら **root の `docs/design/design-language.md`**（視覚の正）

**UI を実装するときは `../../_shared/visual.md` も読む**（§V1 視覚の正 / §V2 intake）。
UI に触れない feature（API・バッチ等）では読まなくてよい。

WHAT でも HOW でも、**変えたくなったら build で辻褄を合わせず `/pj:change` に投げる**（concepts.md §8）。
どの層が正かの判定と鎖の伝播は `/pj:change` が引き受けるので、build 側で層を選び分けない。

## ゴール（build レイヤー）

対象 feature を **全受入条件のテストが緑**になるまで実装する。これが完了の唯一の基準。
「受入条件 = test = 完了」の契約を守る。

**package product を実装するときは、完了条件がもう1本ある**（concepts §2）:

- **app / 他 feature を import していない。** 相対パスで `packages/<name>/` の外へ出ていない
- **`package.json` の dependencies に書いたものしか使っていない**（bare specifier で解決できる）
- **app 固有の語彙が型名・識別子・テスト名に混ざっていない**（その package の責務の語彙は可）
- 外界（永続・外部送信・ドメイン解決）は**注入**で受け取っている

> 判断に迷ったら: 「これを別リポジトリに切り出したとき、そのまま動くか？」
> 動かないなら app を知ってしまっている。**辻褄を合わせず `/pj:change` に投げる。**

## 置き場は表を引くだけ。新しい箱を作らない（concepts §12）

**実装を置く場所は root の `docs/design/conventions.md` の「配置の表」が決める。**
build は表を引くだけで、**新しいトップレベルのディレクトリを作らない**。

- 表に無いものが要ると気づいたら、**その場で作らず `/pj:change` に投げる**（作法＝ HOW の変更）。
- 迷ったら concepts §12 の3分類で当てる: **挙動**（テストできる）／ **見た目**（全画面に効く）／
  **まとめる**（上2つを繋ぐだけ）。3つのどれにも当てはまらないなら、分類のほうが間違っている。
- **package の中身も同じ分類で組む。** 完了条件は「別リポジトリに install してそのまま動くか」（§2）。

> これを怠ると、置き場の無いコードがその場しのぎの新ディレクトリを生み、
> **次に触る人が「なぜここにあるか」を辿れなくなる**。

## build の中心契約: 受入条件 → test → code

- 対象 feature の `## 受入条件` の各チェックボックスを、**最低1つの test に 1:1 で写す**
- **落ちるテストを先に書く**（テストファースト）。それから実装して緑にする
- その feature の**全受入条件テストが緑になって初めて実装完了**とみなす
- 完了したら `build_progress`、overview の2軸ダッシュボード、変更履歴を同期（下記）

## モード別の振る舞い（dispatch は concepts.md §7）

### build（既定: `/pj:build <feature>`）
feature 名が無ければ next の結果から「どれを実装します?」と一言。手順:
1. **読む**: 対象 feature の spec（＋ `[[関連]]` 1階層）と `docs/design/` 一式を直読みする
   （spec/design が直接の入力。concepts §7）。
2. **前提チェック**: 次の3つを見て、欠けていたら警告して続行確認する
   （不足はユーザーに確認、または該当スキルへ戻す）。
   - 対象 feature の `progress`（spec）が 3 未満
   - design に未決の技術論点が残る
   - **土台が無い**——`stack.md` の採用技術が入っていない／テストランナーが起動しない／
     対象が package なら `package.json` が無い。**この場合は `/pj:setup stack`・`/pj:setup sync` へ戻す。**
     **ここで自分で FW やランナーを入れ始めない**（特定 feature の build に土台が埋もれ、
     後から「なぜこの構成か」を辿れなくなる）。
3. **計画**: 実装単位に分解。**依存順を尊重**（多数から参照される基盤 = データモデル・共通エンティティを先に）。
4. **テスト先行**: 受入条件 → test を書く（落ちる状態で）。
5. **実装**: design の stack / conventions に従って緑になるまで実装。横断方針（conventions.md）を必ず踏襲。
   - **UI を実装するとき**は `docs/design/design-language.md`（視覚の正・visual.md §V1）を正とし、満たす原則を
     共通コンポーネント先頭に `@satisfies DL-NN` で注釈する。完了前に **`pj:design-reviewer`** で違反ゼロを確認
     （コード品質は従来どおり `/code-review` `/security-review` `/verify`）。
   - **対象に `docs/design/intake/<intake-id>/` があれば、実装前に `intake.md` と実体を必ず読む**
     （visual.md §V2）。**拘束範囲は忠実度（fidelity）が決める**:
     `sketch` = 構造と並び順 / `structure` = ＋挙動 / `hifi` = ＋明記された寸法・閾値・時間。
     - **`hifi` なら数値を落とさない。** 数値は `intake.md` の振り分け表から **AC** を辿る
       （表に無い数値を実体から目測で拾わない）。
     - **部品が DS に置き換わるのは正常。配置・列順・密度が変わるのは違反。** ここを取り違えない。
     - DS を当てると出典の意図が壊れる箇所が出たら、**黙って折衷せず** `/pj:change` で DL-NN を1本足して記録する。
     - 出典を見て**振る舞い（WHAT）の誤り**に気づいたら build で辻褄を合わせず `/pj:change` へ。
6. **同期**: 全受入条件テストが緑になったら
   - 対象 feature の `build_progress` を更新（concepts.md §5。5 = 全受入条件テスト緑）
   - **その product の** overview の2軸ダッシュボードの build 列を更新（**進捗の正はここ**。
     コア機能リストは触らない）。**package を実装したら続けて root の product 表（上段）も引き直す**
     （下段が正・上段は導出。concepts §9）
   - feature.md の `## 変更履歴` に `YYYY-MM-DD: 実装（受入条件 N 件テスト緑）` を追記、`updated` を今日に
7. コミット・push は**ユーザーが明示するまでしない**（プロジェクトの CLAUDE.md 準拠）。

### まとめて build（`/pj:build <A> <B> …` / `/pj:build packages`）

対象が**複数**のときのモード。**やることは上の既定 build と同じ**で、それを対象ごとに繰り返すだけ。
新しい実装作法も新しい足切り基準も作らない（**判断は next と上の前提チェックが既に持っている**）。

**対象の決め方**:
- **product 名を並べて打たれたら**それが対象（`/pj:build user-group access-control valid-period`）。
- **`packages` / `all` のような範囲キーワード**なら、**next の優先順位そのまま**
  （依存順 × spec readiness）で対象を並べる。ここで独自の閾値を発明しない。
- **層に割る**。層は **root `docs/specs/overview.md` の products 表の「依存」列**から引く
  （宣言＝あるべき姿なので、まだ実装の無い package も層に割れる）。
  `docs/design/dependencies.md` は**実体からの生成物なので使わない**（未実装分が落ちる）。

**承認は最初の1回だけ**（このモードの本体）。開始前に「層 → 対象 → 本数」を一覧で出して確認を取る。
**通ったら以降は product ごとに聞かない。** 最後の報告まで走り切る。

**実行**: 同じ層の product は **1 product = 1 sub-agent で並列**。各 sub-agent は
「その product の未緑 feature を、上の既定 build 手順 1〜7 で緑にする」だけを負う。
メインは要約（緑本数・blocked とその理由）だけ受け取る（concepts §14）。

**書き込み境界（並列で壊さないための線）**:
- sub-agent が書いてよいのは **`packages/<name>/` 配下だけ**（`src/` ・その product の `docs/` ・`package.json`）。
- **root の `docs/specs/overview.md` の products 表・ルートの workspace 設定・lockfile は触らせない。**
  products 表の引き直しは**全層が終わってからメインが1回**やる（concepts §9 上段は導出）。
  依存インストールが要るなら、**層を跨ぐ前にメインがまとめて実行**する。

**詰まっても止めない**: 前提チェックの欠け（spec が薄い・土台が無い）／受入条件が design と矛盾する／
テストが緑にならない のいずれかに当たったら、その product を **blocked** として記録して**次へ進む**。
**バッチ中に `/pj:change` や `/pj:setup` を起動しない**（対話が要るので必ずそこで止まってしまう）。
blocked の product は**緑でないものを緑と記録しない**。書きかけは**落ちるテストごとそのまま残し**、
`build_progress` は実際のテスト緑率のまま置く（先へ進むために辻褄を合わせて上げない・concepts §15）。

**報告**: 論点を最後まで溜めない（concepts §15）。**1本終わるたびに1行**出す
（`[4/9] workflow/core … BLOCKED — wf-AC-07 の前提が design と矛盾`）。
そのうえで最後に `緑 N 本 / blocked M 本` を集約し、blocked は1本ずつ**理由と次の一手**
（どの skill に何を投げるか）を出す。**止まらないことと黙ることは別**。

**plan**: `/pj:build plan packages` は**コードを書かず**、層割り・対象一覧・各 product の受入条件数・
想定リスクだけ出す。数が多いときは先にこれを見てもらう。

### 軽量修正（実装中に仕様/設計の誤りに気づいたとき）
build で勝手に辻褄を合わせない。**`/pj:change <気づいたこと>` に投げる**（concepts.md §8 典型シナリオ1）。
どの層が正かの判定・AC の採番・鎖（受入条件 ↔ test ↔ code）の伝播は `/pj:change` が引き受けるので、
build 側で層を選び分けない。

### status / next
- status は2軸ダッシュボード（spec / build）。「spec 済・未実装」「実装中」「テスト緑」を一目で。
- next は次に実装すべき feature を提案。優先: **依存順（基盤が先）× spec readiness の高い順**。
  spec が 3 未満の feature は「先に /pj:spec で詰める」と促す。

### plan（`/pj:build plan <feature>` / `plan <product…>` / `plan packages`）
コードは書かず、上記「計画」までを出す（分解・依存順・受入条件→テスト一覧・想定リスク）。
大きい feature は実装前にこれでレビューしてもらう。
**対象が複数なら**「まとめて build」の層割り・対象一覧・各 product の受入条件数までを出す（実装しない）。

### review / audit
- review はコード品質の確認。専用 agent を作らず**既存の `/code-review` `/security-review` `/verify`
  に委譲**する（再発明しない）。
- audit は **concepts.md §14「audit の起動仕様」に従う**（全 skill 共通の1つの監査。対象は**対象 product の
  トライアド** `docs/specs/` ＋ `docs/design/` ＋ `src/` ＋ root の作法）。product が明示されなければ
  **全 product を順に**見る。実装と仕様/設計の乖離を報告し、**product 境界の違反はその前に**出す
  （concepts §14 観点B）。編集しない。

## 進捗管理

各 feature の `build_progress`（build readiness）を concepts.md §5 の軸で測る。基準は明快:
**0 = 未着手 / 進行中はテスト緑率に応じて / 5 = 全受入条件テスト緑**。spec の `progress` は build では
さわらない。`updated` は必ず今日の日付。状態は必ず overview の2軸ダッシュボードに反映する。
