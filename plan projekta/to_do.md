# ShopHub — TO-DO i plan implementacije

> **Tim:** 3 člana | **Stack:** Go (svuda) | **Cilj:** položiti projekat (≥15/50), realno težiti 35–45/50.
>
> **Eliminacioni zahtev (zahtev 5.3):** organizacija IaC u `helm-charts` i `kube-state` repozitorijumima. **Bez ovoga projekat se ne pregleda.** Zato je to Faza 0 — ne fini retouch na kraju.
>
> Ovaj dokument je živ. Kako se zadaci završavaju, čekirajte ih. Kad iskrsne nešto što specifikacija ne pokriva, dogovorite na licu mesta i upišite odluku u sekciju **Decision log** na kraju.

---

## Sadržaj

- [0. Faza 0 — preduslov za predaju (eliminaciono)](#0-faza-0--preduslov-za-predaju-eliminaciono)
- [1. Faza 1 — temelji (sedmica 1)](#1-faza-1--temelji-sedmica-1)
- [2. Faza 2 — Shop operator (sedmice 2–3)](#2-faza-2--shop-operator-sedmice-23)
- [3. Faza 3 — Shop aplikacija (sedmica 3)](#3-faza-3--shop-aplikacija-sedmica-3)
- [4. Faza 4 — ShopHub aplikacija (sedmica 4)](#4-faza-4--shophub-aplikacija-sedmica-4)
- [5. Faza 5 — Web3 plaćanje (sedmica 4–5)](#5-faza-5--web3-pla%C4%87anje-sedmica-45)
- [6. Faza 6 — Observability stack (sedmica 5)](#6-faza-6--observability-stack-sedmica-5)
- [7. Faza 7 — Helm Chart-ovi i kube-state](#7-faza-7--helm-chart-ovi-i-kube-state)
- [8. Faza 8 — CI/CD pipeline-ovi](#8-faza-8--cicd-pipeline-ovi)
- [9. Faza 9 — Završno testiranje i demo](#9-faza-9--zavr%C5%A1no-testiranje-i-demo)
- [Podela posla po članovima tima](#podela-posla-po-%C4%8Dlanovima-tima)
- [Status (ažurirano 2026-05-28)](#status-a%C5%BEurirano-2026-05-28)
- [Decision log](#decision-log)

---

## Status (ažurirano 2026-05-29)

### Šta je gotovo (merge-ovano u main)

| Komponenta | PR-ovi | Procenat |
|---|---|---|
| Faza 0: 5 repo-a + branch protection + commitlint | shop #1, shop-operator #1, shophub #1, helm-charts #1, kube-state #1 | 100% |
| Faza 1: k3d klaster + CNPG + MongoDB operator (umesto Redis) | kube-state #3 | 95% |
| Faza 2: Shop operator (sva 3 CRD-a sa reconciler-ima) | shop-operator #2-6 + #7 (ServiceMonitor) | **100%** |
| Faza 3: Shop backend (Go/Gin CRUD items/orders, /metrics, /probe/*) + **frontend** (storefront + Web3 USDT checkout + payment animacija) | shop #2-4, payment-flow | **100%** |
| Faza 4: ShopHub backend (REST CRUD + **JWT auth + per-user multi-tenancy**) + **Helm chart deploy-ovan** + **Railway-style frontend** (landing/auth/dashboard, dokazano e2e) | shophub #2,#5,#6, frontend | **100%** |
| Faza 6: Observability — **KOMPLETNO** (metrics + per-Shop Grafana dashboard + Loki logs + Tempo traces + Alertmanager→Discord alarmi, spec 4.1) | kube-state #4+, shop-operator #7+, helm-charts #6+ | **100%** |
| Faza 7.1: Shop operator Helm chart (eliminacioni zahtev) | helm-charts #2 | 100% |
| Faza 7.2: ShopHub Helm chart (backend + RBAC + CNPG + JWT + ServiceMonitor, spec 3.3) | helm-charts (shophub) | **100%** |
| Faza 8: CI/CD pipeline — test/lint/docker-build/docker-publish (shop-operator, shophub) + helm-publish (OCI) | sve faze A+B | **85%** |

Ukupno: **~88%** projekta.

### Završeno u sesiji 2026-05-29 (D-items)

- **D5** ✅ `postInitApplicationSQL` + `OWNER TO` (items/orders šema u CNPG bootstrap-u) — shop-operator
- **D6** ✅ Hadolint Docker Build workflow (shop-operator + shop)
- **D9** ✅ Per-Shop Grafana dashboard, 100% po spec 4.1 (HTTP total/2xx/4xx/404/GB + CPU/RAM/FS-volume/net + latency). Jedinstveni posetioci odloženi za D8/Loki.
- **D8** ✅ Observability dovršen: Loki (logging) + Tempo (OTLP tracing, OTel u backend-u, OTEL env injektuje operator) + unique visitors (Loki distinct). Trace-ovi dokazani u Grafani (demo-shop span-ovi).
- **D13** ✅ ShopHub JWT auth + per-user tenant namespace multi-tenancy (Postgres users, bcrypt, JWT nosi namespace, middleware scope-uje per-tenant).
- **D14** ✅ ShopHub Helm chart deploy-ovan u cluster (backend + RBAC + CNPG users DB + JWT secret + ServiceMonitor). Register→login→shop u svom tenant ns dokazano in-cluster.
- **D1** ✅ Status Conditions (Available/Progressing/Degraded + Reason taksonomija) u Shop reconciler-u. Dokazano u clusteru.
- **D10** ✅ Per-Shop alerting → Discord. PrometheusRule (shop+cluster alarmi) + operator-kreiran AlertmanagerConfig (apiURL secret-ref, OnNamespace tenant izolacija) → Discord webhook. Dokazano end-to-end (ShopHighErrorRate firing → poruka u Discord kanalu).
- **D11** ✅ Shop backend (Go/Gin, items+orders CRUD, Prometheus `/metrics`, probes) + frontend skela + operator wiring (DATABASE_URL env, probes)
- **CI/CD Faza A+B** ✅ kompletni validation + publish workflow-i kroz sva 3 app repo-a + helm-charts OCI

**4 bug-a otkrivena+popravljena tokom D9 demo-a:**
1. Operator `containerHTTPPort` 80→8080 (backend CrashLoopBackOff)
2. Operator Service bez `app` labele (ServiceMonitor discovery fail)
3. Operator: dodato `ensureServiceMonitor` (auto-kreira SM po Shop-u) + RBAC
4. Backend: guard na negativan `Writer.Size()` (Prometheus counter panic → 500 umesto 404)

### Šta SLEDI (po prioritetu, prema vežbama 5 + asistent feedback)

**Hitno** (asistent eksplicitno tražio):
- **Faza 8 — kompletan CI/CD pipeline.** Detalji u [`ci-cd-plan.md`](./ci-cd-plan.md). Bez ovog `make deploy` ne radi end-to-end.

**Dodaci iz vežbi 5** (asistent verovatno tražio na odbrani — vidi [`../vezbe pomocni materijali/vezbe-5-operatori-helm-discord.md`](../vezbe%20pomocni%20materijali/vezbe-5-operatori-helm-discord.md)):

| # | Stavka | Težina | Gde u to_do.md |
|---|---|---|---|
| ~~**D1**~~ ✅ | ~~**Conditions u Status-u**~~ — GOTOVO: Available/Progressing/Degraded sa Reason-ima (DatabaseFailed/DatabaseProvisioning/Deploying/Ready), `meta.SetStatusCondition` (LastTransitionTime na pravim prelazima). Dokazano: `kubectl get shop` → Available=True (Ready). | ~1h | — |
| ~~**D2**~~ ✅ | ~~**`+kubebuilder:subresource:scale`**~~ — GOTOVO: opciono `spec.replicas` override (availability ostaje default), `status.selector`, scale marker. `kubectl scale shop demo-shop --replicas=4` skalirao deployment na 4/4 i ostao (dokazano u klasteru). HPA-ready preko scale subresource-a. | ~30 min | — |
| ~~**D3**~~ ✅ | ~~**Watches sa predicate-om** za CNPG Secret~~ — GOTOVO: Secret watch sa `cnpg.io/cluster` predikatom → reconcile Shop čim CNPG objavi `<shop>-app` secret (bez buđenja na nevezane secret-e). | ~45 min | — |
| ~~**D4**~~ ✅ | ~~**FieldIndexer**~~ — GOTOVO: index Shop-ova po `spec.discordWebhookSecretRef.name`; drugi Secret watch nalazi referencirajuće Shop-ove u O(1). | ~30 min | — |
| ~~**D5**~~ ✅ | ~~**`postInitApplicationSQL` sa `OWNER TO`**~~ — GOTOVO (shop-operator) | ~15 min | — |
| ~~**D6**~~ ✅ | ~~**Hadolint** workflow~~ — GOTOVO (shop-operator + shop) | ~10 min | — |
| ~~**D7**~~ ✅ | ~~**Pravi testovi**~~ — GOTOVO: ShopHub auth (Testcontainers Postgres + fake kube, 6 testova), shop-operator reconcile (fake client + WithStatusSubresource, 5 testova), shop backend handleri (Testcontainers Postgres, 3 testa). Svi prolaze u CI. | ~1.5h | — |
| ~~**D8**~~ ✅ | ~~**Loki + Promtail** za logove + **Tempo** za tracing~~ — GOTOVO. Loki (logging) + Tempo (OTLP tracing, OpenTelemetry u backend-u) + unique visitors (4.1.d, Loki distinct query). Observability 100% (metrics+logs+traces+alerti). | ~2h | — |
| ~~**D9**~~ ✅ | ~~**Per-Shop Grafana dashboard**~~ — GOTOVO, 100% po spec 4.1 (osim unique visitors → D8) | ~1.5h | — |
| ~~**D10**~~ ✅ | ~~**PrometheusRule alarmi + Alertmanager → Discord**~~ — GOTOVO, per-Shop preko operator-kreiranog AlertmanagerConfig (apiURL secret-ref, OnNamespace izolacija) | ~2h | — |
| ~~**D11**~~ ✅ | ~~**Shop backend** (Go/Gin) sa CRUD-om i `/metrics`~~ — GOTOVO + frontend skela + operator wiring | ~2h | — |
| ~~**D12**~~ ✅ | ~~**Web3 plaćanje** (Sepolia + USDT + MetaMask)~~ — GOTOVO i dokazano protiv prave Sepolia mreže. Self-deployed ERC-20 "USDT" (Remix, 6 decimala, open mint), backend verifikuje Transfer event preko go-ethereum (poller pending→confirmed, idempotentan po replikama), operator injectuje WALLET_ADDRESS, frontend MetaMask connect + transfer (ethers v6) + polling. E2e: MetaMask poslao 5 USDT → Etherscan Success → order confirmed → stock smanjen. | ~2-3h | — |
| ~~**D13**~~ ✅ | ~~**ShopHub auth** (JWT)~~ — GOTOVO + **puni multi-tenancy**: register/login (bcrypt+JWT, Postgres users), svaki korisnik dobija svoj `tenant-<id>` namespace, JWT nosi namespace, middleware scope-uje /api/shops per-tenant. Dokazano lokalno (register→shop u svom ns→401 bez tokena). | ~2h | Deployment ide sa D14 (CNPG + JWT_SECRET + RBAC) |
| ~~**D14**~~ ✅ | ~~**ShopHub Helm chart**~~ — GOTOVO + deploy-ovan u cluster. Backend + ClusterRole (namespaces + cluster-wide shops) + CNPG users DB + JWT secret (lookup-preserve) + ServiceMonitor (spec 3.3). Dokazano: register→login→shop u svom tenant ns, RBAC radi. | ~1.5h | — |
| ~~**D15**~~ ✅ | ~~**Shop Helm chart**~~ — GOTOVO: tanak chart koji renderuje Shop CR (+ opcioni Discord `webhook-url` secret) iz values-a; manual-install shortcut za operator-managed Shop. `helm lint` + `helm template` (default i custom) verifikovani. | ~1h | — |

### Pomocni dokumenti

- [`ci-cd-plan.md`](./ci-cd-plan.md) — detaljan plan kompletnog CI/CD-a (workflows YAML, DockerHub secrets, SemVer)
- [`../vezbe pomocni materijali/vezbe-5-operatori-helm-discord.md`](../vezbe%20pomocni%20materijali/vezbe-5-operatori-helm-discord.md) — sažetak vežbi 5 (Conditions, scale subresurs, Watches predicate, finalizeri, helm filozofija)
- [`../vezbe pomocni materijali/kubernetes_operatori_elegancija.md`](../vezbe%20pomocni%20materijali/kubernetes_operatori_elegancija.md) — kompletan operator dev vodič
- [`../vezbe pomocni materijali/docker_elegancija.md`](../vezbe%20pomocni%20materijali/docker_elegancija.md) — Docker prakse
- [`../vezbe pomocni materijali/kubernetes_elegancija.md`](../vezbe%20pomocni%20materijali/kubernetes_elegancija.md) — K8s osnovni resursi
- [`../workflow-cheatsheet.md`](../workflow-cheatsheet.md) — git/WSL workflow gotchas (Spotahome dead, kubebuilder multigroup, itd.)

---

## Podela posla po članovima tima

Pošto se mnogo zadataka preklapa, fiksiramo **vlasnike** ali jedni drugima pomažemo. Vlasnik = osoba koja pravi PR i odgovorna je da stvar radi do kraja.

| Član | Glavna oblast | Sekundarno |
|------|---------------|------------|
| **A — Platform/K8s** | Shop operator (Go/Kubebuilder), CRD-ovi, finalizer za Discord/Wallet, Helm chart operatora | RBAC, kube-state repo |
| **B — Backend/Apps** | Shop backend (Go/Gin), ShopHub backend (Go + client-go za pravljenje CR-ova) | Testcontainers, integracioni testovi |
| **C — DevOps/Frontend** | Frontend-ovi (Shop i ShopHub), CI/CD pipeline-ovi, Observability (Grafana/Prometheus/Loki/Tempo), Helm-ovi za aplikacije | Web3 integracija u Shop frontend |

> **Pravilo o code review-u:** PR mora odobriti **bar 1 drugi član tima** (zahtev 5.1). Ne self-merge. Tim od 3 to lako podnosi.

---

## 0. Faza 0 — preduslov za predaju (eliminaciono)

> **Trajanje: dan 1.** Ako ovo ne uradite na vreme, ostalo nema smisla.

Cilj: kreirati **svih 5 repozitorijuma na GitHub-u** sa pravilnom konfiguracijom branch protection-a i strukturom direktorijuma. **Tek posle ovoga** počnite sa kodom.

### 0.1. Kreiranje repozitorijuma

Mora 5 odvojenih repo-a (zahtev 5.1: "svaki mikroservis ima svoj repozitorijum"):

- [ ] `shop-operator` — Go/Kubebuilder operator + CRD-ovi
- [ ] `shop` — Shop aplikacija (backend + frontend monorepo, ili razdvojeno)
- [ ] `shophub` — ShopHub aplikacija (backend + frontend)
- [ ] `helm-charts` — svi Helm Chart-ovi koje sami pišete
- [ ] `kube-state` — stanje klastera (ArgoCD/Flux ili sirov values override)

> **Napomena o "monorepo vs polyrepo" za Shop/ShopHub:** specifikacija kaže "svaki mikroservis svoj repo". Shop backend i Shop frontend su tehnički ista aplikacija (vlasnik shop-a), pa idu zajedno u `shop` repo-u sa folderima `backend/` i `frontend/`. Isto za `shophub`. Ako budete imali više backend mikroservisa unutar Shop-a (npr. `shop-api` + `shop-payments`), svaki dobija svoj repo.

### 0.2. Branch protection (na svakom repo-u)

Ide u **Settings → Branches → Add rule** za `main`:

- [ ] Require pull request before merging
- [ ] Require approvals: **1**
- [x] Require status checks to pass before merging — required check imena moraju da odgovaraju **job-name**-u (ne workflow-name-u) i da postoje u tom repo-u. Po repo-u: shop/shophub/shop-operator = Docker Build/Lint/Tests/commitlint; helm-charts = Helm Lint/commitlint; kube-state = commitlint.
- [ ] **Require linear history** ✅ (zahtev 5.1)
- [ ] Do not allow bypassing the above settings

> **Trunk-Based Development**: radimo iz `main` (= trunk), feature grane su kratkoživeće (1–3 dana). Bez `develop` grane — TBD eksplicitno preporučuje samo trunk + release grane po potrebi. README pominje "linearnu istoriju master i develop grana", ali u praksi ako ide TBD, samo `main` je dovoljno; ako radite klasičan GitFlow, dodajte i `develop`.

### 0.3. Conventional Commits — pravilo za poruke

Sve commit poruke u formatu `<type>(<scope>): <opis>`. Tipovi:

- `feat` — nova funkcionalnost
- `fix` — bagfix
- `chore` — administrativno (build, deps)
- `docs` — dokumentacija
- `refactor` — refactor bez promene ponašanja
- `test` — dodavanje/ispravka testova
- `ci` — izmene CI/CD pipeline-a

Primeri:
```
feat(shop-operator): add Wallet CRD spec validation
fix(shop): correct cart total calculation for USDT
ci(shophub): add integration test stage with testcontainers
```

> **Tip:** dodajte `commitlint` GitHub Action u svaki repo da automatski validira commit poruke. Ako ne želite da gnjavite, bar pišite ručno po pravilima.

### 0.4. Inicijalna struktura `helm-charts` repozitorijuma

```
helm-charts/
├── README.md
└── charts/
    ├── shop/                 # Shop aplikacija (backend + frontend)
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    ├── shophub/              # ShopHub aplikacija
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   ├── charts/           # subchart-ovi (npr. kube-prometheus-stack)
    │   └── templates/
    │       ├── NOTES.txt
    │       ├── _helpers.tpl
    │       ├── deployment.yaml
    │       ├── hpa.yaml
    │       ├── ingress.yaml
    │       ├── service.yaml
    │       └── serviceaccount.yaml
    └── shop-operator/        # Helm chart Shop operator-a (zahtev 3.2)
        ├── Chart.yaml
        ├── values.yaml
        ├── crds/             # CRD YAML-ovi (auto-instaliraju se prvi)
        └── templates/
```

> **Bitno za `crds/` folder Helm-a:** Helm 3 automatski instalira sve YAML-ove iz `crds/` foldera **pre** template-ova, i **nikad ih ne dira na upgrade**. Zato CRD-ovi idu tu, ne u `templates/`. Ako želite da se CRD-ovi update-uju na helm upgrade (rizično), morate ih prebaciti u `templates/` sa `helm.sh/hook: crd-install` (deprecated) ili koristiti `kubebuilder edit --plugins=helm/v2-alpha` koji generiše chart na svoj način — pogledajte sekciju 12 u `kubernetes_operatori_elegancija.md`.

### 0.5. Inicijalna struktura `kube-state` repozitorijuma

```
kube-state/
├── README.md
└── clusters/
    └── local/
        ├── cluster.yaml          # k3d cluster config
        ├── shop-operator/
        │   ├── helm.yaml         # OCI ref Helm chart-a
        │   └── values.yaml       # value override
        ├── shophub/
        │   ├── helm.yaml
        │   └── values.yaml
        ├── cnpg/                 # CloudNativePG operator
        │   ├── helm.yaml
        │   └── values.yaml
        ├── redis-operator/       # ako idete sa Redis-om kao "light" bazom
        │   ├── helm.yaml
        │   └── values.yaml
        └── kube-prometheus-stack/
            ├── helm.yaml
            └── values.yaml
```

`cluster.yaml` (preuzeto iz operator eleganta):
```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: local
servers: 1
agents: 2          # 2 worker noda za realniji raspored
ports:
  - port: 8080:80
    nodeFilters:
      - loadbalancer
  - port: 8443:443
    nodeFilters:
      - loadbalancer
```

`helm.yaml` (primer za shop-operator):
```yaml
chart: oci://docker.io/<vas-username>/shop-operator
version: 0.1.0
namespace: shop-operator-system
createNamespace: true
```

> **Opciono ali preporučeno:** koristite **ArgoCD** ili **Flux**. Ako idete sa ArgoCD-om, struktura `kube-state` postaje skup `Application` manifest-a. Najlakše je krenuti bez GitOps alata, pa ga dodati u Fazi 7 ako stignete — to nosi **opcione bodove**.

### Checkpoint 0 ✅
- [ ] Svih 5 repo-a postoji na GitHub-u sa branch protection-om.
- [ ] `helm-charts` i `kube-state` imaju strukturu foldera (može i samo prazni README + `.gitkeep`).
- [ ] U svakom repo-u postoji prvi commit "chore: initial structure" po Conventional Commits.

---

## 1. Faza 1 — temelji (sedmica 1)

Cilj: imati **lokalni klaster sa CNPG i Redis operatorom**, **2 docker image-a koja se pokreću u njemu**, i **prvu skicu Shop CRD-a** (još bez kontrolera).

### 1.1. Alati — proveriti verzije (svi članovi)

```bash
go version          # >= 1.25.6
kubebuilder version # >= 4.11.0 (kubebuilder version)
k3d version         # >= 5.8.3
docker version
helm version        # >= 3.14
kubectl version --client
```

Ako neko nema neku verziju, instalira pre dalje. Kubebuilder eleganta su **stroge oko verzija** — ne odstupati.

### 1.2. Pokretanje lokalnog klastera (svi članovi, lokalno svako kod sebe)

```bash
cd kube-state/clusters/local
k3d cluster create --config cluster.yaml
kubectl cluster-info
kubectl get nodes
```

### 1.3. Instalacija CNPG operatora

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  cnpg/cloudnative-pg
kubectl -n cnpg-system get pods
```

### 1.4. Instalacija Redis operatora

> **Odluka tima:** Redis Enterprise (REDB) traži licencu i komplikovaniju instalaciju. **Preporuka: koristiti `spotahome/redis-operator`** (otvorena alternativa) ili **MongoDB community operator** ako vam je MongoDB komotniji. Specifikacija eksplicitno kaže "može MongoDB ako se koristi operator". **Predlog: idemo sa Spotahome Redis operatorom** — najlakše + ostaje "Redis" kao u README-u.

```bash
helm repo add redis-operator https://spotahome.github.io/redis-operator
helm upgrade --install redis-operator \
  --namespace redis-operator \
  --create-namespace \
  redis-operator/redis-operator
```

> Pre commit-a u `kube-state`, dodajte `helm.yaml` i `values.yaml` za oba operatora. Tako kompletan tim zna kako se klaster postavlja.

### 1.5. Inicijalizacija Shop operatora (član A)

```bash
mkdir -p shop-operator && cd shop-operator
kubebuilder init \
  --domain shophub.local \
  --project-name shop-operator \
  --repo github.com/<org>/shop-operator
```

Dodaj 3 API-ja (jedan po CRD-u):

```bash
kubebuilder create api --group apps --version v1 --kind Shop \
  --resource --controller --plural shops

kubebuilder create api --group notify --version v1 --kind DiscordChannel \
  --resource --controller --plural discordchannels

kubebuilder create api --group payments --version v1 --kind Wallet \
  --resource --controller --plural wallets
```

Dobija se 3 grupe: `apps.shophub.local`, `notify.shophub.local`, `payments.shophub.local`. To je čistije nego sve u istoj grupi.

### 1.6. Skeleton Shop backend-a (član B)

```bash
mkdir -p shop && cd shop
go mod init github.com/<org>/shop
mkdir -p backend frontend
cd backend
go get github.com/gin-gonic/gin
go get github.com/jackc/pgx/v5
```

`backend/main.go` minimum:
```go
package main

import (
    "log"
    "net/http"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    r.GET("/probe/liveness", func(c *gin.Context) { c.String(http.StatusOK, "ok") })
    r.GET("/probe/readiness", func(c *gin.Context) { c.String(http.StatusOK, "ok") })
    r.GET("/", func(c *gin.Context) { c.JSON(200, gin.H{"app": "shop"}) })
    log.Fatal(r.Run(":8080"))
}
```

> Ovo je dovoljno da napravimo Dockerfile i image — funkcionalnost dolazi u Fazi 3. **Liveness/readiness probe su obavezni** (vidi `kubernetes_elegancija.md` sekcija 2).

### 1.7. Multi-stage Dockerfile za Shop backend (član B/C)

Po `docker_elegancija.md` sekcija 1.1: multi-stage, non-root user, slim base, exec CMD, hadolint pass.

`shop/backend/Dockerfile`:
```dockerfile
# syntax=docker/dockerfile:1.6
ARG GO_VERSION=1.25.6

FROM golang:${GO_VERSION}-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux go build -ldflags='-s -w' -o /out/shop ./...

FROM alpine:3.20 AS release
RUN apk add --no-cache ca-certificates && \
    addgroup -S app && adduser -S app -G app
COPY --from=builder /out/shop /app/shop
RUN chown -R root:root /app && chmod 0755 /app/shop
USER app
WORKDIR /app
EXPOSE 8080
ENTRYPOINT ["/app/shop"]
```

**Provera (zahtev iz checklist-e u docker eleganti):**
```bash
docker run --rm -i hadolint/hadolint < Dockerfile
docker buildx build -t shop-backend:dev .
docker run --rm -p 8080:8080 shop-backend:dev
```

### 1.8. Učitavanje image-a u k3d klaster

```bash
docker tag shop-backend:dev shop-backend:dev
k3d image import shop-backend:dev -c local
```

### Checkpoint 1 ✅
- [ ] Lokalni k3d klaster radi (svi članovi tima isto reproduktivno).
- [ ] CNPG operator instaliran (`kubectl get pods -n cnpg-system` → Running).
- [ ] Redis operator instaliran.
- [ ] `shop-operator` projekat inicijalizovan, 3 CRD skeletona generisana.
- [ ] Shop backend skeleton + Dockerfile prolazi `hadolint` i pokreće se u kontejneru.
- [ ] Identičan setup za ShopHub backend (član B/C — paralelan zadatak Fazi 1.6).

---

## 2. Faza 2 — Shop operator (sedmice 2–3)

> **Vlasnik: član A.** Najveći komad posla u projektu. **Sve odluke ovde su detaljno pokrivene u `kubernetes_operatori_elegancija.md`** — pratiti ga doslovno.

### 2.1. Spec i Status — Shop CRD

Cilj: korisnik kreira `Shop` CR, operator orkestrira sve šta Shop aplikacija zahteva (Deployment, Service, Ingress, baza preko CNPG/Redis CR-a, Wallet, DiscordChannel, ServiceMonitor za Prometheus).

`api/v1/shop_types.go` (delovi):
```go
package v1

import (
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +kubebuilder:validation:Enum=standard;high
type Availability string

const (
    AvailabilityStandard Availability = "standard"
    AvailabilityHigh     Availability = "high"
)

// +kubebuilder:validation:Enum=postgres;redis
type DatabaseKind string

const (
    DatabasePostgres DatabaseKind = "postgres"
    DatabaseRedis    DatabaseKind = "redis"
)

type ShopSpec struct {
    // Title je ime prodavnice — obavezno polje, koristi se i kao display name.
    Title string `json:"title"`

    // +kubebuilder:default:=standard
    Availability Availability `json:"availability"`

    // +kubebuilder:default:=postgres
    Database DatabaseKind `json:"database"`

    // WalletAddress je adresa kripto wallet-a vlasnika prodavnice.
    WalletAddress string `json:"walletAddress"`

    // +optional
    // DiscordWebhookSecretRef pokazuje na Secret koji čuva Discord webhook URL.
    DiscordWebhookSecretRef *corev1.SecretReference `json:"discordWebhookSecretRef,omitempty"`

    // +optional
    // Image override — koristi se ako CI/CD pushuje novu verziju Shop image-a.
    Image *string `json:"image,omitempty"`
}

type ShopStatus struct {
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // +optional
    URL string `json:"url,omitempty"`           // ingress hostname
    // +optional
    DatabaseSecret string `json:"databaseSecret,omitempty"`
    // +optional
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`
    // +optional
    DesiredReplicas int32 `json:"desiredReplicas,omitempty"`
    // +optional
    Selector string `json:"selector,omitempty"` // za /scale subresource
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.availability,statuspath=.status.readyReplicas,selectorpath=.status.selector
// +kubebuilder:resource:shortName=sh,categories={shophub}
// +kubebuilder:printcolumn:name="TITLE",type="string",JSONPath=".spec.title"
// +kubebuilder:printcolumn:name="DB",type="string",JSONPath=".spec.database"
// +kubebuilder:printcolumn:name="READY",type="integer",JSONPath=".status.readyReplicas"
// +kubebuilder:printcolumn:name="URL",type="string",JSONPath=".status.url"
// +kubebuilder:printcolumn:name="AGE",type="date",JSONPath=".metadata.creationTimestamp"
type Shop struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   ShopSpec   `json:"spec,omitempty"`
    Status ShopStatus `json:"status,omitempty"`
}
```

> **Pravila iz operator eleganta koja primenjujemo:**
> - Obavezni atributi su vrednosti (`Title`, `WalletAddress`), opcioni su pokazivači (`Image`, `DiscordWebhookSecretRef`).
> - Sve u `Status`-u je `+optional`.
> - `subresource:scale` da `kubectl scale shop my-shop --replicas=3` radi (replica koju mapuje *na osnovu* `availability`).
> - Print kolone čine `kubectl get shops` čitljivim.

> **Pažnja na `subresource:scale`:** mapira numerički put. `availability` je string (`standard`/`high`), pa to ne radi direktno. **Dva izlaza:**
> 1. Dodajte u Spec eksplicitno `Replicas *int32` (override) i mapirajte na `.spec.replicas`. Tada `availability` postaje *default* — kontroler na osnovu njega popunjava `replicas` ako je prazno.
> 2. Bez `subresource:scale` (specifikacija ga ne traži, HPA radi i bez toga preko Deployment-a).
>
> **Preporuka: opcija 1.** Daje fleksibilnost a ne razbija specifikaciju.

### 2.2. DiscordChannel CRD

```go
type DiscordChannelSpec struct {
    // GuildID je ID Discord servera.
    GuildID string `json:"guildId"`
    // Name kanala koji se kreira.
    Name string `json:"name"`
    // BotTokenRef pokazuje na Secret koji sadrži Discord bot token.
    BotTokenRef corev1.SecretReference `json:"botTokenRef"`
}

type DiscordChannelStatus struct {
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
    // +optional
    ChannelID string `json:"channelId,omitempty"`
    // +optional
    WebhookURL string `json:"webhookUrl,omitempty"` // upisuje se u Secret, ovde samo metadata
    // +optional
    WebhookSecretName string `json:"webhookSecretName,omitempty"`
}
```

> **OBAVEZAN finalizer** (eksterni resurs — Discord API). Vidi sekciju 13 operator eleganta. Bez finalizer-a, brisanje `DiscordChannel` CR-a ostavlja siroče kanale na Discord serveru.

### 2.3. Wallet CRD

```go
// +kubebuilder:validation:Enum=ethereum;solana;sepolia;polygon-amoy
type Network string

type WalletSpec struct {
    Network Network `json:"network"`
    // OwnerAddress (opciono) — adresa vlasnika kojem se prebacuju sredstva.
    // Ako nije zadato, generiše se nov private key i čuva u Secret-u.
    // +optional
    OwnerAddress *string `json:"ownerAddress,omitempty"`
}

type WalletStatus struct {
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
    // +optional
    Address string `json:"address,omitempty"`
    // +optional
    PrivateKeySecretRef *corev1.SecretReference `json:"privateKeySecretRef,omitempty"`
}
```

> **Finalizer takođe?** Tehnički adresa na blockchain-u se ne briše (transakcije ostaju zauvek), ali **lokalno generisani Secret sa private key-em treba obrisati**. Vidi: kontroler može Secret postaviti kao "owned" preko `OwnerReference` — Kubernetes GC tada brine. Tu nema potrebe za finalizer-om.

### 2.4. Reconciler logika — Shop kontroler

Ključ idemoidentnosti i level-based pristupa. Svaka iteracija zna sve iz `Spec`-a, čita stanje child resursa, dovodi u željeno stanje. Skelet:

```go
func (r *ShopReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)

    // 1. Učitaj Shop CR
    shop := &appsv1.Shop{}
    if err := r.Get(ctx, req.NamespacedName, shop); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Postavi Progressing=True na ulazu (sa pomoćnom metodom)
    if err := r.setProgressStatus(ctx, shop, "Reconciling", "Reconciliation in progress"); err != nil {
        return ctrl.Result{}, err
    }

    // 3. Garantuj postojanje baze (CNPG Cluster ili Redis)
    dbSecretName, err := r.ensureDatabase(ctx, shop)
    if err != nil {
        return ctrl.Result{}, r.setUnrecoverableErrorStatus(ctx, shop, "DatabaseFailed", err.Error())
    }

    // 4. Garantuj postojanje Wallet CR-a
    walletAddress, err := r.ensureWallet(ctx, shop)
    if err != nil {
        return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
    }

    // 5. Garantuj postojanje DiscordChannel CR-a (ako je definisano)
    if shop.Spec.DiscordWebhookSecretRef != nil {
        if err := r.ensureDiscordChannel(ctx, shop); err != nil {
            log.Error(err, "discord channel not ready, will retry")
        }
    }

    // 6. Deploy Shop aplikacije (Deployment + Service + Ingress + ServiceMonitor)
    if err := r.ensureDeployment(ctx, shop, dbSecretName, walletAddress); err != nil {
        return ctrl.Result{}, err
    }
    if err := r.ensureService(ctx, shop); err != nil {
        return ctrl.Result{}, err
    }
    if err := r.ensureIngress(ctx, shop); err != nil {
        return ctrl.Result{}, err
    }
    if err := r.ensureServiceMonitor(ctx, shop); err != nil {
        // Prometheus operator možda nije instaliran — log ali ne fail
        log.Info("ServiceMonitor not created", "reason", err.Error())
    }

    // 7. Update Status na osnovu trenutnog stanja Deployment-a
    if err := r.updateStatusFromDeployment(ctx, shop); err != nil {
        if apierrors.IsConflict(err) {
            return ctrl.Result{}, nil // benigno, petlja će se okinuti
        }
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```

### 2.5. Mapiranje availability na replike

```go
func replicasFor(shop *appsv1.Shop) int32 {
    if shop.Spec.Availability == appsv1.AvailabilityHigh {
        return 3
    }
    return 2 // standard
}
```

### 2.6. ensureDatabase — primer za CNPG

Ovo je gde se najlakše zaglaviti. Logika: **ako Spec.Database == postgres**, kreiraj `postgresql.cnpg.io/v1.Cluster` u istom namespace-u; sačekaj da CNPG postavi `<name>-app` Secret; kopiraj refs u Status.

```go
func (r *ShopReconciler) ensureDatabase(ctx context.Context, shop *appsv1.Shop) (string, error) {
    secretName := shop.Name + "-app" // CNPG default

    if shop.Spec.Database == appsv1.DatabasePostgres {
        cluster := &cnpgv1.Cluster{
            ObjectMeta: metav1.ObjectMeta{Name: shop.Name, Namespace: shop.Namespace},
            Spec: cnpgv1.ClusterSpec{
                Instances: 1,
                Bootstrap: &cnpgv1.BootstrapConfiguration{
                    InitDB: &cnpgv1.BootstrapInitDB{
                        Database: shop.Name,
                        Owner:    shop.Name,
                        PostInitApplicationSQL: []string{
                            `CREATE TABLE IF NOT EXISTS items(
                                id text PRIMARY KEY,
                                name text NOT NULL,
                                price_usdt numeric(36,18) NOT NULL,
                                stock int NOT NULL DEFAULT 0
                            )`,
                            `ALTER TABLE items OWNER TO ` + shop.Name,
                            `CREATE TABLE IF NOT EXISTS orders(
                                id text PRIMARY KEY,
                                buyer_wallet text NOT NULL,
                                tx_hash text,
                                amount_usdt numeric(36,18) NOT NULL,
                                created_at timestamptz DEFAULT now()
                            )`,
                            `ALTER TABLE orders OWNER TO ` + shop.Name,
                        },
                    },
                },
                StorageConfiguration: cnpgv1.StorageConfiguration{Size: "1Gi"},
            },
        }

        if err := ctrl.SetControllerReference(shop, cluster, r.Scheme); err != nil {
            return "", err
        }

        // Idempotent create-or-update
        existing := &cnpgv1.Cluster{}
        err := r.Get(ctx, client.ObjectKeyFromObject(cluster), existing)
        if apierrors.IsNotFound(err) {
            if err := r.Create(ctx, cluster); err != nil {
                return "", err
            }
        } else if err != nil {
            return "", err
        }
        // Posmatraj da postoji Secret
        sec := &corev1.Secret{}
        if err := r.Get(ctx, client.ObjectKey{Namespace: shop.Namespace, Name: secretName}, sec); err != nil {
            return "", err // sledeća iteracija će probati ponovo
        }
        return secretName, nil
    }

    // Redis case — kreiraj `databases.spotahome.com/v1.RedisFailover`
    // analogan kod
    return "", fmt.Errorf("unsupported database: %s", shop.Spec.Database)
}
```

> **Bitno za eksplicitan `OWNER TO`** — pokrivено u operator eleganti sekcija 10. Bez ovoga aplikacija dobija "permission denied" na svojim tabelama jer ih je `postgres` root kreirao.

### 2.7. SetupWithManager — Watches, Indeksi

```go
func (r *ShopReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // Indeks za Watch kroz CNPG Secret-e
    if err := mgr.GetFieldIndexer().IndexField(ctx, &appsv1.Shop{},
        ".spec.title",
        func(obj client.Object) []string {
            shop := obj.(*appsv1.Shop)
            return []string{shop.Spec.Title}
        }); err != nil {
        return err
    }

    cnpgSecretPredicate := predicate.NewPredicateFuncs(func(obj client.Object) bool {
        _, ok := obj.GetLabels()["cnpg.io/cluster"]
        return ok
    })

    return ctrl.NewControllerManagedBy(mgr).
        For(&appsv1.Shop{}).
        Owns(&appsv1k8s.Deployment{}).
        Owns(&corev1.Service{}).
        Owns(&netv1.Ingress{}).
        Owns(&cnpgv1.Cluster{}).
        Watches(
            &corev1.Secret{},
            handler.EnqueueRequestsFromMapFunc(r.findShopForCNPGSecret),
            builder.WithPredicates(cnpgSecretPredicate),
        ).
        Named("shop").
        Complete(r)
}
```

> **Predikat za Secret je krucijalan.** Bez njega operator se okida na svaku izmenu svakog Secret-a u klasteru — CPU/memorija eksplodira (vidi operator eleganta sekciju 8).

### 2.8. RBAC markup-i

Ide na `internal/controller/shop_controller.go`, iznad `Reconcile`:

```go
// +kubebuilder:rbac:groups=apps.shophub.local,resources=shops,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps.shophub.local,resources=shops/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps.shophub.local,resources=shops/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=services;secrets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=networking.k8s.io,resources=ingresses,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=postgresql.cnpg.io,resources=clusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=databases.spotahome.com,resources=redisfailovers,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=monitoring.coreos.com,resources=servicemonitors,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=notify.shophub.local,resources=discordchannels,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=payments.shophub.local,resources=wallets,verbs=get;list;watch;create;update;patch;delete
```

Posle dodavanja: `make manifests`.

### 2.9. Discord kontroler i finalizer

```go
const discordFinalizer = "shophub.local/discord-channel"

func (r *DiscordChannelReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    ch := &notifyv1.DiscordChannel{}
    if err := r.Get(ctx, req.NamespacedName, ch); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Dodaj finalizer ako nedostaje
    if ch.DeletionTimestamp.IsZero() {
        if !controllerutil.ContainsFinalizer(ch, discordFinalizer) {
            controllerutil.AddFinalizer(ch, discordFinalizer)
            if err := r.Update(ctx, ch); err != nil {
                if apierrors.IsConflict(err) { return ctrl.Result{}, nil }
                return ctrl.Result{}, err
            }
        }
    } else {
        // Cleanup eksternog resursa pre brisanja
        if controllerutil.ContainsFinalizer(ch, discordFinalizer) {
            if err := r.deleteDiscordChannel(ctx, ch); err != nil {
                return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
            }
            controllerutil.RemoveFinalizer(ch, discordFinalizer)
            if err := r.Update(ctx, ch); err != nil && !apierrors.IsConflict(err) {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }

    // Reconcile na Discord API: idempotentno (ako kanal već postoji, ne pravi nov)
    channelID, webhookURL, err := r.upsertDiscordChannel(ctx, ch)
    if err != nil {
        return ctrl.Result{}, err
    }

    // Sačuvaj webhook u Secret
    secretName := ch.Name + "-webhook"
    sec := &corev1.Secret{
        ObjectMeta: metav1.ObjectMeta{Name: secretName, Namespace: ch.Namespace},
        StringData: map[string]string{"webhook-url": webhookURL},
    }
    ctrl.SetControllerReference(ch, sec, r.Scheme)
    if err := r.upsertSecret(ctx, sec); err != nil {
        return ctrl.Result{}, err
    }

    ch.Status.ChannelID = channelID
    ch.Status.WebhookSecretName = secretName
    if err := r.Status().Update(ctx, ch); err != nil && !apierrors.IsConflict(err) {
        return ctrl.Result{}, err
    }
    return ctrl.Result{}, nil
}
```

### 2.10. Sample-ovi i unit testovi

`config/samples/apps_v1_shop.yaml`:
```yaml
apiVersion: apps.shophub.local/v1
kind: Shop
metadata:
  name: clothing-store
  namespace: tenant-alice
spec:
  title: "Alice's Clothing Store"
  availability: high
  database: postgres
  walletAddress: "0xAB...123"
```

Kubebuilder generiše prazne `*_controller_test.go` fajlove — popuniti **bar po jedan integracioni test** (envtest) za **svaki** kontroler. Vidi sekciju 11 operator eleganta — svaki test mora ići u **poseban namespace** (envtest ne podržava brisanje namespace-a).

### Checkpoint 2 ✅
- [ ] `make manifests generate` prolazi bez grešaka.
- [ ] `make install` instalira CRD-ove na klaster, `kubectl get crds | grep shophub` pokazuje 3.
- [ ] `make run` pokreće operator lokalno; `kubectl apply -f config/samples/...` na Shop CR pokreće Reconcile petlju.
- [ ] Apply Shop sample-a kreira: Deployment (2 ili 3 replike), Service, Ingress, CNPG Cluster, Wallet CR.
- [ ] Brisanje Shop CR-a → Garbage Collector briše sve child resurse.
- [ ] Brisanje DiscordChannel CR-a → finalizer briše Discord kanal pre Kubernetes GC-a.
- [ ] Bar 1 unit test po kontroleru prolazi (`make test`).
- [ ] Sve stavke iz **kontrolne liste pre commit-a** (sekcija 14 operator eleganta) ispoštovane.

---

## 3. Faza 3 — Shop aplikacija (sedmica 3)

> **Vlasnik: član B (backend), član C (frontend).**

### 3.1. Backend (Go/Gin)

Endpoint-i:
- `GET  /api/items` — lista artikala (admin + korisnik)
- `POST /api/items` — kreiraj artikal (admin)
- `PUT  /api/items/:id` — izmeni artikal (admin)
- `DELETE /api/items/:id` — obriši artikal (admin)
- `GET  /api/orders` — listaj porudžbine (admin)
- `POST /api/orders` — kreiraj porudžbinu (korisnik, sa `tx_hash`)
- `GET  /api/orders/:id/verify` — verifikuj transakciju na blockchain-u
- `GET  /probe/liveness`
- `GET  /probe/readiness`
- `GET  /metrics` — Prometheus exporter

Konekcija ka bazi se uzima iz **env varijabli koje dolaze iz Secret-a** koji generiše CNPG (`<shop-name>-app`):
- `PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`, `PGDATABASE`, `URI`

To je već pripremljeno operatorom — operator pri kreiranju Deployment-a pravi `envFrom: secretRef: <shop-name>-app`.

### 3.2. Auth — admin vs korisnik

Pošto je u istom Shop frontend-u i admin (vlasnik) i korisnik (kupac), **dva login flow-a**:
- Admin: classic JWT (email + password). U bazi: `admins(id, email, password_hash)`.
- Korisnik: Web3 wallet sign-in (npr. `eth_sign` / SIWE — Sign-In With Ethereum). U bazi: `users(wallet_address, last_login)`.

> **Ovo je veliki obim posla.** Ako vam vreme curi, **Web3 sign-in se može preskočiti** za MVP — samo "connect wallet" za plaćanje, bez login-a (anonimna kupovina, identifikacija preko wallet adrese tokom plaćanja). Ovo je razumna pozicija jer zahtev 2.4 pominje **plaćanje kriptovalutom**, ne **autentifikaciju**.

### 3.3. Funkcionalnosti

- 2.1 — CRUD artikala (admin).
- 2.2 — listanje porudžbina (admin).
- 2.3 — pretraga + korpa (korisnik). Korpa može biti samo client-side state.
- 2.4 — plaćanje (vidi Faza 5).

### 3.4. Frontend (Next.js ili React/Vite)

> **Predlog tima — React + Vite + TypeScript** ili Next.js. Obe rade. Vite je manji i brže se startuje. Tailwind za styling. Komponente:
> - `AdminLogin`, `AdminDashboard` (CRUD artikala, lista porudžbina)
> - `Storefront` (lista artikala, pretraga, korpa)
> - `Checkout` (Web3 connect, "Pay with USDT" dugme, MetaMask integracija)

### 3.5. Dockerfile za frontend

Frontend **interpretirani** — multi-stage svejedno:

```dockerfile
# syntax=docker/dockerfile:1.6
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

FROM deps AS builder
COPY . .
RUN npm run build

FROM nginx:1.27-alpine AS release
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
USER app
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

> **Bitno:** non-root nginx mora imati port > 1024. Stoga port 8080 (ne 80) i `nginx.conf` koji listen-uje na 8080.

### Checkpoint 3 ✅
- [ ] Shop backend prolazi liveness/readiness na portu 8080.
- [ ] Endpoints rade lokalno (`go run` + lokalni Postgres preko docker compose ili Testcontainers).
- [ ] Frontend se gradi i serve-uje statički.
- [ ] Oba image-a su build-ovana, hadolint čist, učitana u k3d (`k3d image import`).
- [ ] Apply-ovan Shop sample → operator deploy-uje aplikaciju → otvori Ingress URL u browser-u → vidi se storefront.

---

## 4. Faza 4 — ShopHub aplikacija (sedmica 4)

> **Vlasnik: član B (backend), član C (frontend).**

### 4.1. Backend (Go/Gin + client-go)

ShopHub backend ima dve glavne odgovornosti:
1. **Upravljanje korisnicima** (registracija, login).
2. **Upravljanje Shop CR-ovima** preko Kubernetes API-ja — `client-go` ili `controller-runtime/client`.

Endpoint-i:
- `POST /api/auth/register`
- `POST /api/auth/login`
- `GET  /api/shops` — lista shop-ova trenutnog korisnika
- `POST /api/shops` — kreira Shop CR u namespace-u korisnika
- `PUT  /api/shops/:name` — update Spec-a (availability, walletAddress)
- `DELETE /api/shops/:name` — obriši Shop CR
- `GET  /api/shops/:name/status` — status Shop-a (čitanje Status sekcije CR-a)

### 4.2. Multi-tenancy preko namespace-a

Svaki korisnik ShopHub-a dobija **svoj namespace** (`tenant-<userId>`). Sve njegove Shop CR-ove operator pravi tu. Razlozi:
- Izolacija kvota (`ResourceQuota` po namespace-u).
- Lakše brisanje (drop namespace = drop sve).
- Prirodno mapuje na RBAC.

Pri registraciji ShopHub backend kreira namespace:
```go
ns := &corev1.Namespace{ObjectMeta: metav1.ObjectMeta{Name: "tenant-" + userID}}
if err := k8sClient.Create(ctx, ns); err != nil && !apierrors.IsAlreadyExists(err) {
    return err
}
```

### 4.3. RBAC za ShopHub backend

ShopHub backend ima **ServiceAccount** u svom namespace-u. ClusterRole sa pravima:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: shophub-backend
rules:
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "create", "delete"]
  - apiGroups: ["apps.shophub.local"]
    resources: ["shops"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

ClusterRoleBinding na ServiceAccount.

### 4.4. Frontend ShopHub-a ✅ GOTOVO

GOTOVO (Railway-style klon, vidi repo-root `frontend_guide.md`): Vite+React+TS+
Tailwind+framer-motion. Landing (hero + animirani mockup-ovi, scroll reveals,
žive data-transfer čestice), JWT auth (login/register), dashboard (lista/create/
delete Shop-ova preko `/api/shops`). Dokazano e2e: register→login→create shop→
kartica (Playwright demo protiv pravog klastera).


- `Login` / `Register` (klasično email + password ili Web3 wallet — opcionalan zahtev).
- `Dashboard` — lista shop-ova, "Create new shop" forma:
  - Title (text)
  - Availability (radio: standard / high)
  - Wallet address (text)
  - Database (select: postgres / redis)
- "Edit shop" form, "Delete shop" dugme.
- Klik na shop → otvara Ingress URL u novom tabu (preko `target="_blank"`).

### Checkpoint 4 ✅
- [ ] ShopHub backend deploy-ovan u namespace `shophub-system`.
- [ ] Korisnik se registruje → namespace se kreira.
- [ ] Korisnik kreira shop preko UI → Shop CR se pojavi (`kubectl get shops -A`).
- [ ] Operator hvata CR i deploy-uje shop.
- [ ] Brisanje shop-a iz UI → Shop CR briše → GC briše child-ove.

---

## 5. Faza 5 — Web3 plaćanje (sedmica 4–5)

> **Vlasnik: član C uz pomoć člana B.**

### 5.1. Stack izbor

**Preporuka tima:**
- **Mreža:** Sepolia testnet (Ethereum) — najlakše, najviše tutoriala, MetaMask radi out-of-the-box.
- **Token:** USDT on Sepolia — postoji testnet adresa, može da se nabavi preko faucet-a.
- **Wallet:** MetaMask.
- **Library:** `ethers.js` u frontend-u, `go-ethereum` (`geth`) u backend-u za verifikaciju transakcija.

> Solana (Phantom) je takođe dobra, ali Solana SDK je manje podržan u Go-u. Sepolia + MetaMask = najmanje rizika.

### 5.2. Flow plaćanja

1. Korisnik popuni korpu, klikne "Pay with USDT".
2. Frontend čita `walletAddress` Shop-a iz `/api/shop-info`.
3. Frontend traži MetaMask da pošalje USDT na taj address (ERC-20 `transfer`).
4. MetaMask vraća `txHash`.
5. Frontend POST-uje `{ items: [...], txHash, buyerAddress }` na `/api/orders`.
6. Shop backend snima `pending` order, pokreće verifikaciju u pozadini (worker goroutine):
   - Periodično (`time.Tick(15s)`) traži tx preko `eth_getTransactionReceipt`.
   - Kad nađe, parsira logove, verifikuje da je `Transfer` event ka pravom address-u za pravu sumu.
   - Update status: `confirmed`, smanji stock, snimi.
7. Frontend polluje `/api/orders/:id` dok ne vidi `confirmed`.

### 5.3. Korisni resursi

- `ethers.js` v6 dokumentacija (`Contract`, `parseUnits`, `formatUnits`).
- USDT ERC-20 ABI (samo `transfer`, `Transfer`, `decimals` — minimalno).
- Sepolia faucet: `sepoliafaucet.com` ili Alchemy faucet.
- Sepolia USDT testnet adresa — proverite na ažurnom resursu, varira.

### Checkpoint 5 ✅
- [x] Demo wallet ima Sepolia ETH (za gas) i Sepolia USDT (self-deployed ERC-20 `0x74b0ef...`).
- [x] Frontend može da inicira `transfer` preko MetaMask (ethers v6, Buy with USDT).
- [x] Backend uspešno verifikuje testnu transakciju (go-ethereum, Transfer event match).
- [x] Order prelazi iz `pending` → `confirmed` automatski (poller, idempotentan po replikama).
- [x] Stock se smanjuje.

---

## 6. Faza 6 — Observability stack (sedmica 5)

> **Vlasnik: član C.** Najsitničavije zahtevi su **u README sekciji 4.1**. Pažljivo!

### 6.1. Komponente

- **kube-prometheus-stack** (Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics) — za metrike + Grafana.
- **Loki + Promtail** — za logove.
- **Tempo + OpenTelemetry collector** — za tracing.

> Sve tri stvari (metrics, logs, traces) traži zahtev 4.1 ("tracing, logging i metrike za svaku Shop aplikaciju"). Loki/Tempo se mogu instalirati posebnim Helm chart-ovima ili kroz `grafana/loki-stack` i `grafana/tempo` charts.

### 6.2. Inštrumentacija Shop backend-a

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequests = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "shop_http_requests_total"},
        []string{"method", "path", "status"},
    )
    uniqueVisitors = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "shop_unique_visitors_total"},
        []string{"shop_name"},
    )
    bytesTransferred = prometheus.NewCounter(
        prometheus.CounterOpts{Name: "shop_bytes_transferred_total"},
    )
    notFoundEndpoints = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "shop_not_found_endpoints_total"},
        []string{"path"},
    )
)
```

> **Šta zahtev 4.1 traži tačno (ponavljam radi jasnoće):**
> - HTTP zahteva ukupno (24h)
> - Uspešni 2xx/3xx (24h)
> - Neuspešni 4xx/5xx (24h)
> - Jedinstveni posetioci (IP + timestamp + browser kao kompozitni ključ — držati hash u memoriji ili Redis-u sa TTL 24h)
> - 404 sa endpoint-ovima (24h)
> - Ukupan saobraćaj u GB
> - CPU, RAM, FS, network — to dolazi besplatno preko `node-exporter` i `cAdvisor`-a.

### 6.3. ServiceMonitor — operator pravi automatski

U Shop kontroler-u, `ensureServiceMonitor`:
```go
sm := &monv1.ServiceMonitor{
    ObjectMeta: metav1.ObjectMeta{Name: shop.Name, Namespace: shop.Namespace,
        Labels: map[string]string{"release": "kube-prometheus-stack"}}, // VAŽNO
    Spec: monv1.ServiceMonitorSpec{
        Selector: metav1.LabelSelector{MatchLabels: map[string]string{"app": shop.Name}},
        Endpoints: []monv1.Endpoint{{Port: "http", Path: "/metrics", Interval: "30s"}},
    },
}
```

> **Zašto `release: kube-prometheus-stack` label?** Default Prometheus selector pokupi samo ServiceMonitor-e sa tim label-om. Bez ovoga metrike se neće skupljati. Alternative: konfigurišite Prometheus `serviceMonitorSelector: {}` (sve).

### 6.4. Grafana dashboard po Shop-u

Per-Shop dashboard se može:
- **Ručno** napisati JSON dashboard sa template varijablom `$shop_name` i sačuvati u `helm-charts/charts/shophub/dashboards/`. Helm `ConfigMap` sa labelom `grafana_dashboard: "1"` se automatski importuje (ako je `grafana.sidecar.dashboards.enabled: true`).
- **Preko `GrafanaDashboard` CRD-a** ako instalirate `grafana-operator`.

> **Najjednostavnije:** jedan parametrizovan dashboard. Korisnik bira `shop_name` iz dropdown-a koji se popunjava iz `label_values(shop_http_requests_total, shop_name)`.

### 6.5. Alarmi

Definisati `PrometheusRule` resurse. Primeri pravila (slobodno smišljati svoja, README to ostavlja studentima):
- "Shop down": `up{job="shop"} == 0` for 2m.
- "High 5xx rate": `rate(shop_http_requests_total{status=~"5.."}[5m]) > 0.1`.
- "Pod restart loop": `kube_pod_container_status_restarts_total > 5` for 5m.
- "Node CPU saturation": `node_load1 > <num_cpu_cores>` for 10m.

Alertmanager config za Discord — koristi `discord_configs` (od Prometheus 0.27+) ili **webhook na shophub-discord servis** koji prevodi i šalje na Discord.

### 6.6. Pristup Grafani

> README: "Pristup grafani imaju samo maintainer-i koji održavaju aplikacije. Korisnici ShopHub-a nemaju pristup Grafana dashboard-u."

Najjednostavnije: Grafana iza basic auth-a (default), credentials se daju samo timu. Ingress sa whitelist-om IP-ja je takođe varijanta.

> **Opcionalna sekcija** za bonus: per-tenant pristup Grafani. To traži Grafana sa OAuth-om (Keycloak ili `auth.proxy`), team-ove po tenant-u, pristup samo svojim dashboard-ima. **Veliki obim posla.** Preskočiti za MVP, ostaviti za "ako stignemo".

### Checkpoint 6 ✅
- [ ] kube-prometheus-stack u klasteru, Prometheus, Grafana, Alertmanager Running.
- [ ] Loki + Promtail instalirani, logovi Shop-a vidljivi u Grafani (Explore → Loki).
- [ ] Tempo instaliran, traces stižu (ako ste integrisali OpenTelemetry).
- [ ] ServiceMonitor automatski kreiran za svaki Shop, `targets` u Prometheusu pokazuje UP.
- [ ] Grafana dashboard za Shop prikazuje SVIH 6 metrika iz zahteva 4.1.
- [ ] Klaster dashboard pokazuje CPU/RAM/FS/network nodova.
- [ ] Bar 2 alarma definisana, demo simulira okidanje (npr. ručno ubiti pod), Discord poruka stigla.

---

## 7. Faza 7 — Helm Chart-ovi i kube-state

> **Vlasnik: zajednički, koordinator A.** **Ovo je deo eliminacionog zahteva 5.3.**

### 7.1. `helm-charts/charts/shop-operator/`

```
shop-operator/
├── Chart.yaml
├── values.yaml
├── crds/
│   ├── apps.shophub.local_shops.yaml
│   ├── notify.shophub.local_discordchannels.yaml
│   └── payments.shophub.local_wallets.yaml
└── templates/
    ├── deployment.yaml
    ├── serviceaccount.yaml
    ├── clusterrole.yaml
    ├── clusterrolebinding.yaml
    ├── service.yaml          # za /metrics
    └── servicemonitor.yaml   # da operator sam šalje metrike
```

`Chart.yaml`:
```yaml
apiVersion: v2
name: shop-operator
description: Shop operator for ShopHub platform
type: application
version: 0.1.0
appVersion: "0.1.0"
```

> CRD-ovi prepisati iz `shop-operator/config/crd/bases/*.yaml` (output od `make manifests`). **Automatizovati:** Make target `make helm-sync` koji kopira CRD-ove i deployment template iz `config/` u `helm-charts/charts/shop-operator/`.

### 7.2. `helm-charts/charts/shophub/`

ShopHub aplikacija sa Helm dependency-jima:

`Chart.yaml`:
```yaml
apiVersion: v2
name: shophub
version: 0.1.0
dependencies:
  - name: kube-prometheus-stack
    version: "65.x.x"
    repository: https://prometheus-community.github.io/helm-charts
  - name: loki-stack
    version: "2.10.x"
    repository: https://grafana.github.io/helm-charts
```

> Ovo zadovoljava zahtev 3.3 ("ShopHub Helm Chart koji koristi CRD-ove iz Shop operator-a i prometheus-stack-a").

`values.yaml` — sva podesiva polja (image tagovi, resources, ingress hostname, kreds...).

### 7.3. `helm-charts/charts/shop/`

> **Pažnja:** `shop` chart **ne koristi se direktno** u smislu da `helm install shop` korisnik ručno pokreće. Operator pravi resurse direktno preko `client-go`. Šta onda ovaj chart radi?
>
> **Dve opcije:**
> 1. **Ne pravi se** — operator radi sve. README sekcija 5.3 pokazuje `shop/` u strukturi, ali nije jasno da li mora postojati.
> 2. **Postoji kao referenca/template** — operator interno koristi go-template logiku iz Helm chart-a (preko `helm template` library-ja) da generiše YAML-ove. Ovo je naprednije, ali vrlo elegantno.
>
> **Preporuka:** napraviti **minimalan Shop chart** kao "fallback" za ručnu instalaciju (debug bez operatora) — zadovoljava strukturu iz README-a a ne kompikuje operator.

### 7.4. Linting

```bash
helm lint charts/shop-operator
helm lint charts/shophub
helm lint charts/shop
```

CI mora da prolazi `helm lint` na svakom PR-u.

### 7.5. Publish kao OCI

DockerHub može hostovati Helm chart-ove kao OCI artefakte:
```bash
helm package charts/shop-operator
helm push shop-operator-0.1.0.tgz oci://docker.io/<vas-username>
```

`kube-state` onda referencira:
```yaml
chart: oci://docker.io/<vas-username>/shop-operator
version: 0.1.0
```

### Checkpoint 7 ✅
- [ ] Sva 3 chart-a u `helm-charts/charts/`, `helm lint` clean.
- [ ] Chart-ovi push-ovani na DockerHub kao OCI.
- [ ] `kube-state/clusters/local/` ima `helm.yaml` + `values.yaml` za sve komponente.
- [ ] Ručna provera: na svežem k3d klasteru, prateći samo upute u `kube-state/README.md`, ceo sistem se diže.

---

## 8. Faza 8 — CI/CD pipeline-ovi

> **Vlasnik: član C.** Zahtev 5.2.

### 8.1. Tri pipeline-a (`shop-operator`, `shop`, `shophub`)

GitHub Actions, fajl `.github/workflows/ci.yml` u svakom repo-u:

```yaml
name: CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with: { go-version: '1.25.6' }

      - name: Cache go modules
        uses: actions/cache@v4
        with:
          path: |
            ~/.cache/go-build
            ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}

      - name: Lint
        run: |
          go install honnef.co/go/tools/cmd/staticcheck@latest
          staticcheck ./...

      - name: Hadolint Dockerfile
        uses: hadolint/[email protected]
        with: { dockerfile: Dockerfile }

      - name: Unit tests
        run: go test ./... -short

      - name: Integration tests (testcontainers)
        run: go test ./... -run Integration

  build-image:
    needs: build-test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Determine version
        id: ver
        run: |
          # SemVer iz git-a (tag ili "0.0.0-<sha>")
          VERSION=$(git describe --tags --always)
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build & push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/shop-operator:${{ steps.ver.outputs.version }}
            ${{ secrets.DOCKERHUB_USERNAME }}/shop-operator:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### 8.2. Integracioni testovi sa Testcontainers

Backend testovi koji traže pravu bazu — koristiti `testcontainers-go`:
```go
import "github.com/testcontainers/testcontainers-go/modules/postgres"

func TestOrderCreation(t *testing.T) {
    ctx := context.Background()
    pg, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("test"),
        postgres.WithUsername("u"), postgres.WithPassword("p"))
    // ...
}
```

### 8.3. Branch protection — povezivanje sa pipeline-om

Settings → Branches → main rule → Require status checks → odaberi `build-test` i `build-image` kao required. Bez prolaska, PR ne može merge-ovati (zahtev 5.1).

### Checkpoint 8 ✅
- [x] CI prolazi na svaki PR (build + lint + test + hadolint).
- [x] Push na main automatski build-uje image i push-uje na DockerHub sa SemVer tag-om.
- [x] Branch protection blokira merge ako CI ne prolazi (required checks po repo-u, name-matched).

---

## 9. Faza 9 — Završno testiranje i demo

### 9.1. Demo skripta

Pre odbrane, napravite **`demo.md`** sa skriptom (10–15 min):
1. Pokazati `helm-charts` strukturu i kube-state strukturu (eliminacioni!).
2. Sa nule podići k3d klaster (`k3d cluster create --config cluster.yaml`).
3. Helm install: cnpg, redis-operator, shop-operator, shophub.
4. Otvoriti ShopHub UI → registracija → kreiranje shop-a.
5. `kubectl get shops -A`, `kubectl describe shop ...` — pokazati Status, Conditions.
6. Otvoriti Shop UI → admin login → dodati artikle.
7. Iz drugog browser-a → kupac flow → korpa → pay sa MetaMask (Sepolia).
8. Verifikovati order u Shop adminu, smanjenje stock-a.
9. Otvoriti Grafana dashboard → pokazati metrike.
10. Simulirati "Shop down" alarm → Discord poruka.
11. Brisanje shop-a iz ShopHub UI → pokazati GC (Deployment, Service, CNPG Cluster nestaju).

### 9.2. Stvari koje se često zaborave

- [ ] README.md za **svaki** repo (kako se gradi, kako se pokreće lokalno).
- [ ] `LICENSE` fajl (MIT je default).
- [ ] `.gitignore` — `bin/`, `dist/`, `.idea/`, `*.local` itd.
- [ ] `.dockerignore` — `node_modules`, `.git`, `*.md` da ne nadu nepotrebnog smeća u build context-u.
- [ ] Verifikovati da svi image-i postoje **na javnom DockerHub-u** (ne samo lokalno).
- [ ] Kratak **arhitektura dijagram** u glavnom README-u (PlantUML ili ekscalibur — može i ručno crtano).

### Checkpoint 9 ✅
- [ ] Demo skripta odrađena bar 1x kompletno bez grešaka.
- [ ] Sve checkpoint stavke iz Faza 0–8 zelene.
- [ ] Tim ima podelu uloga za samu odbranu (ko priča o čemu).

---

## Bonus / opcioni zadaci (ako stignete)

| Bonus | Bodovi (procena) | Napomena |
|-------|------------------|----------|
| Web3 sign-in (SIWE) za ShopHub | +2–3 | Lepa stvar, traži vremena |
| Per-tenant Grafana pristup | +2–3 | Naprednije |
| GitOps preko ArgoCD u kube-state | +3 | Dobro za odbranu |
| MongoDB umesto/uz Redis | +1 | Pokazuje fleksibilnost |
| Kompletan test suite (≥80% coverage) | +2 | Profesionalan utisak |

---

## Decision log

> Ovde se beleže odluke koje pravimo u toku, posebno za stvari koje specifikacija ne pokriva eksplicitno. Format: datum, odluka, razlog.

| Datum | Odluka | Razlog |
|-------|--------|--------|
| 2026-05-25 | **MongoDB Community Operator umesto Spotahome Redis** za "light" DB opciju | Spotahome image `quay.io/spotahome/redis-operator:v1.3.0` vraća 404 (napušten projekat). REDB traži trial licencu. Spec sekcija 1.2 eksplicitno dozvoljava substituciju ("npr. MongoDB"). |
| 2026-05-25 | **Multi-group kubebuilder layout** (`kubebuilder edit --multigroup=true` pre `create api`) | Imamo 3 API grupe (`apps`, `notify`, `payments`) pod `shophub.local` domenom — default single-group layout ne dozvoljava. |
| 2026-05-25 | **WSL Ubuntu za operator dev**, Windows za k3d/frontend/Docker | kubebuilder + make najbolje rade na Linux-u. Docker Desktop ostaje na Windows-u zbog k3d integracije. Vidi memory `wsl-operator-dev-setup`. |
| 2026-05-26 | **MongoDB API workaround**: postaviti `mdb.ObjectMeta.OwnerReferences = [...]` umesto `controllerutil.SetControllerReference` | `mongodbv1.MongoDBCommunity` override-uje `GetOwnerReferences()` da vraća sintetičku self-ref, što razbija standardni controllerutil. Direct field access bypass-uje method override. Vidi memory `mongodb-operator-gotchas`. |
| 2026-05-26 | **Shop operator materijalizuje `mongodb-database` SA + Role + RoleBinding** u tenant namespace-u | MongoDB community operator helm chart kreira SA samo u svom install namespace-u. Cross-namespace MongoDBCommunity bi stagnirao sa "SA not found". Naš operator to rešava preko `ensureMongoDBRBAC` helpera. |
| 2026-05-26 | **MongoDB operator `watchNamespace="*"`** preko helm upgrade | Default je `watchNamespace=<install-ns>` što sprečava operator da vidi CR-ove iz tenant namespace-a. |
| 2026-05-26 | **ServiceMonitor auto-create iz Shop operator-a** (`ensureServiceMonitor`) | Po vežbama 5 + spec 4.1 — svaki Shop dobija automatski Prometheus scraping bez ručne konfiguracije. |
| 2026-05-26 | **`serviceMonitorSelectorNilUsesHelmValues=false`** za kube-prometheus-stack | Bez ovog Prometheus default selector traži label `release: kube-prometheus-stack` na SM-ovima. Naš operator ne stavlja taj label jer bi to vezalo operator za specifičan helm release. |
| 2026-05-26 | **shophub backend pokreće se na portu 9090 (lokalno)**, ne 8080 | k3d `cluster.yaml` mapira host:8080 → loadbalancer:80, blokira backend default port. |
| 2026-05-28 | **CI/CD prioritet eskaliran iz Faze 8 u "hitno"** | Asistent na poslednjim vežbama (5) eksplicitno tražio kompletan CI/CD što pre, ne na kraju projekta. Detaljan plan u `ci-cd-plan.md`. |

---

## Završna napomena

**Eliminacioni zahtev (5.3) je u Fazi 0 namerno.** Ako je 1 sedmica pre roka i nemate kompletnu strukturu `helm-charts` i `kube-state` repo-a sa tačnim folder-ima, **prekinuti druge zadatke i to popraviti** — projekat se inače ne pregleda i sve ostalo nije bitno.

Druga grupa stvari na koju paziti tokom celog projekta:
- **Idempotentnost reconciler-a** (operator eleganta sekcija 5).
- **Multi-stage + non-root + slim** za sve Dockerfile-ove (docker eleganta sekcija 1.1).
- **Liveness + readiness probe** za svaki Pod (k8s eleganta sekcija 2).
- **Conventional Commits** za sve commit-ove (zahtev 5.1).
- **PR review obavezan** — ne self-merge.

Srećno. 🎯
