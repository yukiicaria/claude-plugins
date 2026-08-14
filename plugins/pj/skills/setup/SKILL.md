---
name: setup
description: pj パイプラインで「決めたことを実体にする」担当。引数なしで新規プロジェクトの足回り（git/lint/format/husky）＋ pj-managed な CLAUDE.md ＋ デザインシステムを一括で用意し、package <name> でライブラリの器を packages/ に生やし、stack で design が決めた技術（FW/DB/ORM/テストランナー）をインストールして配線し、sync で docs と実体のズレ（器の無い product・workspace 未登録・lint 未設定）を埋める。Day 0 だけでなく決定が起きるたびに呼ばれる。
argument-hint: "[<なし=フル立ち上げ> | ds | package <name> | stack（design の決定を実体化） | sync（docs と実体のズレを埋める）]"
allowed-tools: Bash(*), Write, Edit, Read, Glob
---

**決めたことを実体にするコマンド。** docs に書かれた決定（product が増えた・技術が決まった）を、実際の
ファイル・依存・設定に落とす。**フェーズではない**——spec / design / build に直交し、決定のたびに呼ばれる。

**中身（docs 本体）は作らない。器と土台だけを用意して spec / design に引き渡す。**

引数: $ARGUMENTS

## モード（＝この skill の目次）

| モード | やること | 完了条件 | モデル起動時 |
|---|---|---|---|
| **引数なし** | A〜E（フル立ち上げ） | 足回り＋ pj-managed な CLAUDE.md ＋ DS。`/pj:spec` を叩ける | **一行確認** |
| **`ds`** | E だけ | DS が入り、UI 作業時に必ずそれを見る導線ができている | **一行確認** |
| **`package <name>`** | F だけ | 器が生え、workspace に載り、依存方向が lint で塞がれている | **無確認** |
| **`stack`** | G だけ（design の後に呼ぶ） | `stack.md` の採用技術が入って配線され、**テストが1本走る** | **一行確認** |
| **`sync`** | H だけ（**冪等**。何度でも可） | docs に在る product が全部実体を持ち、workspace と lint が追いついている | **検出は無確認・埋める前に一覧報告** |

> **`stack` と `sync` は「後から」呼ばれる前提のモード。** pj は spec → design の順に docs を育てるので
> **実体はどうしても後追いになる**。それを異常として扱わず、**追いつく手段を常に用意する**のが役割。

パッケージマネージャーは **bun**（npm / npx は使わない）。既存ファイルは壊さず**不足分のみ追記**する
（A〜E・G 共通）。

## モデル起動時のゲート（`/pj:setup` と打たれた起動では読み飛ばす）

ユーザーが明示的に `/pj:setup ...` と入力したなら、この節は**丸ごと無視**する。
**モデル（Claude）の判断で起動したときだけ**通る。

- 扱いは**モードで分かれる**（上の表の右端）。軸は**コストと不可逆性**であって好みではない——
  `package` は器を生やすだけで判断の余地が無く、`stack` / `ds` / フル立ち上げは依存を実際に install して
  repo の骨組みを触る。
- 確認は AskUserQuestion で**1問だけ**。提示するのは「**対象 product ／ 何を入れるか ／ どこを触るか**」を1文で。
- **承認が得られなければ何も書かずに降りる。** 代案を押し売りしない。
- **`package` を無確認にしたのがモデル起動可にした主目的**——「これは package だ」と気づくのは会話の途中で、
  そこに居るのはモデル。ここでコマンドを要求すると **docs だけ手で書かれて器が永久に生まれない**（concepts §1）。

## 最初に必ず読む

- **`../../_shared/concepts.md`**（背骨）— 起動したらまず読む。
- **`package` モードなら `../../_shared/product-model.md`** — そこで作る器は「package は app を知らない」と
  「可搬」を構造として持たせるものなので、規約を外すと意味がない。
- **`stack` モードなら対象 product の `docs/design/stack.md`** — **そこが入力**。書かれていない技術を
  勝手に入れない（決まっていなければ `/pj:design` に戻す）。
- **DS を導入する（E）なら `../../_shared/visual.md` §V1**。UI を持たない（CLI 等）なら不要。

---

## A. Git アカウント設定（確認 ①）

git リポジトリでなければ `git init`。**アカウントの識別情報はプラグインに持たせない**（プラグインは
共有されうるので、個人のメールアドレスや SSH key パスを焼き込まない）。上から順に:

1. **プリセットがあれば使う**: `~/.claude/pj-git-accounts.md` があれば読み、表のキーを選択肢として提示する
   （「どのアカウントで作りますか: <キー1> / <キー2>」）。選ばれた行の `user.name` / `user.email` を
   `git config` に設定し、remote origin があればその行の **remote 形式**に URL を合わせる。
