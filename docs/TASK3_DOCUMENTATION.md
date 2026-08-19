# Task 3 — Dockerization Documentation

**Project:** Rent-a-Ride (MERN stack)
**Environment:** AWS EC2, Ubuntu, eu-north-1 (Stockholm), t3.small
**Branch:** feature/task03-dockerization

## Objective

Containerize the backend and frontend applications separately using Docker,
connect the backend to MongoDB, verify communication between containers,
and confirm the full stack works the same way it did in Task 2 (non-Docker).

---

## 1. Backend Containerization

### Dockerfile (`backend/Dockerfile`)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

Build is run from the project root (not from `backend/`) because `package.json`
lives at the root:

```bash
docker build -t rent-a-ride-backend -f backend/Dockerfile .
```

Because the build context is the root, `CMD` had to reference the nested path:

```dockerfile
CMD ["node", "backend/server.js"]
```

### Running the container

```bash
docker run -d -p 3000:3000 \
  -e mongo_uri=mongodb://172.17.0.1:27017/rent-a-ride \
  -e ACCESS_TOKEN=<secret> \
  -e REFRESH_TOKEN=<secret> \
  --name backend-container rent-a-ride-backend
```

`172.17.0.1` is Docker's default bridge gateway address, which routes traffic
from inside a container back to the host machine. MongoDB runs directly on
the EC2 host (not in a container), so the backend container reaches it
through this address instead of `127.0.0.1` (which inside a container would
just point back to the container itself).

Verified with:
```bash
docker logs backend-container
```
Output: `connected` + `server listening !` — confirms the backend container
successfully reaches MongoDB on the host.

---

## 2. Frontend Containerization

### Dockerfile (`client/Dockerfile`) — multi-stage build

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY client/package*.json ./
RUN npm install
COPY client/ .
RUN NODE_OPTIONS="--max-old-space-size=1536" npm run build

# Stage 2: Serve
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY client/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

A multi-stage build is used because building the React app requires Node.js
and all dev dependencies, but running it in production only requires the
compiled static files. Stage 1 builds the app; Stage 2 starts from a fresh,
minimal Nginx image and copies in only the compiled output — keeping the
final image small and free of build tooling and source code.

### Running the container

```bash
docker run -d -p 5173:80 --name frontend-container rent-a-ride-frontend
```

---

## 3. Issues Encountered, Root Cause, Fix

### Issue 1 — `COPY failed: no source files were specified`
- **Root cause:** Build was run from `backend/`, but `package.json` lives at
  the project root, so Docker's build context didn't include it.
- **Fix:** Run the build from the project root with `-f backend/Dockerfile .`
  instead of `cd backend && docker build .`

### Issue 2 — `Cannot find module '/app/server.js'`
- **Root cause:** Because the build context was the root, `COPY . .` copied
  the entire project (including the `backend/` folder itself), so the file
  actually landed at `/app/backend/server.js`, not `/app/server.js`.
- **Fix:** Updated `CMD` to `["node", "backend/server.js"]`.

### Issue 3 — `npm error Missing script: "build"` (frontend)
- **Root cause:** Same root-vs-subfolder issue as above — `COPY package*.json ./`
  picked up the root `package.json` (backend's) instead of `client/package.json`.
- **Fix:** Explicitly prefixed copy paths with `client/`:
  `COPY client/package*.json ./` and `COPY client/ .`

### Issue 4 — `FATAL ERROR: Reached heap limit ... JavaScript heap out of memory`
- **Root cause:** The Vite production build is memory-intensive; the EC2
  instance's available RAM + swap wasn't enough during the Docker build step.
- **Fix:** Increased swap to 4GB and capped Node's memory usage during the
  build with `NODE_OPTIONS="--max-old-space-size=1536"` in the Dockerfile.

### Issue 5 — 404 on page refresh / direct route access (frontend)
- **Root cause:** React Router handles routing client-side, but Nginx (by
  default) looks for a matching physical file for every URL. Refreshing on
  a route like `/profile` had no matching file, so Nginx returned 404.
- **Fix:** Added a custom `client/nginx.conf` with a fallback rule so any
  unmatched route serves `index.html`, letting React Router take over:
  ```nginx
  server {
      listen 80;
      root /usr/share/nginx/html;
      index index.html;

      location / {
          try_files $uri $uri/ /index.html;
      }
  }
  ```

### Issue 6 — `port is already in use` when starting backend container
- **Root cause:** The non-Docker backend process (from Task 2 testing) was
  still running on port 3000, conflicting with the container.
- **Fix:** Stopped the old process before starting the container.

---

## 4. Verification

- Restarted both containers (`docker stop` + `docker start`) and confirmed
  the app still worked correctly afterward — verifying the containers don't
  depend on any one-time setup state.
- Tested the full flow in the browser (signup/login/profile) through the
  containerized frontend and backend — confirmed data from earlier testing
  (Task 2) was still present in MongoDB, proving the backend container
  successfully connects to the database on the host.

---

## 5. MongoDB

MongoDB was **not** containerized. The task allows either MongoDB Atlas or
a MongoDB container — the existing host-installed MongoDB (set up in Task 2)
was reused instead, and the backend container connects to it via the Docker
bridge gateway address (`172.17.0.1`).

---

## 6. Outcome

- Backend runs in a Docker container ✅
- Frontend runs in a Docker container (served via Nginx) ✅
- Backend successfully connects to MongoDB ✅
- Frontend and backend containers communicate successfully ✅
- Containers survive stop/start cycles ✅
