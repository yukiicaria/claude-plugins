---
name: setup
description: pj パイプラインの Day 0（立ち上げ）レイヤー。新規プロジェクトの足回り（git/lint/format/husky）を一括で作り、ルート CLAUDE.md を pj-managed として生成し、デザインシステムを導入する。package <name> モードで、自分で作るライブラリ（package product）の器を packages/ に生やす。言語非依存。完了後は /pj:spec で WHAT を書き始める。
disable-model-invocation: true
argument-hint: "[<なし=フル立ち上げ> | ds（DSだけ入れ直す） | package <name>（ライブラリの器を作る）]"
allowed-tools: Bash(*), Write, Edit, Read, Glob
---

新規プロジェクトを **pj で回せる状態**にして生み出すコマンド。足回り（汎用の開発環境）を一括で整え、
ルート `CLAUDE.md` を **pj-managed として生成**し、**デザインシステムを導入**する。その後 `/pj:spec` で
WHAT を書き始めるのが標準導線。

引数: $ARGUMENTS

## 最初に必ず読む

共通の軸・モード・保守規律・**運用宣言の管理ブロック（§16）**は **`../../_shared/concepts.md`**
（pj パイプラインの背骨）。起動したらまず読む。**デザインシステムを導入する（E）なら
`../../_shared/visual.md` §V1（視覚の正・DS の3スロット契約）も読む。** UI を持たない（CLI 等）なら不要。

**`package <name>` モードなら concepts §2（product モデル）を必ず読む。**
そこで作る器は「package は app を知らない」を構造として持たせるものなので、規約を外すと意味がない。

setup は pj の**最上流（Day 0）**で、spec/design/build の**手前**に立つ。ここで作るのは「中身」ではなく
「中身を育てられる器」。

## ゴール（setup レイヤー）

- 言語・FW を問わず動く**汎用の足回り**（git / .gitignore / lint / format / 型チェック / husky）が整っている
- ルート `CLAUDE.md` が **pj-managed ブロック入りで存在**（pj で回すプロジェクトとして生まれている）
- **デザインシステムが入り、Claude が UI 作業時に必ずそれを見る導線**ができている
- → この状態で `/pj:spec` を叩けば、追加の環境構築なしに WHAT を書き始められる

**途中でユーザーに確認を取るのは「Git アカウント選択」と「デザインシステム選択」の2点だけ。**
それ以外は自動実行し、最後にまとめて報告する。パッケージマネージャーは **bun**（npm / npx は使わない）。

## モード

- **引数なし** → フル立ち上げ（下記 A〜E を全部）。既存ファイルは壊さずマージ／不足分のみ追記。
- **`ds`** → E（デザインシステム導入）だけを実行（既存プロジェクトに後から入れる・入れ直す）。
- **`package <name>`** → G（package product の器を作る）だけを実行。A〜E は動かさない。

---

## A. Git アカウント設定（唯一の確認 ①）

git リポジトリでなければ `git init`。**アカウントの識別情報はプラグインに持たせない**（プラグインは
共有されうるので、個人のメールアドレスや SSH key パスを焼き込まない）。手順は上から順に:

1. **プリセットがあれば使う**: `~/.claude/pj-git-accounts.md` が存在すればそれを読み、表のキーを
   選択肢としてユーザーに提示する（「どのアカウントで作りますか: <キー1> / <キー2>」）。選ばれたら
   その行の `user.name` / `user.email` を `git config` に設定し、remote origin があれば
   その行の **remote 形式**に URL を合わせる。
2. **プリセットが無く、既に `git config user.name` / `user.email` が設定済み**なら、それをそのまま使う
   （何も聞かない）。
3. **どちらも無い**場合だけ、`user.name` と `user.email` を1回聞いて設定する。

> プリセットを使いたい場合の書式は `~/.claude/pj-git-accounts.md` を参照（無ければ作らなくてよい。
> pj はこのファイルの存在を前提にしない）。

## B. 足回りのファイル（存在しなければ作成・あれば不足分のみ追記）

1. **.gitignore** — node_modules / bun.lockb / dist / build / out / .next / .cache / .env* / *.log /
   .DS_Store / coverage / *.tsbuildinfo / .vscode / .idea / **.claude/settings.local.json**
2. **package.json**（無ければ。name = ディレクトリ名）— scripts: `format`(prettier --write .) /
   `lint`(eslint .) / `type-check`(tsc --noEmit) / `prepare`(husky)。devDeps: husky / lint-staged /
   prettier / eslint / @eslint/js / @typescript-eslint/{eslint-plugin,parser}。
   lint-staged: `*.{ts,tsx}`→eslint --fix+prettier、`*.{js,jsx,mjs,cjs}`→同左、`*.{json,css,md}`→prettier。
