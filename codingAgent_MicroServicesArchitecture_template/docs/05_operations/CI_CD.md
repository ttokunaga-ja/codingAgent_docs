# CI_CD

## 目的
変更を安全に本番へ届けるための最小CI/CD要件を定義する。

## CI（必須）
- Unitテスト
- Integrationテスト（必要なモジュールのみ。Testcontainers活用）
- 契約（.proto / OpenAPI）と生成物の整合性チェック（再生成差分の検出）
- Atlas差分の検証（plan相当）

## セキュリティ（推奨）
- 依存関係の脆弱性検査（Go / Node）
- コンテナイメージスキャン
- SBOM生成（必要に応じて成果物へ添付）
- secret scanning（コミット/PR）
- SAST（静的解析。適用範囲は段階的に拡張）
- 設定/IaCの検査（k8s/コンテナ設定のlint等）
- CI 実行環境の最小権限化（OIDC等の短命クレデンシャルを優先）

> 供給網（サプライチェーン）観点の詳細は `SUPPLY_CHAIN_SECURITY.md` を参照。

## CIの推奨ステージ（例）
### 1) Lint / Static
- Go lint（方式はプロジェクトで統一）
- 依存・生成ツールの整合（tooling を固定している場合は差分が出ないこと）

### 1.5) Security（推奨）
- secret scanning
- SAST
- 依存スキャン（SCA）/コンテナスキャン

### 2) Contract / Codegen
- `.proto` を更新した場合:
	- `buf lint`（推奨）
	- `buf breaking`（推奨。比較対象は `main` またはリリースタグ）
	- `protoc` / `buf generate` の再実行で差分が出ないこと（生成物手編集の検出）
- OpenAPI を更新/生成した場合:
	- `docs/openapi.yaml` の再生成で差分が出ないこと
	- Orval 等のフロント向け生成を前提に、破壊的変更がないこと（レビューで確認）

> 補足: Buf の breaking 検査は `google.api.http` などのカスタムオプション変更を互換性判定に含めないため、
> 外向き契約（OpenAPI）は「生成物差分検出」を CI に含める。

### 3) Tests
- `go test ./...`
- Integration（DB/ES 等）: Testcontainers で必要最小の範囲を実行

### 4) Provenance / Attestation（推奨）
- SBOM と provenance（出自）を生成し、成果物に紐づける
- 可能なら CD 側で「署名/検証」をゲートにする

## CD（推奨）
- ステージング自動デプロイ
- 本番は承認付き（小規模なら手動トリガーでも可）

## 関連
- `05_operations/SUPPLY_CHAIN_SECURITY.md`
- `03_integration/CONTRACT_TESTING.md`
- `05_operations/VULNERABILITY_MANAGEMENT.md`

