# <package 名>

**これは独立したライブラリ（package product）。** この repo の app のために作っているが、
**app を知らない前提で作る**（pj concepts §2）。

## 絶対に守ること

- **app / 他 feature を import しない。** 相対パスで `packages/` の外へ出ない。
  必要なものは**引数かアダプタとして注入させる**。
- **app 固有の語彙をこの package の中に持ち込まない**（spec・glossary・型名・識別子すべて）。
  ただし**この package の責務そのものの語彙は持ってよい**（組織図ライブラリが「組織」を知るように）。
  判定は「**別の app がこのまま使えるか**」。語彙の正はこの package 自身の `docs/specs/glossary.md`。
- **依存は一方向のみ。** 他 package への依存は `package.json` に書いたものだけ。循環させない。
- **足回りを root から借りない**（可搬 / pj concepts §2）。
  - `devDependencies` に**テストランナー・型定義を自分で書く**。
    「root にあるから動く」は通らない —— workspace の hoisting が隠しているだけで、
    切り出した瞬間に動かなくなる
  - `tsconfig` は**自己完結**させる。app の FW プラグイン・`paths`・`jsx` 設定を継承しない
  - テスト設定は**この package だけで走る**こと。setup ファイルが app を指していたら違反
  - FW・DS ライブラリは `peerDependencies`（`dependencies` に入れない）

> 判断に迷ったら: 「これを別リポジトリにコピーして `bun install` したら、
> **型検査とテストがそのまま通るか**？」 通らないなら、まだ独立していない。
> **実際に試せる。** 疑わしいときは空ディレクトリにコピーして走らせる。

## 真実の場所

| 何 | どこ |
|---|---|
| 振る舞い（WHAT） | `docs/specs/overview.md`（育ったら `docs/specs/features/`） |
| 技術・構造（HOW） | `docs/design/` |
| 実装 | `src/` ＋ tests |
| **作法（全 product 共通）** | **repo root の `docs/design/conventions.md`**（準拠する。所有はしない） |
| **視覚の正**（UI を持つ場合） | **repo root の `docs/design/design-language.md`**（準拠する。所有はしない） |

## 変えたいとき

`/pj:change <やりたいこと>` 一本。層も product も自動判定される。
