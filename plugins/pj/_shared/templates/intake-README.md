# 取り込んだ設計成果物（索引）

外から来た設計成果物を**出典として保存**した場所。方法論は pj プラグインの `_shared/visual.md` §V2。

> **出典は正ではない。** 正は、振り分けて `AC-NN` / `DL-NN` になったものだけ。
> **どこまで再現するかは忠実度（fidelity）が決める** — `sketch` は構造だけ、`hifi` は寸法まで。

## 一覧

| 取り込み | 対象 | 忠実度 | 実体 | 備考 |
| --- | --- | --- | --- | --- |
| `<intake-id>` | <画面 / 部品 / ライブラリ> | `<fidelity>` | [`<intake-id>/`](<intake-id>/) | <一行> |

各件の詳細（出典・振り分け表・拾わなかったもの）は `<intake-id>/intake.md` にある。

## 置き場と構成のルール

```
docs/design/intake/
  README.md              # このファイル（索引）
  <intake-id>/           # 1件1ディレクトリ。kebab-case（成果物の名前。feature 名に縛らない）
    intake.md            # この取り込みの正
    <実体>               # index.html / screenshot.png / api.md など
```

- **実体はここに置く**（URL 参照だけにしない）。再現するには正確な寸法・閾値・算出式を読める必要がある。
- **同梱アセットは剥がす。目安 100KB / 件。** DS バンドル・フォント・画像は参照に留める。
- **原典（Artifact の URL 等）は `intake.md` に必ず残す。**
- **product ごとに持つ。** 画面は root、部品やライブラリはその package。

## 受け口

外から来たものはまず **`design-inbox/`**（リポジトリ直下・gitignore 済み）に置き、
`/pj:design intake <パス>` で取り込む。取り込んだら inbox からは消してよい。

## 変更履歴

- YYYY-MM-DD: <変更内容>
