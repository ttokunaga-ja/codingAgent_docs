# CI/CD（Frontend）

このドキュメントは、フロントエンド（Next.js + FSD）の CI/CD で **最低限守るゲート** を固定します。

目的：
- 境界違反（FSD）が本番へ入る
- 生成物ズレ（OpenAPI/Orval）
- `next build` は通るが本番で壊れる
を早期に止める。

関連：
- FSD境界：`../01_architecture/FSD_LAYERS.md`
- API契約：`../03_integration/API_CONTRACT_WORKFLOW.md`
- テスト：`../04_testing/TEST_STRATEGY.md`

---

## 結論（Must）

- `lint`（boundaries含む）/ `typecheck` / `test` / `build` を必須ゲートにする
- 翻訳キーの不足・未使用キーを検出する i18n チェック（例：`i18next-scanner` / 専用スクリプト）を CI に組み込み、UI文言のハードコーディングを防ぐ
- API生成は **差分検知** をCIで行う（生成し忘れ防止）
- main へのマージは “再現可能なビルド” が条件

---

## 最低限のパイプライン（テンプレ）

PR（必須）：
1. `npm ci`
2. `npm run lint`
3. `npm run typecheck`
4. `npm test`
5. `npm run build`
6. （採用するなら）`npm run api:generate` → diff チェック
7. （最小の）Playwright smoke（主要導線だけ）

main（任意）：
- デプロイ（staging → production）
- 監視（エラー率/Vitals）を確認し、ロールバック判断できる状態

---

## API生成のCIチェック（推奨）

目的：OpenAPI/OrvalのズレをPRで止める。

- CIで `npm run api:generate` を実行し、生成物に差分が出ないことを確認
- 差分が出たら、生成し忘れ or OpenAPIの更新漏れ

---

## i18n / 翻訳ファイルのCIチェック（推奨）

目的：UI文言のハードコーディングや翻訳漏れを早期に検出し、PRでブロックする。

- `npm run i18n:extract` を実行して翻訳キーを抽出（`i18next-scanner` の使用を推奨）
- `npm run i18n:check` を実行して、コードで参照されている翻訳キーが `public/locales/{lang}/*.json` に存在するかを検証する（missing keys は CI でエラーにする）
- `npm run lint` で `react/jsx-no-literals` を有効にして、JSX中のハードコーディング文字列を防ぐ

GitHub Actions のサンプルワークフローは `.github/workflows/i18n-check.yml` を参照してください。
---

## キャッシュとビルド（注意）

- `next build` は production の前提（開発サーバで動く＝本番OKではない）
- 動的化（CSP nonce 等）でコスト/キャッシュが変わるため、
  意図したレンダリング戦略になっているか確認する

関連：`../03_integration/SECURITY_CSP.md`

---

## 禁止（AI/人間共通）

- boundaries 違反を例外設定で握りつぶす
- 生成物を手編集する
- build を通さずにマージする
