# Replicating the eXide CI Locally

This document describes how to reproduce the eXide GitHub Actions CI pipeline on your local machine.

## Prerequisites

- **Node.js 24** (LTS) — https://nodejs.org
- **Docker Desktop** — https://www.docker.com/products/docker-desktop
- **GitHub CLI (`gh`)** — https://cli.github.com (needed to download `.xar` dependencies)
- **Git** with submodule support

---

## Step 1 — Clone with submodules

```sh
git clone --recurse-submodules https://github.com/eXist-db/eXide.git
cd eXide
```

If you already have the repo cloned:

```sh
git submodule update --init --recursive
```

---

## Step 2 — Install Node dependencies

```sh
npm ci
```

---

## Step 3 — Run unit tests

```sh
npm test
```

---

## Step 4 — Validate OpenAPI schema

```sh
npx --yes @redocly/cli lint modules/api.json
```

---

## Step 5 — Build the frontend + EXPath package

```sh
npm run build
```

This produces the `.xar` package and all frontend assets in `build/`.

---

## Step 6 — Download `.xar` dependencies into `build/`

You need the GitHub CLI authenticated first (`gh auth login`), then run:

```sh
gh release download v1.12.0 --repo eeditiones/roaster --pattern "roaster-1.12.0.xar" --output build/roaster-1.12.0.xar

gh release download v0.9.5 --repo eXist-db/existdb-openapi --pattern "existdb-openapi-*.xar" --dir build/
```

---

## Step 7 — Start the eXist-db Docker container

The `build/` folder is mounted as the autodeploy directory — eXist-db automatically installs all `.xar` files found there on startup.

**Linux / macOS:**

```sh
docker run --rm --name exist \
  --volume $(pwd)/build:/exist/autodeploy:ro \
  --publish 8080:8080 \
  --detach existdb/existdb:latest
```

**Windows (PowerShell):**

```powershell
docker run --rm --name exist `
  --volume "${PWD}/build:/exist/autodeploy:ro" `
  --publish 8080:8080 `
  --detach existdb/existdb:latest
```

---

## Step 8 — Wait for eXist-db to be ready

Poll until eXide responds with HTTP 200 (allow up to ~2 minutes for startup and package deployment).

**Linux / macOS (bash):**

```sh
for i in $(seq 1 24); do
  HTTP_CODE=$(curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/exist/apps/eXide/index.html || true)
  echo "Attempt $i: HTTP $HTTP_CODE"
  if [ "$HTTP_CODE" = "200" ]; then
    echo "eXide is ready"
    break
  fi
  sleep 5
done
```

**Windows (PowerShell):**

```powershell
for ($i = 1; $i -le 24; $i++) {
    try {
        $code = (Invoke-WebRequest -Uri "http://localhost:8080/exist/apps/eXide/index.html" -UseBasicParsing -ErrorAction Stop).StatusCode
    } catch { $code = 0 }
    Write-Host "Attempt $i : HTTP $code"
    if ($code -eq 200) { Write-Host "eXide is ready!"; break }
    Start-Sleep -Seconds 5
}
```

---

## Step 9 — Run Cypress E2E tests

```sh
npx cypress run --spec 'cypress/e2e/*.cy.js'
```

Cypress is configured to target `http://localhost:8080/exist/apps` (see `cypress.config.js`).

- Screenshots are saved to `cypress/screenshots/` on failure
- Videos are always saved to `cypress/videos/`

---

## Step 10 — Stop the container when done

```sh
docker stop exist
```

---

## Summary

| Step | Command |
|------|---------|
| Install deps | `npm ci` |
| Unit tests | `npm test` |
| Validate OpenAPI | `npx --yes @redocly/cli lint modules/api.json` |
| Build | `npm run build` |
| Download roaster | `gh release download v1.12.0 --repo eeditiones/roaster ...` |
| Download existdb-openapi | `gh release download v0.9.5 --repo eXist-db/existdb-openapi ...` |
| Start eXist-db | `docker run ... existdb/existdb:latest` |
| Wait for ready | poll `http://localhost:8080/exist/apps/eXide/index.html` |
| E2E tests | `npx cypress run --spec 'cypress/e2e/*.cy.js'` |
| Stop container | `docker stop exist` |

> **Note:** eXide 4.x requires eXist-db 7.0+. Always use the `latest` Docker tag which tracks the develop branch.