2. **プリセットが無く、既に `git config` 済み**なら、それをそのまま使う（何も聞かない）。
3. **どちらも無い**場合だけ、`user.name` と `user.email` を1回聞いて設定する。

> pj はこのファイルの存在を前提にしない（無ければ作らなくてよい）。

## B. 足回りのファイル（無ければ作成・あれば不足分のみ追記）

1. **.gitignore** — node_modules / bun.lockb / dist / build / out / .next / .cache / .env* / *.log /
   .DS_Store / coverage / *.tsbuildinfo / .vscode / .idea / **.claude/settings.local.json** / **design-inbox/**
2. **package.json**（name = ディレクトリ名）— scripts: `format`(prettier --write .) / `lint`(eslint .) /
   `type-check`(tsc --noEmit) / `prepare`(husky)。devDeps: husky / lint-staged / prettier / eslint /
   @eslint/js / @typescript-eslint/{eslint-plugin,parser}。
   lint-staged: `*.{ts,tsx}`→eslint --fix+prettier、`*.{js,jsx,mjs,cjs}`→同左、`*.{json,css,md}`→prettier。
3. **tsconfig.json** — strict / ESNext / bundler / outDir dist / rootDir src。
4. **eslint.config.js** — @eslint/js recommended ＋ @typescript-eslint recommended。
5. **.prettierrc** / **.prettierignore** — semi / singleQuote / printWidth 100 / tabWidth 2。
6. **design-inbox/** — **外から来た設計成果物の受け口**（visual.md §V2）。Artifact のエクスポート・
   ハンドオフ・スクショを**そのまま置く場所**で、**gitignore する**（作業場であって正ではない）。
   `README.md` を1枚置き、次を書く:

   > ここに置いたものは `/pj:design intake <パス>` で取り込む。取り込むと、忠実度を宣言したうえで
   > `<product>/docs/design/intake/<id>/` に**出典として保存**され、中身は `AC-NN` / `DL-NN` に振り分けられる。
   > **取り込むまでは何の拘束力も無い。** 取り込んだらここからは消してよい。

   **`docs/design/intake/` は先回りして作らない**（取り込むときに `/pj:design intake` が作る）。
   UI を持たないプロジェクトでは design-inbox ごと不要。

## C. .claude/ 構成

`.claude/settings.json`（git 管理・チーム共有）と `.claude/settings.local.json`（git 無視・個人用）を作る。
permissions.allow は Bash/Read/Write/Edit/Glob/WebFetch/WebSearch/Agent/NotebookEdit のベースライン
（既存があればマージ）。**pj はプラグインなので、空の skills/ agents/ hooks/ は作らない。**

## D. ルート CLAUDE.md を pj-managed として生成

CLAUDE.md が無ければ作り、**マーカー外（人間の領域）**に検出できた情報を埋める（Overview / Tech Stack /
Architecture はスタブ可）＋ Commit Rules（husky の pre-commit/pre-push）・Branch Strategy（main 直 push 禁止 /
develop 起点 / feature・fix）・「bun を使う／`--no-verify` 禁止」。

そのうえで **`../../_shared/templates/managed-block.md` の正準ブロックを stamp** する（concepts §17）。
**CLAUDE.md が既存なら中身は作り直さず、stamp だけ行う。**

## E. デザインシステム導入（確認 ②・DS 非依存）

> 目的: **Claude が UI 作業をするとき、勝手にアイコン・色・コンポーネントを発明せず、決められた語彙に従う**
> 状態を作る。仕組みは特定 DS に依存しない「**3スロット契約**」（正は visual.md §V1）。

| スロット | 入れ方 |
|---|---|
| **実体** | DS パッケージを install／preset は最小トークンを scaffold |
| **導線** | `src/components/CLAUDE.md` に DS の AI ドキュメント（llms.txt 等）を指すポインタを置く（UI 作業時に自動ロード） |
| **正の宣言** | `docs/design/design-language.md` §F に「語彙・トークンの正 = <DS>」を書く |

ユーザーに確認する: 「デザインシステムは? **① 既存を入れる（例: oyster-lib）／② preset（外部DSなし）**」。
UI を持たない（CLI 等）なら**スキップ**し、design-language は N/A 宣言にする。

**① 既存 DS**
1. `bun add <DS パッケージ>`。CSS が要れば import を案内。install 方式（公開 npm / GitHub Packages の
   認証等）が不明なら**一言だけ確認**する。
2. `../../_shared/templates/components-CLAUDE.md` を `src/components/CLAUDE.md` に置き、`{{DS}}` 等を
   実 DS（パッケージ名・AI ドキュメントのパス・アイコンの単一窓口）で埋める。
3. `design-language.md` が無ければ `../../_shared/templates/design-language.md` から作り、§F に
   「**ベース語彙・トークンの正 = <DS>**」を明記。**DS が決めない差分だけ DL-NN を採番**する。

**② preset（外部 DS なし）**
1. `../../_shared/templates/tokens.css` を `src/styles/tokens.css` に置く。React なら既定アイコンとして
   **lucide-react** を入れてよい。
2. `components-CLAUDE.md` を置き、`{{DS}}` を「self / preset」、AI ドキュメントの参照先を
   `docs/design/design-language.md` に向ける。
3. `design-language.md` §F のプリミティブ族（**アイコン筆頭**）に preset の既定を種付けし、そこから
   **DL-NN で全部を育てる**（preset は DL-NN が厚くなる）。

どちらでも **§F の「単一窓口」原則**（アイコン = 単一ライブラリ／色 = トークンのみ・生値禁止）を明記する。

> **lint によるブロック強制（生 color・生 svg・別ライブラリ）はここでは入れない。** 当面は導線
> （design-language ＋ CLAUDE.md ポインタ）と audit の `pj:design-reviewer` で運用する
> （まず warning、ブロックは規律が固まってから）。

---

## F. package product の器を作る（`/pj:setup package <name>`）

> 「◯◯をライブラリとして切り出したい」「共通化して package にしたい」と言われたらここ。
> **大前提: package は app を知らない。** 作る器はその制約を**構造として**持つ。
> **中身（spec / design）は作らない。** 器だけ作って `/pj:spec` に引き渡す。

### F-1. 事前確認

1. **root product であることを確認**（`docs/specs/overview.md` が repo 直下）。**既存 product の配下から
   呼ばれていたら止める**（product の中に product は作らない・concepts §2）。
2. **`<name>` に `/` が含まれていたらグループ配下**（`packages/<group>/<name>/`）。グループ自身は
   product ではなく、深さは `packages/` 配下2段まで。**先回りしてグループを作らない。**
3. **名前の重複チェック**（`packages/<name>/` が既にあれば止めて既存を案内）。
4. **feature 名と衝突していないか**確認する（repo 内一意・concepts §4）。衝突していたらどちらを改名するか一言確認。

### F-2. 器を作る

```
packages/<name>/
  package.json          name は root の scope に合わせる（例 @<scope>/<name>）。
                        dependencies に **app を書かない**。依存する他 package だけを workspace 依存で書く
  tsconfig.json         **自己完結させる**
  <test>.config.ts      この package だけで走るテスト設定
  src/index.ts          公開 API の入口（空でよい）
  CLAUDE.md             ../../_shared/templates/package-CLAUDE.md から
  docs/
    specs/
      overview.md       ../../_shared/templates/package-overview.md から（progress: 0）
      glossary.md       ../../_shared/templates/glossary.md から（**この package の語彙だけ**）
      CLAUDE.md         ../../_shared/templates/specs-CLAUDE.md から
    design/
      CLAUDE.md         ../../_shared/templates/design-CLAUDE.md から
```

**可搬（product-model.md §P2）を最初から満たす。** 後から足すと root に寄せた設定を1つずつ剥がす作業になり、
hoisting が不足を隠すので**壊れていても静かに壊れる**。特に: **devDependencies（テストランナー・型定義・
testing-library）を必ず書く** ／ **tsconfig が app 固有設定（FW プラグイン・`paths`・`jsx`）を継承しない** ／
**`type-check` と `test` の script を両方書く** ／ **FW・DS は `peerDependencies`**。

- **`docs/design/conventions.md` と `design-language.md` は作らない**（root 所有・この package は準拠する側）。
  `docs/design/CLAUDE.md` にその旨を1行書く。
- **`docs/specs/features/` は作らない**（overview 1枚から始める・concepts §6）。
- data-model / stack は**必要になってから** `/pj:design` が作る（先回りしない）。

### F-3. workspace に載せる（install して使う形にする）

1. root の `package.json` の `workspaces` に **`"packages/*"` と `"packages/*/*"` の2本**が揃っているか
   確認し、無ければ追加する（グループを作れる以上、1本だけでは載らない）。
