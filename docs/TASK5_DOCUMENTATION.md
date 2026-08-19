# Task 5 — Docker Multi-Stage, Hardening & Registry Deployment

**Project:** Rent-a-Ride (MERN stack)
**Environment:** AWS EC2, Ubuntu, eu-north-1 (Stockholm), t3.small, Elastic IP: 16.171.104.76
**Branch:** feature/task05-multistage-registry

## Objective

Harden the backend and frontend Docker images for production readiness —
multi-stage builds, minimal base images, non-root users — then publish both
images to three container registries (Docker Hub, GitHub Container Registry,
AWS ECR) and verify the full stack still runs correctly.

---

## 1. Backend — Multi-Stage Build + Non-Root User

Previously (Task 3), the backend used a single-stage Dockerfile. It was
rebuilt as a two-stage image:

```dockerfile
# ===== STAGE 1: Builder =====
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

# ===== STAGE 2: Runtime =====
FROM node:20-alpine
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app .

USER appuser

EXPOSE 3000
CMD ["node", "backend/server.js"]
```

**Why a builder stage, even though Node doesn't need compiling:** `npm ci`
installs dependencies (including anything only needed at install time), and
Stage 2 starts from a completely fresh base image, pulling in only the
`node_modules` and application code it actually needs to run — nothing from
the install process itself carries over.

**Why `npm ci --omit=dev` instead of `npm install`:** `npm ci` installs
exact versions from `package-lock.json` (more reliable/reproducible than
`npm install`, which can resolve slightly different versions), and
`--omit=dev` excludes development-only dependencies from the final image.

**Why a non-root user:** By default, containers run as `root`. If an
attacker manages to exploit a vulnerability inside the running app, running
as root gives them full control of the container's filesystem and
significantly increases the risk of a "container escape" onto the host. A
dedicated low-privilege user (`appuser`) limits the blast radius of any
compromise to only what that user is permitted to do.

---

## 2. Frontend — Non-Root Nginx

The existing multi-stage frontend Dockerfile (build stage + Nginx serve
stage) was updated to run as non-root:

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY client/package*.json ./
RUN npm install
COPY client/ .
RUN NODE_OPTIONS="--max-old-space-size=1536" npm run build

# Stage 2: Serve
FROM nginxinc/nginx-unprivileged:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY client/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

**Why `nginxinc/nginx-unprivileged` instead of manually creating a user:**
Standard Nginx needs root to bind to port 80 (ports below 1024 are
"privileged" and require elevated permissions). Rather than reimplementing
that logic, an official pre-built non-root Nginx image was used — it already
runs as a non-root user and listens on port 8080 by default.

**Result:** switching to the unprivileged base also happened to reduce the
final image size (96.1MB → 84.8MB), since the unprivileged image is a
leaner build.

---

## 3. Container Registries

Both images were pushed to three separate registries:

### Docker Hub
```bash
docker tag rent-a-ride-backend ahmedmateen07/rent-a-ride-backend:latest
docker push ahmedmateen07/rent-a-ride-backend:latest

docker tag rent-a-ride-frontend ahmedmateen07/rent-a-ride-frontend:latest
docker push ahmedmateen07/rent-a-ride-frontend:latest
```

### GitHub Container Registry (GHCR)
```bash
docker login ghcr.io -u <github-username>
# password: GitHub Personal Access Token with write:packages / read:packages scopes

docker tag rent-a-ride-backend ghcr.io/<github-username>/rent-a-ride-backend:latest
docker push ghcr.io/<github-username>/rent-a-ride-backend:latest

docker tag rent-a-ride-frontend ghcr.io/<github-username>/rent-a-ride-frontend:latest
docker push ghcr.io/<github-username>/rent-a-ride-frontend:latest
```

### AWS Elastic Container Registry (ECR)
```bash
aws configure   # Access Key ID, Secret Access Key, region: eu-north-1

aws ecr create-repository --repository-name rent-a-ride-backend --region eu-north-1
aws ecr create-repository --repository-name rent-a-ride-frontend --region eu-north-1

aws ecr get-login-password --region eu-north-1 | \
  docker login --username AWS --password-stdin 448674443582.dkr.ecr.eu-north-1.amazonaws.com

docker tag rent-a-ride-backend 448674443582.dkr.ecr.eu-north-1.amazonaws.com/rent-a-ride-backend:latest
docker push 448674443582.dkr.ecr.eu-north-1.amazonaws.com/rent-a-ride-backend:latest

docker tag rent-a-ride-frontend 448674443582.dkr.ecr.eu-north-1.amazonaws.com/rent-a-ride-frontend:latest
docker push 448674443582.dkr.ecr.eu-north-1.amazonaws.com/rent-a-ride-frontend:latest
```

**Why three registries instead of one:** Each serves a different purpose in
practice — Docker Hub is the most common general-purpose registry; GHCR ties
naturally into a GitHub-hosted codebase and GitHub Actions CI/CD; ECR is
AWS-native, giving the fastest pull times when deploying onto AWS
infrastructure (ECS/EKS/EC2) since traffic never leaves AWS's network.

---

## 4. Issues Encountered, Root Cause, Fix

### Issue 1 — `npm warn config only Use --omit=dev`
- **Root cause:** `npm ci --only=production` uses an older/deprecated flag.
- **Fix:** Switched to `npm ci --omit=dev`, the current recommended syntax.

### Issue 2 — Frontend build killed with `SIGKILL` / heap out of memory
- **Root cause:** Same recurring memory pressure issue from Task 3/4 — the
  Vite production build is memory-intensive, and by this point MongoDB and
  backend containers were also running and consuming RAM on the same
  instance.
- **Fix:** Increased swap to 6GB and temporarily stopped the backend
  container during the frontend build to free up memory, then restarted it
  afterward.

### Issue 3 — Frontend container crashed immediately: `host not found in upstream "backend-container"`
- **Root cause:** Nginx resolves the `proxy_pass` target (`backend-container`)
  at startup. When the frontend container was started before the backend
  container was running, Nginx couldn't resolve the hostname and exited
  immediately.
- **Fix:** Established a strict startup order — MongoDB first, then backend,
  then frontend — so every hostname the frontend depends on already exists
  on the Docker network before Nginx starts.

### Issue 4 — Frontend container ran but connections were reset (`Connection reset by peer`)
- **Root cause:** The Dockerfile declared `EXPOSE 8080` and the container was
  run with `-p 5173:8080`, but `client/nginx.conf` still had `listen 80;`
  left over from the previous (root) Nginx setup — so nothing was actually
  listening on port 8080 inside the container.
- **Fix:** Updated `nginx.conf` to `listen 8080;` to match the unprivileged
  image's actual listening port, then rebuilt and redeployed.

---

## 5. Verification

- All three containers (frontend, backend, MongoDB) confirmed running with
  non-root users where applicable, on the shared Docker network.
- Full signup/login flow retested through the browser after switching to
  the hardened images — confirmed working end-to-end.
- Both backend and frontend images successfully pulled and confirmed present
  in Docker Hub, GHCR, and AWS ECR.

---

## 6. Outcome

- Backend uses a multi-stage build with a dedicated non-root user ✅
- Frontend uses a multi-stage build with a non-root Nginx base image ✅
- Both images use minimal (Alpine-based) base images ✅
- Both images published to Docker Hub, GHCR, and AWS ECR ✅
- Full stack verified working after switching to hardened images ✅
