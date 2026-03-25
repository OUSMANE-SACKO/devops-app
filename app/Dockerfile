# ── Stage 1 : Dependencies ──
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts && mkdir -p /app/node_modules

# ── Stage 2 : Test ──
FROM deps AS test
COPY . .
RUN npm test

# ── Stage 3 : Production ──
FROM node:20-alpine AS production

# Sécurité : user non-root
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser

WORKDIR /app

# Copier seulement les deps de production
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --ignore-scripts && \
    npm cache clean --force

# Copier le code source
COPY --from=deps /app/node_modules ./node_modules
COPY src/ ./src/

# Métadonnées
LABEL maintainer="devops-team"
LABEL version="1.0"

# Sécurité : filesystem read-only compatible
RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/ || exit 1

CMD ["node", "src/index.js"]
