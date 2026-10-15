# Day 14 — Docker in CI/CD

A small Flask API packaged with Docker and checked automatically by GitHub Actions.

## Pipeline

```text
Push / Pull Request
        |
        v
  Hadolint (Dockerfile lint)
        |
        v
     Build image
        |
        v
  Smoke test /health
        |
        v
 Trivy HIGH/CRITICAL scan
        |
        v
Push to Docker Hub (main branch only)
with the full Git commit SHA as the tag
```

## Repository files

- `app.py` — Flask API with `/` and `/health`
- `requirements.txt` — Python dependencies
- `Dockerfile` — container build instructions
- `.dockerignore` — excludes unnecessary files from build context
- `.github/workflows/docker-ci.yml` — automated pipeline

## Run locally

Requirements: Docker Desktop running.

```bash
docker build -t winter-arc-app:local .
docker run -d --name local-api -p 5000:5000 winter-arc-app:local
curl.exe -f http://localhost:5000/health
docker logs local-api
docker rm -f local-api
```

Expected health response: `ok`.

## GitHub and Docker Hub setup

1. Create a public or private Docker Hub repository named `winter-arc-app`.
2. Create a Docker Hub access token with the minimum permissions needed to push.
3. Push this project to a GitHub repository with the default branch named `main`.
4. In GitHub, open **Settings → Secrets and variables → Actions** and add:
   - `DOCKERHUB_USERNAME` — your Docker Hub username
   - `DOCKERHUB_TOKEN` — the Docker Hub access token (not your account password)
5. Push to `main`, then open **Actions** and inspect the `docker-ci` workflow.

Never put tokens or private keys in this repository. Pull requests from forks do not receive repository secrets; publishing is restricted to pushes to `main`.

## Verify the published image

Copy the full commit SHA from the successful workflow run. Replace `DOCKERHUB_USERNAME` and `FULL_COMMIT_SHA` below with your own values.

```bash
docker pull DOCKERHUB_USERNAME/winter-arc-app:FULL_COMMIT_SHA
docker run -d --name published-api -p 5001:5000 DOCKERHUB_USERNAME/winter-arc-app:FULL_COMMIT_SHA
curl.exe -f http://localhost:5001/health
docker logs published-api
docker rm -f published-api
```

Expected health response: `ok`.

## Rollback

Keep a record of the last known-good SHA before each release. To restore it on a test machine:

```bash
docker pull DOCKERHUB_USERNAME/winter-arc-app:PREVIOUS_GOOD_SHA
docker rm -f winter-app
docker run -d --name winter-app -p 5000:5000 --restart unless-stopped DOCKERHUB_USERNAME/winter-arc-app:PREVIOUS_GOOD_SHA
curl.exe -f http://localhost:5000/health
```

Replace `PREVIOUS_GOOD_SHA` with the actual earlier SHA. This is a manual rollback procedure for a test machine, not an automatic production deployment.

## Demonstrate a failure safely

Use a temporary branch and make `/health` return a non-success status or add a failing test. Push the branch and verify the workflow fails before publishing. Revert the temporary change afterward. Do not publish a vulnerable image.

## Evidence checklist

- [ ] Successful GitHub Actions workflow run
- [ ] Docker Hub repository showing the full SHA tag
- [ ] Local or published `/health` response showing `ok`
- [ ] README pipeline diagram and rollback instructions
- [ ] No tokens, SSH keys, or `.env` files committed

## Interview readiness

1. **Why use a Git SHA tag?** It connects an image to an exact source revision and supports reliable rollback.
2. **When should scanning happen?** After build and before publishing.
3. **Where do registry credentials belong?** In GitHub Secrets, never in source code.
4. **Docker Hub vs ECR?** Docker Hub is a general-purpose registry; ECR integrates with AWS IAM and services.
5. **How can builds be faster?** Use good layer ordering, BuildKit cache, a small build context, and multi-stage builds where useful.
6. **How do you roll back?** Redeploy a previously tested SHA-tagged image.
7. **Continuous delivery vs deployment?** Delivery keeps changes ready for release, often with approval; deployment automatically releases passing changes.
8. **Why not expose secrets to pull requests?** Untrusted code should not receive credentials.
9. **What is a pipeline gate?** A required check that stops later steps if it fails.

> Note: Trivy may correctly fail if it finds HIGH/CRITICAL vulnerabilities in the image. Review findings and update dependencies/base images rather than blindly bypassing the gate.
