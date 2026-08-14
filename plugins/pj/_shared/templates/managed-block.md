# 運用宣言の正準ブロック（concepts §17）

> **これはテンプレートではなく「正準テキスト」。** 下のマーカーごと、そのままプロジェクトの
> ルート `CLAUDE.md` に stamp する。**ブロック内は pj が生成し、人は手編集しない。マーカー外は保全する。**
> stamp の条件・冪等性のルールは `concepts.md` §17。

```markdown
<!-- pj:managed start — pj パイプラインが生成・更新する。ブロック内は手で書き換えない -->
## 運用モデル（pj パイプライン）

このプロジェクトは pj パイプラインで運用する。真実は docs/ の3層（下は上から導出、必ず上へ辿れる）:
- 振る舞い(WHAT) → `docs/specs/` ／ 技術・構造(HOW) → `docs/design/` ／ 実装(テスト先行) → `src/` ＋ tests

**単位は product**（spec+design+build を1周持ち、独立して作れる）。リポジトリのゴール本体が root product、
自分で作るライブラリは `packages/<name>/` に置く product。**feature は product の中の振る舞いの単位**。
- **package は app を知らない**（import も spec からの参照もゼロ）。app は package を install して使う。
  判定は2つ ——「**別の app がそのまま使えるか**」（語彙）と「**別 repo にコピーして
  `bun install` し、型検査とテストが通るか**」（可搬）。package が自分の責務のドメイン語彙を
  持つのは構わない。**可搬は最初から満たす**（devDependencies を書く・tsconfig を自己完結させる）。
- **package をいくつに割るかは、層の名前ではなく基準で決める**（依存の重さ・第三者が実装を書くか・
  利用者が選んで入れるか・テスト時間・オーナー）。当てはまらないなら割らず、
  1 package の中で lint により依存方向を強制する。
- 全 product が従う作法は `docs/design/conventions.md`（root）1箇所。これは準拠であって依存ではない。
- 新しいライブラリを作りたくなったら **`/pj:setup package <name>`**。

- **直したい・変えたいときの入口は `/pj:change <やりたいこと>` 一本**（層も product も意識しなくてよい）。自然文で言えば
  対象 product と変更の高度(L0コスメ〜L4根本)と正レイヤーを自動判定し、正しい層から直して下流（受入条件 ↔ test ↔ code）へ伝播する。
- 状況確認は `/pj:spec status`・`next`、客観チェックは `review`・`audit`。仕様=`/pj:spec`／設計=`/pj:design`／実装=`/pj:build`。
- 受入条件には安定 ID `(<feature>-AC-NN)` を振り、test がそれを参照する（鎖を grep で機械検証）。feature 名は repo 内で一意。
<!-- pj:managed end -->
```
