# ShopHub — priprema za odbranu projekta

> Moje beleške za odbranu. Cilj: da na svako pitanje mogu da odgovorim u dve ravni —
> prvo **šta** smo uradili, pa odmah **zašto baš tako** (koja je alternativa bila i zašto je odbačena).
> Asistent najviše ceni kad vidi da odluka nije copy-paste sa tutorijala nego svesna odluka.
>
> Format: pitanja i odgovori po oblastima, plus "💡 dodatni poeni" — stvari koje mogu da ubacim
> ako hoću da pokažem dublje razumevanje, i "⚠️ zamka" — mesta gde pitanje zvuči bezazleno a lako se pogreši.

---

## Sadržaj

1. [Elevator pitch — kako otvaram priču](#1-elevator-pitch)
2. [Arhitektura i tok podataka](#2-arhitektura-i-tok-podataka)
3. [Mapiranje specifikacije na implementaciju](#3-mapiranje-specifikacije-na-implementaciju)
4. [Kubernetes osnove — pitanja](#4-kubernetes-osnove)
5. [Shop operator, CRD-ovi i reconciler — najveća oblast](#5-shop-operator-crd-ovi-i-reconciler)
6. [Baze: CNPG i MongoDB operator](#6-baze-cnpg-i-mongodb-operator)
7. [Shop aplikacija (backend + storefront)](#7-shop-aplikacija)
8. [Web3 plaćanje](#8-web3-placanje)
9. [ShopHub platforma (auth, multi-tenancy, client-go)](#9-shophub-platforma)
10. [Observability: metrike, logovi, trace-ovi, alarmi](#10-observability)
11. [Docker i slike kontejnera](#11-docker)
12. [Helm chart-ovi i IaC (eliminacioni zahtev)](#12-helm-i-iac)
13. [GitOps i ArgoCD](#13-gitops-i-argocd)
14. [CI/CD i Git workflow](#14-cicd-i-git)
15. [Testiranje](#15-testiranje)
16. [Zeznuta pitanja i edge case-ovi](#16-zeznuta-pitanja)
17. [Demo scenario — šta pokazujem uživo](#17-demo-scenario)
18. [Cheat sheet — brojevi, imena, verzije](#18-cheat-sheet)

---

## 1. Elevator pitch

Ako me pitaju "opiši projekat u minut", ovo je verzija koju izgovaram:

> ShopHub je multi-tenant platforma na kojoj korisnik kroz web UI kreira sopstvene online prodavnice,
> a svaka prodavnica se **dinamički deploy-uje u Kubernetes klaster**. Srce sistema je **Shop operator**
> koji sam napisao u Go-u sa Kubebuilder-om: on definiše tri CRD-a — `Shop`, `DiscordChannel` i `Wallet` —
> i za svaki `Shop` CR orkestrira kompletan stack: bazu (PostgreSQL preko CNPG operatora ili MongoDB preko
> MongoDB Community operatora), Deployment sa 2 ili 3 replike u zavisnosti od availability-ja, Service,
> Ingress, ServiceMonitor za Prometheus, Grafana dashboard i Alertmanager rutiranje alarma na Discord kanal
> te prodavnice. Kupci u prodavnici plaćaju kriptovalutom — ERC-20 USDT tokenom na Sepolia testnetu preko
> MetaMask-a, a backend verifikuje transakciju direktno na blockchain-u. Ceo sistem je pokriven
> observability stack-om (Prometheus + Grafana + Loki + Tempo), CI/CD pipeline-ovima na svih pet
> repozitorijuma, Helm chart-ovi se publish-uju kao OCI artefakti na DockerHub, a stanje klastera se
> deklarativno vodi u kube-state repozitorijumu kroz ArgoCD app-of-apps pattern.

Ključne stvari koje želim da "prodam" u prvih pet minuta, jer su iznad minimuma specifikacije:

- **ArgoCD GitOps** (spec kaže opciono → bonus bodovi)
- **Per-tenant Grafana organizacije** (spec 4.1 opciono → bonus bodovi)
- **Pravi finalizeri** za eksterne resurse (Discord kanal, Grafana dashboard)
- **Scale subresource** na Shop CRD-u (`kubectl scale shop` radi, HPA-ready)
- **Idempotentna verifikacija plaćanja** koja preživljava restart poda i radi sa više replika
- **Testcontainers integracioni testovi** u CI-ju

---

## 2. Arhitektura i tok podataka

### 2.1 Pet repozitorijuma (zahtev 5.1: svaki mikroservis svoj repo)

| Repo | Šta je unutra | Jezik/alat |
|---|---|---|
| `shop-operator` | Kubebuilder operator, 3 CRD-a + 3 kontrolera | Go, controller-runtime |
| `shop` | Shop aplikacija: Go/Gin backend + React/Vite storefront (jedan image) | Go + TypeScript |
| `shophub` | ShopHub platforma: Go/Gin backend + React/Vite frontend (jedan image) | Go + TypeScript |
| `helm-charts` | Naša tri chart-a: `shop-operator`, `shophub`, `shop` | Helm |
| `kube-state` | Deklarativno stanje klastera: k3d config, helm.yaml + values.yaml po komponenti, ArgoCD aplikacije | YAML |

### 2.2 Tok od registracije do žive prodavnice

1. Korisnik se **registruje** na ShopHub (email + lozinka → bcrypt hash u Postgres `users` tabeli).
2. ShopHub backend mu kreira **sopstveni namespace** `tenant-<random-hex>` i (opciono) Grafana org.
3. Backend vraća **JWT koji u claims nosi taj namespace** — svaki naredni API poziv je automatski
   scope-ovan na korisnikov namespace, bez dodatnog upita u bazu.
4. Korisnik u dashboard-u popuni formu (naziv, availability, baza, wallet adresa) → ShopHub backend
   preko controller-runtime client-a **kreira `Shop` CR** u korisnikovom namespace-u.
5. **Shop operator** uhvati event i reconcile petlja radi redom: baza (CNPG `Cluster` ili
   `MongoDBCommunity` CR) → čeka connection Secret → admin lozinka Secret → Deployment (2/3 replike)
   → Service → Ingress (`<ime>.localhost:8080`) → ServiceMonitor → Grafana dashboard ConfigMap →
   AlertmanagerConfig. Status se vodi kroz Conditions (`Available`/`Progressing`/`Degraded`).
6. Korisnik klikne na URL prodavnice → otvara se storefront. Admin prodavnice se uloguje lozinkom
   koju je operator generisao (Secret `<shop>-admin`), doda artikle.
7. Kupac ubaci u korpu → backend **rezerviše stock i kreira pending order** → MetaMask šalje USDT
   `transfer` na wallet adresu prodavnice → frontend zakači `txHash` na order → **sweep petlja** u
   backendu svakih 15s proverava receipt na Sepoliji → `confirmed` → order potvrđen.
8. Metrike sa `/metrics` skuplja Prometheus (preko ServiceMonitora koje operator pravi), logove Loki,
   trace-ove Tempo; alarmi (npr. visok error rate) idu kroz Alertmanager na Discord kanal prodavnice
   (koji je takođe napravio operator kroz `DiscordChannel` CRD).
9. Brisanje prodavnice iz UI-ja → briše se Shop CR → **finalizer** prvo počisti Grafana dashboard u
   tenant org-u, a Kubernetes GC preko owner referenci briše sve child resurse; DiscordChannel
   finalizer obriše kanal na Discord serveru.

### 2.3 Dijagram (crtam na tabli ako treba)

```
                 ┌─────────────────────────────────────────────────────────┐
                 │                     k3d klaster                          │
  browser        │                                                          │
  ────────────►  │  Ingress (traefik, host :8080 → :80)                     │
                 │     │                                                    │
                 │     ├── shophub.localhost ──► ShopHub (Deployment)       │
                 │     │        │ JWT auth, users u CNPG Postgres           │
                 │     │        │ client-go: CRUD Shop/Wallet/DiscordChannel│
                 │     │        ▼                                           │
                 │     │   tenant-ab12cd (namespace po korisniku)           │
                 │     │      Shop CR ◄─────────────┐                       │
                 │     │        │                   │ reconcile             │
                 │     │        ▼                   │                       │
                 │     │   ┌────────────────  Shop operator  ────────────┐  │
                 │     │   │ CNPG Cluster / MongoDBCommunity             │  │
                 │     │   │ Deployment(2/3) + Service + Ingress         │  │
                 │     │   │ ServiceMonitor + Dashboard CM + AMConfig    │  │
                 │     │   └─────────────────────────────────────────────┘  │
                 │     └── mojshop.localhost ──► Shop storefront            │
                 │                                    │  USDT verify        │
                 │  monitoring ns: Prometheus/Grafana/│  (Sepolia RPC)      │
                 │  Alertmanager/Loki/Tempo           ▼                     │
                 └──────────────────────────────► Ethereum Sepolia testnet  │
                                                       ▲
                                          MetaMask (kupac) ── Discord (alarmi)
```

---

## 3. Mapiranje specifikacije na implementaciju

Ovo držim u glavi da mogu da odgovorim na "gde je u projektu ispunjen zahtev X":

| Spec | Zahtev | Kako je implementirano |
|---|---|---|
| 1.1 | Prijava/registracija | ShopHub JWT auth (bcrypt + HS256, 24h TTL); **opcioni Web3 wallet sign-in takođe postoji** (`auth/wallet.go`) |
| 1.2 | Upravljanje sajtovima | Dashboard CRUD nad Shop CR-ovima; naziv, availability standard/high, wallet adresa, baza standard/light |
| 1.2 napomena | Baze preko operatora | CNPG za Postgres; **MongoDB Community operator umesto Redis-a** (spec eksplicitno dozvoljava) |
| 2.1 | CRUD artikala | Shop backend `/api/items` (admin gate) |
| 2.2 | Pregled porudžbina | `/api/orders` GET — samo admin |
| 2.3 | Pretraga + korpa | Storefront (client-side korpa, pretraga po nazivu) |
| 2.4 | Kripto plaćanje | ERC-20 USDT na Sepoliji, MetaMask + ethers v6, backend verifikacija go-ethereum-om |
| 3.1 | Shop operator | 3 CRD-a: `Shop` (apps grupa), `DiscordChannel` (notify), `Wallet` (payments) |
| 3.2 | Operator Helm chart | `helm-charts/charts/shop-operator` (CRD-ovi u `crds/`, RBAC, Deployment) |
| 3.3 | ShopHub Helm chart | `helm-charts/charts/shophub` — koristi Shop CRD-ove, deklariše kube-prometheus-stack kao dependency |
| 4.1 | Observability | Prometheus + Grafana (per-Shop dashboard) + Loki (logovi) + Tempo (trace-ovi) + alarmi → Discord |
| 4.1 opciono | Korisnik vidi samo svoje dashboarde | **Per-tenant Grafana organizacije** (operator + ShopHub pišu u Grafana HTTP API) |
| 5.1 | Git konfiguracija | TBD, branch protection (1 approval, required checks, linear history), Conventional Commits + commitlint |
| 5.2 | CI pipeline | test/lint/docker-build na PR; docker-publish + helm-publish na main/tag; Testcontainers u testovima |
| 5.3 | **IaC (eliminacioni)** | `helm-charts` + `kube-state` tačno po strukturi iz spec-a; ArgoCD kao opcioni deo |

⚠️ **Zamka:** ako pitaju "zašto nemate Redis kad spec kaže Redis" — odgovor je spreman:
spec u napomeni kaže *"Umesto Redis baze može se koristiti neka druga baza (npr. MongoDB) ali je
obavezno koristiti operator te baze"*. Probali smo prvo Redis: Spotahome redis-operator je napušten
projekat (image `quay.io/spotahome/redis-operator:v1.3.0` vraća 404), a zvanični Redis Enterprise
(REDB) traži enterprise licencu i trial registraciju. MongoDB Community operator je živ, open-source
i uklapa se u "light" ulogu. Ovo je zabeleženo i u decision log-u projekta.

---

## 4. Kubernetes osnove

Asistent često krene od osnova da proveri da li razumem šta koristim, pa tek onda ide na operator.

**P: Šta je Pod i zašto skoro nikad ne pravite Pod direktno?**
O: Pod je najmanja deployable jedinica — grupa od jednog ili više kontejnera koji dele mrežu
(localhost) i storage. Direktan Pod nema samoizlečenje: ako padne, niko ga ne podiže ponovo. Zato
sve ide kroz Deployment koji preko ReplicaSet-a garantuje željeni broj replika. U projektu jedini
"sirovi" pod je debug (`kubectl run curl ...`).

**P: Razlika liveness i readiness probe? Kako ste ih postavili?**
O: Liveness odgovara na pitanje "da li je proces živ" — ako padne, kubelet **restartuje** kontejner.
Readiness odgovara "da li sme da prima saobraćaj" — ako padne, pod se **izbacuje iz Service
endpoint-a** ali se ne restartuje. Kod nas: liveness (`/probe/liveness`) je jeftin i bezuslovan —
uvek vraća 200 dok proces radi. Readiness (`/probe/readiness`) radi `Ping()` ka bazi — ako baza
kratko ne radi, pod prestane da prima saobraćaj ali se **ne restartuje bespotrebno**. To je svesna
odluka: da smo bazu vezali za liveness, kratkotrajan problem sa bazom bi izazvao restart lavinu
(pod restartuje → opet ne može do baze → CrashLoopBackOff), a restart aplikacije problem sa bazom
ionako ne rešava.

**P: Čemu služi Service? Zašto ne gađate pod po IP-ju?**
O: Pod IP nije stabilan — menja se na svaki restart/reschedule. Service daje stabilno DNS ime i
load balancing preko replika (koristi readiness da zna kome sme da šalje). Discovery unutar istog
namespace-a: `service:port`; iz drugog: `service.namespace.svc.cluster.local`. Naš operator za
svaki Shop pravi ClusterIP Service `<shop>` na portu 8080.

**P: Šta radi Ingress i koja je razlika u odnosu na Service?**
O: Service rešava komunikaciju **unutar** klastera; Ingress izlaže HTTP/HTTPS **van** klastera —
L7 ruter (kod nas traefik, dolazi uz k3s/k3d) koji na osnovu hostname-a i putanje rutira ka
Service-ima. Svaki Shop dobija svoj host `<ime>.localhost`; k3d mapira loadbalancer :80 na host
:8080, pa je klikabilan URL `http://<ime>.localhost:8080` — zato operator ima `INGRESS_URL_PORT`
env da u Status.URL doda `:8080` sufiks (sam Ingress Host nikad ne nosi port, to je samo za browser).

**P: ConfigMap vs Secret? Kada koje?**
O: Oba su key-value konfiguracija koju pod konzumira kao env, argumente ili mount-ovan fajl.
Secret za sve osetljivo (base64 + može da se enkriptuje at rest + RBAC se obično striktnije daje).
Kod nas ConfigMap nosi npr. Grafana dashboard JSON, a Secreti nose: DB kredencijale (pravi ih CNPG/
MongoDB operator), admin lozinku prodavnice, Discord bot token, Discord webhook URL, JWT signing
secret, Grafana admin lozinku, privatni ključ Wallet-a. ⚠️ Bitno pravilo koje znam: mount-ovan
ConfigMap se auto-ažurira u podu, ali **env varijable i subPath mount ne** — treba restart poda.

**P: Šta je namespace i kako ste ga iskoristili?**
O: Logička izolacija imena i RBAC/kvota granica. Mi ga koristimo kao **granicu tenanta**: svaki
ShopHub korisnik dobija svoj `tenant-<id>` namespace i svi njegovi Shop-ovi (i sve što operator za
njih napravi) žive tamo. Time dobijamo: izolaciju imena (dva korisnika mogu imati shop `moj-shop`),
prirodan RBAC scope, mogućnost ResourceQuota po tenantu, i "obriši namespace = obriši sve od
korisnika". Sistemske komponente imaju svoje namespace-e: `shophub`, `shop-operator-system`,
`monitoring`, `cnpg-system`, `mongodb-operator`, `argocd`.

**P: Objasnite RBAC.**
O: Role-Based Access Control: `ServiceAccount` (identitet poda) + `Role`/`ClusterRole` (skup
dozvola: apiGroups/resources/verbs) + `RoleBinding`/`ClusterRoleBinding` (vezivanje). Namespaced
vs cluster-wide: Role važi u jednom namespace-u, ClusterRole svuda. Kod nas postoje tri bitna RBAC
paketa: (1) operator — ClusterRole generisan iz kubebuilder RBAC markera (sme da dira shops,
deployments, services, secrets, ingresses, CNPG clusters, mongodbcommunity, servicemonitors,
alertmanagerconfigs, configmaps, SA/Role/RoleBinding...); (2) ShopHub backend — ClusterRole da
kreira namespace-e i upravlja Shop/Wallet/DiscordChannel CR-ovima u svim tenant namespace-ima;
(3) `mongodb-database` SA + Role + RoleBinding koje **naš operator materializuje u svakom tenant
namespace-u** jer ih MongoDB operatorov chart pravi samo u svom install namespace-u.

**P: Šta se desi kad uradite `kubectl apply`? Ko sve učestvuje?**
O: kubectl šalje REST poziv API serveru → API server je **jedini** koji piše u etcd → pre upisa se
pozivaju admission webhook-ovi i validacija (za CRD-ove: OpenAPI šema generisana iz naših Go tipova)
→ posle upisa API server broadcast-uje event svim zainteresovanim watcher-ima (naš operator, scheduler,
kube-controller-manager...). Scheduler bira nod za pod, kubelet na tom nodu pokreće kontejner.
Ovo je i uvod u priču o informer-u operatora (sekcija 5).

**P: requests vs limits?**
O: `requests` je garantovana rezervacija koju scheduler koristi pri odluci gde pod staje; `limits`
je plafon (CPU se throttle-uje, memorija → OOMKill). Definišu QoS klasu poda. Postavili smo ih za
operator (50m/64Mi — 500m/256Mi), Prometheus, Grafanu... jer su k3d nodovi docker kontejneri sa
ograničenom memorijom.

---

## 5. Shop operator, CRD-ovi i reconciler

Ovo je najveći i najverovatniji blok pitanja. Materijal sa vežbi je asistentov, pa pitanja
verovatno prate njegove naglaske: informer dijagram ("morate ga znati usred noći"), idempotentnost,
finalizeri, watches + predikati, subresursi.

### 5.1 Koncepti

**P: Šta je operator pattern?**
O: Kombinacija **Custom Resource-a** (deklarativni API — korisnik opiše *šta* želi) i **custom
kontrolera** (petlja koja dovodi stvarno stanje u željeno). Operator je u suštini Deployment koji
sluša izmene "svojih" objekata na API serveru i procesira ih. Time domensko znanje ("kako se
deploy-uje prodavnica sa bazom i monitoringom") pretvaramo u Kubernetes-native automatiku — umesto
runbook-a, kod.

**P: Šta je CRD, a šta CR?**
O: CRD (CustomResourceDefinition) je **definicija tipa** — registruje novi RESTful endpoint na API
serveru sa OpenAPI v3 šemom (kod nas generisana iz Go structova kubebuilder markerima preko
`make manifests`). CR je **konkretna instanca** tog tipa. Analogno: CRD = klasa, CR = objekat.
Naša tri CRD-a žive u tri API grupe pod domenom `shophub.local`: `apps.shophub.local/v1 Shop`,
`notify.shophub.local/v1 DiscordChannel`, `payments.shophub.local/v1 Wallet`.

**P: Zašto tri API grupe, a ne sve u jednoj?**
O: Semantička podela po domenu odgovornosti (aplikacije / notifikacije / plaćanja) — isto kao što
Kubernetes deli `apps/v1`, `networking.k8s.io/v1`... Praktična posledica: kubebuilder default
layout dozvoljava samo jednu grupu, pa je **pre** `create api` moralo `kubebuilder edit
--multigroup=true` (to sam naučio na teži način — druga `create api` komanda puca sa "multiple
groups are not allowed by default").

**P: Nacrtajte i objasnite arhitekturu kontrolera (informer dijagram).**
O: (crtam)

```
API server (jedini piše u etcd)
   │ watch (broadcast eventova)
   ▼
Reflector ── kešira objekte u memoriji procesa (Informer cache)
   │ Predicate (filter na ulazu) + EventHandler (mapiranje na zahteve)
   ▼
Work queue ── drži samo namespace/name, DEDUPLIKUJE ključeve
   │
   ▼
Reconciler petlja (naš kod)
```

Ključne posledice koje umem da izvedem iz dijagrama:
- `r.Get()` u reconciler-u čita **iz keša**, ne sa API servera → zato je moguć stale read i `409
  Conflict` na update (postoji `r.APIReader` za direktno čitanje, ali se ne zloupotrebljava zbog
  rate limita API servera).
- Work queue dedup po `namespace/name` → ako se CR promeni 5 puta dok petlja radi, reconciler se
  okine **jednom za poslednje stanje** → to je **level-based** dizajn (reagujemo na trenutno vs
  željeno stanje, ne na svaki pojedinačni event/edge).
- Bez predikata na `Watches()`, reflector kešira **sve** objekte tog tipa u klasteru → memorija i
  CPU rastu sa klasterom. Zato svaki naš Secret watch ima predikat.

**P: Šta znači da je reconciler idempotentan i zašto mora biti?**
O: Ista operacija primenjena više puta daje isti rezultat kao prva. Reconciler je **stateless** —
ne pamti prethodnu iteraciju, a petlja se legitimno okida više puta za isti objekat (svaki update
child resursa je opet trigger). Da nije idempotentan, svaki prolaz bi npr. pravio novi Deployment
ili duplirao Discord kanal, i sistem bi divergirao ili ušao u beskonačnu petlju. Postižemo ga
"ensure" pattern-om: svaka `ensureX` funkcija prvo pročita postojeće stanje (`CreateOrUpdate` /
`Get` pa `Create` samo ako je `NotFound`), i **Status-om** kao zapisom dokle se stiglo (npr.
DiscordChannel čuva `channelID` u Status pre nego što nastavi — retry onda ne pravi drugi kanal).

**P: Koje su moguće povratne vrednosti Reconcile i šta znače?**
O: `{}, nil` = uspeh, ne diraj me dok se objekat ne promeni; `{}, err` = greška → exponential
backoff requeue; `{RequeueAfter: t}, nil` = polling (npr. čekamo da CNPG objavi Secret — vraćamo
`RequeueAfter: 10s`); `{Requeue: true}` koristimo samo za benigni retry posle `409 Conflict`.
Nikad ne čekamo resurs u petlji (`wait until running` je antipattern — petlja ima timeout i blokira
worker); umesto toga završimo iteraciju i requeue-ujemo.

**P: Kako rešavate 409 Conflict?**
O: API server verzioniše objekte kroz `metadata.resourceVersion`; update sa starom verzijom vraća
409. To je kod nas **očekivana i benigna** situacija jer paralelno iste objekte piše više aktera
(deployment kontroler piše status Deploymenta, CNPG svoj Cluster...). Rešenje na dva nivoa:
u `setConditions` i status update-ovima konflikt gutamo (sledeći event ponovo okine petlju sa svežim
stanjem), a na vrhu `Reconcile` postoji wrapper: ako bilo šta ispod vrati `IsConflict`, vraćamo
`{Requeue: true}, nil` umesto greške — da se u logovima ne gomilaju lažne greške i da ne trošimo
exponential backoff na nešto što nije greška.

### 5.2 Shop CRD — dizajn

**P: Prošetajte kroz Shop Spec i objasnite odluke.**
O:
```go
type ShopSpec struct {
    Title         string          // obavezno → vrednost, ne pokazivač
    Availability  Availability    // enum standard|high, default standard
    Replicas      *int32          // OPCIONO → pokazivač; override za scale subresource
    Database      DatabaseKind    // enum postgres|mongodb, default postgres
    WalletAddress string          // obavezno
    DiscordWebhookSecretRef *corev1.SecretReference // opciono → pokazivač
    Image         *string         // opciono → pokazivač; CI može da pin-uje verziju
}
```
Pravilo sa vežbi koje smo dosledno primenili: **obavezni atributi su vrednosti, opcioni su
pokazivači**. Razlog: za opcioni `string` prazna vrednost `""` je dvosmislena — ne zna se da li je
korisnik namerno postavio prazno ili nije dirao polje; `nil` tu dvosmislenost eliminiše, što je
bitno za Server-Side Apply merge semantiku. Enumi su ograničeni `+kubebuilder:validation:Enum`
markerima pa API server odbije nevalidnu vrednost pre nego što do operatora uopšte dođe.

**P: A Status?**
O: `Conditions []metav1.Condition` (+listType=map po `type`), `URL`, `DatabaseSecret`,
`ReadyReplicas`, `Selector`. **Sve je `+optional`** — pri prvom apply-u Status je prazan i tek ga
kontroler popunjava; da nije optional, prvi apply bi pao na validaciji. Status je "observed state"
i piše ga isključivo operator kroz `/status` subresource (`r.Status().Update()`); Spec je
vlasništvo korisnika — da reconciler pokuša da menja Spec, to bi bio antipattern (i tehnički je
sprečeno jer status subresource update ignoriše Spec).

**P: Objasnite Conditions model.**
O: Tri standardna tipa: `Available` (spremno za korišćenje), `Progressing` (radi se ka željenom
stanju), `Degraded` (neoporavljiva greška). `Degraded` i `Progressing` su uzajamno isključivi;
`Available` nije — može biti `Available=True, Progressing=True` (radi ali se skalira). Koristimo
`meta.SetStatusCondition` koji `LastTransitionTime` menja **samo na stvarnu promenu stanja** — pa
istorija prelaza ostaje verna. Uz svaki condition ide `Reason` taksonomija: `Ready`, `Deploying`,
`DatabaseProvisioning`, `DatabaseFailed`. Poenta Reason-a: `kubectl describe shop` i alati poput
ArgoCD/Headlamp prikazuju razlog direktno — korisnik odmah vidi da li je problem njegov YAML ili
infrastruktura. Punimo i `ObservedGeneration` da se vidi na koju generaciju Spec-a se status odnosi.

**P: Kako radi `kubectl scale shop` kod vas? (scale subresource)**
O: Marker:
```go
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas,selectorpath=.status.selector
```
Tri putanje: specpath (gde piše željeni broj), statuspath (šta autoscaler čita kao trenutno),
selectorpath (label selector da HPA nađe podove — operator u status upisuje `app=<ime>`).
⚠️ Fora koju očekujem da pitaju: spec kaže da availability određuje replike, a scale subresource
mora da mapira **numeričko** polje — `availability` je string. Rešenje: opciono polje
`spec.replicas *int32` koje je **override** — kad je `nil`, važi availability mapiranje
(standard→2, high→3); kad ga `kubectl scale`/HPA postavi, on pobeđuje:
```go
func replicasFor(shop *appsv1.Shop) int32 {
    if shop.Spec.Replicas != nil { return *shop.Spec.Replicas }
    if shop.Spec.Availability == appsv1.AvailabilityHigh { return 3 }
    return 2
}
```
Dokazano u klasteru: `kubectl scale shop demo-shop --replicas=4` → deployment 4/4 i **ostane** 4
(operator ga ne vraća, jer i operator računa kroz `replicasFor`).

**P: Print kolone?**
O: `+kubebuilder:printcolumn` markeri → `kubectl get shops` prikazuje TITLE, DB, AVAILABILITY,
READY, URL, AGE. Plus `shortName=sh` i kategorija `shophub`. Sitno, ali odbrana izgleda bolje kad
`kubectl get sh` vrati čitljivu tabelu.

### 5.3 Reconcile tok Shop kontrolera

**P: Prošetajte me kroz jednu reconcile iteraciju za novi Shop.**
O: Redosled u `reconcile()`:
1. `Get` Shop; `client.IgnoreNotFound` (ako je obrisan, nema šta da se radi — GC je preuzeo).
2. Ako ima `DeletionTimestamp` → `reconcileDelete` (finalizer put, vidi 5.5). Inače dodaj finalizer
   ako fali.
3. `ensureDatabase` → grana po `spec.database`:
   - **postgres**: kreiraj CNPG `Cluster` (instances=1, bootstrap initdb sa našom šemom kroz
     `postInitApplicationSQL`, storage 1Gi) sa `SetControllerReference`; zatim `Get` Secret
     `<shop>-app`. Ako ga još nema → vrati `""` → petlja postavi `Progressing=True,
     Reason=DatabaseProvisioning` i requeue za 10s.
   - **mongodb**: prvo `ensureMongoDBRBAC` (SA/Role/RoleBinding u tenant ns), pa password Secret,
     pa `MongoDBCommunity` CR (SCRAM auth, user sa `dbOwner` rolom, **ConnectionStringSecretName
     pin-ovan na `<shop>-app`** da downstream logika bude ista za obe baze).
4. `ensurePasswordSecret` za `<shop>-admin` (random 16 bajtova hex) — admin lozinka prodavnice.
5. `ensureDeployment` — `CreateOrUpdate`: replika po `replicasFor`, image iz `spec.image` ili
   `DEFAULT_SHOP_IMAGE` env-a (chart value), probe, env (vidi 5.4).
6. `ensureService` — ClusterIP; ⚠️ labela `app: <shop>` mora na **sam Service** (ne samo selector)
   jer ServiceMonitor selektuje po labelama Service-a. To je bio pravi bug koji smo uhvatili na
   demou: Service bez labele → Prometheus ne otkrije target.
7. `ensureIngress` — host `<shop>.<baseDomain>`.
8. `ensureServiceMonitor`, `ensureDashboard`, `ensureAlertmanagerConfig` — **best-effort**: log i
   nastavi ako npr. Prometheus operator nije instaliran. Svesna odluka: observability ne sme da
   obori funkcionalnost prodavnice.
9. `updateStatusFromDeployment` — prepiše `readyReplicas`, `selector`, `url`, `databaseSecret` i
   postavi Conditions (`ready >= desired` → Available=True/Progressing=False/Degraded=False).

Svaki child ima `SetControllerReference(shop, obj, scheme)` → owner reference sa `controller: true`
→ Kubernetes GC briše sve pri brisanju Shop-a, bez ijedne linije našeg koda.

**P: Šta je `controllerutil.CreateOrUpdate` i zašto ga koristite umesto ručnog Get/Create/Update?**
O: Helper koji: Get-uje objekat, izvrši našu mutate funkciju nad željenim stanjem, pa Create ako ne
postoji ili Update **samo ako se nešto promenilo** (no-op update se ne šalje → nema novog eventa →
nema beskonačne petlje). To nam daje idempotentnost i "jedna izmena po iteraciji" disciplinu
praktično besplatno.

### 5.4 Env ugovor operator → Shop backend

**P: Kako Shop aplikacija dobija konekciju ka bazi?**
O: Operator u Deployment ubacuje `DATABASE_URL` env **preko secretKeyRef** — vrednost nikad ne
prolazi kroz operator niti završava u YAML-u u git-u. Izvor zavisi od baze: CNPG objavljuje
connection string pod ključem `uri` u `<shop>-app` Secretu; MongoDB Community pod
`connectionString.standard`. Backend onda po **šemi** URL-a (`postgres://` vs `mongodb://`) bira
store implementaciju — operator i backend komuniciraju isključivo tim jednim env ugovorom.
Uz to idu: `OTEL_EXPORTER_OTLP_ENDPOINT` (Tempo in-cluster), `OTEL_SERVICE_NAME` (= ime shopa, da
se trace-ovi grupišu po tenantu), `WALLET_ADDRESS` (za verifikaciju uplata), `SHOP_DB_NAME` (Mongo
connection string ne nosi default bazu, Postgres URI nosi — pa ime baze šaljemo posebno) i
`ADMIN_PASSWORD` (secretKeyRef na `<shop>-admin`).

### 5.5 Finalizeri

**P: Šta je finalizer i gde ga koristite?**
O: String marker u `metadata.finalizers` koji **zaustavlja brisanje**: kad korisnik obriše objekat,
API server samo postavi `deletionTimestamp` i objekat visi u `Terminating` dok svi finalizeri ne
budu uklonjeni. Kontroler detektuje `DeletionTimestamp != nil` → odradi cleanup **eksternog**
resursa → ukloni finalizer → tek tada GC završi brisanje. Pravilo: finalizer treba **uvek kad
kontroler upravlja nečim van Kubernetes-a** — GC briše samo K8s objekte.
Kod nas dva finalizera:
1. `DiscordChannel` — bez finalizera bi brisanje CR-a ostavilo kanal-siroče na Discord serveru.
2. `Shop` — `apps.shophub.local/grafana-tenant-dashboard`: per-tenant Grafana dashboard se pushuje
   preko HTTP API-ja (nije K8s objekat, GC ga ne vidi), pa ga finalizer briše iz tenant org-a.
A `Wallet` **namerno nema** finalizer — blockchain adresa se ne može "obrisati" (lanac je
append-only), a privatni ključ živi u Secret-u koji je owned → GC ga počisti. Znati kada finalizer
*ne* treba je jednako bitno kao znati kada treba.

**P: Šta ako Discord API ne radi u trenutku brisanja?**
O: `deleteChannel` fail → `RequeueAfter: 30s`, CR ostaje u Terminating i pokušava dok Discord ne
proradi. A za slučaj koji bi zauvek blokirao brisanje — **bot token Secret je već obrisan** — imamo
eksplicitan izlaz: ako je token Secret `NotFound` na delete putu, logujemo, preskočimo Discord
cleanup i skinemo finalizer, jer je bolje prihvatiti kanal-siroče nego CR koji se nikad ne može
obrisati. To je trade-off koji umem da obrazložim.

**P: Duplikat kanala — kako ste ga sprečili?**
O: Redosled upisa: čim Discord vrati channel ID, **prvo ga persistujemo u Status**, pa tek onda
pravimo webhook. Da smo išli obrnuto i webhook korak pukne, retry iteracija ne bi znala da kanal
već postoji i napravila bi drugi (a finalizer bi znao samo za poslednji → orphan). Ovo je primer
"jedna izmena po iteraciji + Status kao checkpoint" principa sa vežbi.

💡 Bonus detalj koji niko ne očekuje: Discord API **odbija webhook imena koja sadrže reč
"discord"** (`USERNAME_INVALID_CONTAINS`). Pošto se ime webhooka izvodilo iz imena kanala/shopa,
korisnik može legitimno da nazove prodavnicu tako da webhook pukne — zato je ime webhooka fiksna
konstanta `shophub-alerts`.

### 5.6 Watches, predikati, indeksi

**P: Razlika `For`, `Owns`, `Watches`?**
O: `For(&Shop{})` — primarni tip, svaki event nad njim ide u queue. `Owns(&Deployment{})` — child
resursi koje smo mi kreirali: event nad njima se mapira na **owner** Shop (filtrira se po owner
referenci). `Watches` — objekti koje **ne posedujemo** ali nas se tiču: kod nas Secreti koje pravi
CNPG/MongoDB operator, i webhook Secreti koje pravi DiscordChannel kontroler. `Watches` ima tri
dela: tip objekta, **Predicate** (filter na ulazu — pre keša) i **EventHandler** (mapiranje
objekat → lista reconcile zahteva; rešava 1-na-više: jedan Secret može da tiče 5 Shop-ova).

**P: Koje Watches imate i zašto?**
O: Dva Secret watch-a na Shop kontroleru:
1. **Connection Secret** — predikat: ime se završava na `-app` (CNPG konvencija `<cluster>-app`, a
   Mongo connection Secret smo *namerno pin-ovali na isto ime* da jedan predikat pokrije obe baze).
   Handler skine `-app` sufiks i enqueue-uje Shop istog imena. Efekat: operator reaguje **momentalno**
   kad baza objavi kredencijale, umesto da čeka requeue tick od 10s.
2. **Webhook Secret** — predikat: sufiks `-webhook`; handler koristi **FieldIndexer** nad
   `.spec.discordWebhookSecretRef.name` da u O(1) nađe Shop-ove koji referenciraju taj Secret
   (umesto O(N) List + iteracija). Indeks se registruje u `SetupWithManager` pre nego što manager
   krene.
Bez predikata bi se handler okidao na **svaki** Secret event u klasteru — na vežbama je to
eksplicitno nazvano CPU/memory eksplozijom, pa je predikat obavezan deo svakog našeg watch-a.

**P: Zašto FieldIndexer kad imate malo Shop-ova?**
O: Iskreno — za naš obim radilo bi i O(N). Ali indeks je bio preporuka sa vežbi, cena je ~10 linija,
a semantika je ispravnija: kešbuilding radi informer, lookup je mapa. Pokazuje da razumemo kako se
skalira, i asistent je to eksplicitno pominjao.

### 5.7 make run vs make deploy, leader election

**P: Razlika `make run` i `make deploy`?**
O: `make run` pokreće operator **lokalno kao Go proces** sa mojim kubeconfig-om (admin prava) —
brza iteracija u razvoju, ali maskira RBAC greške. `make deploy` (odnosno kod nas: helm install
chart-a) pokreće operator **u klasteru kao Deployment** sa svojim ServiceAccount-om i RBAC-om iz
markera — tek tu se vidi da li su RBAC markeri kompletni (fali marker → `403 Forbidden` od API
servera). Na odbrani operator radi in-cluster, instaliran iz **OCI Helm chart-a sa DockerHub-a**,
što je i jači dokaz od `make deploy` (koji koristi kustomize).

**P: Šta je leader election i da li vam treba?**
O: Kad operator ima više replika, samo lider izvršava Reconcile; ostale su hot standby. Bez toga bi
dve replike paralelno pisale iste objekte → race + konstantni 409. Kubebuilder skelet to već ima
(`--leader-elect` flag); naš chart ima `leaderElection` value, default `false` jer vozimo 1 repliku
— ali znam tačno šta bih uključio za HA.

**P: Šta vam kubebuilder daje "besplatno" u main.go?**
O: Manager sa: leader election, health/readiness probe endpointi (`/healthz`, `/readyz` na :8081 —
naš chart ih vezuje u liveness/readiness probe operatora), Prometheus `/metrics` endpoint (:8443,
metrike tipa `controller_runtime_reconcile_total`, `workqueue_depth`), scheme registracija,
graceful shutdown na SIGTERM. Pravilo sa vežbi: to se ne piše ručno, naša briga je samo Reconcile
logika.

---

## 6. Baze: CNPG i MongoDB operator

**P: Zašto niste sami pravili StatefulSet za Postgres?**
O: Pravilo sa vežbi: za standardne sisteme (baza, broker) se **koriste gotovi operatori** — naša
mini-implementacija nikad ne bi bila industrijskog kvaliteta (failover, backup, rotacija
kredencijala, upgrade...). CNPG je CNCF projekat i de-facto standard za Postgres na K8s. Naš
operator je "operator koji diriguje drugim operatorima": on kreira CNPG `Cluster` CR i dalje samo
posmatra.

**P: Kako izgleda CNPG Cluster koji pravite?**
O: `instances: 1` (demo; produkcija bi digla), `storage: 1Gi`, i `bootstrap.initdb` sa: database i
owner = ime shopa, plus `postInitApplicationSQL` koji kreira naše tabele `items` i `orders`.
⚠️ Detalj sa vežbi: `postInitApplicationSQL` izvršava **postgres superuser**, pa bez eksplicitnog
`ALTER TABLE ... OWNER TO <app-user>` aplikacija dobije "permission denied" na sopstvenim tabelama.
Naš dodatni detalj: ime shopa je DNS-1123 (sme crtica), a crtica u neqoutovanom Postgres
identifikatoru nije validna — pa owner ide **pod navodnicima**: `ALTER TABLE items OWNER TO "moj-shop"`.

**P: Šta CNPG objavi kad baza bude spremna?**
O: Set Secreta, za nas bitan `<cluster>-app` sa: username, password, host (cluster DNS), port,
dbname i gotovim `uri` connection stringom. Mi u Deployment mapiramo samo `uri` → `DATABASE_URL`.

**P: Koje ste probleme imali sa MongoDB operatorom? (ovde imam tri prave ratne priče)**
O:
1. **OwnerReference workaround:** `MongoDBCommunity` Go tip **override-uje `GetOwnerReferences()`**
   da vraća sintetičku self-referencu, što slomi `controllerutil.SetControllerReference` (misli da
   owner već postoji). Rešenje: ručno sagradimo `metav1.OwnerReference` (GVK preko
   `apiutil.GVKForObject`) i dodelimo ga **direktno na embedded `ObjectMeta` polje** — pristup polju
   zaobilazi method override. GC posle radi normalno.
2. **watchNamespace:** operatorov chart default gleda samo svoj install namespace → CR-ovi u tenant
   namespace-ima bi večno stajali. Instaliran je sa `watchNamespace="*"`.
3. **ServiceAccount materializacija:** podovi koje MongoDB operator pravi zahtevaju SA
   `mongodb-database` (+ Role za secrets/pods), a chart ga kreira samo u svom namespace-u. Naš
   operator zato u **svakom tenant namespace-u** materializuje SA + Role + RoleBinding
   (`ensureMongoDBRBAC`), owned by Shop → GC ih čisti.
Sve tri su u decision log-u sa datumima — mogu da pokažem da je to bio proces, ne sreća.

**P: Šta se desi ako neko ručno obriše `<shop>-app` Secret?**
O: Za CNPG: Secret je CNPG-ov objekat i CNPG ga po potrebi regeneriše; naš watch na `-app` Secrete
okine reconcile kad se ponovo pojavi. U međuvremenu postojeći podovi rade (env je već injektovan),
a novi podovi bi pali na startu — readiness ih drži van saobraćaja. Na vežbama je rečeno otvoreno:
za Delete event na tuđem Secretu nema elegantnog rešenja — dokumentuje se "ne dirajte ovaj Secret".

---

## 7. Shop aplikacija

**P: Arhitektura Shop backenda?**
O: Go + Gin. Slojevi: `httpapi` (router + handleri), `store` (persistence interfejs sa dve
implementacije — Postgres/pgx i MongoDB driver), `payment` (verifikacija na chainu + sweep),
`observability` (Prometheus middleware, strukturirani logovi, OTel tracing), `config` (env).
Jedan image služi i **storefront SPA**: Vite build se u multi-stage Dockerfile-u iskopira u
`/app/web` i Gin ga servira kroz NoRoute fallback (`/api`, `/metrics`, `/probe` ostaju API).

**P: Zašto interfejs `store.Store` sa dve implementacije?**
O: Spec traži dve vrste baze po izboru vlasnika prodavnice ("standard"/"light"). Umesto dva backend-a,
jedan backend bira implementaciju po šemi `DATABASE_URL`-a (`postgres://` ili `mongodb://`).
Interfejs vraća **domenske greške** (`ErrNotFound`, `ErrInsufficientStock`) pa HTTP sloj mapira na
statuse (404/409) a da store ne zna ništa o HTTP-u. Dobili smo i lakše testiranje (Testcontainers
po implementaciji).

**P: Kako radi admin deo prodavnice?**
O: Operator generiše random lozinku u `<shop>-admin` Secret i injektuje je kao `ADMIN_PASSWORD`;
mutacioni endpointi (`POST/PUT/DELETE /api/items`, `GET /api/orders`) su iza admin gate-a; vlasnik
lozinku vidi u ShopHub dashboardu (ShopHub sme da pročita Secret u tenant namespace-u). Listanje
artikala i kreiranje porudžbine je javno — kupci ne moraju nalog (spec 2.4 traži plaćanje, ne
autentifikaciju kupca — svesno pojednostavljenje koje umem da obrazložim).

**P: Objasnite rezervaciju stock-a — zašto ne skidate stock tek po potvrdi plaćanja?**
O: Tok je **reserve → pay → attach → verify**. `CreateOrder` atomski radi guarded decrement
(`UPDATE items SET stock = stock - $qty WHERE id=$id AND stock >= $qty` u istoj transakciji sa
insertom ordera; Mongo verzija koristi conditional update) — dva kupca ne mogu preprodati iste
komade. Da rezervišemo tek po potvrdi, u prozoru od par blokova bi dvojica platila isti poslednji
komad — a vraćanje para na chainu nije trivijalno kao vraćanje rezervacije. Naličje rezervacije je
da napušteni checkout drži stock — zato **pending order bez tx hash-a ističe posle 30 min** (sweep
ga faila i vrati rezervaciju).

**P: Šta ako klijent pošalje cenu/iznos u requestu?**
O: Ne veruje mu se: `AmountUSDT` je **output-only** — store ga računa iz **sačuvane** cene artikla
× količina u trenutku kreiranja ordera. Kupac ne može da "kupi za 0.01" krafted requestom, jer
verifikator na chainu očekuje server-side izračunat iznos.

---

## 8. Web3 plaćanje

**P: Kroz šta prolazi jedno plaćanje, korak po korak?**
O:
1. Kupac klikne Buy → frontend `POST /api/orders` → backend rezerviše stock, kreira `pending` order
   i vrati iznos.
2. Frontend (ethers v6) traži od MetaMask-a ERC-20 `transfer(shopWallet, amount)` na našem TestUSDT
   ugovoru; adresa prodavnice dolazi iz `/api/shop-info` (backend je ima iz `WALLET_ADDRESS` env-a
   koji je injektovao operator iz Shop Spec-a).
3. MetaMask vrati `txHash` → frontend `POST /api/orders/:id/tx`.
4. **Sweep petlja** (goroutina, tick 15s) uzme sve pending ordere: za svaki sa tx hash-om pozove
   `eth_getTransactionReceipt` na Sepolia RPC. Receipt nema → i dalje pending. Receipt failed →
   order failed + vrati stock. Receipt success → parsira **logove** receipta i traži `Transfer`
   event: emitovao ga **naš token ugovor**, `topics[0]` je keccak256 od
   `Transfer(address,address,uint256)`, `topics[2]` (to) je wallet prodavnice, a `data` (iznos) je
   ≥ očekivanog → `confirmed`.
5. Frontend poll-uje order dok ne vidi `confirmed`.

**P: Zašto sweep petlja, a ne goroutina po porudžbini?**
O: Per-request goroutina umire sa podom — ako se pod restartuje između uplate i potvrde, porudžbina
bi zauvek ostala pending. Sweep čita **iz baze** ("koji su orderi pending?") pa je stanje uvek
rekonstruktibilno: novi pod nastavlja tačno gde je stari stao. Idempotentno je i sa **više replika**
(standard=2!): potvrda je conditional update (`WHERE status='pending'`), pa ako dve replike
istovremeno potvrde isti order, druga je no-op. Ovo je isti "level-based, stateless" princip kao
kod operatora — namerno.

**P: Šta sprečava replay — da dva ordera prilože isti txHash?**
O: `RequiredForTx` sabere iznose **svih ne-failed ordera koji dele isti tx hash** i zahteva da
on-chain transfer pokrije **sumu**. Legitimni slučaj je korpa (više ordera plaćeno jednim
transferom — suma se poklapa); zloupotreba (dodaj još jedan order na stari hash) gura sumu iznad
on-chain iznosa i verifikacija pada.

**P: Zašto Sepolia? Zašto svoj token umesto "pravog" USDT?**
O: Sepolia je aktuelni Ethereum testnet sa najboljom podrškom (MetaMask out-of-the-box, javni
RPC-ovi, faucet-i za gas ETH). Spec eksplicitno kaže da je testnet dovoljan. "USDT na Sepoliji" ne
postoji kanonski — razne kopije sumnjive dostupnosti i bez faucet-a. Zato sam **deploy-ovao
sopstveni ERC-20** (preko Remix-a): simbol USDT, **6 decimala kao pravi USDT**, i **otvoren `mint`**
koji služi kao faucet — svako na demou može sebi da iskuje token i proba kupovinu. Backend-u je
svejedno: ERC-20 interfejs je standard, `TOKEN_CONTRACT` je konfiguracija.

**P: Objasnite decimale — šta znači 6 decimala?**
O: ERC-20 iznosi su celi brojevi u "base units"; `decimals` kaže gde je zarez. 5 USDT sa 6 decimala
= 5.000.000 jedinica. Konverziju radimo `big.Rat`-om (`ToBaseUnits("5.5", 6) = 5500000`) — nikad
float, da ne bude drift. Iz istog razloga i cene kroz ceo sistem putuju kao **stringovi** i u
Postgresu su `numeric(36,18)`.

**P: Šta je gas i ko ga plaća?**
O: Naknada za izvršavanje transakcije na mreži, plaća je pošiljalac (kupac) u ETH (testnet ETH sa
faucet-a). ERC-20 `transfer` je poziv funkcije ugovora → troši više gasa nego prost ETH transfer,
jer menja storage ugovora (balances mapa) i emituje event.

**P: Zašto verifikujete kroz event logove, a ne `tx.to` i `tx.value`?**
O: Kod ERC-20 plaćanja `tx.to` je **adresa token ugovora**, ne primaoca, a `tx.value` je 0 ETH —
stvarni transfer se dešava unutar izvršavanja ugovora. Jedini pouzdan trag je `Transfer` event u
receipt logovima (uz proveru `receipt.Status == success` — jer failed transakcija takođe ima
receipt). Zato čitamo receipt, filtriramo logove po adresi ugovora + event signature + primaocu.

**P: Šta je Wallet CRD onda, ako Shop već ima walletAddress?**
O: Spec 3.1 traži Wallet CRD koji "kreira account na blockchainu". Naš Wallet kontroler: ako je
`ownerAddress` zadat — samo ga upiše u Status; ako nije — **generiše secp256k1 keypair**
(go-ethereum `crypto.GenerateKey`), adresu izvede iz javnog ključa, a privatni ključ upiše u Secret
`<wallet>-key` (owned → GC). ShopHub ima dugme "generiši mi wallet": napravi Wallet CR, sačeka
(bounded poll, 10s) da operator objavi adresu, i korisnik je iskoristi za Shop. Race pri kreiranju
Secreta (dve replike operatora / dupli event) je pokriven: na `AlreadyExists` bacimo naš ključ i
pročitamo adresu iz postojećeg Secreta.

⚠️ Tehnički pošteno je reći: "kreiranje accounta na blockchainu" u Ethereum modelu ne postoji kao
on-chain operacija — adresa postoji čim postoji keypair; na chain "uđe" prvom transakcijom. Naš
kontroler dakle generiše i čuva keypair, što je maksimum smisla tog zahteva.

---

## 9. ShopHub platforma

**P: Kako radi autentifikacija?**
O: Register: bcrypt hash lozinke (default cost) → insert u `users` (CNPG Postgres u `shophub`
namespace-u; unique violation → 409) → kreiranje tenant namespace-a → JWT. Login: bcrypt compare →
JWT. JWT je **HS256**, TTL 24h, i u claims nosi `ns` — tenant namespace korisnika. Middleware
verifikuje potpis (+ eksplicitno proverava da je alg HMAC familija — brani od alg-swap napada,
npr. `alg: none`), i stavlja namespace u request context. Poruke grešaka na loginu su iste za
"nema korisnika" i "pogrešna lozinka" — bez user enumeration-a.

**P: Zašto namespace u JWT claims?**
O: Svaki API poziv treba scope; da je u bazi, svaki request bi imao dodatan DB lookup. JWT je
potpisan — korisnik ne može da izmeni `ns` claim bez invalidacije potpisa, pa je to bezbedan nosač.
Handleri onda rade `client.InNamespace(nsFromCtx(c))` na svakom List/Get/Create — korisnik
**fizički ne može** da vidi tuđe Shop-ove jer se tuđi namespace nigde ne pojavljuje u njegovom toku.

**P: Kako ShopHub priča sa Kubernetesom?**
O: `controller-runtime` client (isti kao u operatoru, sa registrovanim našim šemama) — ne sirovi
client-go, jer nam treba typed CRUD nad našim CRD-ovima i isti API smo već znali iz operatora.
In-cluster config + ServiceAccount; ClusterRole daje: namespaces (create/get), shops/wallets/
discordchannels (CRUD u svim namespace-ima), secrets get (za admin lozinku prodavnice). ShopHub je
**jedini** deo sistema koji sme da pravi namespace-e.

**P: Šta ako kreiranje namespace-a pukne posle upisa korisnika u bazu?**
O: Redosled je namerno: prvo DB insert (da duplikat emaila pukne čisto pre nego što diramo klaster),
pa namespace. Ako namespace korak pukne, nalog postoji ali bez namespace-a — i tu je self-healing:
`ensureNamespace` je idempotentan i **poziva se ponovo na svakom loginu**. Znači sledeći login
popravi stanje. Isti mehanizam leči i namespace obrisan ručno.

**P: DiscordChannel flow iz perspektive ShopHub-a?**
O: Platforma ima jedan Discord server (guild) i jednog bota; guild ID i ime bot-token Secreta su
chart values, a **sam token je isključivo u pre-kreiranom Secretu** — nikad u git-u. Kad korisnik
štiklira "Discord notifikacije", ShopHub kreira `DiscordChannel` CR (ime = ime shopa) **sa owner
referencom na Shop** — pa brisanje shopa povuče i brisanje kanala (kroz finalizer kontrolera).
Operator-ov DiscordChannel kontroler onda napravi kanal + webhook i objavi `<shop>-webhook` Secret,
koji Shop CR referencira u `discordWebhookSecretRef`, a Shop kontroler od toga napravi
AlertmanagerConfig. Ceo lanac: checkbox u UI → kanal na Discordu → alarmi te prodavnice u njemu.

**P: Multi-tenancy — šta konkretno garantuje izolaciju?**
O: Tri sloja: (1) JWT nosi namespace i middleware ga nameće — API nikad ne prima namespace od
klijenta; (2) svi K8s pozivi su `InNamespace`; (3) observability izolacija — AlertmanagerConfig sa
OnNamespace strategijom rutira samo alarme svog namespace-a, Grafana per-tenant org pokazuje samo
svoje dashborde. Ako pitaju šta bih dodao za "pravu" produkciju: NetworkPolicy između tenant
namespace-a i ResourceQuota po tenantu — znam da fali, svesno van obima.

---

## 10. Observability

### 10.1 Metrike

**P: Kako Prometheus saznaje za novu prodavnicu? (service discovery lanac)**
O: Lanac: Shop backend izlaže `/metrics` (promauto counteri) → operator za svaki Shop kreira
**ServiceMonitor** (selektor `app: <shop>`, port `http`, path `/metrics`, interval 15s) →
prometheus-operator (deo kube-prometheus-stack-a) od svih ServiceMonitora generiše scrape config →
Prometheus scrape-uje. Nula ručne konfiguracije po prodavnici — to je poenta operatora.
⚠️ Detalj koji su svi na vežbama zeznuli: default Prometheus bira samo ServiceMonitore sa labelom
`release: kube-prometheus-stack`. Mi smo u kube-state values postavili
`serviceMonitorSelectorNilUsesHelmValues: false` (i pod/rule varijante) → Prometheus prihvata SM-ove
iz **svih** namespace-a bez tog label-a. Razlog: da tenant SM-ovi ne budu vezani za ime konkretnog
helm release-a. (U SM koji operator pravi ipak stavljamo release labelu — belt & suspenders, radi u
oba režima.)

**P: Koje metrike izlaže Shop backend i kako pokrivaju spec 4.1?**
O: Tri custom instrumenta:
- `shop_http_requests_total{method,route,status}` — counter; iz njega PromQL-om izvodim sve iz
  spec-a: ukupno u 24h = `sum(increase(shop_http_requests_total[24h]))`; uspešni =
  `... {status=~"2..|3.."}`; neuspešni = `{status=~"4..|5.."}`; 404 po endpointu =
  `sum by (route) (increase(...{status="404"}[24h]))`.
- `shop_http_request_duration_seconds` — histogram; p95 latencija kroz `histogram_quantile`.
- `shop_http_response_bytes_total` — protok; GB = `sum(increase(...[24h])) / 1e9`.
CPU/RAM/FS/mreža po podu dolaze "besplatno" iz cAdvisor-a (`container_*` metrike), a za nodove/
klaster iz node-exporter-a + kube-state-metrics.
Dva prava bug-fixa vezana za metrike koje volim da ispričam: (1) Gin `c.FullPath()` je prazan kad
ruta ne postoji — za 404 beležimo sirovu putanju (da dashboard "404 endpoints" pokaže šta je
gađano), a sve ostalo bez rute ide u `"unknown"` da ne eksplodira label cardinality; (2) Gin-ov
`Writer.Size()` je **-1** pre nego što je body upisan, a `counter.Add(negativno)` **panikuje** —
guard `size > 0`. Taj panic je u demou pravio 500 umesto 404.

**P: Jedinstveni posetioci (IP + timestamp + browser)?**
O: To namerno **nije** Prometheus metrika: kombinacija IP+UA je visoke kardinalnosti, a Prometheus
sa label-om po poseti bi eksplodirao. Rešenje: backend loguje strukturirano (IP, user-agent) →
Promtail → **Loki**, i dashboard panel radi Loki query sa distinct agregacijom u 24h prozoru.
Pravi alat za high-cardinality podatke su logovi, ne metrike — to je poenta koju želim da izgovorim.

### 10.2 Logovi i trace-ovi

**P: Kako ste rešili logging i tracing?**
O: **Loki + Promtail** (Promtail kao DaemonSet kupi stdout svih podova, labelira po namespace/pod)
— logovi svake prodavnice dostupni u Grafani (Explore → Loki, filter po tenant namespace-u).
**Tempo + OpenTelemetry**: backend je instrumentiran otelgin middleware-om, span-ovi se šalju
OTLP/HTTP na in-cluster Tempo (endpoint injektuje operator kroz env), `OTEL_SERVICE_NAME` = ime
shopa pa se trace-ovi grupišu per tenant. Grafana ima Tempo datasource (dodat eksplicitno u values;
Loki datasource dolazi automatski iz loki chart-a preko `grafana_datasource` ConfigMap-a — pazili
smo da ga ne dupliramo).

### 10.3 Dashboardi

**P: "Svaka Shop aplikacija treba da ima svoj dashboard" — kako?**
O: Operator za svaki Shop renderuje dashboard iz **embedovanog JSON template-a** (`go:embed
dashboard.json`, placeholder `$shop` → ime shopa, uid `shop-<ime>`) i upiše ga u ConfigMap sa
labelom `grafana_dashboard: "1"`. Grafana sidecar (deo stack-a) automatski importuje takve
ConfigMap-e. Dakle bukvalno **poseban dashboard objekat po prodavnici**, ne jedan zajednički sa
dropdown-om — jer spec kaže "svoj dashboard". ConfigMap je owned by Shop → dashboard nestane sa
prodavnicom. Uz to, anotacija `grafana_folder: <tenant-namespace>` + sidecar `folderAnnotation` →
dashboardi se grupišu u folder po tenantu.

**P: Opcioni zahtev — korisnik vidi samo svoje dashborde. Kako ste to izveli u OSS Grafani?**
O: Prva ideja — folder permisije — **ne radi u OSS Grafani**: basic role (Viewer) je org-wide, pa
Viewer vidi sve foldere; fine-grained RBAC je Enterprise feature. Zato smo otišli na **Grafana
Organizations**: org je hard granica izolacije u OSS-u. Pri registraciji ShopHub kreira org
`tenant-<id>` + Viewer korisnika u **samo tom** org-u (admin API, verifikovano da user kreiran sa
`OrgId` ne završi u Main Org). Operator pri reconcile-u svaki tenant dashboard pushuje i u tenant
org preko HTTP API-ja (u org-u ponovo kreira i datasource-e jer su org-scoped). Maintaineri i dalje
koriste Main Org (vide sve — spec kaže da pristup Grafani imaju samo maintaineri, korisnici samo
opciono svoje). Cleanup tenant dashboard-a pri brisanju shopa radi **Shop finalizer** — jer HTTP
objekat u Grafani nije K8s resurs pa ga GC ne vidi.

### 10.4 Alarmi

**P: Kako putuje alarm od metrike do Discord poruke?**
O: `PrometheusRule` (ship-ujemo ga u operator chart-u) definiše pravila — npr. `ShopHighErrorRate`:
udeo 4xx/5xx > 10% u 5m prozoru, grupisano `by (namespace, service)` → **svaki firing alert nosi
namespace labelu**. Prometheus evaluira → šalje Alertmanager-u → Alertmanager mata alert prema
**AlertmanagerConfig** CR-ovima. Naš operator za svaki Shop (koji ima Discord webhook ref) pravi
AlertmanagerConfig u namespace-u shopa sa `discordConfigs` receiver-om. Stack je konfigurisan sa
`alertmanagerConfigMatcherStrategy: OnNamespace` — Alertmanager **automatski ograniči** route tog
config-a na alarme iz istog namespace-a. Rezultat: alarmi prodavnice A idu samo na Discord kanal
prodavnice A. Webhook URL ide kao **`apiURL` secret-ref** — URL nikad nije u renderovanom configu
ni u git-u.
⚠️ Ratna priča: prvi pokušaj je bio inline Alertmanager config sa `webhook_url_file` — ali
prometheus-operator validira inline config protiv svoje šeme koja to polje **nema**, pa odbija ceo
config. Jedini način da webhook bude secret-ref je AlertmanagerConfig CRD sa `apiURL` poljem. To je
tačno vrsta detalja koja pokazuje da smo stvarno ovo terali do kraja.

**P: Kako demonstrirate alarm?**
O: Pucam petlju 404 zahteva na prodavnicu (`while ($true) { curl .../nepostojeci }`) → error rate
pređe 10% → posle `for: 10s` (demo-kratko namerno) rule pređe u firing → Alertmanager → Discord
poruka u kanalu prodavnice. `SendResolved: true` pa stigne i "resolved" poruka kad prestanem.
Imamo i klasterske alarme: NodeHighCPU/NodeHighMemory > 90% (spec traži i alarme za sam klaster).

---

## 11. Docker

**P: Objasnite svoj Dockerfile.**
O: Multi-stage, tri faze (na primeru shophub/shop unified image-a):
1. `node:20-alpine` — Vite build frontenda (sa `--mount=type=cache` za npm keš).
2. `golang:1.26-alpine` — Go build; `CGO_ENABLED=0` (statički binarni, bez libc zavisnosti),
   `-ldflags='-s -w'` (skida simbole i DWARF → manji binarni); cache mount za module i build keš —
   rebuild sa istim go.sum ne skida ništa ponovo.
3. `alpine:3.20` release — samo ca-certificates + binarni + SPA dist. **Non-root**: `adduser -S app`,
   fajlovi `chown root:root` + `chmod 0755` (app korisnik može read+execute, ne write — ne može da
   prepiše sopstveni binarni), `USER app`. `ENTRYPOINT ["/app/shophub"]` — **exec forma**.
Zašto exec forma: shell forma pokreće proces kao dete `sh`-a → aplikacija nije PID 1 → SIGTERM od
`docker stop`/K8s ne stiže do nje → nema graceful shutdown-a, ubija je tek SIGKILL posle timeout-a.
Brz test: ako `docker stop` traje 10s, verovatno je shell forma.

**P: Zašto multi-stage — šta tačno dobijate?**
O: (1) veličina — finalna slika nema Go toolchain, node_modules, source; (2) bezbednost — manja
attack surface, niko u kontejneru ne može da rebuild-uje izmenjen kod jer nema ni kompajlera ni
koda; (3) keširanje — slojevi za zavisnosti se ne invalidiraju izmenom koda (COPY go.mod/go.sum pa
download, tek onda COPY source).

**P: hadolint?**
O: Linter za Dockerfile — imamo ga kao korak u docker-build workflow-u na svakom PR-u, pa
Dockerfile regresije ne mogu da se merguju. Port > 1024 zbog non-root (8080), fiksne verzije baznih
slika, no-cache za apk — sve po checklisti sa vežbi.

**P: Kako lokalna slika uđe u k3d klaster?**
O: `k3d image import <image> -c local` — k3d nodovi su docker kontejneri sa sopstvenim containerd
image store-om, ne vide hostov docker daemon. Za odbranu je ipak sve na DockerHub-u (public), pa
klaster normalno pull-uje; import koristim samo za lokalne dev iteracije.
⚠️ Gotcha koju znam iz iskustva: posle promene image-a sa istim tagom, `kubectl rollout restart`
na operator-managed Deployment ne pomaže trajno — operator na sledećem reconcile-u vrati svoj
template. Ispravan tok je import → delete podova (ili bump taga).

---

## 12. Helm i IaC

Ovo je **eliminacioni zahtev (5.3)** pa strukturu znam napamet.

**P: Struktura helm-charts repoa?**
O: `charts/shop-operator`, `charts/shophub`, `charts/shop`. Sve naše chart-ove CI lint-uje
(`helm lint` + `helm template` sanity) i publish-uje kao **OCI artefakte** na DockerHub
(`oci://registry-1.docker.io/urospetraskovic/<chart>`).

**P: Šta je u shop-operator chartu i zašto su CRD-ovi u `crds/` folderu?**
O: Templates: Deployment operatora, ServiceAccount, ClusterRole + Binding (RBAC generisan iz
markera pa prenet u chart), metrics Service, PrometheusRule (alarmi), opcioni grafana-admin Secret.
CRD YAML-ovi (output `make manifests`) idu u **`crds/`**: Helm 3 ih instalira **pre** template-ova
i **nikad ih ne dira na upgrade/delete**. To je svesni trade-off: + siguran install redosled (CRD
postoji pre nego što bilo šta pokuša da napravi CR) i nema slučajnog brisanja CRD-a (što bi obrisalo
sve CR-ove i sve prodavnice!); − upgrade CRD šeme zahteva ručni `kubectl apply` novih CRD-ova. Za
CRD apply postoji još jedna caka: veliki CRD prelazi limit `last-applied-configuration` anotacije →
`kubectl apply --server-side`.

**P: Čemu služi shop chart ako Shop-ove pravi operator?**
O: Tanak chart koji renderuje **Shop CR** (+ opcioni Discord webhook Secret) iz values-a. Namena:
(1) struktura iz spec-a 5.3 ga eksplicitno prikazuje; (2) ručni/debug put — prodavnica bez ShopHub
UI-ja (`helm install moja charts/shop --set name=...`); operator normalno preuzima od CR-a dalje.
Razmatrali smo i varijantu da operator interno renderuje helm template — presloženo, odbačeno.

**P: ShopHub chart i zahtev 3.3 ("koristi CRD-ove iz operatora i prometheus-stack")?**
O: ShopHub chart: Deployment (unified image), RBAC (ClusterRole za namespaces + CRD-ove — koristi
CRD-ove iz operatora, deo 1 zahteva), CNPG `Cluster` za users bazu, JWT Secret
(lookup-preserve trik: ako Secret već postoji, upgrade ga ne regeneriše — inače bi svaki upgrade
izlogovao sve korisnike), ServiceMonitor (vezuje se na prometheus-stack, deo 2 zahteva), Ingress.
Plus `kube-prometheus-stack` je deklarisan kao **uslovni dependency** u Chart.yaml
(`condition: kube-prometheus-stack.enabled`, default off): zahtev 3.3 je time formalno ispunjen, a
u praksi stack instaliramo jednom cluster-wide kroz kube-state — monitoring je infrastruktura
klastera, ne privezak jedne aplikacije; dva Prometheusa nemaju smisla. Ovu odluku znam da odbranim.

**P: Verzija chart-a vs appVersion?**
O: `version` je verzija **pakovanja** (chart-a) — SemVer, po njoj se povlači iz registra;
`appVersion` je verzija aplikacije koju chart podrazumevano deploy-uje (default image tag).
Odvojeno se bump-uju: promena samo template-a digne chart version, novi image digne appVersion.

**P: Struktura kube-state repoa?**
O: `clusters/local/` sa `cluster.yaml` (k3d config: 1 server + 2 agenta, port mape 8080:80,
8443:443) i po folder za svaku komponentu: `helm.yaml` (OCI/repo referenca chart-a + verzija +
namespace) + `values.yaml` (overrides za taj klaster). Komponente: cnpg, mongodb-operator,
kube-prometheus-stack, loki, tempo, shop-operator, shophub. Poenta: **klaster je rekonstruktibilan
iz git-a** — SETUP.md vodi od praznog laptopa do radnog sistema, i to smo bukvalno testirali od
nule. Plus `argocd/` folder (sledeća sekcija).

**P: Zašto helm install svega odjednom ne valja? (pitanje sa vežbi)**
O: Helm je **templating + apply**, ne orkestrator stanja: sve manifeste preda API serveru
paralelno, "success" znači "applied", ne "radi". Ako app krene pre nego što operator objavi Secret
baze → race, podovi crashuju a helm kaže da je sve ok. Zato je bring-up **iterativan po talasima**:
prvo operatori koji donose CRD-ove (cnpg, mongodb, prometheus-stack), pa naš operator + loki/tempo,
pa shophub. U ručnoj (Opcija B) proceduri to su eksplicitni koraci sa `kubectl wait`; u ArgoCD
varijanti to elegantno rešavaju retry + eventual consistency (i sync wave anotacije po potrebi).

**P: Gde su tajne, ako je sve u git-u?**
O: **Nigde u git-u.** Tri tajne se pre-kreiraju ručno kao Secreti: Grafana admin lozinka
(`grafana-admin`, ista u monitoring/shop-operator-system/shophub namespace-ima — jer Secret ne može
da se referencira cross-namespace), Discord bot token (`discord-bot-token` u shophub ns), i JWT
secret (chart ga generiše i čuva lookup-om). Values fajlovi referenciraju samo **imena** Secreta
(`existingSecret: grafana-admin`). U CI-ju su tajne GitHub Actions Secrets. Sve ostale tajne
(DB kredencijali, webhook URL-ovi, admin lozinke prodavnica, wallet ključevi) generišu operatori
u klasteru i nikad ne napuštaju klaster.

---

## 13. GitOps i ArgoCD

**P: Šta je GitOps i šta vam ArgoCD konkretno radi?**
O: GitOps = git je **jedini izvor istine** o stanju klastera; agent u klasteru neprestano poredi
živo stanje sa git-om i konvergira ga (pull model — klaster vuče, niko ne push-uje kubectl-om).
ArgoCD kod nas: `Application` CR po komponenti (u `kube-state/argocd/apps/`) pokazuje na helm chart
(naši preko OCI DockerHub-a, upstream preko helm repo-a) + values iz kube-state repoa. Sa
`syncPolicy.automated: {prune: true, selfHeal: true}`: commit u kube-state → ArgoCD sam primeni;
ručna izmena u klasteru (drift) → ArgoCD je **vrati na git stanje**; obrisano iz git-a → prune.

**P: App-of-apps pattern?**
O: `root.yaml` je Application koji pokazuje na folder `argocd/apps/` — dakle aplikacija čiji su
"resursi" druge aplikacije. Bootstrap celog klastera je: instaliraj ArgoCD + apply root → ArgoCD
sam podigne svih 7 komponenti. Jedan manifest = ceo sistem. To je i demo highlight: pokazati ArgoCD
UI gde je sve zeleno/Synced, pa obrisati npr. operator Deployment i gledati selfHeal kako ga vraća.

**P: Zašto ArgoCD, a ne Flux?**
O: Oba su ravnopravna CNCF Graduated rešenja; ArgoCD je izabran zbog UI-ja (vizuelno stablo resursa
je odlično i za debugging i za odbranu) i jer je bio pokriven materijalima. Flux je više
"headless/toolkit" pristup — legitiman, samo drugačiji izbor.

---

## 14. CI/CD i Git

**P: Kako izgleda vaš git workflow?**
O: **Trunk Based Development**: jedan `main` (trunk), kratkoživeće feature grane (1-3 dana), bez
`develop` — TBD eksplicitno kaže trunk + eventualno release grane. Svaki PR: obavezan **1 approval**
+ svi required checks zeleni + **linear history** (squash merge) — sve enforced kroz branch
protection, bez bypass-a. Conventional Commits format poruka, automatski validiran `commitlint`
workflow-om u svih 5 repo-a.

**P: Zašto linearna istorija / squash?**
O: `main` istorija = niz atomskih, imenovanih promena (jedan PR = jedan commit) — bisect,
changelog i revert su trivijalni; nema merge-commit špageta. Praktična cena koju znam iz iskustva:
posle squash-a lokalna grana "nije merged" po git-u (drugi SHA) → `git branch -D`, i lokalni main
se resetuje na origin.

**P: Šta se dešava na PR, a šta na merge u main?**
O: **PR (CI):** commitlint; lint (golangci-lint); test (go build + go vet + go test sa
Testcontainers integracionim testovima); docker-build (hadolint + build image-a **bez** push-a —
validacija da je slika build-abilna); helm-lint u helm-charts repou. Required checks se u branch
protection vezuju po **imenu job-a** (ne workflow-a) i moraju postojati u tom repou — naučeno
podešavanjem. **Merge u main (CD):** docker-publish build-uje i push-uje image na DockerHub (tag
`main` + SHA), helm-publish pakuje chart-ove kao `verzija-dev.YYYYMMDD.sha` i push-uje na OCI.
**Git tag `v*` (release):** docker-publish sa čistom SemVer verzijom iz taga; helm-publish objavi
**čistu verziju iz Chart.yaml** (suffix se skida).
⚠️ Fina caka: kod helm-publish-a release verzija dolazi iz **Chart.yaml**, ne iz git taga — tag je
samo okidač. Znači bump verzije u Chart.yaml je deo release PR-a.

**P: SemVer strategija?**
O: MAJOR.MINOR.PATCH; `fix:` → patch, `feat:` → minor, breaking → major (mapiranje na Conventional
Commits). Image-i i chart-ovi su pin-ovani po verziji u kube-state (npr. operator chart 0.1.7+,
shophub 0.2.x) — nikad `latest` u deklaraciji klastera, jer bi klaster prestao da bude
reproducibilan.

**P: DockerHub nalog vs GitHub org?**
O: GitHub org je `shophub-devoops`, DockerHub nalog `urospetraskovic` (org imena na DockerHub-u su
naplatna). I jedna netrivijalna kolizija: chart `shop-operator` i image operatora bi na OCI
registru delili isto ime (`urospetraskovic/shop-operator`) i pregazili se — pa je **image
preimenovan u `shop-operator-controller`**, chart zadržao `shop-operator`.

---

## 15. Testiranje

**P: Kakve testove imate?**
O: Tri nivoa:
1. **Unit/reconcile testovi operatora** — fake client (`fake.NewClientBuilder`) sa našim šemama +
   `WithStatusSubresource` (bez toga fake client ne ume `r.Status().Update()`), pa ručno pozivamo
   `Reconcile()` i assertujemo child resurse (Deployment sa 2/3 replike, Service, uslovi u Statusu...).
2. **Integracioni sa Testcontainers** — ShopHub auth testovi dižu **pravi Postgres u docker
   kontejneru** (bcrypt+JWT+unique email flow, uz fake kube client za namespace-e); shop store/API
   testovi isto (i Mongo varijanta). Zahtev 5.2 eksplicitno traži Testcontainers ili compose za
   integracione testove — imamo Testcontainers, radi i u GitHub Actions (runner ima docker).
3. **E2E workflow** operatora (kind klaster u CI-ju, kubebuilder scaffold).
Envtest ograničenje koje znam: **brisanje namespace-a ne radi** u envtest-u (nema kontrolera koji
finalizuje namespace) → svaki test dobija svež namespace sa random sufiksom umesto cleanup-a.

**P: Šta je Testcontainers i zašto je bolji od mock-a baze?**
O: Biblioteka koja iz testa programatski podigne pravi kontejner (postgres:16) i vrati connection
string; test tako vežba **pravi SQL** (constraint-e, transakcije, `ON CONFLICT`...), što mock ne
može. Cena je sporiji test — zato su odvojeni od unit testova.

---

## 16. Zeznuta pitanja

Skupljeno sve što bih ja pitao da hoću da vidim da li je kandidat stvarno radio projekat.

**P: Obrišem vam Shop CR. Šta se tačno, hronološki, dešava?**
O: (1) API server vidi finalizer → postavi deletionTimestamp, CR u Terminating. (2) Reconcile uđe u
delete granu: obriše tenant Grafana dashboard preko HTTP-a (best-effort — Grafana koja ne radi ne
sme da zaglavi brisanje), skine finalizer. (3) CR nestane. (4) GC dependency walk: briše owned
resurse — Deployment (→ ReplicaSet → podovi), Service, Ingress, ServiceMonitor, AlertmanagerConfig,
dashboard ConfigMap, admin Secret, CNPG Cluster (CNPG dalje briše svoje podove/PVC), odnosno
MongoDBCommunity + RBAC trio. (5) DiscordChannel CR je owned by Shop → GC krene da ga briše → ali
on ima svoj finalizer → njegov kontroler prvo obriše kanal na Discord serveru, pa pusti. Ništa od
ovoga nije eksplicitni "delete" kod u Shop kontroleru — sve nosi ownership + finalizeri.

**P: Šta ako dva korisnika kreiraju shop sa istim imenom?**
O: Radi — imena su namespace-scoped, a svaki korisnik ima svoj namespace. Isti korisnik dva puta
isto ime → API server vrati AlreadyExists → 409 korisniku. (Ingress host je `<ime>.<domen>` pa bi
dva istoimena shopa različitih korisnika delila host — poštena poznata limitacija za localhost demo;
u produkciji bi host uključivao i tenant.)

**P: Operator vam padne / bude ugašen. Šta se dešava sa prodavnicama?**
O: Ništa dramatično — to je lepota level-based dizajna. Postojeći Deploymenti/Service-i nastavljaju
da rade (operator nije na putanji saobraćaja). Promene u međuvremenu se ne procesiraju, ali kad se
operator vrati, informer napravi **full resync** i reconcile dovede sve u željeno stanje — nema
"propuštenih eventova" jer ne zavisimo od eventova nego od trenutnog stanja.

**P: Zašto vam ShopHub sme da čita Secrete u tenant namespace-ima? Nije li to širok RBAC?**
O: Treba mu za admin lozinku prodavnice (prikaz vlasniku). Jeste šire nego idealno — least
privilege bi bio verb `get` na secrets (bez list), što i imamo; sledeći korak pooštravanja bi bio
resourceNames pattern ili poseban "credentials broker". Svestan trade-off, zapisan.

**P: Kako znate da alert ide baš u pravi Discord kanal, a ne u sve?**
O: Dva mehanizma zajedno: PrometheusRule grupiše `by (namespace, ...)` pa svaki alert nosi namespace
labelu, a `OnNamespace` matcher strategy tera Alertmanager da AlertmanagerConfig iz namespace-a X
primenjuje **samo** na alerte sa `namespace=X`. Test: dva shopa, 404 flood na prvi → poruka stiže
samo u kanal prvog.

**P: Šta se desi ako MetaMask transakcija bude uspešna, ali korisnik zatvori tab pre attach-a?**
O: Order ostane pending bez tx hasha i posle 30 min istekne (stock se vrati). Para je otišla — to je
poznata limitacija flow-a (kupac ima tx hash u svom walletu i podrška bi ga ručno rešila). Rešenje
"kako treba" bi bilo da backend indeksira sve Transfer evente ka wallet adresi pa sam upari uplate,
ali to je van obima — znam da postoji i kako bi se radilo.

**P: BlockOwnerDeletion deadlock — o čemu se radi?**
O: Objekat može imati više owner referenci (samo jedan `controller: true`). Ako više ownera drži
`blockOwnerDeletion: true` na istom dependent objektu, može se napraviti ciklična blokada gde niko
ne može da se obriše. Preporuka sa vežbi: blockOwnerDeletion samo u 1-na-1 vezama; za 1-na-više
koristiti labele + sopstvenu logiku. Kod nas je svaki objekat owned od tačno jednog CR-a, pa smo
bezbedni.

**P: Zašto Reconcile ne sme da menja Spec?**
O: Spec je deklaracija korisnika — vlasništvo korisnika. Ako ga kontroler menja, korisnik i
kontroler se biju oko istog polja (i GitOps alat bi drift vraćao unazad — ArgoCD selfHeal bi se
tukao sa operatorom). Jedini legitimni izuzeci su scale subresource i finalizeri. Kod nas: HPA/
kubectl scale piše `spec.replicas` kroz scale subresource — to je korisnikova volja izražena kroz
standardni interfejs, ne operator koji sam sebi menja spec.

**P: Šta bi se desilo da nemate predikat na Secret watch-u?**
O: Svaki Secret event u klasteru (a stack tipa prometheus/grafana ih ima dosta i rotira ih) bi:
(a) ušao u reflector keš → memorija raste sa brojem Secreta u klasteru; (b) okidao naš EventHandler
→ enqueue → prazni reconcile-i → CPU. Sa predikatom kešira se i procesira samo ono što prolazi
filter. Na malom klasteru se ne vidi, na pravom je to inženjerska razlika.

**P: Grafana lozinke korisnika u bazi — plain?**
O: Ne — tenant Grafana lozinka se čuva **enkriptovana at rest** (simetrična enkripcija u
`grafana/crypto.go`, ključ van baze) jer mora moći da se **prikaže** korisniku (nije naš auth,
Grafana je treći sistem gde se on loguje). ShopHub lozinke su bcrypt hash — nikad se ne prikazuju.
Razlika hash vs enkripcija i kad se koje koristi — umem da objasnim.

**P: Zašto ne koristite `latest` tag?**
O: `latest` nije verzija nego pokretna meta: klaster prestaje da bude reproducibilan (isti YAML,
različit sadržaj), rollback je nemoguć, cache node-a odlučuje šta stvarno radi. Sve u kube-state je
pin-ovano; `latest`/`main` tagovi postoje na registru samo kao pogodnost za dev.

**P: Koliko replika ima operator i šta bi se desilo sa dve bez leader election-a?**
O: Jedna (chart default). Dve bez leader election-a: obe drže isti informer state i obe reconcile-uju
iste objekte → utrkuju se u pisanju → konstantni 409 (benigni ali bučni), a kod eksternih resursa
(Discord!) i **pravi duplikati** — dva createChannel poziva pre nego što prvi upiše Status. Zato
je multi-replica dozvoljen samo sa leaderElection=true.

**P: Po čemu se vaš pristup razlikuje od "helm chart po prodavnici" bez operatora?**
O: Helm je jednokratan apply — nema petlju. Operator neprestano konvergira (obrišeš Service →
vrati se; CNPG rotira lozinku → watch reaguje), nosi domensku logiku (čekanje Secreta, redosled,
availability→replike), radi cleanup eksternih sistema kroz finalizere i izlaže domenski API
(`kubectl get shops` sa statusom). ShopHub bi sa helm pristupom morao da bude "CD server" — ovako
je samo tanki API koji piše CR-ove.

---

## 17. Demo scenario

Redosled koji vežbam (15-ak minuta), sa fallback planovima:

1. **IaC prvo** (eliminacioni!): pokažem helm-charts strukturu, kube-state strukturu, OCI chart-ove
   na DockerHub-u (`helm pull oci://...`).
2. **ArgoCD UI** — sve Synced/Healthy; usput objasnim app-of-apps root.
3. `kubectl get pods -A` — operator, cnpg, mongodb-operator, monitoring, shophub.
4. **ShopHub UI**: registracija novog korisnika → pokažem da je nastao `tenant-*` namespace.
5. Kreiranje prodavnice (high availability, postgres, Discord opcija) → uživo:
   `kubectl get shops -n tenant-... -w`, `kubectl describe shop` (Conditions:
   DatabaseProvisioning → Deploying → Ready), `kubectl get all -n tenant-...`.
6. Klik na URL → storefront; admin login (lozinka iz dashboarda) → dodam artikal.
7. **Kupovina**: korpa → MetaMask (Sepolia, TestUSDT) → potvrda → order pending → confirmed →
   stock smanjen. Pokažem tx na Etherscan-u. *(Fallback ako RPC/mreža zeza: unapred potvrđena
   transakcija + snimak.)*
8. **Observability**: Grafana kao admin (Main Org — dashboardi svih prodavnica po folderima), pa
   logout → login kao tenant Viewer (vidi SAMO svoj org/dashboard). Loki logovi, Tempo trace
   zahteva.
9. **Alarm**: 404 flood petlja → ShopHighErrorRate firing → Discord poruka u kanalu prodavnice →
   prekinem → resolved poruka.
10. **Skaliranje**: `kubectl scale shop demo --replicas=4` → 4/4.
11. **Brisanje** iz UI → sve nestaje (`kubectl get all -n tenant-...`), Discord kanal obrisan
    (finalizer priča).
12. Ako traže CI: pokažem zeleni PR sa checks + DockerHub tagove.

Pravila za sebe: svaku komandu pratim jednom rečenicom "šta očekujem da vidimo"; ako nešto pukne,
ne panika — debug uživo je često jači utisak (describe → events → logs).

---

## 18. Cheat sheet

**Verzije/alati:** Go 1.26 · Kubebuilder 4.x (multigroup) · k3d (1 server + 2 agenta) ·
kube-prometheus-stack 85.3.3 · Helm OCI na DockerHub.

**Imena:** GitHub org `shophub-devoops` · DockerHub `urospetraskovic` · operator image
`shop-operator-controller` (≠ ime chart-a, zbog OCI kolizije) · unified shop image `shop-backend` ·
domen CRD-ova `shophub.local`.

**Portovi:** Shop/ShopHub backend u klasteru **8080** · k3d host mapping **8080→80, 8443→443** ·
operator health **8081**, metrics **8443** · Grafana `grafana.localhost:8080` · Tempo OTLP **4318**.

**Secreti (ključevi):** `<shop>-app` (`uri` / `connectionString.standard`) · `<shop>-admin`
(`password`) · `<shop>-mongo-pw` · `<channel>-webhook` (`webhook-url`) · `<wallet>-key`
(`private-key`, `address`) · `grafana-admin` · `discord-bot-token` (`token`).

**Web3:** Sepolia testnet · TestUSDT ERC-20 `0x74b0ef...7ff6f`, **6 decimala**, otvoren mint
(faucet) · sweep 15s · pending TTL 30 min · Transfer topic = keccak256
`Transfer(address,address,uint256)`.

**Replike:** standard=2, high=3, `spec.replicas` override pobedjuje.

**Conditions:** Available / Progressing / Degraded; Reasons: Ready, Deploying,
DatabaseProvisioning, DatabaseFailed.

**Alarmi:** ShopHighErrorRate (>10% 4xx/5xx, 5m) · ShopHighLatency (p95>1s) · NodeHighCPU/Memory
(>90%).

**Eliminaciono:** helm-charts (3 charta, crds/ folder) + kube-state (clusters/local, helm.yaml +
values.yaml po komponenti) — struktura 1:1 sa spec-om 5.3.

---
---

# DODATAK A — Šira teorijska pitanja (van samog projekta)

Asistenti vole da provere i opšte razumevanje alata, ne samo projekat. Ovo su pitanja koja mogu da
padnu "usput".

## A.1 Kubernetes internals

**P: Koje komponente čine Kubernetes control plane?**
O: **API server** (jedina ulazna tačka, jedini piše u etcd), **etcd** (distribuirani key-value
store, izvor istine), **scheduler** (bira nod za pod na osnovu requests, affinity, taints),
**controller-manager** (ugrađeni kontroleri: deployment, replicaset, namespace, GC...). Na svakom
nodu: **kubelet** (pokreće podove, javlja status, izvršava probe) i **kube-proxy** (Service
routing). Naš operator je konceptualno ista stvar kao controller-manager, samo za naše CRD-ove.

**P: Deployment vs StatefulSet — i zašto Shop koristi Deployment iako ima bazu?**
O: Deployment = stateless replike, zamenjive, random imena, ne dele ništa. StatefulSet = stabilan
identitet (pod-0, pod-1), stabilan storage po replici (volumeClaimTemplates), uređen start/stop —
za baze i klastere sa identitetom. Shop **backend** je stateless (sve stanje je u bazi) → Deployment.
Baza sama jeste stateful, ali nju ne vodimo mi nego CNPG/MongoDB operator — koji ispod haube
upravlja svojim podovima/PVC-ovima (CNPG čak namerno koristi svoje podove umesto golog StatefulSet-a
da bi imao punu kontrolu nad failover-om). To je i odgovor na "zašto ne pišemo svoj DB operator".

**P: Šta je HPA i da li bi radio sa vašim CRD-om?**
O: HorizontalPodAutoscaler — skalira broj replika po metrici (CPU, custom). Da — baš zato smo
implementirali **scale subresource** na Shop-u: HPA ne mora da zna šta je Shop, dovoljno mu je
standardni `/scale` endpoint (specpath/statuspath/selectorpath). HPA bi pisao `spec.replicas`,
operator ga poštuje jer override pobeđuje availability default.

**P: Taints/tolerations, affinity — koristite li?**
O: Ne aktivno (k3d demo klaster, 3 noda, nema potrebe), ali znam čemu služe: taint odbija podove sa
noda osim ako pod ima toleration (npr. control-plane nod); affinity/anti-affinity privlači/razdvaja
podove (npr. anti-affinity da 3 replike high shopa ne završe na istom nodu — to bi bio prirodan
sledeći korak za pravi HA, i chart operatora već ima affinity value placeholder).

**P: Šta je PV/PVC?**
O: PersistentVolume = komad storage-a u klasteru; PersistentVolumeClaim = zahtev poda za storage.
Kod nas ih ne pišemo ručno — CNPG iz `storage.size: 1Gi` sam pravi PVC po instanci; k3d ima
local-path provisioner koji dinamički veže PV. Brisanje CNPG Cluster-a briše i PVC (owned).

**P: Kako radi DNS u klasteru?**
O: CoreDNS. Svaki Service dobija zapis `<svc>.<ns>.svc.cluster.local`. Zato CNPG connection string
sadrži host tipa `<cluster>-rw.<ns>.svc` i radi iz bilo kog poda, a naš operator env-om samo
prosledi gotov URI.

## A.2 Docker dublje

**P: Šta je image, a šta kontejner? Šta su layeri?**
O: Image = read-only šablon složen iz layera (svaka Dockerfile instrukcija = layer, adresiran
hash-em, deli se između slika). Kontejner = pokrenuta instanca sa tankim writable layerom preko
image-a. Layer keširanje je razlog za redosled u Dockerfile-u: ono što se retko menja (deps) ide
pre onoga što se često menja (source) — pa se rebuild svodi na poslednje layere.

**P: CMD vs ENTRYPOINT?**
O: ENTRYPOINT = fiksni izvršni program; CMD = default argumenti (ili default komanda ako
ENTRYPOINT-a nema); `docker run img args` pregazi CMD, ne ENTRYPOINT. Mi koristimo
`ENTRYPOINT ["/app/shophub"]` exec formu — binarni je PID 1, prima SIGTERM, graceful shutdown radi.

**P: Zašto se tajne ne smeju prosleđivati kao build ARG?**
O: ARG vrednosti ostaju vidljive u image istoriji/metadata (`docker history`, `image inspect`);
brisanje fajla u kasnijem layeru ne pomaže jer stariji layer i dalje postoji (alati tipa skopeo ga
izvuku). Ispravno: `RUN --mount=type=secret` — tajna je dostupna samo tokom tog RUN-a i ne završi
ni u jednom layeru. Nama u build-u tajne nisu ni potrebne (sve tajne su runtime, kroz K8s Secrete)
— što je i najčistije.

**P: Šta je .dockerignore i zašto je bitan?**
O: Isključuje fajlove iz build contexta (node_modules, .git, dist...) → manji context (brži build),
manja šansa da tajna/smeće uleti u sliku, bolje keširanje (izmena README-a ne invalidira COPY).

## A.3 Git dublje

**P: merge vs rebase vs squash?**
O: Merge spaja grane merge commit-om (čuva topologiju, pravi "špagete"); rebase prepiše commit-e
preko novog osnova (linearno, ali menja SHA — ne raditi na deljenim granama); squash merge (naš
izbor) spljošti ceo PR u jedan commit na main — linearna istorija, atomski revert. Trade-off koji
priznajem: gubi se granularnost commit-a unutar PR-a — zato PR-ovi treba da budu mali.

**P: Zašto Conventional Commits — šta konkretno dobijate?**
O: Mašinski čitljive poruke (`feat(scope): ...`, `fix:`, `ci:`): automatska validacija
(commitlint na svakom PR-u), mogućnost automatskog changelog-a i SemVer bump-a (feat→minor,
fix→patch, breaking→major), i čitljiva istorija. Na svih 5 repo-a.

**P: Šta je branch protection tačno sprečio kod vas?**
O: Direktan push na main (sve kroz PR), merge bez approvala, merge sa crvenim CI-jem, ne-linearnu
istoriju. Praktično: bug koji obori testove fizički ne može u main. Plus caka koju smo naučili:
required check se vezuje po **imenu job-a** i mora da postoji u tom repou — ako zahtevaš check
workflow-a koji se na tom repou ne okida (npr. paths-filtered), PR-ovi ostanu zauvek blokirani.

## A.4 Go pitanja (jer je ceo backend Go)

**P: Zašto Go za sve tri komponente?**
O: Operator praktično mora Go (kubebuilder/controller-runtime ekosistem); backend-i Go radi
deljenja tipova — ShopHub **importuje `api/` pakete operatora** i radi typed CRUD nad Shop CR-ovima
(nema ručnog YAML-a/unstructured mapa); go-ethereum za verifikaciju plaćanja; jedan statički binarni
u slici od par desetina MB; i iskreno — jedan jezik = manje konteksta za tim.

**P: Goroutine u sweep verifikatoru — kako se gasi čisto?**
O: `PaymentVerifier.Run(ctx)` — petlja na `time.Ticker` sa `select` na `ctx.Done()`. Main na
SIGTERM otkaže context → sweep izađe, HTTP server radi graceful `Shutdown`. To je standardni Go
context lifecycle pattern; vezuje se na priču o exec formi i PID 1 (signal mora da stigne).

**P: Zašto big.Int/big.Rat za novac umesto float64?**
O: float64 ne može tačno da predstavi decimalne iznose (0.1+0.2≠0.3) — kod novca to je korupcija
podataka. On-chain iznosi su 256-bitni celi brojevi (ne staju ni u int64 u opštem slučaju) → big.Int
za base units, big.Rat za konverziju decimalnog stringa, Postgres `numeric`, JSON stringovi.

## A.5 Security pitanja

**P: bcrypt vs SHA-256 za lozinke?**
O: SHA je brz — napadač proba milijarde kandidata u sekundi na GPU. bcrypt je **namerno spor** i
podesive cene, sa ugrađenim salt-om po lozinki (duplirane lozinke → različiti hash-evi, rainbow
tabele beskorisne). Zato ShopHub lozinke: bcrypt. (A Grafana tenant lozinka: enkripcija, ne hash —
jer mora da se prikaže korisniku; razlika: hash je jednosmeran, enkripcija reverzibilna ključem.)

**P: Šta štiti JWT od falsifikovanja? Šta su mu slabosti?**
O: HMAC-SHA256 potpis sa server-side tajnom — izmena payload-a (npr. `ns` claim) invalidira potpis.
Slabosti koje aktivno adresiramo: alg-swap (`alg:none`/RS↔HS zamena) — middleware eksplicitno
zahteva HMAC familiju; curenje tajne — tajna je u K8s Secret-u, lookup-preserve u chartu; TTL 24h
ograničava prozor zloupotrebe ukradenog tokena. Poznata slabost JWT-a uopšte: nema server-side
revokacije pre isteka (trebala bi blacklist/kratki TTL + refresh — van obima, znam za nju).

**P: Gde bi napadač prvo gledao u vašem sistemu?**
O: Iskren odgovor koji pokazuje razmišljanje: (1) ShopHub API — zato je svaki endpoint iza JWT
middleware-a i namespace-scoped; (2) admin endpoints prodavnice — iza admin gate-a; (3) payment
flow — zato server računa iznos, verifikuje on-chain i brani replay sumiranjem po tx hash-u;
(4) tajne u git-u — zato ih tamo nema (proveravano i kroz istoriju); (5) kontejneri — non-root,
read-only binarni, minimalna baza. Šta bih dodao: NetworkPolicy, rate limiting šire (postoji na
auth), image scanning (trivy) u CI.

---

# DODATAK B — Pitanja o procesu rada

Na odbrani skoro sigurno padne i "kako ste radili", ne samo "šta ste napravili".

**P: Kojim redosledom ste gradili projekat i zašto?**
O: Faza 0 je bila **eliminacioni zahtev** — svih 5 repo-a sa branch protection-om i strukturom
helm-charts/kube-state, pre ijedne linije koda (ako to ne valja, projekat se i ne pregleda — nema
smisla rizikovati). Zatim temelji: k3d klaster + CNPG + MongoDB operator; pa operator (najveći
komad — CRD-ovi pa reconcileri jedan po jedan); pa Shop backend + frontend; ShopHub sa auth-om i
multi-tenancy; Web3 plaćanje; observability; i **CI/CD smo digli rano** a ne na kraju — na vežbama
je eksplicitno rečeno da pipeline treba što pre, i isplatilo se: svaki naredni PR je već bio
validiran.

**P: Šta je bilo najteže?**
O: Iskren, konkretan odgovor: (1) MongoDB operator integracija — tri odvojena problema
(GetOwnerReferences override, watchNamespace, SA materijalizacija) od kojih nijedan nije dokumentovan
na jednom mestu; (2) Alertmanager → Discord sa webhook-om iz Secreta — prometheus-operator odbija
webhook_url_file pa se do AlertmanagerConfig.apiURL rešenja došlo probavanjem; (3) prvi put
razumeti zašto se reconcile "vrti" — dok nisam skapirao no-op update pravilo i jednu-izmenu-po-
iteraciji, imao sam beskonačnu petlju. Svaka od ovih priča je dobra jer ima problem → dijagnozu →
rešenje → zapis u decision log.

**P: Šta biste drugačije uradili?**
O: (1) envtest/e2e testove operatora pisao bih paralelno sa reconcilerima, ne posle — par bugova
(port 80→8080, Service labela) bi se uhvatilo pre demo-a; (2) uveo bih NetworkPolicy između tenant
namespace-a od starta; (3) dashboard JSON bih generisao iz koda umesto održavanja embedded JSON-a;
(4) razmislio bih o webhook validaciji CRD-a (admission webhook) za stvari koje marker validacija
ne pokriva (npr. format wallet adrese).

**P: Kako ste donosili odluke kad spec ne pokriva nešto?**
O: Spec sam kaže da se takvi slučajevi rešavaju kako studentima odgovara — ali smo svaku takvu
odluku **zapisali u decision log** (datum, odluka, razlog): MongoDB umesto Redis-a, multigroup
layout, serviceMonitorSelectorNilUsesHelmValues, pin Mongo connection Secreta na `-app` ime, unified
image, per-tenant Grafana org umesto folder permisija... Na odbrani mogu da otvorim taj log — to je
možda najjači dokaz da je projekat rađen svesno.

**P: Kako je izgledao vaš dev loop za operator?**
O: Kod u WSL Ubuntu (kubebuilder/make/go žive tamo), klaster k3d na Windows Docker Desktop-u,
kubeconfig deljen (`KUBECONFIG=/mnt/c/...`). Brza iteracija: `make manifests generate` →
`make install` (CRD-ovi) → `make run` (operator lokalno, admin kubeconfig) → apply sample CR →
gledaj logove. Kad logika stane: build image → push/import → helm upgrade operator chart-a →
provera **in-cluster** (jer make run maskira RBAC greške). Za odbranu sve ide iz OCI chart-a kao
prava instalacija.

**P: Koliko je šta trajalo / gde je otišlo najviše vremena?**
O: Grubo: operator ~40% vremena (od toga trećina na integracije sa tuđim operatorima), observability
~20% (najviše Alertmanager/Grafana fino podešavanje), aplikacije ~20%, Web3 ~10% (manje nego što
sam očekivao — ERC-20 je jednostavan standard), CI/CD + IaC ~10% (ali raspoređeno kroz ceo projekat).

---

# DODATAK C — Komande koje kucam na odbrani (spremne, bez razmišljanja)

```bash
# --- stanje klastera ---
kubectl get nodes
kubectl get pods -A
kubectl get crds | grep shophub.local

# --- shopovi ---
kubectl get shops -A                          # print kolone: TITLE, DB, READY, URL
kubectl get sh -A                             # shortName radi
kubectl describe shop <ime> -n <tenant-ns>    # Conditions + Events
kubectl get all,secrets,ingress -n <tenant-ns>
kubectl get shop <ime> -n <ns> -o jsonpath='{.status.conditions}' | jq

# --- operator ---
kubectl -n shop-operator-system logs deploy/shop-operator -f
kubectl scale shop <ime> -n <ns> --replicas=4        # scale subresource demo

# --- baza ---
kubectl -n <ns> get clusters.postgresql.cnpg.io      # CNPG
kubectl -n <ns> get mongodbcommunity                 # Mongo
kubectl -n <ns> get secret <shop>-app -o jsonpath='{.data.uri}' | base64 -d

# --- observability ---
kubectl get servicemonitors -A
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090
#   → Status > Targets: shop target UP
# Grafana: http://grafana.localhost:8080  (admin i tenant login odvojeno)

# --- alarm demo (PowerShell varijanta) ---
while ($true) { curl -s http://<shop>.localhost:8080/api/nepostojeci | Out-Null }

# --- GC demo ---
kubectl delete shop <ime> -n <ns>
kubectl get all -n <ns> -w                    # gledaj kako sve nestaje

# --- helm/OCI dokazi (eliminacioni) ---
helm pull oci://registry-1.docker.io/urospetraskovic/shop-operator --version <v>
helm lint charts/shop-operator                # u helm-charts repou
helm template demo charts/shop --set name=proba

# --- argocd ---
kubectl -n argocd get applications
kubectl -n argocd port-forward svc/argocd-server 8081:443
```

Mini PromQL podsetnik (za Grafana Explore uživo):

```promql
sum(increase(shop_http_requests_total[24h]))                          # ukupno 24h
sum(increase(shop_http_requests_total{status=~"2..|3.."}[24h]))       # uspešni
sum by (route) (increase(shop_http_requests_total{status="404"}[24h]))# 404 po endpointu
sum(increase(shop_http_response_bytes_total[24h])) / 1e9              # GB
histogram_quantile(0.95, sum by (le) (rate(shop_http_request_duration_seconds_bucket[5m])))
```

---

# DODATAK D — Rečnik (jednom rečenicom, za brze odgovore)

- **CRD** — definicija novog tipa resursa na API serveru (šema + endpoint); **CR** je instanca.
- **Reconcile petlja** — funkcija koja trenutno stanje dovodi u željeno; stateless, idempotentna,
  level-based.
- **Informer** — klijentska mašinerija (Reflector + Cache + WorkQueue) koja watch evente pretvara u
  keš i redosled poslova.
- **Predicate** — filter eventova pre keša; **EventHandler** — mapiranje event → reconcile zahtevi.
- **FieldIndexer** — sekundarni indeks nad kešom za O(1) List po polju.
- **OwnerReference** — "ovo pripada onome"; osnova za kaskadni Garbage Collection.
- **Finalizer** — marker koji zadržava brisanje dok kontroler ne počisti eksterno stanje.
- **Subresource** — pod-endpoint objekta (`/status`, `/scale`, `/finalizers`) sa odvojenim RBAC-om.
- **Leader election** — više replika operatora, samo jedna radi.
- **ServiceMonitor** — CRD prometheus-operatora: deklaracija "šta scrape-ovati".
- **PrometheusRule** — CRD sa alert/recording pravilima.
- **AlertmanagerConfig** — CRD za per-namespace rutiranje alarma (naš most ka Discord-u).
- **OCI chart** — Helm chart kao artefakt u container registriju (isti registri kao image-i).
- **App-of-apps** — ArgoCD Application koji pokazuje na folder drugih Application-a; bootstrap
  jednim manifestom.
- **TBD (Trunk Based Development)** — kratkoživeće grane + jedan trunk (main); suprotno GitFlow-u.
- **SemVer** — MAJOR.MINOR.PATCH; breaking/feature/fix.
- **ERC-20** — standardni interfejs za tokene na Ethereumu (transfer, balanceOf, decimals, Transfer
  event).
- **Receipt** — potvrda izvršene transakcije sa statusom i event logovima; jedini pouzdan dokaz
  ERC-20 uplate.
- **Testcontainers** — biblioteka koja u testu podiže prave zavisnosti kao docker kontejnere.
- **bcrypt** — spor, salted hash za lozinke; **JWT** — potpisan token sa claims; **HS256** — HMAC
  potpis simetričnim ključem.

---

*Kraj priprema. Redosled učenja: sekcije 5 (operator) i 10 (observability) su najverovatnije i
najteže — njih prvo; pa 8 (Web3) i 12 (IaC — eliminacioni, mora se znati bez zastajkivanja); ostalo
je nadgradnja. Demo (17) provežbati bar dvaput end-to-end.*
