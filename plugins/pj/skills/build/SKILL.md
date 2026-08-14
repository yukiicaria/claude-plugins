---
name: build
description: pj パイプラインの DOING レイヤー。spec（WHAT）と design（HOW）から、受入条件をテストに写してテスト先行で実装する。対象は1つでも複数でもよく、複数なら依存層ごとに並列で回す（承認は最初の1回・詰まった product は blocked にして次へ）。status・next・plan・review・audit を文脈で切り替える。完了は「全受入条件のテストが緑」で測る。
disable-model-invocation: true
argument-hint: "[status|next|plan <対象…>|review|audit|<feature名>|<product名…>|packages]"
---

spec と design を実コードに落とすコマンド。テストファーストで feature を実装し、状態を overview に反映する。

引数: $ARGUMENTS

## 最初に必ず読む

**`../../_shared/concepts.md`**（背骨）。そのうえで:

1. **対象 product を決める**（concepts §2 の3箇所を glob。引数の feature 名がどの product のものかで決まる
   ——同名 feature は repo 内で禁止なので一意に定まる・concepts §4）。**一行宣言してから動く。**
   **対象は1つとは限らない**——product 名が複数並ぶ・`packages` のような範囲キーワードなら
   「まとめて build」に入る（**中身は既定 build と同じ**）。
2. **spec と design を直読みする**（build は両方の下流。パスは**対象 product 相対**）:
   `docs/specs/`（特に対象 feature の受入条件）／`docs/design/`（stack / data-model / 関連 ADR＝拘束条件）／
   **root の `docs/design/conventions.md`**（全 product が準拠する作法）と、UI なら **root の `design-language.md`**。
   **複数対象なら product ごとに読み直す**（1本目の design を2本目に流用しない）。
3. **UI を実装するなら `../../_shared/visual.md`**（§V1 視覚の正 / §V2 intake）。UI に触れないなら不要。
4. **package を実装するなら `../../_shared/product-model.md`**（可搬・境界の実務）。

WHAT でも HOW でも、**変えたくなったら build で辻褄を合わせず `/pj:change` に投げる**（concepts §8）。
どの層が正かの判定と鎖の伝播は `/pj:change` が引き受けるので、**build 側で層を選び分けない**。

## ゴール（build レイヤー）

対象 feature を **全受入条件のテストが緑**になるまで実装する。**これが完了の唯一の基準**で、
「受入条件 = test = 完了」の契約を守る。

**package product では完了条件がもう1本ある**（concepts §2 / product-model.md）:

- **app / 他 feature を import していない**（相対パスで `packages/<name>/` の外へ出ていない）
- **`package.json` に書いたものしか使っていない**（bare specifier で解決できる）
- **app 固有の語彙が型名・識別子・テスト名に混ざっていない**（その package の責務の語彙は可）
- 外界（永続・外部送信・ドメイン解決）は**注入**で受け取っている

> 迷ったら: 「これを別リポジトリに切り出したとき、そのまま動くか？」動かないなら app を知ってしまっている。
> **辻褄を合わせず `/pj:change` に投げる。**

## build の中心契約: 受入条件 → test → code

- 対象 feature の `## 受入条件` の各チェックボックスを、**最低1つの test に 1:1 で写す**
- **落ちるテストを先に書く**。それから実装して緑にする
- **全受入条件テストが緑になって初めて実装完了**とみなす
- **置き場は root の `conventions.md` の「配置の表」を引くだけ。新しいトップレベルの箱を作らない**
  （表に無いものが要ると気づいたら、その場で作らず `/pj:change` に投げる・concepts §12）

## モード別の振る舞い（dispatch は concepts §7）

### build（既定: `/pj:build <feature>`）

feature 名が無ければ next の結果から「どれを実装します?」と一言。手順:

1. **読む**: 対象 feature の spec（＋ `[[関連]]` 1階層）と `docs/design/` 一式。
2. **前提チェック**（欠けていたら警告して続行確認する）:
   - 対象 feature の `progress`（spec）が 3 未満／design に未決の技術論点が残る
   - **土台が無い**——`stack.md` の採用技術が入っていない／テストランナーが起動しない／対象が package なら
     `package.json` が無い。**この場合は `/pj:setup stack`・`/pj:setup sync` へ戻す。ここで自分で FW や
     ランナーを入れ始めない**（特定 feature の build に土台が埋もれ、後から「なぜこの構成か」を辿れなくなる）。
3. **計画**: 実装単位に分解。**依存順を尊重**（多数から参照される基盤 = データモデル・共通エンティティを先に）。
4. **テスト先行**: 受入条件 → test を書く（落ちる状態で）。
5. **実装**: design の stack / conventions に従って緑にする。横断方針（conventions.md）を必ず踏襲。
   - **UI なら** `design-language.md` を正とし、満たす原則を共通コンポーネント先頭に `@satisfies DL-NN` で
     注釈する。完了前に **`pj:design-reviewer`** で違反ゼロを確認。
   - **対象に `docs/design/intake/<intake-id>/` があれば、実装前に `intake.md` と実体を必ず読む**
     （visual.md §V2）。**拘束範囲は忠実度が決める**: `sketch` = 構造と並び順 / `structure` = ＋挙動 /
     `hifi` = ＋明記された寸法・閾値・時間。
     - **`hifi` なら数値を落とさない。** 数値は `intake.md` の振り分け表から **AC** を辿る
       （表に無い数値を実体から目測で拾わない）。
     - **部品が DS に置き換わるのは正常。配置・列順・密度が変わるのは違反。**
     - DS を当てると出典の意図が壊れる箇所が出たら、**黙って折衷せず** `/pj:change` で DL-NN を1本足して記録する。
