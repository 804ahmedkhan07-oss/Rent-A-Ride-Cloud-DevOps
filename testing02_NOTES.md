# Deployment Notes

## Issues Faced & Fixes
1. MongoDB not installed on server → installed mongodb-org, started mongod service
2. EADDRINUSE port conflict → resolved with fuser -k
3. Low disk space during frontend npm install → resized EBS volume, ran growpart + resize2fs
4. Low RAM causing npm install OOM crash → added 2GB swap file

## Status
- Backend: connected + running
- MongoDB: running
- Frontend: install in progress