2. **app 側が使うときは workspace 依存**（`"@<scope>/<name>": "workspace:*"`）。相対パス import で使わせない。
3. `bun install` を流してリンクを張る。

### F-4. 依存方向を lint で塞ぐ

package.json の依存だけでは**相対パスでの脱出**（`../../src/...`）を止められない。root の eslint 設定に
ゾーン制約を足す（**設定の本体は product-model.md §P3**。`eslint-plugin-import` が無ければ入れる）。
既に同等のルールがあれば重複させない。**入らなかったらその旨を完了報告に明記する**（強制が1枚欠けた状態を黙って作らない）。

### F-5. root の product 表に1行足す

root の `docs/specs/overview.md` の product 表（concepts §9 上段）に行を追加する（表が無ければ作る）。
値は `spec: □□□□□ / build: □□□□□`、依存列は空。

### F-6. 引き渡し

「器ができたので `/pj:spec` で `<name>` の WHAT を書き始めてください」と案内する。**overview の中身は書かない。**

---

## G. design が決めた技術を実体化する（`/pj:setup stack`）

> **入力は対象 product の `docs/design/stack.md`。** **design はコードを書かない**ので、決定が実体になる
> 契機がここに要る。**書かれていない技術を勝手に入れない。**

### G-1. 事前確認