3. **tsconfig.json**（無ければ）— strict / ESNext / bundler / outDir dist / rootDir src。
4. **eslint.config.js**（無ければ）— @eslint/js recommended ＋ @typescript-eslint recommended。
5. **.prettierrc** / **.prettierignore**（無ければ）— semi / singleQuote / printWidth 100 / tabWidth 2。

既存ファイルは `allow`・scripts・devDependencies など**不足エントリのみ追記**（既存は消さない）。

## C. .claude/ 構成

`.claude/settings.json`（git 管理・チーム共有）と `.claude/settings.local.json`（git 無視・個人用）を作る。
permissions.allow は Bash/Read/Write/Edit/Glob/WebFetch/WebSearch/Agent/NotebookEdit のベースライン。既存があれば
allow をマージ（既存は消さない）。**pj はプラグインなので、空の skills/ agents/ hooks/ は作らない**（プロジェクト固有の
ものが要るときだけ後で足す）。

## D. ルート CLAUDE.md を **pj-managed として生成**

CLAUDE.md が無ければ作る。**マーカー外（人間の領域）**に検出できた情報を埋める（Overview / Tech Stack /
Architecture はスタブ可）＋ Commit Rules（husky の pre-commit/pre-push）・Branch Strategy（main 直 push 禁止 /
develop 起点 / feature・fix）・「bun を使う／`--no-verify` 禁止」を書く。

そのうえで **concepts.md §16 の正準な管理ブロックを stamp** する（`<!-- pj:managed start -->`〜`end`）。
既にマーカーがあれば重複させない。これにより、**普通のセッションや新しい人も入口（`/pj:change`）に必ず着地**する。
CLAUDE.md が既存の場合は中身を作り直さず、管理ブロックの stamp だけ行う。

## E. デザインシステム導入（唯一の確認 ②・DS 非依存）

> 目的: **Claude が UI 作業をするとき、勝手にアイコン・色・コンポーネントを発明せず、決められた語彙に従う**状態を作る。
> 仕組みは特定の DS に依存しない「**3スロット契約**」で表す。どの DS でも・preset でも同じ枠に収める。

**スロット契約**（このスキルが各 DS をこの枠に当てはめる）:

| スロット | 役割 | 入れ方 |
|---|---|---|
| **実体** | 部品とトークンの本体 | DS パッケージを install／preset は最小トークンを scaffold |
| **導線** | UI 作業時に**自動ロード**され Claude を従わせる | `src/components/CLAUDE.md` に DS の AI ドキュメント（llms.txt 等）を指すポインタを置く |
| **正の宣言** | 視覚の正にベース語彙を固定 | `docs/design/design-language.md` §F に「語彙・トークンの正 = <DS>」を書く |

> **強制（lint で生 color・生 svg・別ライブラリをブロック）はこのスキルでは入れない。**
> 当面は導線（design-language / CLAUDE.md ポインタ）＋ audit の `pj:design-reviewer`（concepts §13）で運用する
> （oyster README と同じ「まず warning、ブロックは規律が固まってから」方針）。

ユーザーに確認する: 「デザインシステムは? **① 既存を入れる（例: oyster-lib）／② preset（外部DSなし）**」
UI を持たない（CLI 等）なら**スキップ**し、design-language は N/A 宣言にする（visual.md §V1）。

### ① 既存 DS を入れる
1. **実体**: `bun add <DS パッケージ>`（例: `@oyster-lib/ui` `@oyster-lib/core`）。CSS が要れば import を案内
   （oyster なら `import '@oyster-lib/ui/styles.css'`）。install 方式（公開 npm / GitHub Packages の認証等）が
   不明なら**一言だけ確認**する。
2. **導線**: `../../_shared/templates/components-CLAUDE.md` を `src/components/CLAUDE.md` に置き、`{{DS}}` 等の
   プレースホルダを実 DS（パッケージ名・AI ドキュメントのパスや llms.txt・アイコンの単一窓口）で埋める。
3. **正の宣言**: `docs/design/design-language.md` が無ければ `../../_shared/templates/design-language.md` から作り、
   §F に「**ベース語彙・トークンの正 = <DS>**」を明記。**DS が決めない差分だけ DL-NN を採番**する（oyster の
   `Icon`＝Lucide のように、DS がプリミティブ族を持つならそれを単一窓口として §F に書く）。

### ② preset（外部 DS なし／自作の出発点）
1. **実体**: `../../_shared/templates/tokens.css` を `src/styles/tokens.css` に置く（color/spacing/type の最小トークン）。
   アイコンの既定ライブラリとして **lucide-react** を入れてよい（UI が React の場合）。
2. **導線**: `components-CLAUDE.md` を `src/components/CLAUDE.md` に置き、`{{DS}}` を「self / preset」、
   AI ドキュメントの指す先を `docs/design/design-language.md` に向ける。
