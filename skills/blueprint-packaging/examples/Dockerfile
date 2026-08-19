# Web app image for an OctoMesh Application workload.
# Multi-stage: SPA build -> slim runtime with a zero-dependency Node proxy.
# Fulfils the web-app chart contract (see chart-contract.md):
#   PORT / UPSTREAM_URL env, GET / -> 200, container port 5055.

FROM node:22-alpine AS build
WORKDIR /build
COPY client/package.json client/package-lock.json ./
RUN npm ci
COPY client/ ./
RUN npm run build

FROM node:22-alpine
WORKDIR /app
# server/server.js: serves the built SPA at GET / (satisfies the probes),
# proxies /api/* to UPSTREAM_URL preserving method, query and JSON body.
COPY server/ ./server/
COPY --from=build /build/dist ./client/
ENV PORT=5055
EXPOSE 5055
CMD ["node", "server/server.js"]
