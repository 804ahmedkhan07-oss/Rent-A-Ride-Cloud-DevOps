# Task 6 — Orchestration Using Docker Compose

**Project:** Rent-a-Ride (MERN stack)
**Environment:** AWS EC2, Ubuntu, eu-north-1 (Stockholm), t3.small, Elastic IP: 16.171.104.76
**Branch:** feature/task06-docker-compose

## Objective

Replace manual `docker run` commands for each container with a single
`docker-compose.yml` file that orchestrates the frontend, backend, and
database together — using the pre-built images already published to Docker
Hub in Task 5, rather than rebuilding them locally.

---

## 1. Design Decision — `image` vs `build`

Compose supports two ways to define a service:
- `build:` — build the image locally from a Dockerfile every time
- `image:` — pull an already-built image from a registry

Since hardened, tested images were already published to Docker Hub in
Task 5, `image:` was used instead of `build:`. This is the more appropriate
choice for a deployment scenario: it avoids rebuilding on every startup,
guarantees the exact same image that was already tested is what runs, and
is faster. `build:` is generally more appropriate for active local
development, where rebuilding on every code change is the point.

```yaml
services:
  mongodb:
    image: mongo:7
    volumes:
      - mongodb-data:/data/db
    restart: always

  backend:
    image: ahmedmateen07/rent-a-ride-backend:latest
    ports:
      - "3000:3000"
    environment:
      - mongo_uri=mongodb://mongodb:27017/rent-a-ride
      - ACCESS_TOKEN=${ACCESS_TOKEN}
      - REFRESH_TOKEN=${REFRESH_TOKEN}
    depends_on:
      - mongodb
    restart: always

  frontend:
    image: ahmedmateen07/rent-a-ride-frontend:latest
    ports:
      - "5173:8080"
    depends_on:
      - backend
    restart: always

volumes:
  mongodb-data:
    external: true
```

---

## 2. Networking — No Manual Network Needed

Unlike the manual setup in Task 4 (`docker network create rent-a-ride-net`),
Compose automatically creates a shared network for all services defined in
the same file. Each service becomes reachable by its **service name** —
e.g. the backend reaches MongoDB at `mongodb:27017`, not by IP or by a
manually created container name.

---

## 3. Volumes — Reusing Existing Data

The `mongodb-data` volume was created manually in Task 4 and already
contained persisted application data. Declaring it as `external: true`
tells Compose "this volume already exists — attach to it, don't create a
new one," so existing data was preserved rather than starting fresh.

---

## 4. Secrets — Moved Out of the Compose File

The task requirements explicitly state not to hardcode credentials directly
in the Compose file. Initially `ACCESS_TOKEN` and `REFRESH_TOKEN` were
written directly as plain values under `environment:`. This was corrected
by moving the actual values into a `.env` file (excluded from Git via
`.gitignore`) and referencing them in `docker-compose.yml` with
`${ACCESS_TOKEN}` / `${REFRESH_TOKEN}` syntax — Compose automatically reads
a `.env` file in the same directory and substitutes the values at runtime.

---

## 5. Issues Encountered, Root Cause, Fix

### Issue 1 — `docker compose` command not found
- **Root cause:** `docker-compose` (the older standalone tool) isn't
  installed by default on a fresh `docker.io` install via `apt`; the
  correct modern tool is the `docker compose` (v2) plugin.
- **Fix:** Installed the Compose v2 plugin, confirmed with `docker compose version`.

### Issue 2 — Frontend crashed immediately: `host not found in upstream "backend-container"`
- **Root cause:** `client/nginx.conf`'s `proxy_pass` still referenced the
  old container name (`backend-container`) from the manual `docker run`
  setup in Task 3/4. Under Compose, the backend service is simply named
  `backend` — that's the hostname other containers must use to reach it.
- **Fix:** Updated `nginx.conf` to `proxy_pass http://backend:3000/api/;`,
  rebuilt the frontend image, and pushed the corrected image back to
  Docker Hub so `docker compose pull` would fetch the fix.

### Issue 3 — Frontend build killed (`SIGKILL`) mid-build
- **Root cause:** The swap file created in a previous session had reset
  after the EC2 instance was stopped/started — `free -h` showed `Swap: 0B`,
  leaving too little memory for the Vite build alongside the already-running
  backend and MongoDB containers.
- **Fix:** Recreated a 6GB swap file, then re-ran the build successfully.

---

## 6. Verification

- `docker compose ps` confirmed all three services (`frontend`, `backend`,
  `mongodb`) running under Compose.
- `docker compose logs backend` confirmed a successful MongoDB connection
  using the `.env`-sourced credentials.
- Full signup/login flow tested through the browser — confirmed working,
  and existing (previously persisted) user data was still present,
  confirming the volume reuse worked correctly.
- `docker compose down` + `docker compose up -d` tested to confirm the
  stack restarts cleanly and data survives.

---

## 7. Outcome

- Full stack (frontend, backend, database) orchestrated with a single
  `docker-compose.yml` and a single `docker compose up -d` command ✅
- Services communicate using Compose service names over an
  automatically-created network ✅
- Database data persists across `docker compose down`/`up` cycles via the
  existing external volume ✅
- Sensitive credentials sourced from a `.env` file, not hardcoded in the
  Compose file ✅
- `restart: always` configured on all services for automatic recovery ✅
