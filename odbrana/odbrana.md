# ShopHub — Priprema za odbranu projekta

> Ovo je moj lični "cheat sheet" za odbranu. Pisano tako da za svaku temu znam: **šta sam uradio, kako, i ZAŠTO baš tako**. Cilj: da na svako asistentovo pitanje mogu da odgovorim mirno, sa razlogom, i da pokažem da razumem — ne da recitujem.
>
> Struktura: prvo veliki pregled arhitekture, pa tema po tema (Docker → Kubernetes → Operatori → Baze → Observability → Web3 → ShopHub → Helm → IaC/GitOps → CI/CD → Git). Na kraju: brza pitanja/odgovori, "teška" pitanja i demo skripta.
>
> **Legenda:** 🎯 = ključna poenta koju asistent voli da čuje · ⚠️ = zamka/gotcha · 💬 = kako to lepo izgovoriti.

---

## Sadržaj

1. [Pregled projekta i arhitektura](#1-pregled-projekta-i-arhitektura)
2. [Docker](#2-docker)
3. [Kubernetes — osnovni resursi](#3-kubernetes--osnovni-resursi)
4. [Kubernetes operatori — teorija](#4-kubernetes-operatori--teorija)
5. [Naši CRD-ovi: Shop, DiscordChannel, Wallet](#5-naši-crd-ovi-shop-discordchannel-wallet)
6. [Reconciler — kako radi naš Shop kontroler](#6-reconciler--kako-radi-naš-shop-kontroler)
7. [Ownership, Garbage Collection, Finalizeri](#7-ownership-garbage-collection-finalizeri)
8. [Watches, predikati, FieldIndexer](#8-watches-predikati-fieldindexer)
9. [Baze podataka — CNPG i MongoDB](#9-baze-podataka--cnpg-i-mongodb)
10. [Observability — metrike, logovi, trace-ovi, alarmi](#10-observability--metrike-logovi-trace-ovi-alarmi)
11. [Web3 plaćanje](#11-web3-plaćanje)
12. [ShopHub — auth, multi-tenancy, RBAC](#12-shophub--auth-multi-tenancy-rbac)
13. [Helm chart-ovi](#13-helm-chart-ovi)
14. [kube-state, IaC, GitOps](#14-kube-state-iac-gitops)
15. [CI/CD](#15-cicd)
16. [Git workflow](#16-git-workflow)
17. [Decision log — odluke i razlozi](#17-decision-log--odluke-i-razlozi)
18. [Brza pitanja / odgovori (rapid fire)](#18-brza-pitanja--odgovori-rapid-fire)
19. [Teška / zeznuta pitanja](#19-teška--zeznuta-pitanja)
20. [Demo skripta](#20-demo-skripta)

---

## 1. Pregled projekta i arhitektura

### 1.1 Šta je ShopHub (u jednoj rečenici)

**Platforma koja krajnjim korisnicima omogućava da dinamički kreiraju (deploy-uju) svoje online prodavnice u Kubernetes klasteru, sa plaćanjem u kriptovaluti, i punim observability stack-om.**

To su zapravo **dve aplikacije + jedan operator**:

- **ShopHub** — "control plane" web aplikacija. Korisnik se registruje, uloguje i preko panela kreira/menja/briše svoje prodavnice. ShopHub **ne deploy-uje ništa sam ručno** — on samo kreira `Shop` custom resurse u Kubernetes-u.
- **Shop** — sama prodavnica (storefront + admin). Vlasnik prodavnice dodaje artikle i gleda porudžbine; kupci pretražuju, dodaju u korpu i plaćaju USDT-om.
- **Shop operator** — Kubernetes operator koji "gleda" `Shop` CR-ove i za svaki od njih orkestrira sve što prodavnica treba: bazu, Deployment, Service, Ingress, monitoring, Discord kanal.

🎯 **Ključna rečenica za asistenta:** *"ShopHub je aplikacija koja govori Kubernetes API-ju šta želi (deklarativno, kroz `Shop` CR), a operator je taj koji to pretvara u stvarne resurse. Nikad ne pravim Deployment-e ručno iz aplikacije — sve ide kroz operator pattern."*

### 1.2 Pet repozitorijuma (zahtev 5.1 + eliminacioni 5.3)

Svaki mikroservis ima svoj repo:

| Repo | Šta je unutra | Jezik/tech |
|------|---------------|-----------|
| `shop-operator` | Kubebuilder operator + 3 CRD-a + reconcileri | Go |
| `shop` | Shop backend (Gin) + frontend (Vite/React) + Web3 | Go + TS |
| `shophub` | ShopHub backend (Gin + client-go) + frontend | Go + TS |
| `helm-charts` | Svi Helm chart-ovi koje sami pišemo (shop-operator, shophub, shop) | Helm/YAML |
| `kube-state` | Stanje klastera — koje komponente, koje verzije, koji override-i + ArgoCD | YAML |

⚠️ **Zahtev 5.3 je ELIMINACIONI** — ako `helm-charts` i `kube-state` nemaju tačnu strukturu foldera, projekat se **ne pregleda**. Zato smo to uradili prvog dana (Faza 0), ne na kraju.

### 1.3 Tok podataka kroz sistem (end-to-end)

```
Korisnik → ShopHub UI → ShopHub backend (client-go)
                              │  kreira Shop CR u tenant namespace-u
                              ▼
                     Kubernetes API server (etcd)
                              │  broadcast event
                              ▼
                     Shop operator (Reconcile petlja)
                              │  za svaki Shop pravi:
        ┌─────────────┬───────┼────────────┬──────────────┬───────────────┐
        ▼             ▼       ▼            ▼              ▼               ▼
   CNPG Cluster   Deployment Service    Ingress    ServiceMonitor  AlertmanagerConfig
   (ili Mongo)    (2/3 rep)             (storefront)  (Prometheus)   (→ Discord)
                              │
                              ▼
                Shop aplikacija radi na <shop>.localhost:8080
                              │
        kupac plaća USDT (MetaMask) → backend verifikuje na Sepolia → order confirmed
```
Digresija: CNPG je gotov Kubernetes operator za PostgreSQL bazu. Ista ideja kao tvoj Shop operator, samo za bazu: ima svoj CRD Cluster. Operator pravi cluster sa 1 instancom, onda 
CNPG kad se bootstrapuje pravi secret sa kredencijalima url host user passowrd dbname, i onda
operator uzme taj secret i ubaci ga u evn kod shap backenda kao DATABASE_URL

Digresija: Ingres je nesto kubernetesovo, neki njegov "resurs" koja izlaze aplikaciju spolja
preko HTTP-a sa lepim imenom, url-om. Znaci prvo imamo pod (aplikaciju) pa onda Service, to je 
pristup unutar klastera i load balancing, i onda na kraju imamo ingress koj predstavlja 
pristup spolja.
Service rešava komunikaciju unutar klastera. Ne možeš mu prići iz browsera. 
Ingress rešava pristup spolja: rutira HTTP saobraćaj po host-imenu ka odgovarajućem Service-u.
⚠️ Bitno (asistent voli): Ingress izlaže samo HTTP (80) i HTTPS (443). Za druge portove spolja koristiš Service tipa NodePort ili LoadBalancer.
💬 Jedna rečenica: "Ingress je ulazna tačka spolja — mapira host-ime na Service. Moj operator za svaku prodavnicu pravi Ingress, pa je svaka dostupna na svom URL-u tipa ime.localhost."

💬 Ako te pita "objasni mi ceo flow" — kreni od korisnika koji klikne "Create Shop", pa prati CR do operatora, pa nabroji šta operator napravi. To pokazuje da razumeš **deklarativni** model.

---

## 2. Docker

Materijal: `docker_elegancija.md`. Naši Dockerfile-ovi: `shop/Dockerfile`, `shop/backend/Dockerfile`, `shophub/Dockerfile`, `shop-operator/Dockerfile`.

### 2.1 Pitanje: "Objasni mi svoj Dockerfile."

Uzmem `shop/Dockerfile` (unified image) — to je najbogatiji primer:

```dockerfile
# 1) storefront build (Node) — kompajlira React u statički /dist
FROM node:20-alpine AS web
...
RUN npm run build

# 2) backend build (Go) — kompajlira binarni fajl
FROM golang:1.26-alpine AS builder
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags='-s -w' -o /out/shop ./cmd/shop

# 3) release — minimalna slika, samo binary + statika
FROM alpine:3.20 AS release
RUN apk add --no-cache ca-certificates && addgroup -S app && adduser -S app -G app
COPY --from=builder /out/shop /app/shop
COPY --from=web /web/dist /app/web
RUN chown -R root:root /app && chmod 0755 /app/shop
USER app
ENTRYPOINT ["/app/shop"]
```

Objašnjenje po tačkama (svaka je "best practice" iz materijala):

1. **Multi-stage build** — build alati (Go kompajler, Node, npm) ostaju u builder stage-ovima; finalna slika ima **samo izvršni binary + statiku**. 🎯 *Razlog: manja slika, manji attack surface. Niko unutar kontejnera ne može da izmeni source i rebuild-uje malicioznu verziju jer source-a nema.*
2. **Non-root korisnik** (`adduser -S app`, `USER app`) — aplikacija se **nikad ne pokreće kao root**. Owner fajlova je root (`chown root:root`), a app korisnik ima samo read+execute. ⚠️ *Ako neko provali u kontejner, nema write permisije.*
3. **Slim/alpine bazna slika** — manje zavisnosti = manje ranjivosti.
4. **`CGO_ENABLED=0`** — statički linkovan Go binary, radi u minimalnoj slici bez libc zavisnosti.
5. **`-ldflags='-s -w'`** — strip debug simbola → manji binary.
6. **Exec forma `ENTRYPOINT ["/app/shop"]`** (ne shell forma) — 🎯 *aplikacija je PID 1, pa joj SIGTERM stiže direktno → graceful shutdown radi. Shell forma bi je pokrenula kao podproces sh-a i signali ne bi stigli → docker/K8s bi je ubio KILL-om posle 10s.*
7. **`--mount=type=cache`** za `/go/pkg/mod` i npm cache — brži rebuild-ovi, cache se ne pakuje u sliku.
8. **hadolint** prolazi (imamo `# hadolint ignore=DL3018` komentar sa objašnjenjem gde svesno ne pinujemo verziju ca-certificates jer Alpine bumpne minor).

### 2.2 Kompajlirani vs interpretirani jezici

💬 *"Go je u prednosti jer se pokreće kao binary — finalna slika ne mora da sadrži source ni runtime. Interpretirani jezici (Node, Python) zahtevaju source u slici da bi radili, pa je multi-stage manje efikasan. Zato je backend u Go-u pravi kompajlirani binary, a frontend se build-uje u statičke fajlove koje Go server servira — i njemu ne treba Node u runtime-u."*

### 2.3 Moguća pitanja

- **"Zašto EXPOSE 8080 a ne 80?"** → Non-root korisnik ne sme da bind-uje portove < 1024. Zato aplikacija sluša 8080. (Isto važi za nginx u frontend-u ako ga koristimo odvojeno.)
- **"Šta radi `.dockerignore`?"** → Izbacuje `node_modules`, `.git`, `*.md` iz build konteksta → brži build, manje smeća, ne curi ništa nepotrebno.
- **"Kako se rukuje secret-ima u build-u?"** → `--mount=type=secret`, **nikad** preko `ARG` (vidljivo kroz `docker image inspect`) ni COPY-pa-obriši (skopeo čita stare layer-e). Mi u build-u ne trebamo secret-e — svi secret-i dolaze u runtime-u kroz Kubernetes Secret-e.
- **"Docker Compose / Swarm?"** → Znam teoriju (templating preko `docker compose config`, Swarm master/slave, Raft za izbor lidera, neparan broj master-a ≤7, Docker secrets/configs sa hash sufiksom za verzionisanje). Ali mi za deployment koristimo **Kubernetes**, ne Swarm — Compose koristimo eventualno za lokalni integ-test (mada mi koristimo Testcontainers).

---

## 3. Kubernetes — osnovni resursi

Materijal: `kubernetes_elegancija.md`.

### 3.1 Resurs vs objekat, scope

- **Resurs** = tip (Pod, Deployment, Service). **Objekat** = instanca (`deployment/demo-shop`).
- **Namespace-scoped** (Pod, Deployment, Secret, ConfigMap, Ingress) vs **cluster-wide** (Namespace, CRD, ClusterRole, PV).
- 🎯 *Sve u Kubernetes-u je resurs. CRD mi dodaje nove tipove resursa (`Shop`, `Wallet`, `DiscordChannel`) koji se ponašaju kao ugrađeni.*

### 3.2 Ključni resursi koje koristimo i zašto

| Resurs | Gde ga koristimo | Zašto |
|--------|------------------|-------|
| **Deployment** | Operator pravi Deployment za svaki Shop | Više replika, self-healing (padne pod → napravi nov), rolling update |
| **Service** (ClusterIP) | Operator pravi Service po Shop-u | Stabilan DNS + load balancing preko replika (IP poda nije statičan) |
| **Ingress** | Operator izlaže storefront | Pristup spolja preko HTTP na `<shop>.localhost:8080` |
| **Secret** | DB kredencijali, JWT, Discord webhook, wallet ključ, admin lozinka | Sve osetljivo — nikad u ConfigMap |
| **ConfigMap** | Grafana dashboard JSON (label `grafana_dashboard=1`) | Nije tajna; sidecar ga auto-importuje |
| **Namespace** | Po tenant-u (`tenant-xxxx`) | Izolacija, lako brisanje, mapira na RBAC |
| **ServiceAccount + Role/ClusterRole** | Operator i ShopHub backend | RBAC least-privilege |

### 3.3 Liveness i readiness probe (asistent OBAVEZNO pita)

Naš Shop backend ima obe (`shop_controller.go` ih injektuje u Deployment):

```go
LivenessProbe:  GET /probe/liveness   // uvek vraća "ok" — jeftino
ReadinessProbe: GET /probe/readiness  // proverava s.Ping(ctx) — DB dostupnost
```

🎯 **Razlika (ovo se pita):**
- **Liveness** — "da li je app živ?" Ako padne → **kubelet restartuje kontejner**. Držimo je jeftinom i bezuslovnom da kubelet ne bi restartovao pod tokom kratkog DB blip-a.
- **Readiness** — "da li app može da prima saobraćaj?" Ako ne → **Service ga izbaci iz load balancing-a** (ali ga ne restartuje). Zato readiness proverava bazu: dok baza nije spremna, pod ne prima saobraćaj, ali se ne ubija.

💬 *"Liveness = restart, readiness = ukloni iz saobraćaja. To je razlog zašto liveness ne sme da zavisi od baze — inače bi mi privremeni DB problem izazvao restart loop cele aplikacije."*

### 3.4 Service komunikacija

- Isti namespace: `service-name:port`
- Drugi namespace: `service-name.namespace.svc.cluster.local:port` (FQDN)
- Primer kod nas: backend šalje trace-ove na `tempo.monitoring.svc.cluster.local:4318`.

### 3.5 Zašto Ingress a ne NodePort

Ingress izlaže samo HTTP/HTTPS (80/443) i daje named routing po host-u (`<shop>.localhost`). Za svaki Shop operator pravi zaseban Ingress sa `ingressClassName` (k3d ima Traefik). NodePort bi otvorio proizvoljan port i ne bi dao lepe URL-ove.

---

## 4. Kubernetes operatori — teorija

Materijal: `kubernetes_operatori_elegancija.md`, `vezbe-5-operatori-helm-discord.md`. Ovo je **najveći deo odbrane** — asistent najviše ovde kopa.

### 4.1 Šta je operator (definicija napamet)

🎯 **"Operator = Custom Resource (CR) + Custom Controller. To je Pod/Deployment koji sluša izmene određenih objekata i procesira ih dovodeći ih u željeno stanje. Sastoji se od skupa kontrolera (control loops)."**

Način proširenja Kubernetes-a: **CRD + operator** (drugi način je Aggregation Layer, njega ne koristimo).

### 4.2 Arhitektura kontrolera — dijagram "koji moramo znati u sred noći"

```
kubectl → API server → (webhooks) → etcd
              │ (API server je JEDINI koji piše u etcd)
              │ broadcast
              ▼
          Operator
          ┌──────────────────────────────┐
          │ Informer:                    │
          │  Reflector → Cache → Workqueue│
          └──────────────┬───────────────┘
                         ▼
                   Reconciler loop
```

Komponente (moram da ih objasnim svojim rečima):

- **API server** — jedini koji upisuje u etcd. Ni kubelet, ni scheduler, ni controller manager ne diraju etcd direktno.
- **Reflector** — posmatra (`Watch`) izmene na API server-u i trpa objekte u **Cache** (u memoriji procesa).
- **Cache** — čuva **kompletne objekte** (Metadata + Spec + Status). Zato `r.Get()` čita iz keša, ne udara API server.
- **Workqueue** — drži samo metapodatke (`namespace/name`) i radi **deduplikaciju**: ako 5 izmena udari na isti CR, reconciler se okine samo jednom za poslednje stanje.
- **Reconciler** — control loop koji implementira našu logiku.

⚠️ **Zašto je predikat na Watches bitan:** bez njega reflector trpa **SVE** resurse tog tipa u cache → memory leak kako klaster raste. (Ovo je omiljena zamka.)

### 4.3 Idempotentnost i level-based dizajn (KLJUČNO)

🎯 **"Reconciler je stateless i mora biti idempotentan — ista operacija primenjena više puta daje isti rezultat. Ne pamti prethodno stanje; sve što zna izvlači iz Spec-a i Status-a objekta."**

🎯 **"Level-based, ne edge-based: sistem radi na osnovu razlike trenutnog vs željenog stanja, bez obzira koliko je međupromena propušteno. Ako se objekat 5 puta promenio dok se petlja izvršavala, interesuje me samo poslednja izmena."**

💬 To je razlog zašto koristim `CreateOrUpdate` pattern (idempotentno kreiraj-ili-ažuriraj) i zašto se ne oslanjam na "event" nego na "trenutno stanje sveta".

### 4.4 Zlatna pravila reconciler-a (moram da nabrojim)

1. **Jedna izmena objekta po iteraciji.** API server prati `resourceVersion`; slanje stare verzije → `409 Conflict`.
2. **Posle svakog `Update` proveri `!apierrors.IsConflict(err)`** — konflikt je benigno, ignoriši i petlja će se ponovo okinuti.
3. **No-op update ne pokreće broadcast** — koristi to da izbegneš beskonačnu petlju.
4. **Reconciler NE menja Spec** — samo Status (i izuzetno Scale/Finalizer). Spec je vlasništvo korisnika. Ako pokušaš PUT na Spec preko `/status` subresursa → 403.
5. **Ne čekaj resurse u petlji** (`wait_until_running` ❌). Petlja ima timeout. Umesto toga: završi iteraciju i vrati `RequeueAfter`.
6. **Normalno je da se petlja okida više puta** dok se ne dostigne željeno stanje.

### 4.5 Povratne vrednosti Reconcile

| Return | Značenje |
|--------|----------|
| `ctrl.Result{}, nil` | Uspeh — stani dok se objekat ne promeni |
| `ctrl.Result{}, err` | Greška — **exponential backoff** requeue |
| `ctrl.Result{RequeueAfter: t}, nil` | Polling — vrati u queue posle vremena t |
| `client.IgnoreNotFound(err)` | Objekat obrisan → nema šta da se radi |

### 4.6 Kategorizacija grešaka + Reason taksonomija

- **Oporavljive** (409 Conflict) → ignoriši, pusti petlju.
- **Neoporavljive** (loš config) → `Degraded=True` sa `Reason`.

`Reason` taksonomija (konvencija):
- `Stalled` — greška u **konfiguraciji korisnika**, mora da menja Spec ("promeni YAML").
- `Failed` — greška u **infrastrukturi klastera**, operator je sve odradio kako treba ("SRE, imaš problem").
- `Init`/`Creating`/`Scaling` — normalan napredak (`Progressing=True`).

Kod nas u `shop_controller.go` koristimo `DatabaseFailed`, `DatabaseProvisioning`, `Deploying`, `Ready`.

### 4.7 Leader election i /metrics (iz kutije od Kubebuilder-a)

🎯 *"Kubebuilder mi je već dao production-ready main.go: leader election (ako operator ima više replika, samo jedna radi Reconcile u datom trenutku — ostale čekaju; sprečava race i 409), Prometheus /metrics endpoint sa `controller_runtime_reconcile_total` itd., i health probe /healthz /readyz. Ja pišem samo Reconcile logiku."*

---

## 5. Naši CRD-ovi: Shop, DiscordChannel, Wallet

### 5.1 Multi-group layout

Imamo 3 API grupe pod domenom `shophub.local`:
- `apps.shophub.local` → **Shop**
- `notify.shophub.local` → **DiscordChannel**
- `payments.shophub.local` → **Wallet**

⚠️ *Multi-group nije default u Kubebuilder-u — moralo je `kubebuilder edit --multigroup=true` PRE `create api`. Razlog: čistije razdvajanje odgovornosti (deployment / notifikacije / plaćanje) nego sve u jednoj grupi.*

### 5.2 Shop CRD — dizajn (`shop_types.go`)

```go
type ShopSpec struct {
    Title         string          // obavezno → vrednost (ne pokazivač)
    Availability  Availability    // enum standard/high, default=standard
    Replicas      *int32          // OPCIONO → pokazivač; override za scale/HPA
    Database      DatabaseKind    // enum postgres/mongodb, default=postgres
    WalletAddress string          // obavezno
    DiscordWebhookSecretRef *corev1.SecretReference // opciono → pokazivač
    Image         *string         // opciono → pokazivač (CI/CD bump)
}
```

🎯 **Pravilo o pokazivačima (asistent pita):** *"Obavezni atributi su vrednosti, opcioni su pokazivači. Razlog: ako je opcioni string prazan (''), ne znam da li je nedodeljen ili namerno prazan — pokazivač eliminiše nejasnoću i sprečava probleme kod Server Side Apply."*

**Status** — sve `+optional` (jer je pri kreiranju prazan, operator ga popunjava):
```go
type ShopStatus struct {
    Conditions     []metav1.Condition  // +listType=map +listMapKey=type
    URL            string              // ingress hostname
    DatabaseSecret string
    ReadyReplicas  int32
    Selector       string              // za /scale subresource
}
```

**Markup-i na Shop struct-u:**
- `+kubebuilder:subresource:status` — reconciler menja samo Status.
- `+kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas,selectorpath=.status.selector` — omogućava `kubectl scale shop demo --replicas=4` i HPA.
- `+kubebuilder:printcolumn:...` — `kubectl get shops` prikazuje TITLE/DB/AVAILABILITY/READY/URL/AGE.
- `+kubebuilder:resource:shortName=sh` — `kubectl get sh`.

⚠️ **Zamka koju je asistent postavljao:** *availability je string (standard/high), a scale subresurs mapira NUMERIČKI put.* Rešenje: dodao sam eksplicitan `Replicas *int32` u Spec i mapirao scale na `.spec.replicas`. Kad je nil → availability odlučuje (2 ili 3); kad je set (kubectl scale/HPA) → pobeđuje. To je `replicasFor()` funkcija.

### 5.3 DiscordChannel CRD

Spec: `GuildID`, `Name`, `BotTokenRef` (Secret sa bot token-om). Status: `ChannelID`, `WebhookSecretName`.
- 🎯 **OBAVEZAN finalizer** — upravlja **eksternim resursom** (Discord kanal van K8s-a). Bez njega bi brisanje CR-a ostavilo siroče kanal na serveru.

### 5.4 Wallet CRD

Spec: `Network` (enum: ethereum/solana/sepolia/...), `OwnerAddress *string` (opciono). Status: `Address`, `PrivateKeySecretRef`.
- Dve grane: (1) korisnik dao adresu → samo je zapiši u Status; (2) nije dao → generiši secp256k1 keypair (`crypto.GenerateKey()` iz go-ethereum), sačuvaj private key u Secret, adresu u Status.
- ⚠️ **Zašto Wallet NEMA finalizer** (a DiscordChannel ima): *on-chain adresa se ne može "obrisati" (transakcije ostaju zauvek). Jedino osetljivo lokalno stanje je private key Secret, koji je `OwnerReference`-ovan na Wallet → Kubernetes GC ga sam obriše. Nema eksternog resursa za cleanup → nema potrebe za finalizer-om.*

💬 Ovo poređenje (DiscordChannel ima finalizer, Wallet nema) je odličan pokazatelj da razumem **kada** treba finalizer, a ne da ga slepo dodajem svuda.

---

## 6. Reconciler — kako radi naš Shop kontroler

Fajl: `shop-operator/internal/controller/apps/shop_controller.go`. Ovo je srce projekta.

### 6.1 Redosled koraka u Reconcile (ume da traži da ispričam)

```
Reconcile(req):
  1. Get(Shop)  → ako NotFound: IgnoreNotFound (obrisan je)
  2. Ako DeletionTimestamp != 0 → reconcileDelete (Grafana cleanup + skini finalizer)
  3. AddFinalizer (za Grafana tenant dashboard cleanup)
  4. ensureDatabase → vrati ime connection Secret-a ("" ako još nije spreman)
        - ako fail → Degraded=True, Reason=DatabaseFailed, RequeueAfter 30s
        - ako "" → Progressing=True, Reason=DatabaseProvisioning, RequeueAfter 10s
  5. ensurePasswordSecret (<shop>-admin) — admin lozinka
  6. ensureDeployment (2/3 replike, env, probe)
  7. ensureService (ClusterIP, label app=<shop>)
  8. ensureIngress (<shop>.localhost)
  9. ensureServiceMonitor (Prometheus) — best effort
  10. ensureDashboard (Grafana ConfigMap + per-tenant org) — best effort
  11. ensureAlertmanagerConfig (→ Discord) — best effort
  12. updateStatusFromDeployment (Conditions: Available/Progressing/Degraded)
```

### 6.2 Zašto "ensure..." funkcije i CreateOrUpdate

Sve su **idempotentne**: `controllerutil.CreateOrUpdate(ctx, client, obj, mutateFn)` — ako objekat ne postoji, kreira ga; ako postoji, ažurira mutate funkcijom. 🎯 *Petlja sme da se okine 100 puta, rezultat je isti — to je idempotentnost u praksi.*

### 6.3 Conditions (D1) — kako ih postavljam

`meta.SetStatusCondition` sam koristi jer **bumpuje `LastTransitionTime` samo na pravi prelaz** (ne na svaki reconcile). Postavljam trojku:
- Ready: `Available=True, Progressing=False, Degraded=False`
- Deploying: `Available=False (x/y replicas), Progressing=True`
- DatabaseFailed: `Degraded=True, Available=False`

⚠️ *Available NIJE isključiv sa ostalima; Degraded i Progressing JESU međusobno isključivi.*

### 6.4 Conflict handling — elegantno

Na vrhu `Reconcile` imam wrapper:
```go
func (r *ShopReconciler) Reconcile(...) {
    res, err := r.reconcile(...)
    if apierrors.IsConflict(err) {
        return ctrl.Result{Requeue: true}, nil  // stale resourceVersion → re-read
    }
    return res, err
}
```
💬 *"Konflikt se dešava kad paralelni writer (K8s deployment controller piše .status, CNPG ažurira Cluster) izmeni objekat između mog Get i Update. To je benigno — samo ponovo pročitam i pokušam."*

### 6.5 Mapiranje availability → replike

```go
func replicasFor(shop) int32 {
    if shop.Spec.Replicas != nil { return *shop.Spec.Replicas } // scale/HPA override
    if shop.Spec.Availability == High { return 3 }
    return 2 // standard
}
```
Tačno po spec-u: standard=2, high=3.

---

## 7. Ownership, Garbage Collection, Finalizeri

### 7.1 OwnerReference i GC

Za svaki child (Deployment, Service, CNPG Cluster, Secret, ConfigMap...) pozivam `controllerutil.SetControllerReference(shop, child, scheme)`. To postavlja `metadata.ownerReferences` sa `controller: true`.

🎯 **Efekat:** *kad obrišem Shop CR, Kubernetes Garbage Collector automatski obriše sve child-ove. Ne moram ručno da brišem Deployment/Service/bazu — to je "cascade delete" preko ownership-a.*

Tri vrednosti koje kontrolišu GC: `apiVersion+kind+name+uid` (ko je owner), `controller: true` (primarni owner — samo jedan), `blockOwnerDeletion: true` (owner ne može da se obriše dok child postoji).

⚠️ **BlockOwnerDeletion deadlock** (asistent voli): *jedan objekat sme imati više owner-a, ali samo jedan `controller:true`. Ako više objekata pokazuje na isti child sa `blockOwnerDeletion:true`, može doći do deadlock-a — niko ne može da se obriše. Preporuka: block samo u 1-na-1 vezama; za 1-na-više koristi labele i selektore.*

### 7.2 Finalizeri — kada i zašto

🎯 **Pravilo: "Uvek kad kontroler upravlja resursom VAN Kubernetes-a."** GC briše samo K8s objekte; eksterni resursi (Discord kanal, cloud resursi, spoljne baze) ostaju siročad bez finalizer-a.

**Finalizer pattern (kod nas u DiscordChannel controlleru):**
1. Pri kreiranju → dodaj string u `metadata.finalizers`.
2. Pri brisanju → Kubernetes stavlja objekat u `Terminating` i **NE briše** dok finalizer postoji.
3. Reconciler vidi `DeletionTimestamp != nil` → uradi cleanup (obriši Discord kanal preko API-ja) → skini finalizer.
4. Kubernetes završi brisanje.

⚠️ **Zeznuti edge case koji sam pokrio:** ako je bot-token Secret već obrisan kad se briše DiscordChannel, ne mogu da pozovem Discord API. Da ne bih zauvek blokirao brisanje CR-a, u tom slučaju **skinem finalizer i pustim brisanje** (bolje siroče kanal nego zaglavljen CR). To je u `Reconcile` na vrhu deletion grane.

⚠️ **Drugi edge case:** persist-ujem `ChannelID` u Status **pre** kreiranja webhook-a. Da webhook fail-uje i requeue se desi bez sačuvanog ChannelID-a, retry bi napravio **duplikat kanal** na guild-u.

### 7.3 Dva finalizera u projektu

- **DiscordChannel** — `discordchannel.notify.shophub.local/finalizer` → briše Discord kanal.
- **Shop** — `apps.shophub.local/grafana-tenant-dashboard` → briše per-tenant Grafana dashboard (koji je pushovan preko HTTP-a, nije K8s objekat, pa ga GC ne dohvata).

💬 To je opet dokaz da razumem finalizer: Shop-ov finalizer NIJE za Deployment/Service (njih GC pokrije), nego **samo** za onaj deo koji je van Kubernetes-a (Grafana HTTP API).

---

## 8. Watches, predikati, FieldIndexer

`SetupWithManager` u `shop_controller.go`.

### 8.1 Owns vs Watches

```go
ctrl.NewControllerManagedBy(mgr).
    For(&Shop{}).
    Owns(&Deployment{}).         // okida petlju za child-ove Shop-a
    Owns(&Service{}).
    Owns(&cnpgv1.Cluster{}).
    Owns(&MongoDBCommunity{}).
    Watches(&Secret{}, ..., predikat).  // za resurse koje NE posedujemo
```

- **`Owns()`** — za objekte koje operator poseduje (ima OwnerReference). Petlja se okida samo za NAŠE child-ove.
- **`Watches()`** — za objekte koje operator NE poseduje ali zavisi od njih. Kod nas: **connection Secret** (`<shop>-app`) koji generiše CNPG/MongoDB operator, i **webhook Secret** koji generiše DiscordChannel controller.

### 8.2 Tri parametra Watches + predikat (asistent pita)

- **Object** — tip (`&corev1.Secret{}`).
- **Predicate** — filter NA ULAZU (pre keširanja). Isključuje događaje koji nas ne zanimaju.
- **EventHandler** — mapping NA IZLAZU: mapira 1 Secret na `[]reconcile.Request` (jedan Secret može da pokrene više Shop-ova).

Naši predikati:
```go
// 1) connection secret: samo Secret-i sa sufiksom "-app"
builder.WithPredicates(predicate.NewPredicateFuncs(isConnectionSecret))
// 2) webhook secret: samo Secret-i sa sufiksom "-webhook"
builder.WithPredicates(predicate.NewPredicateFuncs(hasWebhookSecretSuffix))
```

🎯 **Zašto predikat (KLJUČNO):** *"Bez predikata, EventHandler bi se okinuo na SVAKU izmenu SVAKOG Secret-a u klasteru → CPU i memorija operatora eksplodiraju. Predikat filtrira odmah na ulazu."*

### 8.3 FieldIndexer (D4) — O(1) pretraga

Za pitanje "koji Shop-ovi referenciraju ovaj webhook Secret?" bez indeksa bih morao da listam SVE Shop-ove i iteriram (O(N)). Sa indeksom je O(1):

```go
mgr.GetFieldIndexer().IndexField(ctx, &Shop{}, ".spec.discordWebhookSecretRef.name",
    func(obj) []string { return []string{shop.Spec.DiscordWebhookSecretRef.Name} })
// pa u handleru:
r.List(ctx, &shops, client.MatchingFields{discordWebhookRefField: secret.Name})
```

💬 *"Asistent je rekao da ovo nije obavezno za mali broj Shop-ova, ali sam ga uradio jer je preporuka i pokazuje da razumem zašto — bez indeksa je linearna pretraga kroz sve objekte."*

### 8.4 Zašto reagujem samo na neke event-tipove

Za CNPG connection secret bitni su nam `Create`/`Update` (kad operator objavi/rotira credential). `Delete` ne handlujemo posebno — u dokumentaciji piše "ne dirajte ovaj Secret ručno" (adekvatno rešenje ne postoji, aplikacija prosto neće raditi bez baze).

---

## 9. Baze podataka — CNPG i MongoDB

### 9.1 Filozofija: koristi gotove operatore

🎯 **"Za standardne komponente (baza, message broker) koristim gotove operatore — ne pišem svoj minimalni Postgres kontroler. Moja implementacija ne bi bila industrijska. Moj operator se OSLANJA na CNPG i MongoDB operator; NE pravi StatefulSet-ove direktno."**

Spec traži: `standard` = PostgreSQL (CNPG), `light` = Redis (REDB). Mi za `light` koristimo **MongoDB** umesto Redis-a.

⚠️ **Zašto MongoDB umesto Redis (decision, asistent može pitati):**
- Spotahome Redis operator je **napušten** (image `quay.io/spotahome/redis-operator:v1.3.0` vraća 404).
- REDB (Redis Enterprise) traži **trial licencu** i komplikovanu instalaciju.
- Spec sekcija 1.2 **eksplicitno dozvoljava** substituciju: *"Umesto Redis baze može se koristiti neka druga baza (npr. MongoDB) ali je obavezno koristiti operator te baze."*
- Koristimo **MongoDB Community Operator**. ✅ Zadovoljen uslov: koristi se operator te baze.

### 9.2 CNPG (PostgreSQL) put

`ensurePostgresDatabase`: pravim `postgresql.cnpg.io/v1.Cluster` sa 1 instancom, 1Gi storage, i **bootstrap `initdb`** koji kreira šemu (items, orders tabele).

🎯 **`postInitApplicationSQL` + `OWNER TO` (asistent voli):**
```go
`CREATE TABLE IF NOT EXISTS items (...)`,
`ALTER TABLE items OWNER TO "` + owner + `"`,
```
⚠️ *`postInitApplicationSQL` izvršava ROOT (postgres superuser). Ako ne postavim eksplicitno `OWNER TO <appuser>`, aplikacija koja se loguje kao app-user dobija "permission denied" na svojim tabelama.* Dvostruki navodnici jer Shop imena mogu imati crticu (nevalidan unquoted identifier u Postgres-u).

CNPG posle bootstrap-a pravi Secret `<shop>-app` sa `uri`, `host`, `port`, `user`, `password`, `dbname`. Operator čeka da se taj Secret pojavi, pa ga `envFrom`/secretRef-uje u Deployment kao `DATABASE_URL`.

### 9.3 MongoDB put — i njegove gotcha zamke

`ensureMongoDBDatabase`: pravim `MongoDBCommunity` CR sa 1 članom, SCRAM auth, generisanom lozinkom (random hex u Secret-u).

⚠️ **Tri MongoDB gotcha-e koje sam morao da rešim (odlična priča za odbranu — pokazuje da nije copy-paste):**

1. **OwnerReference override:** `MongoDBCommunity.GetOwnerReferences()` je override-ovan da vraća sintetičku self-referencu, što razbija `controllerutil.SetControllerReference`. Rešenje: ručno gradim `OwnerReference` i dodeljujem direktno preko embedded `ObjectMeta` polja (`mdb.OwnerReferences = ...`) — promocija polja zaobilazi method override.

2. **`mongodb-database` ServiceAccount:** MongoDB operator za svaki spawn-ovan Pod traži SA `mongodb-database`, ali njegov Helm chart pravi taj SA **samo u svom install namespace-u**. Cross-namespace MongoDBCommunity bi stao sa "serviceaccount mongodb-database not found". Rešenje: moj operator **materijalizuje SA + Role + RoleBinding u svakom tenant namespace-u** (`ensureMongoDBRBAC`), owned od Shop-a.

3. **`watchNamespace="*"`:** default MongoDB operator gleda samo svoj namespace. Preko helm override-a sam podesio `watchNamespace="*"` da vidi CR-ove u tenant namespace-ima.

💬 Da bih Deployment env učinio uniformnim, **pin-ujem connection Secret na `<shop>-app`** i za Mongo (isto ime kao CNPG). Razlika je samo u ključu: CNPG `uri`, Mongo `connectionString.standard`. Funkcija `dbEnvFromSecret` bira ključ po tipu baze.

### 9.4 Iterativni helm deploy — zašto ne sve odjednom

⚠️ *"Helm nije za logiku stanja, samo za templating. Ako sve `enabled:true` odjednom, aplikacija pokuša da se digne pre nego što Secret postoji → race condition, pod-ovi crashuju, a helm to ne vidi kao grešku. Pravi pristup: prvo baza (`postgres.enabled`), sačekaj Secret, pa app."* Kod nas operator prirodno rešava ovo: dok nema `<shop>-app` Secret-a, vraća `""` i requeue-uje — Deployment se ne pravi dok baza nije spremna.

---

## 10. Observability — metrike, logovi, trace-ovi, alarmi

Spec 4.1. Ovo je bogata tema. Tri stuba: **metrics + logs + traces**, plus **alarmi → Discord**.

### 10.1 Stack (sve iz kube-state)

| Komponenta | Uloga |
|-----------|-------|
| kube-prometheus-stack | Prometheus (metrike) + Alertmanager + Grafana + node-exporter + kube-state-metrics + prometheus-operator CRD-ovi (ServiceMonitor, PrometheusRule) |
| Loki + Promtail | Agregacija logova (spec 4.1 "logging") |
| Tempo | Distributed tracing (spec 4.1 "tracing"); backend gura span-ove preko OTLP/HTTP |

### 10.2 Metrike — instrumentacija backend-a

Backend (`observability/metrics.go`) izlaže Prometheus counter-e po spec-u 4.1:
- Ukupan broj HTTP zahteva (24h), 2xx/3xx uspešni, 4xx/5xx neuspešni
- 404 sa endpoint-ovima
- Ukupan protok saobraćaja (GB)
- Jedinstveni posetioci (IP+timestamp+browser → Loki distinct query)
- CPU/RAM/FS/network dolaze besplatno preko node-exporter i cAdvisor

⚠️ **Bug koji sam popravio:** Prometheus counter je pucao na negativan `Writer.Size()` → panic → 500 umesto 404. Dodao guard.

### 10.3 ServiceMonitor — operator pravi automatski

`ensureServiceMonitor` pravi `ServiceMonitor` po Shop-u sa label-om `release: kube-prometheus-stack`.

🎯 **Zašto taj label:** *"Default Prometheus iz kube-prometheus-stack-a selektuje samo ServiceMonitor-e sa label-om `release: kube-prometheus-stack`. Bez njega se metrike ne bi skupljale."*

⚠️ **Zašto Service mora imati `app` label** (ne samo Spec.Selector): *ServiceMonitor selektor matchuje **Service labele**, ne Pod selektor. Zato Service sam nosi `app: <shop>`.* (Ovo je bio jedan od 4 bug-a otkrivenih na demo-u.)

### 10.4 Grafana dashboard po Shop-u (spec 4.1: "svaka aplikacija svoj dashboard")

`ensureDashboard`: embed-ujem `dashboard.json` template, zamenim `$shop` placeholder imenom Shop-a, i napravim **ConfigMap sa label-om `grafana_dashboard=1`**. Grafana sidecar iz stack-a auto-importuje takve ConfigMap-e cluster-wide.

💬 *"Umesto jednog deljenog dashboard-a sa dropdown-om, svaki Shop dobija svoj dashboard objekat sa svojim `uid`-om — tačno kako spec traži."*

**Opciono (spec 4.1) — per-tenant pristup:** operator dodatno gura isti dashboard u **per-tenant Grafana org** preko Grafana HTTP API-ja (`grafana_client.go`). ShopHub pri registraciji pravi Grafana org + Viewer login za korisnika, tako da svaki korisnik vidi **samo svoje** dashboard-e. Maintaineri u default org-u vide sve.

🎯 *"OSS Grafana basic role je org-wide, pa folder-permisije ne izoluju dovoljno — zato koristim Grafana Organizacije (jedna po tenant-u) za pravu izolaciju."*

### 10.5 Alarmi + Alertmanager → Discord (D10)

- **PrometheusRule** — pravila za okidanje (npr. `ShopHighErrorRate`: rate 5xx > prag; cluster alarmi: CPU/pod restart loop). U helm-charts operator chart-u.
- **AlertmanagerConfig** — operator pravi po Shop-u (`ensureAlertmanagerConfig`), rutira alarme tog Shop-a na njegov Discord kanal.

🎯 **Detalji koji impresioniraju:**
- Koristim `DiscordConfig` sa `apiURL` kao **secret-ref** (webhook URL nije u renderovanom config-u/git-u). ⚠️ *Probao sam `webhook_url_file` — operator ga odbija; secret-ref je ispravan način.*
- **OnNamespace matcher strategija** — Alertmanager auto-scope-uje rute na namespace tog Shop-a, tako da **samo alarmi tog Shop-a idu na njegov webhook** (tenant izolacija).
- Dokazano end-to-end: kontinuirani 404 loop → `ShopHighErrorRate` firing → poruka u Discord kanalu.

### 10.6 Logging (Loki) i Tracing (Tempo)

- **Loki + Promtail** skupljaju logove svih Pod-ova; vidljivi u Grafana Explore. Jedinstveni posetioci se računaju Loki distinct query-jem.
- **Tempo**: backend je instrumentiran OpenTelemetry-jem (`otelgin` middleware), operator injektuje `OTEL_EXPORTER_OTLP_ENDPOINT` (Tempo) i `OTEL_SERVICE_NAME=<shop>` da su trace-ovi grupisani po tenant-u. Span-ovi dokazani u Grafani.

### 10.7 Pristup Grafani

Spec: samo maintaineri, ne ShopHub korisnici. Grafana admin lozinka živi **samo u klasteru** (Secret `grafana-admin` u 3 namespace-a — Secret-i se ne referenciraju cross-namespace, pa isti u monitoring/shop-operator-system/shophub), **nikad u git-u**.

---

## 11. Web3 plaćanje

Spec 2.4. Stack: **Sepolia testnet + self-deployed USDT ERC-20 + MetaMask + ethers.js (frontend) + go-ethereum (backend)**.

### 11.1 Konstante (memory: d12_web3_constants)

- Mreža: **Sepolia** testnet (Ethereum).
- Token: **self-deployed TestUSDT** ERC-20 na `0x74b0ef...7ff6f`, 6 decimala, open mint faucet (deploy-ovan preko Remix-a).
- ⚠️ *Zašto self-deployed: zvanična Sepolia USDT adresa je nepouzdana/varira; svoj ERC-20 sa open mint-om mi daje deterministički faucet za demo.*

### 11.2 Flow plaćanja (reserve → pay → attach → verify)

1. Kupac napuni korpu, klikne "Buy with USDT".
2. Frontend čita `walletAddress` Shop-a (`/api/shop-info`).
3. Backend kreira order kao **`pending`** i **rezerviše stock odmah** (da dva kupca ne kupe poslednji komad).
4. Frontend traži MetaMask da pošalje USDT `transfer` na tu adresu → dobije `txHash`.
5. Frontend attach-uje tx hash: `POST /api/orders/:id/tx`.
6. Backend **sweep loop** (`verifier.go`, `time.Ticker(15s)`) proverava pending order-e:
   - Preko go-ethereum `eth_getTransactionReceipt` nađe tx, parsira logove, verifikuje `Transfer` event ka pravoj adresi za pravu sumu.
   - `confirmed` → potvrdi order; `failed` → vrati stock.
7. Frontend polluje `/api/orders/:id` dok ne vidi `confirmed`.

### 11.3 Elegantni detalji (impresioniranje)

🎯 **Sweep loop, ne per-request goroutine:** *"Verifikacija je sweep petlja koja preživljava restart Pod-a — pending order-i se re-proveravaju svaki put kad je proces gore. Per-request goroutine bi se izgubila na restartu."*

🎯 **Idempotentnost po replikama:** više replika Shop-a → sweep je idempotentan (baza je izvor istine, `ConfirmOrder`/`FailOrder` su atomične).

🎯 **Zaštita od replay-a tx hash-a (`RequiredForTx`):** *"Jedan cart checkout legitimno pravi više order-a koji dele isti tx (kupac plati ceo total jednim transfer-om). Zato transfer mora da pokrije SUMU svih ne-failed order-a sa tim tx-om. Posledica: replay-ovan tx hash ne može da plati više nego original — dodatni order gurne sumu preko on-chain iznosa i verifikacija fail-uje."*

**TTL na pending:** order bez tx hash-a se expire-uje posle 30 min i vraća rezervisani stock (kupac koji je zatvorio MetaMask ne drži stock zauvek).

### 11.4 Wallet CRD veza

Operator injektuje `WALLET_ADDRESS` u backend (adresa gde padaju uplate). ShopHub ima "Generate wallet" flow → pravi `Wallet` CR → operator generiše keypair → vrati adresu.

---

## 12. ShopHub — auth, multi-tenancy, RBAC

Fajlovi: `shophub/backend/internal/auth/auth.go`, `httpapi/handlers.go`, `httpapi/crds.go`.

### 12.1 Auth (D13)

- Register/Login: email + password, **bcrypt** hash, **JWT** (HS256, TTL 24h).
- JWT nosi `ns` claim (tenant namespace) — middleware ga stavlja u gin context; svaki shop handler scope-uje operacije na taj namespace bez dodatnog DB lookup-a.
- Users tabela u Postgres-u (`EnsureUsersSchema` na startup-u).

🎯 **Middleware proverava signing method:** *odbijam token ako alg nije HMAC — sprečava `alg:none` i RS/HS confusion napade.*

### 12.2 Multi-tenancy preko namespace-a

🎯 **"Svaki ShopHub korisnik dobija SVOJ namespace (`tenant-<random>`). Svi njegovi Shop CR-ovi žive tu."** Razlozi:
- Izolacija (ResourceQuota po namespace-u).
- Lako brisanje (drop namespace = drop sve).
- Prirodno mapira na RBAC.

Register pravi namespace (`ensureNamespace`, idempotentno). Login re-asertuje namespace → account se **self-heal**-uje ako je kreiranje palo pri registraciji.

### 12.3 ShopHub kao Kubernetes klijent

ShopHub backend koristi `controller-runtime/client` da pravi/menja/briše **Shop CR-ove** (i Wallet, DiscordChannel). NE pravi Deployment-e — to je operatorov posao.

⚠️ *ShopHub kreira Shop CR sa binding validacijom (`oneof=standard high`, `oneof=postgres mongodb`) na API nivou pre nego što uopšte udari K8s. Ako CRD validacija fail-uje (`IsInvalid`) → 400; ako već postoji → 409.*

### 12.4 RBAC za ShopHub backend

ServiceAccount + **ClusterRole** (jer radi cross-namespace):
```yaml
- namespaces: get, list, create, delete
- apps.shophub.local/shops: get, list, watch, create, update, patch, delete
```
ClusterRoleBinding na SA. 🎯 *Least privilege: ShopHub sme da pravi namespace-e i Shop CR-ove, ali NE i da direktno pravi Deployment-e/Pod-ove.*

### 12.5 Admin lozinka prodavnice

Operator generiše `<shop>-admin` Secret (random lozinka). ShopHub čita taj Secret (`GetShopAdminCredentials`) da vlasniku prikaže login za admin panel njegove prodavnice. Backend Shop-a čuva `ADMIN_PASSWORD` env iz istog Secret-a i njime gejtuje admin endpoint-e (item writes, order listing — spec 2.2).

---

## 13. Helm chart-ovi

Repo `helm-charts`. Tri chart-a: `shop-operator`, `shophub`, `shop`.

### 13.1 shop-operator chart (zahtev 3.2)

- `crds/` folder sa 3 CRD YAML-a. 🎯 *"Helm 3 automatski instalira sve iz `crds/` PRE template-ova i NIKAD ih ne dira na upgrade. Zato CRD-ovi idu tu, ne u templates/."*
- `templates/`: Deployment operatora, ServiceAccount, ClusterRole + Binding, Service (za /metrics), PrometheusRule (alarmi), grafana-secret.
- CRD-ovi se sync-uju iz `shop-operator/config/crd/bases/*.yaml` (output `make manifests`).

### 13.2 shophub chart (zahtev 3.3)

🎯 **Zahtev 3.3: "ShopHub Helm Chart koji koristi CRD-ove iz Shop operatora i prometheus-stack".**
- **Dependency na kube-prometheus-stack** (`Chart.yaml` dependencies, verzija 85.3.3, vendored kao `.tgz` u `charts/`).
- Templates: Deployment, Service, Ingress, RBAC (ClusterRole cross-namespace), ServiceMonitor, CNPG `db.yaml` (users baza), JWT secret (lookup-preserve da se ne regeneriše na upgrade), grafana-secret.

### 13.3 shop chart (zahtev iz strukture 5.3)

⚠️ *Operator pravi Shop resurse direktno — pa čemu ovaj chart? Tanak chart koji renderuje **Shop CR** iz values-a (+ opcioni Discord secret). Služi kao "fallback" za ručnu instalaciju/debug bez ShopHub UI-ja i zadovoljava strukturu iz spec-a 5.3.*

### 13.4 Publish kao OCI

`helm-publish.yml` na push u main (paths `charts/**`): `helm package` + `helm push oci://docker.io/urospetraskovic`. Verzija = iz `Chart.yaml` (ne git tag).

⚠️ **Naming zamka (memory: cicd_publishing_scheme):** operator **image** je preimenovan u `shop-operator-controller` da se ne sudari sa **chart-om** `shop-operator` na istom OCI registry-ju (isti tag npr. 0.1.0 bi kolidirao).

### 13.5 Helm lint

CI `helm-lint.yml` radi `helm lint charts/*` + `helm template` (sanity render) na svaki PR.

---

## 14. kube-state, IaC, GitOps

Repo `kube-state`. **Ovo je deo eliminacionog zahteva 5.3.**

### 14.1 Struktura

```
kube-state/clusters/local/
├── cluster.yaml              # k3d: 1 server + 2 agenta, portovi 8080→80, 8443→443
├── cnpg/ helm.yaml + values.yaml
├── mongodb-operator/
├── kube-prometheus-stack/
├── loki/
├── tempo/
├── shop-operator/            # OCI chart iz helm-charts
└── shophub/
```

Svaka komponenta = `helm.yaml` (koji chart + verzija + namespace) + `values.yaml` (override).

🎯 **"kube-state je single source of truth ZA STANJE KLASTERA: šta je instalirano i sa kojim override-ima. Bilo ko ko pročita ovaj repo može da digne ceo klaster od nule."** Verzije su pinovane (npr. cnpg 0.28.2, kube-prometheus-stack 85.3.3, shop-operator 0.1.7, shophub 0.2.1).

### 14.2 GitOps sa ArgoCD (opcioni bonus spec 5.3)

`kube-state/argocd/` — **app-of-apps** pattern: jedan `kubectl apply` na `root.yaml` predaje ceo klaster ArgoCD-u, koji instalira svaku komponentu (iz istih pinovanih verzija i values fajlova preko multi-source Application-a) i **self-heal**-uje drift.

💬 *"Manuelni helm install je najjednostavniji za demo, ali ArgoCD app-of-apps daje continuously-reconciled setup — to je opcioni GitOps deo zahteva 5.3."*

### 14.3 Razlika helm-charts vs kube-state

- `helm-charts` = **šta pišem ja** (definicije chart-ova).
- `kube-state` = **šta je deploy-ovano i sa kojim vrednostima** (stanje konkretnog klastera, uključujući i upstream chart-ove kao CNPG/Prometheus koje ne pišem).

---

## 15. CI/CD

Zahtev 5.2. Materijal: `ci-cd-plan.md`.

### 15.1 CI (na svaki PR)

Po repo-u (`shop`, `shophub`, `shop-operator`):
- **commit-lint** — Conventional Commits.
- **test.yml** — `go build` + `go vet` + `go test -race` + coverage.
- **lint.yml** — golangci-lint.
- **docker-build.yml** — build image (bez push) + **hadolint** Dockerfile.
- **e2e** (samo shop-operator) — kind cluster + integracioni testovi.
- `helm-charts`: **helm-lint.yml**.

### 15.2 CD (na merge u main / tag)

- **docker-publish.yml** — build + push na DockerHub. SemVer: iz git tag-a (`v0.1.0`→`0.1.0`+`latest`), inače dev verzija `0.0.0-YYYYMMDD-<sha7>`+`main`.
- **helm-publish.yml** — package + push chart-ova na OCI.

### 15.3 Testovi (D7) — Testcontainers

🎯 *"Integracioni testovi koriste **Testcontainers-go** za pravu bazu (ne mock): shop backend handleri protiv pravog Postgres-a, ShopHub auth protiv Postgres-a + fake kube klijenta, operator reconcile sa fake client-om (`WithStatusSubresource`). Svi prolaze u CI."*

⚠️ **Zašto svaki operator test u svoj namespace:** *envtest ne podržava brisanje namespace-a, pa umesto BeforeEach/AfterEach koji briše, pravim nov namespace sa random sufiksom po testu.*

### 15.4 Branch protection (zahtev 5.1)

- Require PR + 1 approval (ne self-merge).
- **Require status checks** — ⚠️ *required check imena moraju da odgovaraju **job-name**-u (ne workflow-name-u) i da postoje U TOM repo-u.* Po repo-u različiti set (memory: branch_protection_required_checks). Nikad ne zahtevati paths-filtered/push-only workflow.
- **Require linear history**.

### 15.5 Secrets

`DOCKERHUB_USERNAME` (=`urospetraskovic`) + `DOCKERHUB_TOKEN` (Read/Write/Delete) po repo-u. ⚠️ *Token nikad u repo fajlovima.* (Memory: DockerHub acct `urospetraskovic` ≠ GH org `shophub-devoops`.)

### 15.6 make deploy vs make run (asistent naglasio!)

🎯 **"Na odbrani mora `make deploy` / helm install da radi end-to-end, ne samo `make run`."**
- `make run` — operator lokalno kao Go proces, koristi moj kubeconfig (admin prava) → za razvoj.
- `make deploy` / helm install — operator kao **Pod u klasteru**, koristi ServiceAccount + RBAC → produkciona simulacija. Ako RBAC markup-i fale → 403.

---

## 16. Git workflow

### 16.1 Trunk Based Development

- Radimo iz `main` (trunk), kratkoživeće feature grane (1-3 dana).
- Bez `develop` grane — TBD preporučuje samo trunk + release grane po potrebi.

### 16.2 Conventional Commits

`<type>(<scope>): opis` — `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`. Primeri:
```
feat(shop-operator): add Wallet CRD spec validation
fix(shop): correct cart total calculation for USDT
ci(shophub): add integration test stage
```
`commitlint` workflow validira automatski.

🎯 **Veza sa SemVer:** `feat:` → minor, `fix:` → patch, `feat!:`/`BREAKING CHANGE:` → major.

---

## 17. Decision log — odluke i razlozi

Ovo su odluke koje spec ne pokriva eksplicitno, spreman sam da ih odbranim:

| Odluka | Razlog |
|--------|--------|
| **MongoDB umesto Redis** za `light` DB | Spotahome mrtav (404), REDB traži licencu; spec dozvoljava substituciju uz operator |
| **Multi-group kubebuilder** | 3 grupe (apps/notify/payments) — čistije razdvajanje |
| **WSL za operator dev, Windows za k3d/Docker/frontend** | kubebuilder+make najbolje na Linux-u; Docker Desktop+k3d na Windows-u |
| **MongoDB OwnerRef preko direktnog field access-a** | `GetOwnerReferences()` override razbija controllerutil |
| **Operator materijalizuje mongodb-database SA po tenant ns** | MongoDB chart pravi SA samo u svom ns; cross-ns bi stao |
| **ServiceMonitor auto-create iz operatora** | Svaki Shop dobija scraping bez ručne konfiguracije |
| **connection Secret pin na `<shop>-app` i za Mongo** | Uniforman `envFrom` bez obzira na tip baze |
| **image `shop-operator-controller` ≠ chart `shop-operator`** | Da image i chart ne kolidiraju na OCI |
| **Grafana Organizacije za per-tenant izolaciju** | OSS basic role je org-wide; folder-perms nedovoljne |
| **AlertmanagerConfig apiURL secret-ref (ne webhook_url_file)** | Operator odbija webhook_url_file |
| **Self-deployed USDT ERC-20 na Sepolia** | Zvanična adresa nepouzdana; open-mint faucet za demo |

---

## 18. Brza pitanja / odgovori (rapid fire)

**P: Zašto operator, a ne da ShopHub sam pravi Deployment-e?**
O: Deklarativni model + reconciliation. Operator garantuje željeno stanje kontinuirano (self-healing), radi GC preko ownership-a, i odvaja "šta hoću" (CR) od "kako se pravi" (kontroler). ShopHub ostaje tanak.

**P: Šta se desi ako obrišem Shop CR?**
O: GC obriše sve owned child-ove (Deployment, Service, CNPG Cluster/Mongo, Secret-i, ConfigMap, ServiceMonitor, AlertmanagerConfig). Shop finalizer pre toga obriše per-tenant Grafana dashboard (HTTP, van K8s-a).

**P: Šta ako operator padne usred reconcile-a?**
O: Ništa strašno — reconciler je idempotentan i level-based. Kad se digne, pročita trenutno stanje CR-a i child-ova i nastavi. Ako ima više replika, leader election garantuje da samo jedna radi.

**P: Kako operator zna kad je baza spremna?**
O: Watch-uje `<shop>-app` Secret (predikat na sufiks `-app`). CNPG/Mongo ga objavi kad baza bootstrap-uje → petlja se odmah okine (ne čeka requeue). Dok ga nema, vraća "" + requeue 10s.

**P: Zašto 409 Conflict i kako ga rešavaš?**
O: Stale `resourceVersion` — neko izmenio objekat između mog Get i Update. Benigno; ignorišem `IsConflict` i re-reconcile-ujem sa svežim read-om.

**P: Razlika ClusterRole vs Role?**
O: Role je namespace-scoped, ClusterRole cluster-wide. ShopHub ima ClusterRole jer pravi namespace-e i Shop CR-ove cross-namespace. Operator ima ClusterRole jer gleda CR-ove u svim tenant namespace-ima.

**P: Kako radiš observability za jedinstvene posetioce?**
O: Kompozitni ključ IP+timestamp+browser; Loki distinct query (ili in-memory hash sa TTL 24h).

**P: Šta je ServiceMonitor?**
O: prometheus-operator CRD koji Prometheus-u kaže koji Service da scrape-uje. Operator ga pravi po Shop-u; selektuje se labelom `release: kube-prometheus-stack`.

**P: Kako se plaćanje verifikuje bez poverenja frontend-u?**
O: Backend nezavisno proverava tx na Sepolia preko go-ethereum: `Transfer` event ka pravoj adresi, iznos ≥ suma order-a. Frontend samo attach-uje tx hash; sve odluke su on-chain verifikovane.

**P: Zašto testnet a ne mainnet?**
O: Spec dozvoljava testnet; nema realnih para; Sepolia + MetaMask = najmanje rizika, faucet za ETH (gas) i moj USDT.

**P: Kako se skalira Shop?**
O: `availability` (standard=2/high=3) daje default; `kubectl scale shop demo --replicas=4` ili HPA preko scale subresursa override-uje (`spec.replicas`).

**P: Gde su CRD YAML-ovi u Helm chart-u i zašto?**
O: `crds/` folder — Helm 3 ih instalira pre template-ova i ne dira na upgrade.

---

## 19. Teška / zeznuta pitanja

**P: "Šta ako dva korisnika naprave prodavnicu istog imena?"**
O: Shop CR-ovi žive u **različitim tenant namespace-ima**, pa se imena ne sudaraju (namespace-scoped uniqueness). Unutar istog korisnika, duplikat imena → 409 Conflict iz K8s-a, ShopHub vraća 409.

**P: "Reconciler ti radi 10 stvari — nije li to previše za jednu iteraciju? Nisi li rekao 'jedna izmena po iteraciji'?"**
O: Pravilo je "jedna izmena **istog objekta** po iteraciji" (zbog resourceVersion konflikta). Ja menjam **različite** objekte (Deployment, Service, Ingress...) — svaki je zaseban resurs sa svojim resourceVersion-om. Status Shop-a pišem **jednom na kraju**. Ako Status update fail-uje sa konfliktom, benigno je i re-reconcile. Idempotentnost znači da čak i ako stanem na pola, sledeći prolaz dovrši.

**P: "Zašto MongoDB, spec kaže Redis?"**
O: (Vidi 9.1) Spotahome mrtav, REDB licenca; spec 1.2 eksplicitno: "može druga baza (npr. MongoDB) uz operator te baze". Koristim MongoDB Community Operator → uslov ispunjen.

**P: "Kako sprečavaš da korisnik A vidi dashboard/shop korisnika B?"**
O: Tri sloja: (1) K8s namespace izolacija — JWT nosi ns, middleware scope-uje sve operacije; RBAC. (2) Grafana per-tenant organizacije. (3) Alertmanager OnNamespace scope za alarme.

**P: "Šta ako Discord API padne pri brisanju kanala?"**
O: Finalizer requeue-uje (30s) i pokušava ponovo. Edge case: ako je bot-token Secret već obrisan, ne mogu da zovem API — tada skinem finalizer i pustim brisanje (radije siroče kanal nego zaglavljen CR).

**P: "Helm je fail-ovao ali pod-ovi crashuju — zašto?"**
O: Helm samo apply-uje manifeste, ne prati runtime stanje. Race: app pre baze. Rešenje: iterativan deploy (baza pa app), a kod nas operator prirodno gate-uje Deployment dok `<shop>-app` Secret ne postoji.

**P: "Zašto ne praviš StatefulSet za bazu sam?"**
O: Anti-pattern. Baza je standardna komponenta — koristim industrijski operator (CNPG/Mongo). Moj operator orkestrira preko njihovih CRD-ova, ne reimplementira replikaciju/backup/failover.

**P: "Čemu FieldIndexer ako imaš malo Shop-ova?"**
O: Priznajem da za mali skup nije nužan (i asistent je to rekao). Uradio sam ga jer je preporuka i O(1) umesto O(N) — pokazuje da razumem cenu linearne pretrage na Secret event-ima.

**P: "Kako bi ovo skalirao na 1000 prodavnica?"**
O: Operator je već level-based i idempotentan. Uska grla: broj Watch event-ova (rešava predikat), API rate limit (`r.Get` iz keša, ne APIReader), i resource kvote po namespace-u. Leader election + horizontalno skaliranje operatora. Baze su per-tenant pa se prirodno particionišu.

**P: "Šta je razlika između `Owns` i `Watches` u tvom kodu?"**
O: `Owns` za objekte koje posedujem (OwnerReference) — Deployment, Service, CNPG Cluster, Mongo. `Watches` za tuđe objekte od kojih zavisim — connection Secret (CNPG/Mongo pravi) i webhook Secret (DiscordChannel controller pravi), oba sa predikatom.

---

## 20. Demo skripta

Ovako vodim demo (10-15 min), redosled koji sve pokriva:

1. **Pokaži strukturu** `helm-charts` i `kube-state` (eliminacioni 5.3!). Naglasi crds/ folder, pinovane verzije.
2. **Digni klaster od nule:** `k3d cluster create --config clusters/local/cluster.yaml`.
3. **Instaliraj komponente:** cnpg, mongodb-operator, kube-prometheus-stack, loki, tempo, shop-operator, shophub (ili ArgoCD app-of-apps). Pokaži grafana-admin Secret trik (samo u klasteru).
4. **ShopHub UI:** registracija → pokaži da se `tenant-xxxx` namespace kreira (`kubectl get ns`).
5. **Kreiraj prodavnicu** iz UI (title, availability=high, database=postgres, wallet, opciono Discord). Pokaži `kubectl get shops -A` i `kubectl describe shop` → **Status Conditions** (Available/Progressing/Degraded), URL, ReadyReplicas.
6. **Pokaži šta je operator napravio:** `kubectl get all,cluster,servicemonitor -n tenant-xxxx` → Deployment (3 replike jer high), Service, Ingress, CNPG Cluster, ServiceMonitor.
7. **Otvori storefront** `http://<shop>.localhost:8080` → admin login (lozinka iz ShopHub-a) → dodaj artikle.
8. **Kupac flow** (drugi browser): korpa → "Buy with USDT" → MetaMask potpiše na Sepolia → order `pending` → sweep verifikuje → `confirmed`, stock -1.
9. **Grafana:** login kao maintainer → per-Shop dashboard (HTTP total/2xx/4xx/404/GB, CPU/RAM/FS/net, latency) + cluster dashboard. Pokaži per-tenant org izolaciju.
10. **Alarm demo:** kontinuirani 404 loop → `ShopHighErrorRate` firing → **poruka u Discord kanalu**.
11. **Loki/Tempo:** Explore → logovi Shop-a, trace span-ovi.
12. **Brisanje:** obriši shop iz UI → `kubectl get all -n tenant-xxxx` → **GC obrisao sve** (Deployment, Service, CNPG Cluster nestaju). Discord kanal obrisan finalizer-om.
13. **CI/CD:** pokaži zeleni pipeline na PR-u, image na DockerHub-u, chart na OCI, branch protection blokira merge bez approve+checks.

⚠️ **Uradi demo bar 1x kompletno pre odbrane** — da se ne desi iznenađenje uživo.

---

## Završna napomena — kako se držati

- Kad ne znam nešto, **ne izmišljam** — kažem "to nisam implementirao, ali evo kako bih pristupio..." i objasnim princip. Asistent ceni razumevanje procesa više od činjenice.
- Uvek povezujem **odluku sa razlogom** (zašto MongoDB, zašto finalizer ovde a ne tamo, zašto predikat).
- Ključne fraze koje pokazuju zrelost: *idempotentnost, level-based, deklarativno željeno stanje, ownership/GC, least-privilege RBAC, iterativan deploy, single source of truth.*
- Eliminacioni zahtev (5.3, helm-charts + kube-state struktura) — **prvo to pokaži**, to je ulaznica.

Srećno na odbrani. 🎯
