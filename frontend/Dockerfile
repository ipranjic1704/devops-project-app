# Stage 1: install dependencies
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev

# Stage 2: minimal runtime image
FROM node:22-alpine AS runtime
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder /app/node_modules ./node_modules
COPY src/ ./src/

USER appuser
EXPOSE 3000
CMD ["node", "src/server.js"]