6. **同期**: 全受入条件テストが緑になったら —— 対象 feature の `build_progress` を更新（concepts §5）／
   **その product の** overview の2軸ダッシュボードの build 列を更新（**進捗の正はここ**。コア機能リストは
   触らない）／**package を実装したら続けて root の product 表も引き直す**（concepts §9）／feature.md の
   `## 変更履歴` に `YYYY-MM-DD: 実装（受入条件 N 件テスト緑）` を追記し `updated` を今日に。
7. コミット・push は**ユーザーが明示するまでしない**（プロジェクトの CLAUDE.md 準拠）。

### まとめて build（`/pj:build <A> <B> …` / `/pj:build packages`）

対象が**複数**のときのモード。**やることは上の既定 build と同じ**で、それを対象ごとに繰り返すだけ。
**新しい実装作法も新しい足切り基準も作らない**（判断は next と上の前提チェックが既に持っている）。

- **対象の決め方**: product 名が並べば それが対象。**`packages` / `all` のような範囲キーワードなら
  next の優先順位そのまま**（依存順 × spec readiness）。**ここで独自の閾値を発明しない。**
- **層に割る**。層は **root `docs/specs/overview.md` の product 表の「依存」列**から引く
  （宣言＝あるべき姿なので、まだ実装の無い package も層に割れる）。**実体から生成した依存グラフは使わない**
  （未実装分が落ちる）。
- **承認は最初の1回だけ**（このモードの本体）。開始前に「層 → 対象 → 本数」を一覧で出して確認を取る。
  **通ったら以降は product ごとに聞かない。** 最後の報告まで走り切る。
- **実行**: 同じ層の product は **1 product = 1 sub-agent で並列**。各 sub-agent は「その product の未緑
  feature を、既定 build 手順 1〜7 で緑にする」だけを負う。メインは要約だけ受け取る（concepts §14）。
- **書き込み境界（並列で壊さないための線）**: sub-agent が書いてよいのは **`packages/<name>/` 配下だけ**。
  **root の product 表・workspace 設定・lockfile は触らせない**——product 表の引き直しは**全層が終わってから
  メインが1回**、依存インストールが要るなら**層を跨ぐ前にメインがまとめて**実行する。
- **詰まっても止めない**: 前提チェックの欠け／受入条件が design と矛盾／テストが緑にならない のいずれかに
  当たったら、その product を **blocked** として記録して**次へ進む**。**バッチ中に `/pj:change` や
  `/pj:setup` を起動しない**（対話が要るので必ずそこで止まる）。**緑でないものを緑と記録しない**——
  書きかけは落ちるテストごと残し、`build_progress` は実際のテスト緑率のまま置く（concepts §15）。
- **報告**: **1本終わるたびに1行**出す（`[4/9] workflow/core … BLOCKED — wf-AC-07 の前提が design と矛盾`）。
  最後に `緑 N 本 / blocked M 本` を集約し、blocked は1本ずつ**理由と次の一手**を出す。
  **止まらないことと黙ることは別。**

### 軽量修正（実装中に仕様/設計の誤りに気づいたとき）

build で勝手に辻褄を合わせない。**`/pj:change <気づいたこと>` に投げる**（concepts §8 典型シナリオ1）。

### status / next

- status は2軸ダッシュボード（spec / build）。「spec 済・未実装」「実装中」「テスト緑」を一目で。
- next は次に実装すべき feature を提案（優先: **依存順（基盤が先）× spec readiness の高い順**）。
  spec が 3 未満の feature は「先に /pj:spec で詰める」と促す。

### plan（`/pj:build plan <feature>` / `plan <product…>` / `plan packages`）

コードは書かず、上記「計画」までを出す（分解・依存順・受入条件→テスト一覧・想定リスク）。
**対象が複数なら**層割り・対象一覧・各 product の受入条件数まで。数が多いときは先にこれを見てもらう。

### review / audit

- review はコード品質の確認。**専用 agent を作らず既存の `/code-review` `/security-review` `/verify` に委譲**する。
- audit は **concepts §14「audit の起動仕様」に従う**。実装と仕様/設計の乖離を報告し、**product 境界の違反は
  その前に**出す。編集しない。

## 進捗管理

各 feature の `build_progress` を concepts §5 の軸で測る。基準は明快: **0 = 未着手 / 進行中はテスト緑率に応じて /
5 = 全受入条件テスト緑**。spec の `progress` は build ではさわらない。`updated` は必ず今日の日付。
状態は必ず overview の2軸ダッシュボードに反映する。
