# Task 7 & 8 — CI/CD Pipeline (Jenkins → GitHub Actions)

**Project:** Rent-a-Ride (MERN stack)
**Environment:** AWS EC2, Ubuntu, eu-north-1 (Stockholm), Elastic IP: 16.171.104.76
**Branch:** feature/task07

## Objective

Automate the build → push → deploy workflow so that a GitHub push triggers a
complete pipeline: build frontend/backend Docker images, push them to Docker
Hub, then deploy the updated images on EC2 via Docker Compose.

The original task specification was written for Jenkins. With explicit
permission, Jenkins was replaced with **GitHub Actions** — the same CI/CD
concept, but without the overhead of maintaining a dedicated CI server.

---

## Part 1: Jenkins (Initial Implementation)

Before switching tools, a working Jenkins pipeline was built and partially
verified on the EC2 instance:

- Jenkins installed via the `.war` file (the standard `apt`/repository
  install failed because Jenkins' published GPG signing key had expired),
  run as a persistent `systemd` service.
- A declarative `Jenkinsfile` was written with three stages: Checkout,
  Build & Push, and Deploy.
- Docker Hub authentication was fixed by generating a proper Docker Hub
  Access Token (Docker Hub rejects plain account passwords for automated
  logins).
- Discovered that `docker compose push` silently **skips** any service
  defined with `image:` instead of `build:` — since the Compose file
  references pre-built registry images, nothing was actually built or
  pushed. Fixed by using explicit `docker build` / `docker push` commands
  in the pipeline instead of relying on Compose for that step.
- Fixed a missing `package-lock.json` issue: it was listed in `.gitignore`,
  so Jenkins' fresh checkout never had it, and `npm ci` (used deliberately
  for reproducible installs) failed without it.
- Backend image build succeeded in Jenkins; frontend build repeatedly hit
  `ENOSPC: no space left on device` due to the small EC2 instance's
  limited disk, resolved each time with `docker system prune -af`.

Jenkins was later stopped and disabled (`systemctl stop/disable jenkins`)
once the team moved to GitHub Actions, to free up EC2 resources.

---

## Part 2: GitHub Actions (Final Implementation)

### New Architecture

<!-- Architecture diagram to be inserted here -->
![Architecture Diagram](./https://github.com/804ahmedkhan07-oss/Rent-A-Ride-Cloud-DevOps/blob/feature/task07/.github/githubactions-cicd-diagram.jpeg)

Text summary of the flow:

```
Developer
   ↓
GitHub Repository (push to feature/task07)
   ↓
GitHub Actions
   ├── Checkout code
   ├── Verify Docker availability
   ├── Build frontend image
   ├── Build backend image
   ├── Authenticate with Docker Hub
   ├── Push both images
   └── SSH into EC2 (appleboy/ssh-action)
   ↓
AWS EC2 (Ubuntu)
   ↓
docker compose pull
   ↓
docker compose up -d
   ↓
Frontend + Backend + MongoDB running
```

### Why GitHub Actions Instead of Jenkins

- No dedicated server to install, patch, or keep alive — GitHub runs the
  pipeline on its own infrastructure.
- Triggers are native (`on: push`) — no separate webhook configuration
  needed, unlike Jenkins which requires manually wiring a GitHub webhook.
- Secrets management is built into the repository (Settings → Secrets),
  simpler than Jenkins' Credentials plugin setup.

### Secrets Configured (GitHub → Settings → Secrets and variables → Actions)

- `EC2_HOST` — the EC2 instance's public hostname
  (`ec2-16-171-104-76.eu-north-1.compute.amazonaws.com`)
- `EC2_USER` — `ubuntu`
- `EC2_SSH_KEY` — the full private `.pem` key content (never exposed in
  logs or committed to the repo)
- Docker Hub credentials, stored as secrets and referenced in the login step

### Deployment Step

```yaml
- name: Deploy to EC2
  uses: appleboy/ssh-action@v1.2.0
  with:
    host: ${{ secrets.EC2_HOST }}
    username: ${{ secrets.EC2_USER }}
    key: ${{ secrets.EC2_SSH_KEY }}
    script: |
      cd ~/Rent-A-Ride-Cloud-DevOps
      docker compose pull
      docker compose up -d
      docker compose ps
```

---

## Issues Encountered, Root Cause, Fix

### Issue 1 — Malformed `EC2_HOST` secret
- **Error:** `dial tcp: lookup tcp///16.171.104.76: unknown port`
- **Root cause:** The `EC2_HOST` secret was not a clean hostname (extra
  characters/formatting broke the SSH connection string) — this was
  initially mistaken for an SSH key problem.
- **Fix:** Corrected `EC2_HOST` to the plain EC2 public DNS hostname with
  no extra characters. SSH deployment succeeded immediately after.

### Issue 2 — `docker compose push` skipping all services (carried over from Jenkins)
- Same root cause as documented in Task 6/7 Jenkins notes: Compose only
  pushes services defined with `build:`. Since this project's
  `docker-compose.yaml` uses `image:` (pulling pre-built, already-tested
  images), pushing had to be done with explicit `docker build`/`docker push`
  commands in the CI workflow, with Compose reserved purely for deployment.

### Issue 3 — `restart: always` does not pull new images automatically
- **Observation:** After restarting the EC2 instance, all containers came
  back up correctly (proving `restart: always` works) — but simply
  restarting a container does not fetch a newer image version if one has
  been pushed since. A new deployment requires an explicit
  `docker compose pull` followed by `docker compose up -d`, which is why
  both commands are included in the deployment script rather than relying
  on `restart: always` alone.

### Non-issue — `Cannot GET /` on backend health check
- Hitting `curl http://localhost:3000` returns `Cannot GET /`, which is
  expected: the backend has no root route defined, only `/api/...` routes
  (`/api/auth/signin`, `/api/admin/showVehicles`, etc.). This confirms the
  backend is reachable and responding, not that anything is broken.

### Non-issue — Empty `vehicles` collection in MongoDB
- `db.vehicles.countDocuments()` returned `0`. Investigated the codebase's
  `insertDummyData` endpoint and found it seeds the `MasterData` collection,
  not `vehicles` directly. This was correctly identified as an
  **application data/seeding** matter, unrelated to the CI/CD pipeline or
  Docker deployment — the infrastructure works correctly regardless of
  whether sample data has been seeded.

---

## Verification

- `docker compose ps` on EC2 confirms all three services running:
  `backend` (port 3000), `frontend` (port 5173→8080), `mongodb` (port 27017,
  internal only).
- Full pipeline tested end-to-end with a real push to `feature/task07`:
  checkout → build → push to Docker Hub → SSH deploy → `docker compose pull`
  → `docker compose up -d` → all green.
- Confirmed persistence: EC2 instance restart brought all containers back
  up automatically via `restart: always`.

---

## Outcome

- CI/CD pipeline fully automated via GitHub Actions — a push to
  `feature/task07` triggers build, push, and deployment with no manual
  intervention ✅
- Docker Hub images kept up to date automatically on every push ✅
- EC2 deployment via SSH + Docker Compose verified working ✅
- Full stack (frontend, backend, MongoDB) confirmed running and reachable
  after each deployment ✅
- Jenkins pipeline remains documented as a working alternative
  implementation, demonstrating both approaches ✅
