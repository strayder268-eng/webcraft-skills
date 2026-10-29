# GitHub Actions

## CI dan build image

```yaml
# .github/workflows/ci.yml
name: ci
on:
  push: { branches: [main] }
  pull_request:

permissions:
  contents: read            # bawaan seminimal mungkin; job yang butuh lebih menambah sendiri

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_USER: t, POSTGRES_PASSWORD: t, POSTGRES_DB: t }
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U t" --health-interval 5s --health-retries 20
    steps:
      - uses: actions/checkout@<sha-commit>          # kunci ke SHA, tulis versinya di komentar
      - uses: actions/setup-go@<sha-commit>
        with: { go-version-file: api/go.mod }
      - run: cd api && go vet ./... && go test -race -count=1 ./...
        env: { DATABASE_URL: "postgres://t:t@localhost:5432/t?sslmode=disable" }
      - run: cd api && go run golang.org/x/vuln/cmd/govulncheck@<versi> ./...   # versi dikunci
      - uses: actions/setup-node@<sha-commit>
        with: { node-version: 22, cache: npm, cache-dependency-path: web/package-lock.json }
      - run: cd web && npm ci && npx tsc --noEmit && npx eslint . && npm test --if-present

  image:
    needs: test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write       # hanya job ini yang boleh push ke ghcr.io
    strategy:
      matrix: { app: [api, web] }
    steps:
      - uses: actions/checkout@<sha-commit>
      - uses: docker/login-action@<sha-commit>
        with: { registry: ghcr.io, username: "${{ github.actor }}", password: "${{ secrets.GITHUB_TOKEN }}" }
      - uses: docker/build-push-action@<sha-commit>
        with:
          context: ${{ matrix.app }}
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/${{ matrix.app }}:sha-${{ github.sha }}
          cache-from: type=gha,scope=${{ matrix.app }}
          cache-to: type=gha,scope=${{ matrix.app }},mode=max
```

Ganti setiap `<sha-commit>` dengan SHA rilis action yang dipakai (lihat halaman rilis action itu).

## Mencegah injeksi perintah

Salah (judul PR bisa berisi perintah shell):

```yaml
- run: echo "${{ github.event.pull_request.title }}"
```

Benar:

```yaml
- run: echo "$TITLE"
  env: { TITLE: "${{ github.event.pull_request.title }}" }
```

## Sumber

- [GitHub Actions: Security hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [GitHub: Publishing Docker images](https://docs.github.com/en/actions/use-cases-and-examples/publishing-packages/publishing-docker-images)
- [GitHub Actions: About service containers](https://docs.github.com/en/actions/use-cases-and-examples/using-containerized-services/about-service-containers)
