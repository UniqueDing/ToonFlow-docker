# Toonflow Docker

This repository only publishes [HBAI-Ltd/Toonflow-app](https://github.com/HBAI-Ltd/Toonflow-app) as a Docker image.

It does not vendor Toonflow source code or maintain a duplicate Dockerfile. GitHub Actions builds directly from the upstream release tag and uses the Dockerfile shipped by `HBAI-Ltd/Toonflow-app`.

## Local Build

For local testing, keep `Toonflow-app` next to this repository and build with Docker Compose:

```bash
cp .env.example .env
docker compose up --build
```

By default, compose uses `../Toonflow-app` as the build context. Change `TOONFLOW_APP_CONTEXT` in `.env` if your local Toonflow checkout lives somewhere else.

Or build manually from the upstream repository:

```bash
docker build -t toonflow-app:v1.1.7 \
  "https://github.com/HBAI-Ltd/Toonflow-app.git#v1.1.7"

docker run -d \
  --name toonflow \
  -p 10588:10588 \
  -v "$(pwd)/data:/app/data" \
  toonflow-app:v1.1.7
```

Open `http://localhost:10588/index.html`.

Default login from upstream docs:

- Username: `admin`
- Password: `admin123`

## GitHub Actions Publishing

The workflow at `.github/workflows/publish.yml` publishes to GitHub Container Registry:

```text
ghcr.io/<owner>/toonflow-app:<version>
ghcr.io/<owner>/toonflow-app:<upstream-tag>
ghcr.io/<owner>/toonflow-app:<major.minor>
ghcr.io/<owner>/toonflow-app:<major>
ghcr.io/<owner>/toonflow-app:latest
```

It runs every 6 hours, can be started manually with `workflow_dispatch`, and can be triggered immediately with `repository_dispatch` if the upstream repository sends an `upstream-release` event.

Scheduled behavior:

1. Reads the latest upstream GitHub release from `HBAI-Ltd/Toonflow-app`.
2. Checks whether `ghcr.io/<owner>/toonflow-app:<version>` already exists.
3. Builds the upstream release tag with the upstream project's own Dockerfile and pushes only when the tag is missing.

For stable semver releases, the workflow also updates `<major.minor>`, `<major>`, and `latest`. Prerelease builds keep only exact version/tag images and do not move `latest`.

No extra secrets are needed for GHCR in the same GitHub account or organization. The workflow uses the built-in `GITHUB_TOKEN` with `packages: write` permission.

If the upstream repository can cooperate, it can trigger this packaging workflow immediately after publishing a release by sending a repository dispatch event:

```bash
gh api repos/<owner>/ToonFlow-docker/dispatches \
  --method POST \
  -f event_type=upstream-release \
  -f client_payload='{"tag":"v1.1.7"}'
```

Without upstream cooperation, GitHub Actions cannot subscribe directly to another repository's `release` event, so scheduled polling is the automatic update path.

## Runtime Contract

- Container port: `10588`
- Persistent data path: `/app/data`
- Default env:
  - `NODE_ENV=prod`
  - `PORT=10588`
  - `OSSURL=http://127.0.0.1:10588/`

The upstream image entrypoint copies bundled seed data into `/app/data` without overwriting existing files. Keep the whole `/app/data` directory persisted, because it contains the built web UI, models, skills, database, and generated assets.
