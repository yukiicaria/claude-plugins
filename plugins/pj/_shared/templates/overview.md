# <プロジェクト名>

## 目的
（このプロダクトで何を達成したいか）

## ターゲット / ユーザー
（誰のためのものか）

## コア機能

> ここは「**何があるか**」だけを持つ。**完了状態は持たない**（進捗の正は下のダッシュボード1箇所）。

- <feature-name> — <一行説明> · [詳細](features/<name>.md)

## 非機能要件
（パフォーマンス、セキュリティ、運用など）

## 制約 / 前提
（技術スタック、外部依存、納期など。技術の詳細は docs/design/ へ）

## products（この repo が持つ product 一覧）

> **この節は root product だけが持つ。** 値は各 product の overview からの**導出**であって独立した
> 事実ではない（正は各 product 側・concepts §9）。手で動かさない。
> 自分で作るライブラリを足すときは `/pj:setup package <name>`。

| product | 種別 | spec | build | 依存 |
|---|---|---|---|---|
| <repo 名> | app | ■■□□□ | □□□□□ | <使っている package> |
| <package 名> | package | □□□□□ | □□□□□ | <依存する package。app は書けない> |

> **`package → app` の依存は禁止**（concepts §2）。この列に app が現れたら違反。

## 進捗ダッシュボード（2軸: spec / build）

> この product 自身の feature の盤面。**進捗の正はここ**（上の product 表は導出）。

| 機能 | spec | build | 残課題 |
|---|---|---|---|
| <name> | ■■□□□ | □□□□□ | エラー時の挙動 / 未着手 |

**全体 spec 進捗**: ■■■□□ (平均 3.2/5)
**全体 build 進捗**: □□□□□ (平均 0/5)

> spec 軸は /pj:spec が、build 軸は /pj:build が更新する。build 5 = 全受入条件テストが緑。

## 未確定の全体論点
- <論点>
