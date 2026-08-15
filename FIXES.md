# Fixes Made — Task 2 (EC2 Deployment)

Simple explanation of the 2 code changes made while deploying and testing on EC2.

---

## 1. CORS Fix — `backend/server.js`

**Problem:** Backend only accepted requests from 2 fixed addresses (Vercel URL + `localhost:5173`). When testing from the EC2 public IP in a browser, the backend rejected every request with a CORS error ("Failed to fetch").

**Before:**
```js
const allowedOrigins = ['https://rent-a-ride-two.vercel.app', 'http://localhost:5173'];
```

**After:**
```js
const allowedOrigins = ['https://rent-a-ride-two.vercel.app', 'http://localhost:5173', 'http://13.49.231.183:5173'];
```

**Why:** Added the EC2 instance's public IP so the browser (running on the actual EC2 server) is allowed to talk to the backend.

**Note:** If the EC2 public IP changes (e.g. after a restart), this list needs the new IP added.

---

## 2. Duplicate Phone Number Fix — `backend/models/userModel.js`

**Problem:** The signup form doesn't collect a phone number, so every new user gets `phoneNumber: null`. The database had a `unique: true` rule on `phoneNumber`, and MongoDB treats multiple `null` values as duplicates — so the second user to sign up always failed with:
```
E11000 duplicate key error ... phoneNumber: null
```

**Fix:** Removed the strict `unique: true` constraint on `phoneNumber` (or added `sparse: true`), so multiple users can have an empty phone number without conflicting.

---

## Not Pushed (and never should be)

- `backend/.env` — contains `mongo_uri`, `ACCESS_TOKEN`, `REFRESH_TOKEN` (secrets, machine-specific)
- `client/.env` — contains `VITE_PRODUCTION_BACKEND_URL` (points to whichever server it's deployed on)

These are recreated manually on every new server/instance and are excluded via `.gitignore`.

---

## Quick Reference — What To Do On A New Server

Every time this project is deployed on a fresh instance, these `.env` values need to be set again:

**`backend/.env`:**
```
mongo_uri=mongodb://127.0.0.1:27017/rent-a-ride
ACCESS_TOKEN=your_secret_access_token
REFRESH_TOKEN=your_secret_refresh_token
```

**`client/.env`:**
```
VITE_PRODUCTION_BACKEND_URL=http://<your-ec2-public-ip>:3000
```

And if the new server's IP is different, add it to `allowedOrigins` in `backend/server.js` too.
