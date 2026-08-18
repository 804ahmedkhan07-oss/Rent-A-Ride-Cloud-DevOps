# ===== STAGE 1: Builder =====
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

# ===== STAGE 2: Runtime =====
FROM node:20-alpine
WORKDIR /app

# New user 
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# only important depandencies copy 
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app .

# replace root
USER appuser

EXPOSE 3000
CMD ["node", "backend/server.js"]
