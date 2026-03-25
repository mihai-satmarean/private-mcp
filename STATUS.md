# private-mcp — Catalog MCP Servers (obot)

**Data ultima actualizare:** 18 Mar 2026
**Scop:** Catalog privat de MCP servers pentru instanta obot pe Kubernetes (DO cluster, namespace `obot-mcp`)

---

## Arhitectura generala

```
GitHub Repo (jobsuche-mcp-server)
  |
  | push la main
  v
GitHub Actions: docker-build.yml
  - Build Rust binary (musl, static)
  - Wrap in nanobot (stdio -> HTTP :8099)
  - Push ghcr.io/mihai-satmarean/jobsuche-mcp-server:latest
  |
  v
GitHub Actions: k8s-restart.yml (trigger automat)
  - kubectl rollout restart deployment in obot-mcp namespace
  - Obot preia imaginea noua
  |
  v
obot (DO k8s, namespace obot-mcp)
  - Deployment gestionat de obot
  - Citeste catalog din acest repo (private-mcp)
  - Expune MCP server utilizatorilor
```

---

## MCP Servers in catalog

| Server | Status catalog | Status imagine | Note |
|--------|---------------|----------------|------|
| jobsuche | [X] `jobsuche.yaml` creat | [ ] Prima imagine nepublicata inca | Necesita commit + push in jobsuche-mcp-server |

---

## jobsuche-mcp-server

**Repo:** https://github.com/mihai-satmarean/jobsuche-mcp-server
**Imagine:** `ghcr.io/mihai-satmarean/jobsuche-mcp-server:latest`
**Transport:** stdio (Rust) -> HTTP :8099 via nanobot wrapper
**API:** Bundesagentur fuer Arbeit (public, fara autentificare obligatorie)

### Configurare obot (`jobsuche.yaml`)

```yaml
runtime: containerized
containerizedConfig:
  image: ghcr.io/mihai-satmarean/jobsuche-mcp-server:latest
  port: 8099
  path: /
  args:
    - jobsuche-mcp-server
```

### Modificari facute (18 Mar 2026)

- [X] `Dockerfile` - inlocuit cu build multi-stage:
  - Stage 1: `rust:1.82-bookworm` + musl target (static binary)
  - Stage 2: `cgr.dev/chainguard/wolfi-base` + nanobot v0.0.58 + binary jobsuche
  - ENTRYPOINT: `nanobot.sh jobsuche-mcp-server`
- [X] `scripts/nanobot.sh` - entrypoint preluat din `obot-platform/mcp-images`
- [X] `.github/workflows/docker-build.yml` - actualizat: build musl + push ghcr.io + smoke test pe PR
- [X] `private-mcp/jobsuche.yaml` - catalog entry complet cu port/path

### Pasi urmatori

- [ ] Commit + push modificari in `jobsuche-mcp-server` (Dockerfile, nanobot.sh, workflow)
- [ ] Verifica ca imaginea se construieste cu succes in GitHub Actions
- [ ] Configureaza obot sa citeasca acest repo ca MCP catalog
      (Admin > MCP Server Catalogs > adauga URL repo + branch main)
- [ ] Adauga jobsuche server intr-un MCP Registry in obot
- [ ] Test end-to-end: obot porneste containerul si tools-urile apar disponibile

### Dependente / Secrete necesare in jobsuche-mcp-server GitHub repo

| Secret | Utilizare | Status |
|--------|-----------|--------|
| `GITHUB_TOKEN` | Push imagine la ghcr.io | Automat (GitHub) |
| `KUBECONFIG` | k8s-restart.yml - rollout restart in obot-mcp | Necesar configurat |

### Nanobot wrapper - cum functioneaza

Serverul Rust foloseste transport stdio. Obot cere HTTP pentru runtime containerized.
Solutia: `nanobot.sh` primeste comanda (`jobsuche-mcp-server`), genereaza un `nanobot.yaml`
si porneste `nanobot run --listen-address :8099` care proxiaza HTTP -> stdio.

Conventie obot (confirmata din catalog oficial):
- Port: `8099`
- Path: `/`
- Toate imaginile repackaged de obot folosesc acelasi pattern

---

## Cum se adauga un MCP server nou in catalog

1. Creeaza fisierul YAML in acest repo urmand structura din `jobsuche.yaml`
2. Daca serverul e stdio (Rust, Go, alt binar):
   - Adauga nanobot wrapper in Dockerfile (vezi jobsuche ca referinta)
   - `port: 8099`, `path: /` in containerizedConfig
3. Daca serverul e HTTP nativ: specifica direct port + path
4. Daca serverul e npm/python: foloseste `runtime: npx` sau `runtime: uvx`
5. Push in repo - obot preia automat (GitOps)