3. **正の宣言**: `design-language.md` を置き、§F のプリミティブ族（**アイコン筆頭**）に preset の既定を種付けし、
   そこから **DL-NN で全部を育てる**（preset は DL-NN が厚くなる）。

どちらのパスでも、**design-language §F の「単一窓口」原則**（アイコン = 単一ライブラリ／色 = トークンのみ・生値禁止）を
明記する。これが UI 作業時に `src/components/CLAUDE.md` 経由で自動ロードされ、「勝手に発明しない」を導線で支える。

---

## F. package product の生成（`/pj:setup package <name>`）

> **これは「自分で作るライブラリ」の器を生やすモード**（concepts §2）。
> 「◯◯をライブラリとして切り出したい」「共通化して package にしたい」と言われたらここ。
> **中身（spec / design）は作らない。** 器だけ作って `/pj:spec` に引き渡す（このスキルの一貫した役割）。

**大前提: package は app を知らない。** ここで作る器は、その制約を**構造として**持つ
（package.json の依存に app を書かない・lint で相対パス脱出を塞ぐ）。

### F-1. 事前確認

1. **root product であることを確認**（`docs/specs/overview.md` が repo 直下にある）。
   **既存 product の配下から呼ばれていたら止める**（「product の中に product は作らない」・concepts §2）。
2. **グループを使うか決める**。`<name>` に `/` が含まれていたら
   `packages/<group>/<name>/` に作る（例: `/pj:setup package organization/evaluation`）。
   - **グループ自身は product ではない**（`docs/specs/overview.md` を置かない。ただのフォルダ）
   - 深さは `packages/` 配下2段まで
   - **先回りしてグループを作らない。** 数が少ないうちはフラットでよい
3. **名前の重複チェック**: `packages/<name>/` が既にあれば止めて既存を案内する。
4. **名前が feature 名と衝突していないか**確認する（concepts §4 の repo 内一意）。衝突していたら
   どちらを改名するか一言確認する。

### F-2. 器を作る

```
packages/<name>/
  package.json          name は root の scope に合わせる（例 @<scope>/<name>）。
                        dependencies に **app を書かない**。依存する他 package だけを workspace 依存で書く
  tsconfig.json         root の tsconfig を extends（rootDir: src / outDir: dist）
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

- **`docs/design/conventions.md` と `design-language.md` は作らない。** root 所有で、この package は
  **準拠する側**（concepts §3・§12）。`docs/design/CLAUDE.md` にその旨を1行書く。
- **`docs/specs/features/` は作らない。** overview 1枚から始め、育ったら割る（concepts §6）。
- data-model / stack は**この package が必要になってから** `/pj:design` が作る（先回りしない）。

### F-3. workspace に載せる（install して使う形にする）

1. root の `package.json` に `"workspaces": ["packages/*"]` が無ければ追加する。
2. **app 側が使うときは workspace 依存として書く**（`"@<scope>/<name>": "workspace:*"`）。
   相対パス import で使わせない —— これが「install して使う」を形にする部分。
3. `bun install` を流してリンクを張る。

### F-4. 依存方向を lint で塞ぐ

package.json の依存だけでは**相対パスでの脱出**（`../../src/...`）を止められない。root の eslint 設定に
ゾーン制約を足す（`eslint-plugin-import` が無ければ入れる）:

```js
// packages/ から app（src/）への参照を禁止する
'import/no-restricted-paths': ['error', {
  zones: [{ target: './packages', from: './src',
            message: 'package は app を知らない（pj concepts §2）' }],
}]
```

既に同等のルールがあれば重複させない。**この設定が入らなかったら、その旨を完了報告に明記する**
（強制が1枚欠けた状態を黙って作らない）。

### F-5. root の product 表に1行足す

root の `docs/specs/overview.md` の product 表（concepts §9 上段）に新しい行を追加する。
表がまだ無ければ作る。値は `spec: □□□□□ / build: □□□□□`、依存列は空。

### F-6. 引き渡し

「器ができたので `/pj:spec` で `<name>` の WHAT を書き始めてください」と案内する。
**このスキルは overview の中身を書かない。**

---

## G. 完了報告

作成・更新したファイルを一覧で報告し、次のアクションを案内する:
- **`/pj:spec`** で WHAT（要件）を書き始める（pj パイプラインの起点）
- CLAUDE.md のマーカー外スタブ（Overview / Tech Stack / Architecture）を埋める
- DS の差分が出てきたら `docs/design/design-language.md` に DL-NN を足す（`/pj:design` / `/pj:change`）
- **package モードの場合**: 依存 lint が入ったか／workspace リンクが張れたかを明示し、
  `/pj:spec <name>` を案内する

> このスキルは中身（docs/specs・docs/design 本体）を作らない。それは `/pj:spec` `/pj:design` の init が担う
> （概念の重複を避ける）。setup は器だけを用意して引き渡す。
