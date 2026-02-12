# Skill: Deploy to GCP (Cloud Run + Docker + GitHub Actions)

## Purpose
Ship services to production reproducibly with:
- **Docker image** as the deployment unit
- **Cloud Run** as the runtime
- **GitHub Actions** as the CI/CD runner
- **.env (local dev)** / **GitHub Secrets + GCP Secret Manager (release)** for configuration

## SSOT
- CI/CD: `docs/05_operations/CI_CD.md`
- Release & deploy: `docs/05_operations/RELEASE_DEPLOY.md`
- Secrets & key management: `docs/05_operations/SECRETS_KEY_MANAGEMENT.md`
- Supply chain: `docs/skills/SKILL_SUPPLY_CHAIN_SLSA_SBOM.md`

## Rules (non-negotiable)
- **Cloud Run + Docker + GitHub Actions only** for deploy.
- **Local configuration** is managed via **`.env`** (not committed).
- **Release/production configuration** is managed via:
  - **GitHub Secrets**: CI/CD-time values (e.g. GCP project id, WIF provider, service account email, registry/repo identifiers)
  - **GCP Secret Manager**: runtime application configuration (DB URLs, API keys, OAuth secrets, encryption keys, and other env vars)
- **Do not use any external “environment variable management” services** besides GitHub Secrets and GCP Secret Manager.
  - 明記: **GitHub Secrets + GCP Secret Manager 以外の外部依存の変数管理サービスは使用しない**
  - Explicitly forbidden examples: Doppler, 1Password CLI, Vault SaaS, Parameter Store-like third parties, custom hosted secret UIs.
- **No service account key JSON** (long-lived keys) in repo, developer machines, or GitHub Secrets.
  - Use **Workload Identity Federation (WIF)** from GitHub Actions.

## Safe defaults
- Prefer **Artifact Registry** for container images.
- Use **immutable image tags** for releases (git SHA, semver), and deploy by tag.
- Inject configuration via Cloud Run environment variables **sourced from Secret Manager**.
- Keep “non-secret config” minimal; when in doubt, treat as config and store in Secret Manager to avoid drift.
- Enable structured logs and trace propagation; never log secret values.

## Secret & env-var policy (must match all environments)
### Local (developer machine)
- Store configuration in `.env`.
- `.env` MUST be in `.gitignore`.
- The application loads env vars from the OS environment; `.env` is only a convenience for local dev.

### Release / Production
- Runtime env vars are injected from **GCP Secret Manager**.
- GitHub Actions uses **GitHub Secrets** only for CI/CD wiring (authentication, project/repo identifiers), not as the runtime config source.
- Cloud Run MUST reference secrets (env var mapping) instead of hardcoding values in workflows.

## GitHub Actions (reference workflow outline)
Minimal outline (pseudo-config; adapt names to your repo):

- Trigger: on push to `main` and on tags (release)
- Auth to GCP: `google-github-actions/auth` via **WIF**
- Build/push image: Docker Buildx to Artifact Registry
- Deploy: `gcloud run deploy`

Example steps (illustrative):

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4

  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT_EMAIL }}

  - uses: google-github-actions/setup-gcloud@v2

  - run: |
      gcloud auth configure-docker ${REGION}-docker.pkg.dev

  - uses: docker/setup-buildx-action@v3

  - uses: docker/build-push-action@v6
    with:
      context: .
      push: true
      tags: ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}:${GITHUB_SHA}

  - run: |
      gcloud run deploy ${SERVICE_NAME} \
        --image=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}:${GITHUB_SHA} \
        --region=${REGION} \
        --platform=managed \
        --allow-unauthenticated=false \
        --set-secrets=DATABASE_URL=DATABASE_URL:latest,JWT_SIGNING_KEY=JWT_SIGNING_KEY:latest
```

Notes:
- Prefer `--allow-unauthenticated=false` by default; expose via Gateway/IAP/Load Balancer.
- `--set-secrets` supports `ENV_VAR=SECRET_NAME:version`. Use explicit versions for strict rollouts.

## Docker (safe defaults)
- Multi-stage build.
- Run as non-root.
- `PORT` from Cloud Run is honored.
- No `.env` copied into the image.

## “Banned tools” compliance (deployment/config-related)
- Config libraries: **viper** is forbidden. Use simple `os.Getenv` + explicit parsing/validation.
- Secret managers: **Doppler** (and similar) is forbidden; use **GitHub Secrets + GCP Secret Manager** only.
- Logging: avoid `fmt.Println` / `log.Println` / logrus; use `log/slog` (Go 1.21+).

## Checklist
- Cloud Run deploy uses an image from Artifact Registry.
- GitHub Actions auth uses WIF (OIDC), not a service account JSON key.
- Local config comes from `.env` and is not committed.
- Production config is stored in GCP Secret Manager and injected into Cloud Run.
- No third-party env-var management service is introduced.
- Release uses immutable image tags and can be rolled back by tag.