1. **対象 product を決める**（既定は root）。
2. **`stack.md` を読む。** 無ければ「先に `/pj:design` で stack を決めてください」と案内して**止める**。
3. **`<未定>` の行を数える。** 主要な軸（言語 / FW / DB / テスト）が未定なら報告して**確認を取る**
   （部分的に入れるか、design に戻るか）。
4. **既に入っているものは飛ばす**（何度走らせても壊れない）。

### G-2. 入れて配線する

**stack.md の表を依存関係の順に当てていく**:

```
1. FW / ランタイム    プロジェクトの骨組み（既存の src/ を壊さない）
2. UI / CSS          DS のスタイル読み込みをエントリで1回だけ（visual.md §V1）
3. DB / ORM          接続とマイグレーションの置き場（スキーマは各 package が持つ）
4. テストランナー      設定を置き、**空でよいので1本テストを通す**
5. その他             検証・フォーム・状態管理など stack.md に書かれているもの
```

**`conventions.md` の「フォルダ構成 / 命名」に従う。** package のスキーマを app が束ねる構成なら、
その置き場も作る（中身は空でよい）。

### G-3. 動くことを確かめる

**「入れた」で終わらせない**: `bun run type-check` が通る／**テストが1本走る**（空でよい。ランナー起動の確認）／
FW を入れたなら開発サーバーが起動する。

### G-4. 引き渡し

「stack が実体になったので `/pj:build <feature>` で実装を始められます」と案内する。
**stack.md に無いものを入れた場合は必ず報告する**（必要なら `/pj:design` で stack.md に追記してもらう）。

---

## H. docs と実体のズレを埋める（`/pj:setup sync`）

> **docs が先に進み、実体が後から追いつくのは正常。** このモードは**その差分だけ**を埋める。
> **docs の中身は一切書かない。**

### H-1. 検出する（探索は concepts §2 の3箇所）

```
① 器の無い product     docs/specs/overview.md はあるが package.json が無い
② workspace 未登録     package.json はあるが root の workspaces glob に載っていない
③ 依存方向の lint 無し  packages → src を塞ぐルールが root の eslint 設定に無い
④ product 表の欠落      実体はあるが root の docs/specs/overview.md の product 表に行が無い
⑤ 名前の衝突           feature 名が repo 内で一意でない（concepts §4）
```

**まず一覧にして報告してから直す**（黙って大量に生成しない）。

### H-2. 埋める

- **①** → F-2 の器を作る。ただし **`docs/specs/` が既にあれば上書きしない**（中身は spec が書いたもの）。
- **②** → F-3 ／ **③** → F-4 ／ **④** → F-5（値は各 product の overview から引く。**手で決めない**）。
- **⑤** → **直さない。報告して止める。** 改名は ID の鎖に影響するので `/pj:change` の管轄。

---

## I. 完了報告

作成・更新したファイルを一覧で報告し、次のアクションを案内する:

- **引数なし / `ds`** — `/pj:spec` で WHAT を書き始める。CLAUDE.md のマーカー外スタブを埋める
- **`package`** — 依存 lint が入ったか／workspace リンクが張れたかを明示し、`/pj:spec <name>` を案内
- **`stack`** — 入れたもの・**入れなかったもの（stack.md が未定の軸）**・型チェックとテストが通ったかを明示し、
  `/pj:build <feature>` を案内
- **`sync`** — **埋めたズレと埋めなかったズレを分けて**報告。ゼロになったときだけ「ズレはゼロ」と言う

> **どのモードでも「やったつもり」を報告しない。** 入れた・作ったではなく**通った・載った**を確認して書く。
> 通らなかったもの・入らなかったものは黙って残さず必ず明記する。
