# Asistentove vežbe — municija za odbranu

> Izvor: moje beleške sa vežbi 3, 4 i 5 (`planning/vezbe pomocni materijali/vezbe 3/4/5.odt`).
> Ovo NIJE ponavljanje gradiva — to pokriva `odbrana.md`. Ovo je spisak stvari koje je **on lično
> naglašavao na vežbama**, prepričan tako da mogu prirodno da ih ubacim u odgovore. Poenta: kad
> odgovor začinim detaljem koji je on izgovorio na vežbama, on vidi da sam sedeo, slušao i povezao
> — to je razlika između "naučio sam za ispit" i "pratio sam te ceo semestar".
>
> **Kako se ovo koristi:** ne recituješ. Čekaš pitanje iz te oblasti, odgovoriš normalno, pa dodaš
> "…to ste i na vežbama naglasili kad ste pokazivali X" + jedna njegova rečenica + gde je to kod
> nas u projektu. Jedan takav momenat po oblasti je dovoljan; tri u istoj rečenici zvuči kao ulizivanje.

---

## Sadržaj

1. [Vežbe 3 — Docker](#vezbe-3--docker)
2. [Vežbe 4 — Operator, reconciler, conditions](#vezbe-4--operator)
3. [Vežbe 5 — GC, subresursi, CNPG, watches, Helm, finalizeri](#vezbe-5)
4. [Njegove fore i rečenice koje vredi znati](#njegove-fore)
5. [Brza mapa: njegova priča → naš projekat](#brza-mapa)

---

## Vežbe 3 — Docker

### 1. Multi-stage: "mora build faza, i sve što ide u build mora da se razdvoji"

**Šta je pričao:** Dockerfile mora imati odvojene faze — build stage sa svim alatima, release stage
samo sa onim što se izvršava. I tu je razvio celu priču o **kompajliranim vs interpretiranim
jezicima**: kod kompajliranih dobiješ jedan binarni fajl i samo njega nosiš dalje; kod Python/Node
moraš da vučeš ceo source, node_modules, definition fajlove — i **sve to moraš da zaštitiš**, jer
ako neko u kontejneru sme da menja neki JS fajl, može da ubaci malicioznu skriptu. Kod binarnih
štitiš jedan fajl. Njegov zaključak: *"što se tiče security-ja, Python i Node nisu ništa bolji od
kompajliranih — naprotiv."*

**Kako ja to prodajem:**
> "Na vežbama ste naglasili da su kompajlirani jezici u prednosti kod kontejnerizacije jer se štiti
> jedan binarni fajl umesto celog source stabla — to je bio jedan od razloga što je ceo naš backend
> u Go-u. Release slika ima samo statički binary, bez kompajlera i bez koda."

**Kod nas:** svi Dockerfile-ovi su multi-stage (node → go → alpine release), `CGO_ENABLED=0`
statički binary, `-ldflags '-s -w'`.

### 2. Non-root + permisije: "root je vlasnik, app korisnik samo izvršava"

**Šta je pričao:** binary treba da bude **owned od root-a**, pa se doda novi korisnik (`app`) koji
sme samo da ga **izvršava, ne i da ga menja ili briše** — i kontejner se **nikad ne pokreće kao
root, pogotovo ne u produkciji**. Znači nije samo "USER app" nego i vlasništvo fajlova: app ne sme
da može da prepiše sopstvenu aplikaciju.

**Kako ja to prodajem:**
> "Ispoštovali smo tačno onu šemu sa vežbi: `chown root:root` na binarnom + `chmod 0755`, pa tek
> onda `USER app` — app korisnik može read+execute, ne može write. Da neko i upadne u kontejner,
> ne može da zameni binary."

**Kod nas:** identično u sva tri Dockerfile-a (`addgroup -S app && adduser -S app`, chown root,
chmod 0755, USER app).

### 3. Slim vs Alpine: "ako alpine proradi iz prve — super; ako moraš da budžiš, uzmi slim"

**Šta je pričao:** uvek slim/alpine varijante baznih slika — manje dependency-ja = manje
vulnerabilities, manja slika, brži start. Ali je bio iskren oko Alpine-a: ponekad na njemu moraš da
instaliraš gomilu paketa pa se prednost istopi — pragmatično pravilo je "šta radi iz prve".

**Kako ja to prodajem:**
> "Držali smo se vaše preporuke — alpine baze svuda, i u našem slučaju je prošlo iz prve jer je Go
> binary statički pa mu od OS-a treba praktično samo ca-certificates."

### 4. Shell vs exec forma: "root procesnog stabla je shell — i signal ne stigne do aplikacije"

**Šta je pričao:** razlika CMD/ENTRYPOINT i shell/exec varijante. ENTRYPOINT je osnova, CMD je
sufiks/argumenti. Kod shell forme je root procesa shell, pa Ctrl+C / stop signal ne ide aplikaciji
— testiraš tako što gledaš da li kontejner umire odmah ili tek posle timeout-a.

**Kako ja to prodajem:**
> "Exec forma svuda — pominjali ste da se shell forma pozna po tome što `docker stop` visi ~10s pa
> ubije SIGKILL-om. Kod nas je binary PID 1, hvata SIGTERM i radi graceful shutdown — to nam je
> bitno i zbog payment sweep-a, da se lepo ugasi usred posla."

### 5. Clean build context: "jako je loša praksa da staviš tačku kao build kontekst"

**Šta je pričao:** ne gurati ceo repo kao kontekst — poseban folder ili uredan `.dockerignore`;
bolje keširanje, manje smeća. Usput je pričao i da docker klijent-server arhitektura znači da build
može da se radi i na udaljenom serveru (primer sa AWS instancom i GPU driverima).

**Kod nas:** `.dockerignore` u svim repoima, COPY po fazama (prvo go.mod/go.sum pa source) baš zbog
keš slojeva.

### 6. Hadolint: "alat koji skenira Dockerfile i ne pušta te da merguješ dok ne rešiš"

**Šta je pričao:** postoji linter za Dockerfile koji se veže u CI i blokira merge dok problemi ne
nestanu.

**Kako ja to prodajem:**
> "To smo bukvalno implementirali — hadolint je korak u docker-build workflow-u koji je required
> check na PR-u, znači Dockerfile regresija fizički ne može u main."

### 7. JDK vs JRE

**Šta je pričao:** JDK = development kit (kompajleri, alati — za build), JRE = samo runtime (za
izvršavanje). Build stage uzima jdk sliku, release jre.

**Kako ja to prodajem:** to je isti princip kao naš golang builder → alpine release: build okruženje
i runtime okruženje se razdvajaju, release nosi minimum. Ako pomene Javu, znam paralelu.

---

## Vežbe 4 — Operator

Ovo su vežbe gde je pisao Dojo operator uživo — naš Shop operator je rađen tačno po tom šablonu,
pa je svaka njegova rečenica odavde direktno primenljiva.

### 1. "Možemo i ručno da pišemo CRD YAML, ali što bismo"

**Šta je pričao:** CRD je OpenAPI šema registrovana na API serveru; kubebuilder markerima („onim
notacijama") definišeš pravila u Go strukturi i `make manifests generate` izgeneriše CRD sa
validacijom. Ručno pisanje = puno napora i puno prostora za grešku.

**Kako ja to prodajem:**
> "Sve tri naše CRD šeme su generisane iz Go tipova markerima — enum validacija za
> availability/database, defaulti, print kolone. Ručno nismo pisali ni liniju CRD YAML-a, po onom
> vašem 'možemo od nule, ali što bismo'."

### 2. RBAC iz notacija: "reconciler dobija kompletan set, a mi možemo da skinemo šta ne treba"

**Šta je pričao:** kubebuilder auto-generiše RBAC markere za naš resurs (get/list/watch/create/…),
a čim operator počne da dira **tuđe** resurse (deployments, secrets, CNPG), moramo eksplicitno da
dodamo markere — inače 403. I napomena: `make run` radi sa tvojim adminskim kubeconfig-om pa
maskira RBAC rupe; tek in-cluster deploy sa ServiceAccount-om pokaže istinu.

**Kako ja to prodajem:**
> "Imali smo tačno taj slučaj sa vežbi: operator lokalno kroz `make run` sve može, a kad se
> deploy-uje kao pod, radi samo ono što mu markeri daju. Zato naš kontroler ima ceo blok RBAC
> markera za deployments, services, secrets, CNPG clusters, mongodbcommunity, servicemonitors,
> alertmanagerconfigs — svaki je dodat onda kad nam je 403 rekao da fali."

### 3. Stateless reconciler: "šta ako imaš 2 replike operatora? 1000 bugova"

**Šta je pričao:** reconciler **ne sme da pamti stanje u varijablama** između iteracija — sa dve
replike operatora to je katastrofa. Sve što treba da se pamti ide u **Status objekat**: pročitaš
spec, pročitaš status, inkrementalno (kroz više prolaza petlje) dovodiš stanje. I odatle
idempotentnost: `f(f(x)) = f(x)` — kao množenje gde samo 0 i 1 zadovoljavaju A·A=A. Duplirani event
ne sme da napravi dupli efekat.

**Kako ja to prodajem:**
> "Zapamtio sam onu matematičku definiciju sa vežbi — f(f(x)) = f(x). Naš reconciler je čist
> stateless: sve ensure funkcije prvo čitaju postojeće stanje, a checkpoint je Status. Najbolji
> primer je DiscordChannel: channelID upišemo u Status pre nego što idemo dalje, pa retry ne pravi
> drugi kanal na serveru."

### 4. Level-based vs edge-based: "od 5 izmena nas zanima samo poslednja"

**Šta je pričao:** Kubernetes je level-based — ako se objekat promenio 5 puta dok petlja radi, work
queue deduplikuje po `namespace/name` i reconciler se okine jednom, za poslednje stanje. Ne
reagujemo na svaki event (edge), nego na razliku trenutno vs željeno.

**Kako ja to prodajem:**
> "Zato naš operator sme da bude ugašen pa upaljen — ne zavisi od eventova nego od stanja; kad se
> vrati, resync ga dovede u red. Level-based, kako ste crtali."

### 5. Conditions: Available / Progressing / Degraded — "da se ne bismo patili na odbrani"

**Šta je pričao:** konvencija su tri conditiona; Degraded i Progressing su međusobno isključivi,
Available nije (može available+progressing u isto vreme). Bukvalno je rekao da statusi treba da
šalju dobre poruke *"da se ne bismo patili na odbrani"*. I ključna nijansa: **conditions su javni
API operatora** — drugi alati (pominjao je ArgoCD koji gleda progressing/available pa tek onda
deploy-uje dalje) se integrišu na njih, pa mora backward/forward kompatibilnost. Preporučio je
pomoćne metode: set-unrecoverable-status i set-progress-status.

**Kako ja to prodajem:**
> "Implementirali smo tačno tu trojku sa pomoćnim helperima kao na vežbama, sa Reason taksonomijom
> (Ready/Deploying/DatabaseProvisioning/DatabaseFailed) — i pošto vozimo ArgoCD, ta priča da drugi
> alati čitaju naše conditione kod nas nije teorija nego stvarnost: `kubectl describe shop` na
> odbrani pokazuje ceo životni ciklus kroz LastTransitionTime."

### 6. Stalled vs Failed: "stalled je kad si TI zabrljao konfiguraciju, failed je kad klaster nije mogao"

**Šta je pričao:** kod nepopravljivih grešaka Reason pravi razliku: **Stalled** = greška u
konfiguraciji koju je korisnik uneo (npr. string umesto broja replika) — ne rešava se dok korisnik
ne ispravi spec; **Failed** = klaster nije uspeo da kreira resurs — nije do tebe. Poruka mora da
kaže korisniku kome da se žali.

**Kako ja to prodajem:**
> "Naš DatabaseFailed reason je upravo ta kategorija 'failed' sa vežbi — operator je sve uradio
> kako treba a provisioning baze pukao; a validacione greške tipa 'stalled' kod nas uglavnom ne
> stignu do reconcilera jer ih enum markeri odbiju već na API serveru."

### 7. 409 Conflict: "u bilo kom trenutku može da nam uleti — i to će nas najviše nervirati"

**Šta je pričao:** svaki objekat ima resourceVersion; update sa starom verzijom iz keša → 409.
Zato: **jedna izmena po iteraciji**, posle svakog update-a proveri `IsConflict` — konflikt nije
greška nego znak da čekaš svežu verziju i pustiš petlju da se ponovo okine. Rekao je otvoreno da je
to deo koji najviše nervira i da je jako error-prone, pa treba dobro testirati.

**Kako ja to prodajem:**
> "Ta rečenica se obistinila — dok nisam ispoštovao 'jedna izmena po iteraciji' imao sam petlju
> koja se vrti. Sad svaki naš status update guta IsConflict, a na vrhu Reconcile imamo wrapper koji
> 409 pretvara u čist requeue, jer je to očekivano stanje kad i deployment kontroler i CNPG pišu po
> istim objektima."

### 8. Happy path kroz iteracije — tabla

**Šta je pričao:** crtao je na tabli kako se do Available=True dolazi kroz ~4-5 iteracija: (1)
postavi Progressing/Init, (2) kreiraj Deployment + ownership, Reason=Creating, (3) Deployment
postoji ali replike nisu spremne — uporedi spec vs stvarno, (4) ready se poklopi → Available=True,
(5) izmena replika → mismatch → update → nazad na (3). Svaka iteracija = jedna izmena, i izlaz.

**Kako ja to prodajem:**
> "Naš Shop prolazi istu sekvencu sa table, samo sa više koraka jer čekamo i bazu: prvo
> DatabaseProvisioning dok CNPG ne objavi secret, pa Deploying, pa Ready. Mogu uživo da pustim
> `kubectl get shop -w` da se vide ti prelazi."

### 9. "On nije puno filozofirao — ukrao je postojeće koncepte iz Kubernetesa"

**Šta je pričao:** status svog Dojo resursa je prepisao od Deployment statusa (readyReplicas,
updatedReplicas...) — ne izmišljaj svoje konvencije kad Kubernetes već ima ustaljene.

**Kako ja to prodajem:**
> "Isto smo uradili: Shop.Status.ReadyReplicas je ogledalo Deployment statusa, conditions su
> standardna trojka — namerno ništa originalno, da svako ko zna Kubernetes odmah razume naš API."

### 10. Arhitektura: "jedino API server sme da dira bazu"

**Šta je pričao:** API server je jedini koji piše u etcd; kubeleti, scheduleri, controller manageri,
operatori — svi slušaju event bus i **keširaju kod sebe** da ne spamuju API server. U operatoru:
reflektor gleda evente → cache drži kompletan objekat → work queue drži samo namespace/name (dedup)
→ reconciler. **Get čita iz keša, update ide direktno na API server** — i zato stižu 409-ke. I
pazi da ne uvedeš cirkularnu petlju (update koji okida samog sebe).

**Kako ja to prodajem:** ovo je dijagram koji je na vežbama 5 rekao da se zna "usred noći" — crtam
ga bez pitanja čim padne bilo šta o kešu, watchevima ili konfliktu.

---

## Vežbe 5

### 1. Graceful shutdown: "sleep infinity → 10 sekundi do KILL-a"

**Šta je pričao:** proces koji ne obrađuje SIGTERM čeka da istekne timeout pa ga ubije SIGKILL —
zato brisanje traje. Pravilno: uhvati TERM, završi biznis logiku, vrati greške, zatvori konekcije,
izađi.

**Kod nas:** ShopHub i Shop backend rade `http.Server.Shutdown` na SIGTERM; payment sweep živi na
context-u koji se cancel-uje. Vezuje se na exec formu iz vežbi 3 — signal uopšte stigne do procesa
samo zato što je binary PID 1.

### 2. OwnerReferences + blockOwnerDeletion: "tu može da dođe do deadlocka"

**Šta je pričao:** tri vrednosti kontrolišu GC: owner (apiVersion/kind/name/uid), `controller: true`
(samo jedan!), `blockOwnerDeletion`. Brisanje ide kao dependency chain — dok se child ne obriše,
parent visi u Terminating. I upozorenje: ako više objekata sa blockOwnerDeletion pokazuje na isto,
**deadlock** — zato 1-na-1 veze; za 1-na-više koristi labele i sam sprovedi logiku.

**Kako ja to prodajem:**
> "Kod nas je svaki objekat owned od tačno jednog CR-a pa smo bezbedni, ali znam za taj deadlock
> scenario koji ste pominjali — to je i razlog što webhook Secret ne vezujemo owner referencama na
> više Shop-ova nego Shop-ovi referenciraju Secret kroz spec, a mi ih nalazimo indeksom."

### 3. Scale subresource: "veliki deo onoga što menjaš u YAML-u ima svoju kubectl komandu"

**Šta je pričao:** REST analogija — resurs i pod-resurs (`booking` vs `booking/bed_number`): ne
vučeš ceo objekat da bi promenio jedno polje. Kubernetes CRD ima ugrađene subresurse:
`/status`, `/scale`, `/finalizers` (+ /log, /exec…). **Zato** reconciler ne može da menja spec —
`r.Status().Update()` bukvalno gađa drugi endpoint (`dojo/status`), a RBAC je odvojen po
subresursu; PUT na spec iz reconcilera je namerno onemogućen. Za scale trebaju tri putanje
(specpath/statuspath/selectorpath) + selector polje, i onda se i `kubectl scale` i **HPA — a
pominjao je i Karpenter i KEDA — svi oslanjaju na isti /scale API**.

**Kako ja to prodajem:**
> "To objašnjenje da je /status poseban REST endpoint mi je sve razjasnilo — 'zabrana' menjanja
> spec-a nije konvencija nego RBAC na subresursu. Mi smo scale implementirali do kraja: pošto je
> availability string, uveli smo opcioni spec.replicas kao numerički override, pa `kubectl scale
> shop --replicas=4` radi i HPA bi mogao da se zakači bez ikakvog znanja o tome šta je Shop."

### 4. CNPG filozofija: "pisanje kontrolera za postgres je jako teško — i ne bi bilo primenljivo u industriji"

**Šta je pričao:** namerno je zadato da se koriste **gotovi operatori** za baze — poenta zadatka je
da istražimo ekosistem operatora, jer je to ono što se koristi kad se ozbiljno radi sa Kubernetesom.
CNPG: `bootstrap.initdb` (database, owner, secret), operator objavi app secret sa
username/password/uri koje naš operator čita i kači kao env varijable. Njegova rečenica: *"to je
prava moć Kubernetesa — sve što bi inače bile skripte i migracije, operator rešava kroz YAML."*
I princip: **naš operator ne kreira bazu, nego kreira resurs, a tuđi operator rešava bazu** —
"dojo operator se oslanja na druge operatore, zato je bitan redosled".

**Kako ja to prodajem:**
> "Naš Shop operator je tačno taj 'dirigent' sa vežbi: on ne pravi ni jedan StatefulSet — kreira
> CNPG Cluster ili MongoDBCommunity CR i čeka da kolega-operator objavi secret. A pošto Redis
> operatori nisu prošli (Spotahome mrtav, REDB licenca), primenili smo istu filozofiju na MongoDB
> Community operator — spec je to izričito dozvolio."

### 5. Watches + predicate: "posmatraš sve secrete u klasteru → memory leak"

**Šta je pričao:** kad operator zavisi od tuđeg objekta (CNPG secret), proširuješ SetupWithManager
sa `Watches`. Ali ako gledaš **sve** secrete, reflektor ih sve trpa u keš → memorija raste → memory
leak "zato što nismo lepo konfigurisali watches". Predicate filtrira **pre ulaza** u keš; event
handler mapira **na izlazu** — i tu se rešava 1-na-više (jedan secret → reconcile za 5 aplikacija).
Plus indeksi za O(1) umesto O(N) pretrage ("ne morate, ali je preporuka"). Plus filtriranje po tipu
eventa: **za CNPG secret te zanima samo Update** — create stigne pre deploymenta, a za **delete
nema adekvatnog rešenja**: napišeš u dokumentaciji "ne dirajte ovaj secret" i to je legitiman
odgovor na odbrani (sam je to rekao!).

**Kako ja to prodajem:**
> "Oba naša Secret watch-a imaju predikat (sufiks `-app` za connection secrete, `-webhook` za
> Discord) baš zbog te priče o memory leaku, a za webhook lookup koristimo FieldIndexer — ono što
> ste rekli da 'ne moramo ali je preporuka', uradili smo. A za brisanje CNPG secreta imamo tačno
> vaš odgovor: nema elegantnog rešenja, dokumentovano je da se ne dira, operator čeka da se vrati."

### 6. Helm: "nije namenjeno da sve radi u jednom prolazu"

**Šta je pričao:** values.yaml sa `enabled` prekidačima po komponenti (pg/mongo/app). Ako sve
uključiš odjednom → **race condition**: aplikacija traži secret koji baza još nije napravila,
podovi pucaju — **a helm ne prijavi grešku**, jer njegov posao je templating i apply, ne provera da
li sistem radi. "To ljudi zovu najvećim problemom helma, ali to nije njegova nadležnost." Zato:
iterativno — prvo baza, sačekaš, pa aplikacija. Preporučio je bitnami chartove kao referencu
(values od ~1000 linija; nama treba 20-30). I ključna rečenica za naš projekat: **"ono što on radi
uz pomoć helma u više iteracija, mi u admin panelu moramo da izhendlujemo — panel dinamički šalje
REST zahteve i kreira resurse, a operatori rešavaju ostalo."**

**Kako ja to prodajem:**
> "Ta rečenica je bukvalno arhitektura našeg projekta: statički deo (operatori, monitoring) ide
> helm-om po talasima kroz kube-state, a ono što je kod vas bilo 'više iteracija helma' kod nas je
> automatizovano — ShopHub panel kreira Shop CR, a operator interno sprovodi taj redosled: prvo
> baza, čekanje na secret (requeue), pa tek onda Deployment. Race condition koji ste opisali je
> tačno ono što reconcile petlja rešava umesto čoveka."

### 7. make run vs make deploy: "na odbrani mora da radi iz klastera"

**Šta je pričao:** `make run` = operator kao lokalni proces subscribovan na evente (za debug);
`make deploy` = pod u klasteru preko kustomize-a; a očekuje **helm chart koji deploy-uje operator +
README uputstvo kako se instalira i konfiguriše** — i da kroz to na odbrani prođemo i pokažemo da
radi.

**Kako ja to prodajem:**
> "Otišli smo korak dalje od make deploy — operator se instalira iz Helm chart-a koji CI publikuje
> kao OCI artefakt na DockerHub, a kube-state/SETUP.md je to uputstvo koje ste tražili: od praznog
> laptopa do radnog sistema, testirano od nule."

### 8. Finalizeri: "kad god je resurs van Kubernetesa — finalizer"

**Šta je pričao:** Discord kanal ne živi u Kubernetesu, GC ga ne vidi. Finalizer subresurs kaže
GC-u "ne završavaj brisanje dok ja ne potvrdim da je spoljni resurs počišćen". Pravilo: **kad god
manageuješ resurs van Kubernetesa, koristiš finalizer** — uvek, čim pišeš custom resurse koji
diraju spoljni svet.

**Kako ja to prodajem:**
> "Imamo dva: DiscordChannel (kanal na guildu se briše pre CR-a) i Shop (per-tenant Grafana
> dashboard ide preko HTTP API-ja pa ga GC ne vidi). A namerno NEMA finalizera na Wallet-u —
> blockchain adresa se ne može obrisati, a key Secret čisti GC kroz ownership. Znati gde finalizer
> ne treba mi deluje jednako bitno kao znati gde treba."

### 9. Alarmi + Discord (najava verzije 5)

**Šta je pričao:** PrometheusRule (primer: CPU > 80% → alarm na Discord), discord-go biblioteka,
CRD-ovi za Discord server/kanale sa warning/critical sekcijama.

**Kako ja to prodajem:**
> "Našu verziju smo proširili: operator za svaki Shop pravi AlertmanagerConfig sa OnNamespace
> izolacijom — alarmi prodavnice idu samo u njen kanal; klasterski alarmi (NodeHighCPU/Memory) idu
> u poseban maintainers kanal koji je takođe napravljen kroz naš DiscordChannel CRD. I webhook URL
> nikad nije u git-u — ide kao secret-ref."

### 10. Testovi: "ručno okineš reconcile petlju" + envtest ograničenje

**Šta je pričao:** unit test = ručno pozvati Reconcile funkciju i proveriti efekat (ono što je na
tabli bilo iteracija po iteracija — u testu je jedan poziv po iteraciji). I framework ograničenje:
**ne može da se briše namespace u testovima**, pa se ne radi cleanup nego se za svaki test kreira
nov namespace.

**Kako ja to prodajem:**
> "Testovi operatora rade tačno to — ručno okidanje Reconcile sa fake klijentom (uz
> WithStatusSubresource da status update radi), assert na kreirani Deployment i conditione. I da,
> naleteli smo na to ograničenje sa namespace-ima koje ste najavili."

---

## Njegove fore i rečenice koje vredi znati

Sitnice koje pokazuju da sam bio prisutan, ne samo da sam čitao slajdove:

- **"Dijagram morate znati usred noći"** — API server → reflektor → cache → work queue → reconciler.
  Ako me bilo šta pita o operatoru, ovo crtam prvo.
- **"Jedino API server sme da dira bazu (etcd)"** — svi ostali slušaju i keširaju.
- **f(f(x)) = f(x)** — njegova definicija idempotentnosti; plus ona sa množenjem (samo 0 i 1).
- **"Šta ako imaš 2 replike operatora? 1000 bugova"** — zašto je reconciler stateless (i zašto
  postoji leader election).
- **"Stalled = ti si kriv (popravi YAML), Failed = klaster je kriv"** — Reason taksonomija.
- **409 Conflict "će nas najviše nervirati"** — i jeste.
- **"Helm nije kriv što ti sistem ne radi — njegov posao je templating"** + "nije namenjeno da sve
  radi u jednom prolazu" → iterativni deploy.
- **"Ne piši operator za postgres — ne bi bio primenljiv u industriji"** → CNPG; naš operator
  diriguje tuđim operatorima.
- **"Za delete tuđeg secreta nema adekvatnog rešenja — napiši u dokumentaciji da se ne dira"** —
  legitiman odgovor na odbrani, njegove reči.
- **"Da se ne bismo patili na odbrani"** — zašto statusi i conditions moraju biti informativni.
  (Mogu i da se našalim: statusi su nam informativni baš iz tog razloga.)
- **Clear code, ne clean code** — usput je provaljivao da mu je draži "clear" od Uncle Bobovog
  "clean" koda. Ne forsirati, ali ako padne priča o kvalitetu koda, znam referencu.
- **Bitnami charts kao referenca** — "values od 1000 linija je za produkciju; vama treba 20-30".
  Naši values fajlovi su tačno u tom duhu.
- Očekuje da sistem **"korektno radi kad povećaš replike — neće tražiti 100 aplikacija"** — dakle
  demo skaliranja (kubectl scale shop) je tačno ono što želi da vidi.

---

## Brza mapa: njegova priča → naš projekat

| On je na vežbama... | Mi u projektu... |
|---|---|
| multi-stage + non-root + exec forma + hadolint | sva 3 Dockerfile-a po toj checklisti, hadolint required check u CI |
| kompajlirani jezici sigurniji u kontejneru | ceo backend Go, statički binary, release bez source-a |
| CRD iz markera, "što bismo ručno" | 3 CRD-a generisana iz Go tipova, enum validacije, print kolone |
| stateless reconciler + Status kao checkpoint | ensure-pattern svuda; DiscordChannel čuva channelID u Status pre webhook-a |
| conditions trojka + Stalled/Failed | Available/Progressing/Degraded + Reason taksonomija, vidljivo u describe |
| 409 → IsConflict posle svakog update-a | gutanje konflikta u status update + requeue wrapper na vrhu Reconcile |
| watches sa predicate-om (memory leak priča) | `-app` i `-webhook` predikati + FieldIndexer (O(1)) |
| CNPG umesto svog DB operatora | CNPG za postgres + MongoDB Community za "light" (Spotahome mrtav, REDB licenca) |
| samo Update event za CNPG secret; delete se dokumentuje | isto; dokumentovano "ne dirati secret" |
| scale subresource + selector + RBAC | `kubectl scale shop` radi; spec.replicas override preko availability |
| helm iterativno, enabled prekidači | kube-state talasi + ArgoCD; operator automatizuje redosled interno |
| make deploy / helm chart za operator + README | OCI chart na DockerHub-u + SETUP.md od nule, ArgoCD app-of-apps |
| finalizer za sve van Kubernetesa | DiscordChannel (kanal), Shop (Grafana dashboard); Wallet namerno bez |
| PrometheusRule + alarm na Discord | per-Shop AlertmanagerConfig (OnNamespace) + maintainers kanal za klasterske |
| unit test = ručno okini Reconcile; envtest ne briše namespace | fake client + WithStatusSubresource; nov namespace po testu |
| graceful shutdown / SIGTERM | http.Server.Shutdown + context cancel za payment sweep |

---

## Kako da ovo NE upotrebim pogrešno

- **Ne otvaraj razgovor sa "vi ste rekli na vežbama..."** — čekaj pitanje, odgovori suštinski, pa
  tek onda zakači referencu. Redosled: znanje → primena kod nas → "to je i ono što ste pominjali".
- **Ne citiraj ono što ne umem da odbranim dubinski.** Svaka referenca odavde povlači potpitanje —
  za svaku postoji pun odgovor u `odbrana.md`. Ako neku sekciju tamo nisam savladao, ne pominjem je
  ni odavde.
- **Maksimalno 1 referenca po temi.** Cilj je utisak "slušao je i primenio", ne "naštrebao je moje
  rečenice".
- Najjači aduti (ako biram tri): informer dijagram "usred noći", f(f(x))=f(x) + Status kao
  checkpoint, i "operator se oslanja na druge operatore" → CNPG/Mongo priča. Ta tri pokrivaju
  većinu operator pitanja, a operator je srce projekta.
