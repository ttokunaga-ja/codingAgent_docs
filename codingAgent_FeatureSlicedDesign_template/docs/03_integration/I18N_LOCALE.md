# i18n / Locale Policy（Optional）

このドキュメントは、多言語・ロケールを扱うプロジェクトで
ルーティング/フォーマット/メタデータの責務を固定するための契約です。

---

## 結論（Must when i18n enabled）

- ルーティング方式（path prefix など）をプロジェクトとして固定する
- 文字列は “画面直書き” を禁止し、辞書/キーで管理する
- UIに表示する文字列は**すべて翻訳キー（変数）として管理**し、表示内容は各言語ごとのJSONファイル（例：`public/locales/{lang}/common.json` または `src/locales/{lang}.json`）から読み出すことを**必須**とする
- 日付/数値/通貨フォーマットは `shared/lib` に集約する

---

## 実装指針（推奨）

- 推奨ツール: `i18next` / `react-i18next`（またはプロジェクトで合意したi18nライブラリ）
- コンポーネント内では `useTranslation()` / `t('slice.key')` を使用し、コンポーネントに文字列を直書きしない
- 文字列の連結やフォーマットロジックを避け、パラメータ挿入（例: `t('greeting', { name })`）やプレースホルダを利用する
- 翻訳JSONはスライス/ドメインごとに分割し、共通文言は `common.json` に集約する
- 新規の画面・要素を追加する際は必ず翻訳キーを用意し、デフォルト（英語等）と対象言語のJSONを更新する

---

## ルーティング（Should）

- locale は URL で表現するか（例：`/ja/...`）を決める
- `generateMetadata` で locale ごとの title/description を正しく出す

---

## フォーマット（Must）

- `Intl.DateTimeFormat` / `Intl.NumberFormat` を `shared/lib` に集約
- 表示用フォーマットを slice 内でばらまかない

---

## 禁止（AI/人間共通）

- locale が増えたときに一括置換できない構造
- 日付/数値のフォーマットロジックがページごとに散る
