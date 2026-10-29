# Docker

Versi image di contoh ini hanya contoh. Pakai versi yang sesuai proyek dan tulis lengkap (bukan `latest`).

## Go API

```dockerfile
# api/Dockerfile
FROM golang:1.25-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

Image `distroless` tidak punya shell. Health check dijalankan oleh proxy/compose lewat HTTP, atau sediakan
subcommand `server healthcheck` di binary yang sama.

## Next.js (output standalone)

```js
// next.config.mjs
export default { output: 'standalone' };
```

```dockerfile
# web/Dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
# nilai publik, dibekukan saat build
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
RUN npm run build

FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production HOSTNAME=0.0.0.0 PORT=3000
COPY --from=build --chown=node:node /app/.next/standalone ./
COPY --from=build --chown=node:node /app/.next/static ./.next/static
COPY --from=build --chown=node:node /app/public ./public
USER node
EXPOSE 3000
HEALTHCHECK CMD wget -qO- http://127.0.0.1:3000/ >/dev/null || exit 1
CMD ["node", "server.js"]
```

Folder `public` dan `.next/static` tidak ikut ke `standalone` secara otomatis; disalin manual seperti di atas.
[Next.js: output]

## .dockerignore

```
.git
node_modules
.next
.env*
!.env.example
**/*.pem
secrets/
```

## Compose produksi (kerangka)

```yaml
x-logging: &logging
  driver: json-file
  options: { max-size: "10m", max-file: "3" }

services:
  proxy:
    image: caddy:2.10-alpine
    ports: ["80:80", "443:443", "443:443/udp"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
    depends_on: { web: { condition: service_healthy }, api: { condition: service_healthy } }
    restart: unless-stopped
    logging: *logging

  web:
    image: ghcr.io/<owner>/<app>-web:${IMAGE_TAG:?isi IMAGE_TAG}
    restart: unless-stopped
    deploy: { resources: { limits: { memory: 512m } } }
    logging: *logging

  api:
    image: ghcr.io/<owner>/<app>-api:${IMAGE_TAG:?isi IMAGE_TAG}
    env_file: ./api.env                     # chmod 600, tidak di repo
    depends_on: { db: { condition: service_healthy } }
    restart: unless-stopped
    deploy: { resources: { limits: { memory: 512m } } }
    logging: *logging

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets: [db_password]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U app -d app"], interval: 5s, retries: 20 }
    restart: unless-stopped
    logging: *logging

volumes: { pgdata: {}, caddy_data: {} }
secrets:
  db_password: { file: ./secrets/db_password }
```

`db` tidak punya `ports`: hanya bisa dicapai dari dalam jaringan compose.

## Sumber

- [Docker: Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker Compose: secrets](https://docs.docker.com/compose/how-tos/use-secrets/)
- [Next.js: output standalone](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)
- [distroless](https://github.com/GoogleContainerTools/distroless)
