# Task 4 — Docker Volume, Network & Container Setup

**Project:** Rent-a-Ride (MERN stack)
**Environment:** AWS EC2, Ubuntu, eu-north-1 (Stockholm), t3.small, Elastic IP: 16.171.104.76
**Branch:** feature/task04-volume-network

## Objective

Create a Docker volume to persist database data, create a Docker network so
the frontend, backend, and database containers can communicate using
container names instead of IP addresses, then verify the full stack works
end-to-end and survives container restarts.

---

## 1. Docker Volume

```bash
docker volume create mongodb-data
```

A named volume was created to store MongoDB's data outside of the container
filesystem. Containers are disposable — they get removed and recreated on
every rebuild or redeploy. Without a volume, all database data would be
stored inside the container's own filesystem and would be permanently lost
the moment the container is removed. The volume lives independently on the
host and gets re-attached to a new container, so data survives even if the
container itself is deleted.

---

## 2. Docker Network

```bash
docker network create rent-a-ride-net
```

A custom bridge network was created so the frontend, backend, and database
containers can discover and talk to each other **by container name** instead
of by IP address. By default, each container is isolated and can't see
others unless they're placed on the same network. Once containers join a
custom network, Docker's internal DNS resolves container names automatically
— this also means the setup doesn't break if a container's internal IP
changes on restart.

---

## 3. MongoDB Container (with Volume + Network)

The MongoDB service that was previously installed directly on the EC2 host
(from Task 2) was stopped, and MongoDB was containerized instead:

```bash
docker run -d \
  --name mongodb-container \
  --network rent-a-ride-net \
  -v mongodb-data:/data/db \
  -p 27017:27017 \
  mongo:7
```

- `--network rent-a-ride-net` — joins the shared network so other containers
  can reach it by the name `mongodb-container`.
- `-v mongodb-data:/data/db` — mounts the volume created above to `/data/db`,
  which is the exact path MongoDB uses internally to store its data files.
- `-p 27017:27017` — exposes MongoDB's default port to the host (for direct
  inspection/debugging if needed).

---

## 4. Backend Container (updated to use the network)

The backend container was recreated on the same network, and its database
connection string was changed to use the MongoDB **container name** instead
of an IP address:

```bash
docker run -d \
  --name backend-container \
  --network rent-a-ride-net \
  -p 3000:3000 \
  -e mongo_uri=mongodb://mongodb-container:27017/rent-a-ride \
  -e ACCESS_TOKEN=<secret> \
  -e REFRESH_TOKEN=<secret> \
  rent-a-ride-backend
```

Previously (Task 3), the backend connected to MongoDB via `172.17.0.1`
(Docker's host-gateway address), because MongoDB was running directly on the
EC2 host. Now that MongoDB runs in its own container on the same custom
network, the connection string uses the container's name (`mongodb-container`)
instead — this is the standard way containers address each other on a shared
Docker network.

Verified with:
```bash
docker logs backend-container
```
Output: `connected` + `server listening !` — confirms the backend container
reaches MongoDB entirely through the Docker network, with no dependency on
the host machine's MongoDB installation.

---

## 5. Frontend Container (joined to the same network)

```bash
docker run -d \
  --name frontend-container \
  --network rent-a-ride-net \
  -p 5173:80 \
  rent-a-ride-frontend
```

Joined to the same network for consistency, even though the frontend talks
to the backend over the host's public IP (via the browser) rather than the
internal Docker network.

---

## 6. Issues Encountered, Root Cause, Fix

### Issue 1 — EC2 public IP changes on every instance stop/start
- **Root cause:** AWS assigns a new public IP by default each time an
  instance is stopped and started, unless an Elastic IP is allocated.
- **Fix:** Allocated and associated an Elastic IP (`16.171.104.76`) to the
  instance, giving it a permanent public IP that doesn't change across
  restarts.

### Issue 2 — Signup/login failed after switching to the Elastic IP
- **Root cause:** `VITE_PRODUCTION_BACKEND_URL` in `client/.env` and the
  `allowedOrigins` CORS list in `backend/server.js` still referenced the old
  IP. Since Vite bakes environment variables into the build at build time
  (not runtime), simply updating `.env` wasn't enough — the frontend image
  had to be rebuilt.
- **Fix:** Updated both files with the new Elastic IP, then rebuilt both the
  frontend and backend Docker images and recreated the containers.

### Issue 3 — Old MongoDB data not present after containerizing MongoDB
- **Root cause:** The host-installed MongoDB (from Task 2) and the new
  MongoDB container use completely separate storage — moving to a
  containerized database intentionally starts with a fresh, empty database
  unless data is explicitly migrated.
- **Resolution:** Accepted as expected behavior for this task — the old data
  was test data only. Verified the new setup by signing up a new test user
  and confirming it persists correctly in the volume.

---

## 7. Verification

- Confirmed all three containers (`frontend-container`, `backend-container`,
  `mongodb-container`) are on the same custom network and communicate using
  container names.
- Full flow tested through the browser (`http://16.171.104.76:5173`):
  frontend → backend → database, using a freshly signed-up test account.
- Confirmed the backend container connects to MongoDB purely via the Docker
  network (`mongodb-container:27017`), with no dependency on host-level
  MongoDB or host IP addresses.

---

## 8. Outcome

- Docker volume created and attached to the database container ✅
- Docker network created; all containers communicate by name ✅
- MongoDB fully containerized (no longer dependent on host installation) ✅
- Backend successfully connects to MongoDB through the network ✅
- Frontend, backend, and database containers verified working together ✅

---

## 9. Volume Persistence Test (Proof)

To confirm the Docker volume actually persists data independent of the
container lifecycle, the following test was performed:

1. Checked user count in the database:
   ```bash
   docker exec -it mongodb-container mongosh
   use rent-a-ride
   db.users.find().count()
   ```
   Result: `1`

2. Deleted the MongoDB container entirely and recreated it from scratch,
   re-attaching the same volume:
   ```bash
   docker stop mongodb-container
   docker rm mongodb-container
   docker run -d --name mongodb-container --network rent-a-ride-net \
     -v mongodb-data:/data/db -p 27017:27017 mongo:7
   ```

3. Re-checked the user count on the brand-new container:
   ```bash
   docker exec -it mongodb-container mongosh
   use rent-a-ride
   db.users.find().count()
   ```
   Result: `1` (unchanged)

**Conclusion:** The data survived a full container deletion and recreation,
confirming the volume — not the container — is the actual source of
persistence, as intended.

---

## 10. Production Note — Nginx Config Reusability

The custom `client/nginx.conf` (routing fallback + `/api` reverse proxy to
the backend container) is a one-time architectural piece, not something that
needs to be rewritten per environment. In local development, Vite's dev
server handles the `/api` proxy directly (see `vite.config.js`), but that
proxy only works in dev mode — it has no effect on the production build. In
production, Nginx has to take over that responsibility, since Vite is no
longer running. This file, once written, is reused as-is going forward; only
the backend's address (`proxy_pass` target) would need to change if the
backend's container/service name changes in a different environment (e.g.
in Kubernetes).
