# Error Journal — Rent-a-Ride Deployment (Task 2 to Task 6)

Every real error hit during this project, explained simply, for quick
revision and interview prep. Pattern first, then what it means, then the fix.

---

## 🔑 Category 1: ".env" and Configuration Errors

### "`mongo_uri` is undefined" (multiple times, different reasons)
**Like:** Asking someone to bring "the file" without telling them which
drawer it's in. The code went looking for a settings paper (`.env`) in one
folder, but the paper was actually sitting in a different folder.
**Real cause:** `dotenv.config()` looks for `.env` in the current working
directory by default — not wherever the file physically is.
**Fix:** Either put `.env` where the command runs from, or (better)
explicitly point to it: `dotenv.config({ path: path.join(__dirname, ".env") })`.

### Duplicate `dotenv` import
**Like:** Writing your own name twice on the same form — the form doesn't
know which one to use, so it just breaks.
**Fix:** Keep only one `import dotenv from "dotenv"` line.

### Backend Docker container: `mongo_uri undefined` again
**Like:** The recipe (Dockerfile) never mentions "add salt" — nobody's
going to add salt just because the pot needs it. Docker containers get zero
values by default; nothing carries over from the host.
**Fix:** Pass values explicitly at run time with `-e` flags (`docker run -e mongo_uri=...`).

### Secrets hardcoded in `docker-compose.yml`
**Like:** Writing your house alarm code on the front door instead of
keeping it in your head.
**Fix:** Moved real values into a `.env` file (never committed to Git),
referenced them in the Compose file with `${VARIABLE_NAME}`.

---

## 🔑 Category 2: Networking / "Who Can Talk To Whom"

### `ECONNREFUSED 127.0.0.1:27017`
**Like:** Calling a phone number that simply isn't connected to anyone —
no service was even running on the other end.
**Real cause:** MongoDB wasn't installed/running on the host yet.
**Fix:** Installed and started `mongod`.

### `getaddrinfo EAI_AGAIN mongodb-container` / `host not found in upstream`
**Like:** Shouting a nickname in a room where nobody answers to that
nickname anymore — the room (network) changed, but the note (config) still
has the old name written on it.
**Real cause:** Container/service naming mismatch — the config referenced
an old container name (`mongodb-container`, `backend-container`) that no
longer matched the actual name on the current Docker network (especially
after moving from manual `docker run` to Docker Compose, where service
names become the hostnames).
**Fix:** Always match the exact service/container name used in whatever
network setup is currently active.

### "Failed to fetch" from the browser
**Like:** Mailing a letter to your own old house instead of your friend's
new address — it just never arrives, because it was never sent to the
right place.
**Real cause:** Frontend's `.env` (`VITE_PRODUCTION_BACKEND_URL`) pointed at
`localhost`, which inside a browser means "my own laptop," not the server.
**Fix:** Pointed it at the server's actual public IP/Elastic IP.

### CORS errors ("blocked by CORS policy")
**Like:** A bouncer at a club with a guest list — if your name (origin
URL) isn't on the list, you don't get in, even if you're a real guest.
**Fix:** Added the frontend's actual URL to the backend's `allowedOrigins` list.

### `405 Not Allowed` on `/signup`
**Like:** Knocking on the wrong apartment door and asking for a package —
that apartment (Nginx, serving static files) has no idea what you're
talking about; you needed the delivery office (backend) instead.
**Real cause:** In production, Vite's dev-mode `/api` proxy doesn't exist
anymore — requests were hitting Nginx directly instead of being forwarded
to the backend.
**Fix:** Added an explicit reverse-proxy rule in `nginx.conf` so Nginx
itself forwards `/api/*` requests to the backend container.

### "Connection reset by peer" (frontend container)
**Like:** Ringing a doorbell that's wired to a room nobody's actually
sitting in — the door (port 5173, mapped from outside) led to a room
(port 8080) where no one was listening, because Nginx was still configured
to listen on the old port (80).
**Fix:** Updated `nginx.conf` to `listen 8080;` to match the non-root
Nginx image's actual port.

---

## 🔑 Category 3: Resources (Memory, Disk, Ports)

### `EADDRINUSE: address already in use`
**Like:** Two people trying to sit in the exact same chair at the same
time — only one can occupy that seat (port) at once.
**Fix:** Found and stopped the process/container already using the port.

### `ENOSPC: no space left on device`
**Like:** A closet so full nothing else fits, no matter how you try to
squeeze something in.
**Real cause:** A swap file had eaten up most of the disk.
**Fix:** Removed the oversized swapfile, resized the EC2 volume, freed space.

### `JavaScript heap out of memory` / `SIGKILL` during builds
**Like:** Trying to cook five dishes on one small stove at once — nothing
gets enough heat, and something boils over.
**Real cause:** Vite's production build needs a lot of RAM. With MongoDB and
backend containers also running, the small EC2 instance ran out of memory.
**Fix:** Added swap space (and periodically had to recreate it — swap
resets when the EC2 instance is stopped/started).

---

## 🔑 Category 4: Docker Build/Path Mistakes

### `COPY failed: no source files were specified`
**Like:** Asking someone to "bring the file" while standing in the wrong
room — it's not that the file doesn't exist, you're just looking in the
wrong place.
**Real cause:** Build was run from a subfolder, but `package.json` lived at
the project root (or vice versa) — build context and file location didn't line up.
**Fix:** Ran the build from the correct directory, adjusted `COPY` paths to
match where files actually were relative to the build context.

### `Cannot find module '/app/server.js'`
**Like:** The map says "third door on the left," but because the whole
building layout (root vs subfolder) shifted, the actual door was in a
different spot.
**Fix:** Updated `CMD` to the correct nested path (`backend/server.js`).

### Frontend crash: `Missing script: "build"`
**Like:** Ordering from the wrong restaurant's menu — asked a bakery to
serve a burger when the burger menu was next door.
**Real cause:** Same root-vs-subfolder mixup — the wrong `package.json`
(backend's) was picked up instead of `client/package.json`.
**Fix:** Explicitly prefixed copy paths with `client/`.

---

## 🔑 Category 5: MongoDB / Application Logic

### `E11000 duplicate key error ... phoneNumber: null`
**Like:** A cloakroom that only allows one "no name tag" coat at a time —
the second person without a tag gets turned away, even though neither of
them did anything wrong.
**Real cause:** `phoneNumber` had a `unique` index, and the signup form
never collected a phone number, so every user got `null` — which MongoDB
treated as a duplicate after the second signup.
**Fix:** Removed the strict `unique` constraint (or added `sparse: true`).

### `secretOrPrivateKey must have a value`
**Like:** Trying to stamp a document with a stamp that was never given to you.
**Real cause:** `ACCESS_TOKEN`/`REFRESH_TOKEN` env vars were missing, but
the code needed them to sign login tokens (JWT).
**Fix:** Added both values to `.env`.

---

## 🔑 The Overall Pattern (worth remembering)

Nearly every error in this entire project falls into one of three buckets:

1. **"The note and the reality don't match"** — a name, a path, or a port
   written in a config file didn't match what actually existed anymore.
2. **"Not enough resources"** — RAM, disk, or swap ran out under load.
3. **"Nobody told this container/service what it needed to know"** — a
   missing environment variable or credential.

Recognizing which of these three a new error belongs to is 80% of the
debugging work.
