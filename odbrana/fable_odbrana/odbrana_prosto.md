# ShopHub objašnjen od nule — verzija za razumevanje

> Ovo je "spora" verzija pripreme. Ovde nema prečica: svaki pojam se prvo objasni običnim jezikom,
> sa poređenjima iz stvarnog života, pa se tek onda kaže gde je to u našem projektu i šta bih
> odgovorio na odbrani. Čitaj redom — svaka glava se oslanja na prethodne. Kad ovo savladaš,
> `odbrana.md` (gusta verzija) će ti biti lak, jer je to isti sadržaj samo zbijen.
>
> Pravilo dok čitaš: ako ti rečenica nije jasna, nemoj preskočiti — verovatno koristi pojam iz
> ranije glave. Vrati se, pa nastavi.

---

## Sadržaj

- [0. Velika slika: šta smo uopšte napravili](#0-velika-slika)
- [1. Kontejneri i Docker — od nule](#1-kontejneri-i-docker)
- [2. Kubernetes — od nule](#2-kubernetes)
- [3. Kako Kubernetes radi iznutra (dovoljno da razumeš operator)](#3-kubernetes-iznutra)
- [4. Šta je operator i zašto je on srce našeg projekta](#4-operator)
- [5. Naša tri CRD-a, jedan po jedan](#5-nasa-tri-crd-a)
- [6. Baze: zašto ih ne pravimo sami](#6-baze)
- [7. Shop aplikacija: šta se dešava kad neko nešto kupi](#7-shop-aplikacija)
- [8. Web3 i plaćanje kriptovalutom — od nule](#8-web3)
- [9. ShopHub: nalozi, tokeni i zašto svaki korisnik ima svoj namespace](#9-shophub)
- [10. Observability: kako znamo šta se dešava u sistemu](#10-observability)
- [11. Helm: šta je i zašto postoji](#11-helm)
- [12. kube-state i GitOps: klaster koji se sam sređuje](#12-gitops)
- [13. Git, CI/CD: šta se dešava kad pošaljem kod](#13-cicd)
- [14. Najčešća pitanja — kratki, izgovorljivi odgovori](#14-kratka-pitanja)
- [15. Rečnik: svaki pojam u jednoj rečenici](#15-recnik)

---

## 0. Velika slika

Pre bilo kakve tehnologije, evo šta naš sistem radi, ispričano kao priča:

Zamisli sajt koji se zove **ShopHub**. Na njega dođe čovek koji želi da otvori online prodavnicu,
ali ne zna ništa o serverima. Registruje se, klikne "napravi prodavnicu", ukuca ime, izabere da li
hoće "običnu" ili "pojačanu" dostupnost, izabere bazu, unese adresu svog kripto novčanika — i
klikne dugme. Posle minut-dva, njegova prodavnica je **živ sajt na internetu**, sa svojom bazom
podataka, svojim merenjem saobraćaja, svojim alarmima. On u nju doda proizvode, a kupci plaćaju
kriptovalutom.

E sad, ono što je magija ovog projekta: **niko od nas ne diže tu prodavnicu ručno**. Kad korisnik
klikne dugme, u pozadini se u sistem samo upiše jedan mali zapis koji kaže "želim prodavnicu koja
izgleda ovako". A onda naš program — koji se zove **operator** — primeti taj zapis i sam napravi
sve što treba: bazu, kopije aplikacije, mrežna podešavanja, nadzor, alarme. I ne samo da napravi,
nego **stalno proverava** da li je sve kako treba, i popravlja ako nije.

Ceo projekat je podeljen u 5 delova (5 git repozitorijuma):

| Deo | Šta je | Ljudski rečeno |
|---|---|---|
| `shophub` | sajt za upravljanje prodavnicama | "kontrolna tabla" — tu se korisnik registruje i klikće |
| `shop` | sama prodavnica (sajt + pozadinski program) | ono što kupci vide i gde kupuju |
| `shop-operator` | naš robot-automatizator | pretvara "želju" u živu prodavnicu i održava je |
| `helm-charts` | paketi za instalaciju | "instalacioni fajlovi" za naše programe |
| `kube-state` | opis celog sistema u fajlovima | recept po kom se ceo sistem podiže od nule |

Sve to radi unutar **Kubernetes klastera** — a šta je to, kreće od glave 2. Prvo kontejneri, jer je
to temelj svega.

---

## 1. Kontejneri i Docker

### 1.1 Problem koji kontejner rešava

Zamisli da si napisao program i hoćeš da ga pokreneš na tuđem računaru. Klasičan problem: "kod mene
radi, kod tebe ne radi". Zašto? Jer tvoj računar ima instaliran Go verziju X, neku biblioteku,
neka podešavanja — a tuđi nema.

**Kontejner** je rešenje: to je kutija u koju spakuješ **program + sve što mu treba da radi**
(biblioteke, podešavanja, čak i mini operativni sistem). Ta kutija se onda pokreće isto na bilo kom
računaru koji ima Docker. Nije virtuelna mašina — ne nosi ceo operativni sistem sa sobom, deli
jezgro (kernel) sa računarom na kom radi, pa je mnogo lakša i pokreće se za sekund.

Poređenje koje pomaže: **brodski kontejner**. Pre kontejnera, roba se tovarila na brodove svakojako
i svaki pretovar je bio drama. Onda su svi usvojili standardnu metalnu kutiju — i sad bilo koji
brod, kamion ili dizalica ume da radi sa bilo kojom robom, jer je kutija uvek ista spolja. Docker
je to za softver: spolja standardno, unutra šta god hoćeš.

### 1.2 Tri pojma: image, kontejner, registry

- **Image (slika)** je "recept + sastojci" — nepromenjiv paket iz kog se kontejner pravi. Kao
  instalacioni fajl.
- **Kontejner** je pokrenuta instanca image-a. Iz jednog image-a možeš pokrenuti 10 kontejnera.
  (Analogija: image je klasa, kontejner je objekat. Ili: image je recept, kontejner je skuvano jelo.)
- **Registry** je "prodavnica image-a" — server na koji se image-i kače i sa kog se skidaju. Mi
  koristimo **DockerHub** (nalog `urospetraskovic`). Kad naš sistem negde kaže "pokreni
  `urospetraskovic/shop-backend:0.2.1`", to znači: skini taj image sa DockerHub-a i pokreni ga.

Ono `:0.2.1` na kraju je **tag** — verzija image-a. Bitno: mi svuda koristimo tačne verzije, nikad
`latest`, jer "latest" sutra može biti nešto drugo — a mi hoćemo da sistem bude predvidiv.

### 1.3 Dockerfile — recept za image

**Dockerfile** je tekstualni fajl sa uputstvom kako se image pravi, red po red: "kreni od ove
osnove, iskopiraj ove fajlove, pokreni ovu komandu za build, na kraju pokreći ovaj program".

Naš Dockerfile (pojednostavljen, ovako izgleda za shophub):

```dockerfile
# FAZA 1: build frontenda — uzmi image koji ima Node.js i sastavi sajt
FROM node:20-alpine AS web
COPY frontend/ .
RUN npm install && npm run build        # rezultat: gotovi HTML/JS fajlovi

# FAZA 2: build backenda — uzmi image koji ima Go i iskompajliraj program
FROM golang:1.26-alpine AS builder
COPY backend/ .
RUN go build -o /out/shophub ./cmd/shophub   # rezultat: JEDAN izvršni fajl

# FAZA 3: finalna slika — mini-linux + samo ono što je potrebno da RADI
FROM alpine:3.20
COPY --from=builder /out/shophub /app/shophub   # samo izvršni fajl
COPY --from=web /web/dist /app/web              # samo gotovi fajlovi sajta
USER app                                        # ne pokreći kao administrator!
ENTRYPOINT ["/app/shophub"]
```

Ovo se zove **multi-stage build** (višefazna gradnja) i to je najvažnija stvar koju treba da umeš
da objasniš:

- Faza 1 i 2 su "radionica" — tu su svi alati (Node, Go kompajler), i tu se pravi program.
- Faza 3 je "izlog" — u finalni image ide **samo rezultat**, bez alata i bez izvornog koda.

Zašto je to bitno? Dva razloga:
1. **Veličina**: image sa Go kompajlerom ima stotine MB; naš finalni ima nekoliko desetina MB.
   Manji image = brže se skida, brže pokreće.
2. **Bezbednost**: u finalnom image-u nema kompajlera ni koda. Ako neko provali u kontejner, nema
   čime da napravi i pokrene izmenjenu verziju programa. Asistent je na vežbama baš ovo isticao:
   kod kompajliranih jezika (Go) štitiš jedan fajl; kod Pythona/Node-a moraš da vučeš i štitiš ceo
   izvorni kod — zato nam je Go tu u prednosti.

### 1.4 Non-root: zašto program u kontejneru nije "administrator"

U Linux svetu, **root** je korisnik koji sme sve. Ako se aplikacija u kontejneru izvršava kao root
i neko je hakuje, haker sme sve unutar kontejnera (a ponekad i šire). Zato u finalnoj fazi:

- napravimo običnog korisnika `app`,
- fajlove postavimo tako da je **vlasnik root**, a `app` sme samo da ih **čita i izvršava** — ne i
  da ih menja (`chmod 0755`),
- kažemo `USER app` — od tog trenutka sve radi kao običan korisnik.

Rezultat: čak i da neko provali u aplikaciju, ne može da prepiše njen izvršni fajl niti da
instalira nešto. Ovo je tačno onako kako je asistent pokazivao na vežbama.

Praktična posledednja sitnica: običan korisnik ne sme da koristi portove ispod 1024, pa naše
aplikacije slušaju na portu **8080** umesto na 80.

### 1.5 ENTRYPOINT i zašto pišemo `["/app/shophub"]` a ne samo `/app/shophub`

`ENTRYPOINT` kaže šta se pokreće kad kontejner krene. Postoje dva zapisa:
- **exec forma**: `ENTRYPOINT ["/app/shophub"]` — program se pokreće direktno i on je "glavni
  proces" (PID 1) u kontejneru.
- **shell forma**: `ENTRYPOINT /app/shophub` — prvo se pokrene shell (komandna linija), pa on
  pokrene program kao svoje dete.

Zašto je razlika bitna: kad Kubernetes hoće da ugasi kontejner, on glavnom procesu pošalje signal
**SIGTERM** ("molim te, završi šta radiš i ugasi se"). Kod shell forme taj signal dobije shell — a
shell ga **ne prosledi** našem programu. Program ne zna da treba da se gasi, pa ga posle 10 sekundi
sistem ubije na silu (SIGKILL) — usred posla, sa možda otvorenim konekcijama ka bazi. Kod exec
forme signal stiže direktno programu, program uredno zatvori konekcije i izađe. To se zove
**graceful shutdown** (uredno gašenje). Naš Go kod hvata SIGTERM i uredno gasi HTTP server i
pozadinsku petlju za proveru plaćanja.

Trik za prepoznavanje (sa vežbi): ako `docker stop` traje 10 sekundi — verovatno je shell forma.

### 1.6 Hadolint

**hadolint** je alat koji čita Dockerfile i prijavljuje loše prakse (kao lektor za Dockerfile).
Mi smo ga stavili u automatsku proveru: svaki put kad neko predloži izmenu koda, hadolint proveri
Dockerfile, i ako nešto ne valja — izmena ne može da se prihvati dok se ne ispravi.

---

## 2. Kubernetes

### 2.1 Problem koji Kubernetes rešava

Docker ume da pokrene kontejner na jednom računaru. Ali pravi sistemi imaju: više računara, više
kopija aplikacije (da izdrže saobraćaj i kvarove), potrebu da se aplikacija sama restartuje kad
padne, mrežu između njih, tajne (lozinke), verzije... Ručno održavati to je nemoguće.

**Kubernetes** (skraćeno K8s) je "upravnik zgrade" za kontejnere: ti mu **opišeš šta želiš** ("hoću
2 kopije ove aplikacije, dostupne na ovoj adresi"), a on se **stalno stara** da tako i bude. Ako
kopija padne — podigne novu. Ako se računar pokvari — preseli kontejnere na drugi. Ovaj princip
"opiši željeno stanje, sistem ga održava" zove se **deklarativni pristup** i on je ključna ideja
kojom se ceo naš projekat vodi.

**Klaster** = skup računara (zovu se **nodovi**) kojima Kubernetes upravlja kao jednom celinom.
Mi za razvoj koristimo **k3d** — alat koji napravi mali Kubernetes klaster na laptopu tako što
svaki "nod" bude jedan Docker kontejner. Naš klaster ima 1 glavni (server) i 2 radna (agent) noda.

Sa Kubernetesom pričaš komandom **kubectl** (npr. `kubectl get pods` — "pokaži mi šta radi"), a
želje mu opisuješ u **YAML fajlovima** (tekstualni format: ime, tip, podešavanja).

### 2.2 Pod — najmanja jedinica

**Pod** je omotač oko jednog (retko više) kontejnera. Kubernetes ne barata golim kontejnerima nego
podovima. Za nas: pod ≈ jedna pokrenuta kopija aplikacije.

Bitno: pod je **smrtan i zamenjiv**. Kad se restartuje, dobije novu adresu, nov identitet. Zato se
skoro nikad ne pravi pod direktno, nego preko Deployment-a.

### 2.3 Deployment — "hoću N kopija i drži ih u životu"

**Deployment** je zapis koji kaže: "hoću tačno N podova napravljenih iz ovog image-a, sa ovim
podešavanjima". Kubernetes ga čita i stara se: padne li pod, digne nov; promeniš li image verziju,
postepeno zameni stare podove novima.

**Kod nas:** kad operator pravi prodavnicu, napravi Deployment sa **2 replike** (kopije) ako je
korisnik izabrao "standard", ili **3** ako je izabrao "high". To je bukvalno zahtev iz specifikacije.
Zašto više kopija? (1) ako jedna padne, druga i dalje služi kupce; (2) saobraćaj se deli.

### 2.4 Service — stalna adresa ispred promenjivih podova

Problem: podovi se rađaju i umiru, i svaki put imaju drugu IP adresu. Kako da neko stalno može da
ih nađe?

**Service** je stalna, imenovana adresa koja stoji **ispred** grupe podova i raspoređuje saobraćaj
na njih (load balancing). Kao recepcija firme: zaposleni se menjaju, broj recepcije ostaje isti.
Unutar klastera se do servisa dolazi po imenu, npr. `moj-shop:8080`.

Kako Service zna koji su "njegovi" podovi? Preko **labela** (nalepnica). Operator svakom podu
prodavnice zalepi labelu `app: moj-shop`, a Service kaže "moji su svi sa tom labelom".

### 2.5 Ingress — ulaz u klaster spolja

Service radi samo **unutar** klastera. Da bi neko iz browsera došao do prodavnice, treba **Ingress**
— pravilo koje kaže: "kad stigne zahtev za adresu `moj-shop.localhost`, prosledi ga Service-u
`moj-shop`". Ingress je kao portir zgrade koji po imenu na koverti zna na koji sprat da je pošalje.

**Kod nas:** svaka prodavnica dobije adresu `<ime>.localhost`. Pošto naš k3d klaster mapira svoj
port 80 na port 8080 laptopa, u browseru se kuca `http://moj-shop.localhost:8080`.

### 2.6 Namespace — fioke unutar klastera

**Namespace** je logička "fioka" u klasteru: resursi u različitim namespace-ima se ne mešaju,
imena mogu da se ponavljaju, prava pristupa se daju po fiokama.

**Kod nas je namespace granica između korisnika (tenanta)**: svaki registrovani korisnik ShopHub-a
dobije svoju fioku (`tenant-a1b2c3...`), i sve njegove prodavnice žive tamo. Zato dva korisnika
mogu da imaju prodavnicu istog imena, zato korisnik ne može ni da vidi tuđe prodavnice, i zato je
brisanje korisnika prosto — obriši fioku, sve iz nje nestane. Sistemske stvari imaju svoje fioke:
`shophub`, `shop-operator-system`, `monitoring`, itd.

### 2.7 ConfigMap i Secret — podešavanja i tajne

Aplikaciji trebaju podešavanja (npr. na kom portu radi) i tajne (lozinka baze). To se ne upisuje u
kod ni u image, nego u posebne resurse:
- **ConfigMap** — obična podešavanja (ne-tajna).
- **Secret** — tajne: lozinke, tokeni, ključevi.

Pod ih dobija najčešće kao **environment varijable** (promenljive okruženja — vrednosti koje
program pročita pri pokretanju, u Go: `os.Getenv("DATABASE_URL")`).

**Kod nas su Secreti svuda**: lozinka baze (pravi je operator baze), admin lozinka prodavnice
(pravi je naš operator), Discord adrese, ključ za potpisivanje tokena, privatni ključ novčanika...
Zlatno pravilo projekta: **nijedna tajna nije u git-u** — tajne žive samo u klasteru kao Secreti.

### 2.8 Probes — kako Kubernetes zna da je aplikacija dobro

Kubernetes ne ume da pogodi da li tvoja aplikacija radi; ti mu daš dve "kontrolne tačke" (URL-ove
koje on periodično poziva):

- **Liveness probe** ("da li si živ?"): ako ne odgovori — Kubernetes **restartuje** kontejner.
- **Readiness probe** ("da li si spreman za goste?"): ako ne odgovori — Kubernetes ga privremeno
  **isključi iz saobraćaja** (Service mu ne šalje zahteve), ali ga ne restartuje.

**Naša pametna odluka koju treba znati objasniti**: liveness je kod nas maksimalno prost (uvek
kaže "živ sam" dok proces radi), a readiness proverava **konekciju ka bazi**. Zašto tako? Ako baza
nakratko zakuca, restartovanje aplikacije ništa ne rešava (problem je u bazi!) — samo bi napravilo
lančano restartovanje. Ovako aplikacija samo "izađe iz saobraćaja" dok se baza ne vrati, pa se
sama vrati u saobraćaj. Kvar se izoluje umesto da se širi.

### 2.9 RBAC — ko šta sme

**RBAC** (Role-Based Access Control) je sistem dozvola u Kubernetesu: ko (koji program/čovek) sme
šta (čitati/praviti/brisati) nad čim (podovi, secreti...).

Tri dela: **ServiceAccount** (identitet — "lična karta" programa u klasteru), **Role/ClusterRole**
(spisak dozvola — Role važi u jednoj fioci, ClusterRole svuda), **RoleBinding/ClusterRoleBinding**
(spajalica: "ovaj identitet ima ove dozvole").

**Kod nas:** operator ima svoju listu dozvola (sme da pravi Deploymente, Service, Secrete, baze...);
ShopHub backend ima svoju (sme da pravi namespace-e i Shop zapise). Ako operator pokuša nešto za
šta nema dozvolu, dobije grešku **403 Forbidden** — i tačno tako smo tokom razvoja otkrivali koje
dozvole nam fale.

---

## 3. Kubernetes iznutra

Ovu glavu treba razumeti zato što je operator (glava 4) nemoguće razumeti bez nje. Ali dovoljan je
sledeći nivo, ne dublje.

### 3.1 API server i etcd — sveska i jedini pisar

U srcu klastera stoje dve stvari:

- **etcd** — baza u kojoj piše **celo stanje klastera**. Svaki pod, deployment, secret — sve je
  zapis u etcd-u. Zamisli je kao svesku u kojoj piše "ovako sistem treba da izgleda i ovako trenutno
  izgleda".
- **API server** — jedina komponenta koja sme da piše u tu svesku. SVE u Kubernetesu ide preko
  njega: kad ti kucaš `kubectl apply`, kad operator nešto pravi, kad Kubernetes sam nešto menja —
  sve su to zahtevi API serveru. On proveri da li je zahtev ispravan i dozvoljen, upiše u svesku, i
  onda **obavesti sve zainteresovane** da se nešto promenilo.

Asistent je za ovo rekao da se mora znati "usred noći": **jedino API server dira bazu; svi ostali
slušaju obaveštenja i drže sebi kopiju (keš)** — da ne bi svako stalno zapitkivao API server.

### 3.2 Kontroleri — termostati

**Kontroler** je program koji radi u petlji: pogleda **željeno stanje** (šta piše u svesci da
treba), pogleda **stvarno stanje** (šta stvarno radi), i ako se razlikuju — **preduzme akciju** da
ih izjednači. Pa opet. I opet. Zauvek.

Najbolje poređenje: **termostat**. Podesiš 22°C (željeno stanje). Termostat meri temperaturu
(stvarno stanje). Hladnije od 22? Uključi grejanje. Toplije? Isključi. Ne zanima ga kako je do
razlike došlo — samo stalno poravnava stvarno sa željenim.

Kubernetes je gomila ugrađenih termostata: Deployment kontroler (drži N replika), node kontroler,
itd. **Naš operator je isti takav termostat, samo za pojam "prodavnica"** — i to je cela tajna.

---

## 4. Operator

### 4.1 Ideja: naučiti Kubernetes novom pojmu

Kubernetes iz kutije zna za podove, deploymente, servise... ali pojma nema šta je "prodavnica".
Operator pattern je način da ga naučiš:

1. **CRD (Custom Resource Definition)** = definicija novog tipa zapisa. Mi smo definisali tip
   `Shop` — od tog trenutka u klasteru može da postoji zapis vrste Shop, i `kubectl get shops`
   radi kao da je Shop oduvek deo Kubernetesa.
2. **Kontroler** = naš program (termostat) koji gleda te zapise i pretvara ih u stvarnost.

Dakle: **CRD je obrazac, CR (Custom Resource) je popunjen obrazac, operator je službenik koji po
obrascu sve sprovede** — i posle stalno proverava da li je sve po obrascu.

Naš Shop zapis (CR) izgleda ovako, i vredi ga znati napamet:

```yaml
apiVersion: apps.shophub.local/v1
kind: Shop
metadata:
  name: moj-shop
  namespace: tenant-a1b2c3        # fioka korisnika koji ga je napravio
spec:                             # SPEC = ŽELJA (piše korisnik)
  title: "Moja prodavnica"
  availability: high              # standard = 2 kopije, high = 3
  database: postgres              # ili mongodb
  walletAddress: "0x45bE..."      # gde kupci šalju pare
status:                           # STATUS = STVARNOST (piše operator)
  readyReplicas: 3
  url: http://moj-shop.localhost:8080
  conditions: [...]               # Available=True itd.
```

**Spec vs status je najvažnija podela**: spec piše korisnik ("ovako želim"), status piše operator
("ovako trenutno jeste"). Operator NIKAD ne menja spec — želja je korisnikova svojina. To nije samo
konvencija: status se menja preko posebnog endpointa (`/status`) sa posebnim dozvolama, pa operator
fizički i ne može da dira spec.

### 4.2 Kako smo napravili operator: Kubebuilder

**Kubebuilder** je alat koji generiše kostur operatora u Go-u. Mi napišemo Go strukturu (polja
Shop-a), dodamo **markere** — specijalne komentare tipa:

```go
// +kubebuilder:validation:Enum=standard;high
// +kubebuilder:default:=standard
```

i komanda `make manifests generate` iz toga izgeneriše ceo CRD YAML sa validacijom. Validacija
znači: ako neko pokuša da upiše `availability: turbo`, API server to **odbije odmah** — nevalidan
zapis nikad ni ne stigne do našeg operatora. Asistentova rečenica: "možemo CRD i ručno da pišemo,
ali što bismo" — ručno je gomila YAML-a i gomila prostora za grešku.

Naš operator ima **tri CRD-a u tri grupe**: `Shop` (grupa apps), `DiscordChannel` (grupa notify),
`Wallet` (grupa payments) — podela po smislu, kao što i Kubernetes deli svoje resurse po grupama.
(Tehnička caka koju pamtim: kubebuilder default dozvoljava samo jednu grupu, pa se pre generisanja
mora uključiti multigroup režim.)

### 4.3 Reconcile petlja — šta operator tačno radi

Funkcija koja se izvršava svaki put kad se "nešto desi" sa nekim Shop zapisom zove se
**Reconcile** (pomirenje — pomiruje stvarnost sa željom). Naša, korak po korak, ljudski:

1. **Pročitaj Shop zapis.** Ako više ne postoji — ništa, neko ga je obrisao, brisanje rešava druga
   mašinerija (glava 4.6).
2. **Da li je u brisanju?** Ako jeste, idi na proceduru brisanja. Ako nije, pobrini se da ima
   "kočnicu za brisanje" (finalizer — objašnjeno u 4.6).
3. **Sredi bazu.** Ako je izabran postgres: napravi zahtev CNPG operatoru (glava 6) da digne bazu.
   Pa proveri: da li je baza objavila Secret sa lozinkom? **Ako nije — IZAĐI i zakaži ponovni
   pokušaj za 10 sekundi.** (Ne čekamo u mestu! To je bitno pravilo — vidi 4.4.)
4. **Napravi Secret sa admin lozinkom prodavnice** (nasumična lozinka; vlasnik će je videti u
   ShopHub panelu).
5. **Napravi/popravi Deployment**: prava slika, pravi broj replika (2 ili 3), sve environment
   varijable (adresa baze iz Secreta, adresa novčanika, podešavanja za praćenje...).
6. **Napravi/popravi Service i Ingress** — da prodavnica ima stalnu adresu i da je vidljiva spolja.
7. **Napravi stvari za nadzor** (ServiceMonitor, Grafana dashboard, alarmi — glava 10). Ove korake
   radimo "najbolje što možemo": ako nadzorni sistem nije instaliran, samo zabeležimo i nastavimo —
   prodavnica ne sme da bude taoc nadzora.
8. **Upiši status**: koliko je replika spremno, koja je adresa, i "semafore" (conditions).

Svaki korak "napravi/popravi" je zapravo: pogledaj da li već postoji → ako ne postoji, napravi →
ako postoji ali je drugačije nego što treba, ispravi → ako je već kako treba, **ne diraj ništa**.

### 4.4 Četiri pravila reconcile petlje (i zašto)

**Pravilo 1 — idempotentnost.** Petlja se poziva mnogo puta za isti Shop (svaka promena je novi
poziv). Zato mora da važi: ako se pozove 5 puta zaredom, rezultat je isti kao da se pozvala jednom.
Matematički (asistentova omiljena formula): **f(f(x)) = f(x)**. Da ovo ne važi, svaki poziv bi
npr. pravio novi Discord kanal — katastrofa. Postižemo ga upravo onim "pogledaj pa tek onda pravi".

**Pravilo 2 — bez pamćenja (stateless).** Petlja ne sme ništa da pamti u svojim varijablama između
poziva. Sve što treba zapamtiti ide u **status** zapisa. Zašto: operator može da se restartuje
(pamćenje bi nestalo), ili da ima dve kopije (svaka bi pamtila svoje — haos; asistent: "2 replike
= 1000 bugova"). Primer kod nas: DiscordChannel kontroler zapiše ID kreiranog kanala u status ČIM
ga Discord vrati — pa ako sledeći korak pukne i petlja krene ispočetka, vidi u statusu "kanal već
postoji" i ne pravi drugi.

**Pravilo 3 — ne čekaj u mestu.** Ako nešto još nije spremno (baza se diže), petlja NE SME da stoji
i čeka. Izađe i kaže "probaj me opet za 10 sekundi" (`RequeueAfter`). Petlja koja čeka blokira
obradu svih ostalih zapisa.

**Pravilo 4 — jedna izmena po prolazu + računaj na konflikt (409).** Svaki zapis u Kubernetesu ima
broj verzije. Ako dva programa u isto vreme menjaju isti zapis, drugi dobije grešku **409 Conflict**
("radiš sa zastarelom verzijom"). To kod nas NIJE greška nego normalna pojava (i Kubernetes i baza
i mi pišemo po istim objektima) — na 409 samo odustanemo od tog prolaza i pustimo petlju da krene
iznova sa svežim podacima. Asistent je najavio "to će vas najviše nervirati" — i bio je u pravu.

### 4.5 Conditions — semafori u statusu

U statusu Shop-a stoje tri standardna "semafora" (conditions):

- **Available** — "prodavnica radi i može da služi kupce" (zeleno).
- **Progressing** — "upravo radim na tome da stigne u željeno stanje" (žuto).
- **Degraded** — "desila se greška koju sam ne mogu da popravim" (crveno).

Pravila: Degraded i Progressing ne mogu istovremeno (ili se radi, ili je zaglavljeno); Available
može uz Progressing (radi, ali se npr. baš sad skalira). Uz svaki semafor ide **Reason** — kratak
razlog (kod nas: `DatabaseProvisioning` = čekam bazu, `Deploying` = dižem kopije, `Ready` = sve
spremno, `DatabaseFailed` = baza nije uspela).

Čemu sve to: kad na odbrani kucam `kubectl describe shop moj-shop`, vidi se ceo životni put:
prvo žuto "čekam bazu", pa žuto "dižem kopije", pa zeleno "spremno" — sa vremenima prelaza. I ne
samo za ljude: **drugi alati čitaju ove semafore** (ArgoCD, recimo, gleda da li je nešto
Progressing) — conditions su javni "jezik" kojim naš operator priča sa ostatkom ekosistema.

### 4.6 Ownership, Garbage Collection i finalizeri — kako brisanje radi

**OwnerReference (vlasnička veza):** kad operator napravi Deployment za Shop, on na Deployment
zalepi cedulju "moj vlasnik je Shop moj-shop". Kad se Shop obriše, Kubernetesov **Garbage
Collector** (đubretar) automatski obriše SVE što je Shop posedovao: Deployment, Service, Ingress,
Secrete, zahtev za bazu... Mi za brisanje **nismo napisali ni liniju koda** — samo smo uredno
lepili cedulje pri kreiranju.

Ali — đubretar briše samo stvari **unutar Kubernetesa**. Šta sa stvarima napolju? Naš operator
pravi i: kanal na Discord serveru (živi kod Discorda, ne kod nas) i kontrolnu tablu u Grafani
(živi u Grafani). Ako se Shop obriše, te stvari bi ostale da vise kao "siročići".

Rešenje: **finalizer** — "kočnica za brisanje". To je marker na zapisu koji kaže Kubernetesu:
"kad neko zatraži brisanje, NEMOJ još da završiš — prvo ja da počistim svoje spoljne stvari".
Tok: korisnik obriše → zapis pređe u stanje "Terminating" (briše se, ali još postoji) → naša
petlja to vidi → obriše Discord kanal / Grafana tablu → skine kočnicu → tek tada Kubernetes
stvarno obriše zapis, pa đubretar počisti ostalo.

I finesa koju volim da pomenem jer pokazuje razmišljanje: **Wallet CRD namerno NEMA finalizer** —
adresa na blockchainu se ne može "obrisati" (blockchain pamti zauvek), a Secret sa ključem počisti
đubretar preko vlasničke veze. Znati gde kočnica NE treba je jednako važno.

A šta ako spoljni servis ne radi baš kad brišemo? Discord ne odgovara → pokušavamo ponovo na 30s,
zapis strpljivo stoji u Terminating. A za najgori slučaj (neko je već obrisao Secret sa Discord
tokenom pa NIKAD ne bismo mogli da počistimo kanal) imamo izlaz: preskoči čišćenje i pusti brisanje
— bolje jedan zaboravljen kanal na Discordu nego zapis koji se večno ne može obrisati.

### 4.7 Kako operator "čuje" promene: informer, keš, red

Ovo je dijagram "usred noći". Kako naš operator sazna da je neko napravio Shop?

```
API server ("desilo se X!")
     ↓  (operator je pretplaćen na obaveštenja — to se zove watch)
Reflector — prima obaveštenja i sve upisuje u KEŠ (lokalna kopija objekata)
     ↓  (filter: da li me se ovo uopšte tiče? — predicate)
Work queue (red čekanja) — u red ide samo "ime + fioka" zapisa, BEZ duplikata
     ↓
Reconcile petlja — uzima iz reda, radi svoj posao
```

Tri posledice koje treba razumeti:

1. **Čitanje je iz keša, pisanje ide direktno.** Kad petlja radi `Get`, čita svoju lokalnu kopiju
   (brzo, ne davi API server). Kad radi `Update`, ide na API server. Pošto keš ume da kasni koji
   milisekund, otud povremeni 409 — vidi pravilo 4.
2. **Red uklanja duplikate.** Ako se Shop promeni 5 puta dok smo zauzeti, u redu stoji JEDNOM — i
   obradimo samo poslednje stanje. Ovo se zove **level-based** ponašanje: ne jurimo svaki događaj,
   poravnavamo se sa trenutnim stanjem. Zato sme operator i da odspava (padne) — kad se vrati,
   poravna se, ništa nije "propušteno".
3. **Filter (predicate) je obavezan kod tuđih resursa.** Mi pratimo i Secrete (čekamo da baza
   objavi lozinku). Ali da pratimo SVE Secrete u klasteru, keš bi rastao i rastao — asistent je to
   zvao memory leak. Zato filter: nas zanimaju samo Secreti čije se ime završava na `-app`
   (konvencija za bazine Secrete) ili `-webhook` (Discord). Sve ostalo se ignoriše pre keša.

I još jedan pojam odavde: **FieldIndexer** — registar koji nam omogućava da pitanje "koji Shop-ovi
koriste ovaj Secret?" rešimo trenutno (kao indeks u knjizi), umesto da listamo sve Shop-ove redom.
Asistent: "ne morate, ali je preporuka" — uradili smo.

### 4.8 Scale subresource — da `kubectl scale` radi i za prodavnice

Kubernetes ima standardan način da se bilo šta skalira: `kubectl scale ... --replicas=N`. Da bi to
radilo i za naš Shop, CRD mora da "izloži" tri informacije: gde u spec-u piše željeni broj, gde u
statusu piše trenutni broj, i po kojoj labeli se nalaze podovi.

Caka kod nas: broj replika ne piše direktno — piše `availability: standard/high`. A skaliranje
traži broj. Rešenje: dodali smo **opciono** polje `spec.replicas`. Ako je prazno — važi pravilo
standard=2/high=3. Ako ga neko postavi (ručno ili automatika) — **ono pobeđuje**. Na odbrani:
`kubectl scale shop moj-shop --replicas=4` → za par sekundi 4 kopije, i OSTANE 4 (operator ga ne
vraća, jer i on poštuje isto pravilo). Time je Shop spreman i za **HPA** (autoscaler — automatika
koja dodaje/oduzima kopije prema opterećenju) — jer se sva automatika kači na taj isti standardni
"scale" priključak.

### 4.9 make run vs prava instalacija

- `make run` — operator radi kao običan program na mom laptopu, sa MOJIM (admin) pravima. Super za
  razvoj, ali laže: sa admin pravima sve prolazi, pa se ne vidi da li su RBAC dozvole potpune.
- Prava instalacija — operator radi **kao pod u klasteru**, sa svojim identitetom i samo svojim
  dozvolama. Kod nas se instalira iz **Helm paketa sa DockerHub-a** (glava 11). Na odbrani radi
  ovako, jer je asistent izričito rekao da mora end-to-end iz klastera.

Ako operator ima više kopija, mora **leader election** (izbor vođe): samo jedna kopija stvarno
radi, ostale su rezerva — inače bi se dve kopije utrkivale oko istih zapisa. Mi vozimo jednu
kopiju, ali chart ima prekidač za ovo i znam zašto postoji.

---

## 5. Naša tri CRD-a

### 5.1 Shop — prodavnica

Sve već opisano gore: spec (želja korisnika) → operator digne bazu + 2-3 kopije aplikacije +
adresu + nadzor + alarme; status (stvarnost) → semafori, adresa, broj spremnih kopija.

Šta se desi kad korisnik OBRIŠE prodavnicu (ovo vole da pitaju, redosled je zlato):
1. Kubernetes vidi kočnicu (finalizer) → zapis u "Terminating".
2. Naša petlja obriše Grafana tablu tog korisnika (spoljna stvar), skine kočnicu.
3. Zapis nestane → đubretar krene: briše Deployment (pa on svoje podove), Service, Ingress,
   Secrete, zahtev za bazu (pa operator baze obriše bazu i njen disk), nadzorne zapise...
4. Discord kanal zapisa ima svoju kočnicu → njegov kontroler obriše kanal NA Discord serveru, pa
   pusti.
5. Fioka korisnika ostane čista kao da prodavnice nije ni bilo.

### 5.2 DiscordChannel — kanal za obaveštenja

Spec: ID Discord servera + ime kanala + gde je tajni token bota. Kontroler preko Discord API-ja
napravi kanal i **webhook** (webhook = specijalna URL adresa; ko god na nju pošalje poruku, ona se
pojavi u kanalu — ne treba mu nalog, samo ta adresa). Webhook adresu upiše u Secret
`<ime>-webhook`, i tu adresu onda koriste i alarmi i prodavnica (za poruku o porudžbini).

Dve fine cake koje znam da ispričam:
- **Prvo upiši ID kanala u status, pa tek onda pravi webhook.** Da je obrnuto i webhook korak
  pukne, ponovni prolaz ne bi znao da kanal već postoji → napravio bi dupli.
- Ime webhooka je fiksno `shophub-alerts`, jer Discord **odbija** webhook imena koja sadrže reč
  "discord" — a ime izvedeno iz imena prodavnice može slučajno da je sadrži.

### 5.3 Wallet — novčanik

Spec: mreža (Sepolia) i opciono postojeća adresa. Ako adrese nema, kontroler **generiše par
ključeva** (privatni + javni; iz javnog se izvodi adresa — pojmovi u glavi 8), privatni ključ
sačuva u Secret, adresu upiše u status. ShopHub ima dugme "generiši mi novčanik" koje napravi ovaj
zapis i sačeka adresu. Bez finalizera — objašnjeno u 4.6.

---

## 6. Baze

### 6.1 Zašto bazu ne diže naš operator sam?

Jer je to ozbiljno teško: baza mora da preživi restart (podaci na disku!), da ima rezervne kopije,
zamenu pri kvaru, bezbedno čuvanje lozinki... Asistent je rekao doslovno: "pisanje kontrolera za
postgres je jako teško i ne bi bilo primenljivo u industriji". Pravilo struke: **za standardne
stvari koristi gotove operatore**. Naš operator je zato "dirigent": on samo NAPIŠE ZAHTEV
("hoću postgres bazu za moj-shop"), a **tuđi operator** tu bazu stvarno digne i održava.

### 6.2 CNPG — operator za PostgreSQL

**CloudNativePG (CNPG)** je poznat, open-source operator za PostgreSQL. Naš operator napravi njegov
zapis vrste `Cluster` sa: 1 instanca, 1Gi diska, i **početnim SQL-om** koji odmah napravi naše
tabele (`items` — proizvodi, `orders` — porudžbine).

Dve cake iz prakse (i asistent ih je pominjao):
- Taj početni SQL izvršava **glavni (superuser) korisnik baze**, pa tabele pripadaju njemu — a
  aplikacija se kači kao običan korisnik i dobila bi "nemaš pravo". Zato SQL izričito kaže "vlasnik
  ove tabele je aplikacioni korisnik" (`ALTER TABLE ... OWNER TO ...`).
- Imena prodavnica smeju da imaju crticu (`moj-shop`), a u SQL-u ime sa crticom mora **pod
  navodnike** — sitnica koja obara ako se ne zna.

Kad je baza spremna, CNPG objavi **Secret** `<ime>-app` u kome je sve za konekciju (adresa, korisnik,
lozinka, i gotov "connection string" — jedan dugačak URL sa svim tim). Naš operator taj URL prosledi
aplikaciji kao environment varijablu `DATABASE_URL`. Lozinka NIKAD ne prođe kroz naš kod niti kroz
git — samo pokazujemo na Secret.

### 6.3 MongoDB umesto Redis-a — i tri ratne priče

Specifikacija je za "laku" bazu predlagala Redis, ali je izričito dozvolila zamenu (npr. MongoDB)
**pod uslovom da se koristi operator te baze**. Probali smo Redis: jedan operator je mrtav projekat
(image mu više ne postoji na registru!), drugi traži plaćenu licencu. Uzeli smo **MongoDB Community
Operator** — živ i besplatan. Ovu odluku smo zapisali u decision log (dnevnik odluka sa datumima i
razlozima) — na odbrani mogu da ga otvorim.

Tri problema koja smo stvarno rešavali (odlične priče jer pokazuju da je rađeno, ne prepisano):
1. MongoDB-ov Go tip ima bag zbog kog standardna funkcija za lepljenje vlasničke veze ne radi —
   morali smo cedulju da zalepimo "ručno", direktno na polje.
2. MongoDB operator po defaultu gleda samo SVOJU fioku — a naše baze su po korisničkim fiokama.
   Instaliran je sa podešavanjem "gledaj sve fioke".
3. Podovima baze treba određeni identitet (ServiceAccount) koji operator instalira samo u svojoj
   fioci — pa NAŠ operator taj identitet (plus dozvole) sam pravi u svakoj korisničkoj fioci.

I lukavstvo za kraj: MongoDB-ov Secret sa konekcijom smo namerno nazvali isto kao CNPG-ov
(`<ime>-app`), pa ceo ostatak koda ne mora da zna koja je baza u pitanju.

### 6.4 Kako aplikacija bira bazu

Aplikacija prodavnice pri pokretanju pogleda `DATABASE_URL`: ako počinje sa `postgres://` koristi
Postgres kod, ako sa `mongodb://` koristi Mongo kod. Unutra postoji **interfejs Store** — ugovor
("ovo su operacije nad podacima: daj proizvode, napravi porudžbinu...") sa dve implementacije.
Ostatak aplikacije priča samo sa ugovorom i ne zna ni ne mari koja je baza ispod.

---

## 7. Shop aplikacija

### 7.1 Šta je u njoj

Jedan image sadrži i **backend** (Go program koji ima API — adrese tipa `/api/items` preko kojih se
čitaju/menjaju podaci) i **frontend** (sajt u React-u koji kupac vidi; backend ga servira). Dodatno
backend ima: `/metrics` (brojači za nadzor), `/probe/...` (kontrolne tačke za Kubernetes) i
pozadinsku petlju za proveru plaćanja.

Uloge: **admin** (vlasnik) sme da dodaje/menja proizvode i gleda porudžbine — čuva ga lozinka koju
je operator generisao; **kupac** ne treba nalog — razgleda, puni korpu, plaća novčanikom.

### 7.2 Kupovina korak po korak (nauči ovaj tok!)

1. Kupac klikne "kupi". Frontend pozove backend: "napravi porudžbinu: 2 komada proizvoda X".
2. Backend **odmah rezerviše robu**: u JEDNOJ operaciji nad bazom proveri "ima li bar 2 na stanju"
   i smanji stanje za 2. Zato dva kupca ne mogu da kupe isti poslednji komad — baza garantuje da
   samo jedan prođe. Porudžbina se upiše kao **pending** (na čekanju).
3. Backend sam izračuna cenu (cena iz baze × količina). **Nikad ne veruje ceni iz browsera** — u
   suprotnom bi neko veštim zahtevom "kupio za 0.01".
4. Frontend otvori MetaMask (novčanik — glava 8); kupac potvrdi slanje USDT na adresu prodavnice.
5. MetaMask vrati **tx hash** (broj potvrde transakcije); frontend ga zakači na porudžbinu.
6. **Pozadinska petlja** (svakih 15 sekundi) uzme sve pending porudžbine i za svaku proveri na
   blockchainu: da li je transakcija prošla i da li je STVARNO stigao dovoljan iznos na NAŠU adresu?
   Ako da → porudžbina **confirmed** (potvrđena). Ako je transakcija propala → **failed** + roba se
   vrati na stanje.
7. Ako kupac odustane (zatvori MetaMask i ode): porudžbina bez tx hasha posle **30 minuta ističe**
   i roba se vraća na stanje — da napušteni pokušaji ne drže rezervaciju zauvek.

### 7.3 Zašto petlja, a ne "proveri odmah"?

Da provera ide odmah po zahtevu i aplikacija se u tom trenutku restartuje — porudžbina bi zauvek
ostala na čekanju. Petlja umesto toga svako malo pita BAZU "šta je sve pending?" — pa je svejedno
da li se pod restartovao: novi pod nastavi tačno gde je stari stao. I bezbedno je sa više kopija:
potvrda je uslovna izmena ("potvrdi SAMO ako je još pending"), pa ako dve kopije istovremeno
potvrđuju istu porudžbinu, uspe samo jedna — i samo ta jedna pošalje Discord poruku o porudžbini.
Primeti: ovo je ISTA filozofija kao operator (stanje u bazi, ne u pamćenju; ponovljivo bez štete).

Još jedna zaštita: šta ako neko isti tx hash zakači na dve porudžbine ("plaćam obe istom uplatom")?
Petlja sabere iznose SVIH porudžbina sa tim hashom i traži da uplata pokriva ZBIR. Legalno je kad
je to jedna korpa plaćena odjednom; prevara ne prolazi jer zbir premaši uplaćeno.

---

## 8. Web3

### 8.1 Pojmovi od nule

- **Blockchain**: javna knjiga transakcija koju ne vodi jedna firma nego hiljade računara; upisano
  se ne može obrisati ni prepraviti. Mi koristimo **Ethereum**.
- **Novčanik (wallet)**: par ključeva. **Privatni ključ** = tajna kojom potpisuješ (ko ga ima —
  gazduje parama; čuva se kao Secret!). **Javna adresa** (ono `0x...`) = izvedena iz ključa, kao
  broj računa — slobodno se deli. **MetaMask** je browser dodatak koji čuva ključ i potpisuje
  umesto tebe kad klikneš "potvrdi".
- **Token / ERC-20**: valuta koja živi NA Ethereumu kao program (pametni ugovor — smart contract).
  **ERC-20** je standard koji propisuje koje funkcije takav program mora imati (`transfer` — pošalji,
  `balanceOf` — stanje...). **USDT** je najpoznatiji takav token, "vezan" za dolar.
- **Testnet (Sepolia)**: probna kopija Ethereuma sa bezvrednim parama — za učenje i razvoj.
  Specifikacija izričito kaže da je testnet dovoljan.
- **Gas**: naknada mreži za obradu transakcije; plaća je pošiljalac, u ETH (testni ETH se dobija
  besplatno sa "slavina" — faucet sajtova).
- **Tx hash**: jedinstveni broj transakcije — "broj potvrde o uplati" po kom svako može da je nađe
  (npr. na sajtu Etherscan).

### 8.2 Naš mali trik: sopstveni USDT

"Pravi" USDT na testnetu ne postoji kanonski. Zato sam ja preko **Remix-a** (online alat za pametne
ugovore) **objavio sopstveni ERC-20 token** nazvan USDT na Sepoliji: 6 decimala kao pravi, i sa
otvorenom `mint` funkcijom — svako može sebi da "iskuje" tokene, što nam služi kao slavina za demo.
Backendu je svejedno — ERC-20 je ERC-20; adresa ugovora je samo podešavanje.

Decimale, ljudski: token nema zarez — sve su celi brojevi u najsitnijim jedinicama, a "decimals"
kaže gde zamišljeni zarez stoji. 5 USDT sa 6 decimala = 5.000.000 jedinica. U kodu za novac NIKAD
ne koristimo obične decimalne brojeve (float) jer oni greše na sitnicama (0.1+0.2 ≠ 0.3 u floatu!)
— koristimo velike cele brojeve i tekst.

### 8.3 Kako backend PROVERAVA uplatu (česta zamka!)

Intuicija kaže: "pogledaj kome su pare poslate". Ali kod tokena transakcija ide OVAKO: kupac šalje
transakciju **programu tokena** (ne prodavnici!) sa porukom "prebaci N mojih jedinica na adresu
prodavnice". Znači polje "primalac transakcije" je adresa TOKENA, a iznos ETH-a je 0. Pravi dokaz
je u **receiptu** (potvrdi izvršenja): tamo token program ostavi **event** (zapisnik) `Transfer`
sa: ko šalje, kome, koliko.

Naša provera, redom: nađi receipt po tx hashu (nema ga još → sačekaj, transakcija se možda još
"kuva") → da li je transakcija USPELA (i propale imaju receipt!) → prođi kroz evente → postoji li
`Transfer` koji je (a) ispisao BAŠ NAŠ token program, (b) primalac BAŠ NAŠA adresa, (c) iznos ≥
očekivanog? Sve tri da → potvrđeno.

---

## 9. ShopHub

### 9.1 Registracija i lozinke

Lozinke se u bazi NE čuvaju čitljivo, nego kao **bcrypt hash**. Hash = jednosmerno mlevenje: iz
lozinke se lako dobije hash, iz hasha se lozinka ne može vratiti. Pri prijavi se uneta lozinka
samelje i uporedi sa sačuvanim. bcrypt je poseban po tome što je **namerno spor** — pa napadač koji
ukrade bazu ne može brzo da isprobava milione lozinki. (Za poređenje: običan SHA je munjevit — loš
za lozinke.)

### 9.2 JWT — propusnica

Posle prijave server izda **JWT (token)**: tri dela — podaci ("ko si" + **koja je tvoja fioka/
namespace**), rok važenja (24h), i **potpis** napravljen tajnim ključem servera. Browser token
šalje uz svaki zahtev. Server proveri potpis: ako je iko promenio i slovo u podacima, potpis se ne
poklapa → token nevažeći. Zato korisnik NE MOŽE da u tokenu prepravi svoju fioku u tuđu.

Zašto je fioka u tokenu genijalno prosto: svaki zahtev odmah zna gde sme da gleda, bez čitanja
baze. Handler doslovno kaže "listaj Shop-ove u fioci iz tokena" — tuđa fioka se u toku zahteva
nigde i ne pominje. To je celo srce multi-tenancy izolacije.

(Sitnica za ekstra utisak: proveravamo i da je algoritam potpisa baš onaj naš — postoji stari napad
gde napadač podmetne token "bez potpisa".)

### 9.3 Kako ShopHub pravi prodavnicu

ShopHub backend ima **klijent za Kubernetes** (istu biblioteku koristi i operator) i identitet sa
dozvolama: sme da pravi fioke i Shop/Wallet/DiscordChannel zapise. Kad korisnik klikne "napravi":
backend napravi Shop zapis u NJEGOVOJ fioci — i to je SVE. Dalje preuzima operator. ShopHub ne zna
ništa o bazama, replikama, ingresima — i ne treba da zna. Lepa podela: panel piše želje, operator
ih ostvaruje.

Samopopravka koju volim da pomenem: fioka se pravi pri registraciji, ali se "proveri i napravi ako
fali" izvršava i pri SVAKOJ prijavi — pa ako je nešto nekad puklo ili je fioka obrisana, prva
sledeća prijava je tiho popravi. (Idempotentnost opet!)

---

## 10. Observability

**Observability** (uvid u sistem) = da u svakom trenutku možeš da odgovoriš "šta se dešava i
zašto". Tri vrste podataka, tri alata + Grafana kao zajednički ekran:

| Šta | Ljudski | Alat kod nas |
|---|---|---|
| **Metrike** | brojčane vrednosti kroz vreme (broj zahteva, CPU...) | Prometheus |
| **Logovi** | tekstualni dnevnik ("primio zahtev X, vratio 404") | Loki (+ Promtail sakupljač) |
| **Trace-ovi** | put JEDNOG zahteva kroz sistem, sa trajanjima | Tempo (+ OpenTelemetry u kodu) |

### 10.1 Metrike: put od aplikacije do grafika

1. Naš backend na adresi `/metrics` izlaže brojače (naše ime im počinje sa `shop_`): ukupno
   zahteva (po metodi/ruti/statusu), trajanje zahteva, poslati bajtovi.
2. **Prometheus** je server koji periodično (kod nas na 15s) OBILAZI aplikacije i prepisuje
   brojače (to se zove **scrape**). Pitanje: kako zna koga da obilazi, kad prodavnice nastaju
   dinamički? Odgovor: naš operator za svaku prodavnicu napravi **ServiceMonitor** — zapis
   "obilazi i ovu na /metrics". Nula ručnog podešavanja po prodavnici.
3. **Grafana** crta grafike nad Prometheus podacima. Upiti se pišu jezikom **PromQL** — npr.
   "koliko zahteva u 24h" je `sum(increase(shop_http_requests_total[24h]))`.

Sve iz specifikacije (ukupno/uspešni/neuspešni zahtevi u 24h, 404 sa adresama, ukupan saobraćaj u
GB) su varijacije tog jednog upita sa filterima. CPU/RAM/disk/mreža dolaze "besplatno" od
komponenti koje stack instalira (cAdvisor po podu, node-exporter po računaru).

Dva bug-a odavde koja su odlične priče: (1) brojanje "jedinstvenih posetilaca" NE ide u metrike
(previše različitih vrednosti IP+browser ubija Prometheus) — to rešavamo upitom nad LOGOVIMA u
Lokiju; (2) Go biblioteka vraća veličinu odgovora **-1** dok odgovor nije napisan, a dodavanje
negativnog broja u brojač obara program — našli smo to uživo kad je 404 stranica počela da vraća
500. Guard `if size > 0` i priča ima happy end.

### 10.2 Dashboardi i "svako vidi samo svoje"

Spec: svaka prodavnica ima SVOJU kontrolnu tablu. Operator za svaki Shop uzme šablon (ugrađen JSON),
zameni ime, i sačuva kao ConfigMap sa posebnom labelom — Grafanin pomoćnik (sidecar) takve
automatski učita. Tabla je "u vlasništvu" Shop-a → nestane sa prodavnicom.

Opcioni zahtev (bonus bodovi): korisnik sme da vidi SAMO svoje table. U besplatnoj Grafani prava po
folderima ne postoje kako treba — ali postoje **organizacije** (potpuno odvojeni svetovi unutar iste
Grafane). Pa: pri registraciji ShopHub napravi organizaciju za tog korisnika + njegov nalog samo u
njoj; operator svaku tablu gura i u organizaciju vlasnika. Mi (maintaineri) u glavnoj organizaciji
vidimo sve; korisnik u svojoj vidi samo svoje. Čišćenje table pri brisanju prodavnice radi onaj
finalizer iz 4.6.

### 10.3 Alarmi: put od problema do Discord poruke

1. **PrometheusRule** = zapis sa pravilima. Naša: **ShopHighErrorRate** (>10% zahteva su greške
   kroz 5 min), **ShopHighLatency** (95% zahteva sporije od 1s), **ShopDown** (prodavnica ne
   odgovara / ugašena), **NodeHighCPU/NodeHighMemory** (računar klastera >90%).
2. Prometheus pravila stalno računa; kad je uslov ispunjen dovoljno dugo → alarm **fire**-uje.
3. Alarm ode u **Alertmanager** — komponentu za "kome ovo javiti" (grupisanje, gde poslati).
4. Naš operator za svaku prodavnicu napravi **AlertmanagerConfig**: "alarme šalji na Discord
   webhook ove prodavnice". Podešeno je (OnNamespace strategija) da svaki takav config važi SAMO
   za alarme iz svoje fioke — pa alarmi prodavnice A ne mogu u kanal prodavnice B.
5. Klasterski alarmi nemaju "svoju prodavnicu" — za njih postoji poseban **maintainers kanal**
   (`#cluster-alerts`), napravljen kroz naš sopstveni DiscordChannel CRD.

Demo na odbrani: pustim petlju zahteva na nepostojeću adresu (404-e) → posle ~1 min stigne poruka
u Discord kanal te prodavnice; prekinem → stigne "rešeno". Za klasterski: pod koji vrti prazne
petlje na svim jezgrima → NodeHighCPU → poruka u #cluster-alerts.

Ratna priča vredna pominjanja: prvi pokušaj slanja na Discord je bio kroz direktan config sa
poljem koje pokazuje na fajl sa tajnom — ali alat to polje ne poznaje i odbija ceo config. Jedini
način da webhook adresa NE završi u git-u je bio kroz AlertmanagerConfig CRD koji ume da pokaže na
Secret. Takvi detalji se ne nauče iz slajdova — vidi se da je rađeno.

---

## 11. Helm

### 11.1 Šta je i zašto

Da instaliraš aplikaciju u Kubernetes treba ti hrpa YAML fajlova koji se razlikuju od okruženja do
okruženja (druga adresa, druga verzija...). **Helm** je "menadžer paketa" za Kubernetes (kao
prodavnica aplikacija): paket se zove **chart** i sadrži YAML **šablone** + **values.yaml** (spisak
podesivih vrednosti). `helm install` = uzmi šablone, ubaci vrednosti, primeni na klaster.

Naša tri charta (u `helm-charts` repou — ovo je deo ELIMINACIONOG zahteva, struktura mora da se zna):
- **shop-operator** — instalira operator: njegov Deployment, dozvole (RBAC), i **CRD-ove** (koji
  stoje u posebnom `crds/` folderu — Helm njih instalira PRE svega ostalog i ne dira ih pri
  nadogradnji, jer bi brisanje CRD-a obrisalo SVE prodavnice).
- **shophub** — instalira panel: aplikaciju, dozvole, bazu za korisnike, tajni ključ za tokene.
  Deklariše i kube-prometheus-stack kao opcionu zavisnost (zahtev 3.3), ali je podrazumevano
  isključena — jer monitoring instaliramo JEDNOM za ceo klaster, ne uz svaku aplikaciju.
- **shop** — tanak chart koji napravi jedan Shop zapis; služi za ručnu/debug instalaciju prodavnice
  bez panela. (I da struktura odgovara onoj iz specifikacije.)

Verzije: chart ima svoju verziju (verziju PAKETA) i appVersion (verziju APLIKACIJE koju
podrazumevano instalira). CI naše chartove pakuje i kači na DockerHub kao **OCI artefakte** — isti
registar gde su i image-i ume da čuva i chartove.

### 11.2 Asistentova lekcija: ne sve odjednom

Helm samo ŠALJE fajlove — ne proverava da li sistem posle RADI. Ako u jednom potezu instaliraš i
bazu i aplikaciju, aplikacija krene pre nego što baza objavi lozinku → podovi pucaju, a Helm kaže
"uspešno". Zato se sistem podiže **u talasima**: prvo operatori (oni donose CRD-ove), pa naš
operator + alati za logove/trace, pa panel. A unutar jedne prodavnice taj isti redosled (baza → pa
aplikacija) sprovodi NAŠ operator automatski — ono što se kod asistenta radilo ručno u više
`helm install` prolaza, kod nas radi reconcile petlja.

---

## 12. GitOps

### 12.1 kube-state — sistem opisan u fajlovima

Repo `kube-state` (drugi deo eliminacionog zahteva) sadrži: opis samog klastera (k3d podešavanja)
i po folder za svaku komponentu — koji chart, koja verzija, koja podešavanja. Poenta: **ceo sistem
se može podići od nule samo iz git-a** (SETUP.md vodi korak po korak — testirano). Ništa "neko je
nešto nekad ručno kliknuo pa ne znamo šta".

Tajne su izuzetak: tri tajne (Grafana admin lozinka, Discord bot token, JWT ključ) se naprave ručno
kao Secreti pre instalacije — fajlovi u git-u pominju samo NJIHOVA IMENA, nikad sadržaj.

### 12.2 ArgoCD — čuvar koji sistem drži jednakim git-u

**GitOps** = git je jedina istina o tome kako klaster treba da izgleda, a u klasteru radi agent
koji stvarnost stalno poravnava sa git-om. Naš agent je **ArgoCD**:

- za svaku komponentu postoji **Application** zapis ("prati ovaj chart ove verzije sa ovim
  vrednostima");
- **app-of-apps**: jedan koreni Application pokazuje na folder sa svim ostalima — pa je podizanje
  celog sistema: instaliraj ArgoCD + primeni JEDAN fajl;
- uključeno je: automatska primena izmena iz git-a, **selfHeal** (neko ručno promeni nešto u
  klasteru → ArgoCD vrati na git stanje), **prune** (obrisano iz git-a → obriše se iz klastera).

Demo poslastica: obrišem operator Deployment naživo → ArgoCD ga za koji sekund vrati. To je
termostat-ideja treći put: operator čuva prodavnice, Kubernetes čuva podove, ArgoCD čuva ceo sistem.

---

## 13. Git, CI/CD

### 13.1 Kako radimo sa git-om

- **Trunk Based Development**: jedna glavna grana (`main`), izmene idu kroz kratkoživeće grane
  (dan-dva) i **Pull Request** (PR = "molim da se ovo primi u glavno" + prikaz izmena za pregled).
- **Branch protection** (zaštita grane): na `main` se NE MOŽE direktno; PR mora imati bar 1
  odobrenje + sve automatske provere zelene; istorija mora biti **linearna** (svaki PR postane
  tačno jedan commit — "squash"). Praktično: pokvaren kod fizički ne može u main.
- **Conventional Commits**: poruke u formatu `tip(oblast): opis` (`feat(shop): ...`, `fix(...)`).
  Automat (commitlint) odbija pogrešan format. Zašto: iz ovakvih poruka se automatski izvlači šta
  je nova funkcionalnost a šta popravka → verzije i changelog.

### 13.2 Šta se automatski dešava (CI/CD)

**CI** (continuous integration) — na svaki PR: provera poruka, lint (analiza koda), build + testovi
(uključujući **Testcontainers**: test sam podigne PRAVU bazu u docker kontejneru, izvrti test nad
njom i ugasi je — mnogo verodostojnije od lažne baze), probni build image-a + hadolint.

**CD** (continuous delivery) — na prihvatanje u main: image se sagradi i okači na DockerHub; chart
se zapakuje i okači kao OCI. Na **git tag** `vX.Y.Z` se objavi "prava" verzija po **SemVer**
pravilima (MAJOR.MINOR.PATCH: popravka diže PATCH, nova funkcionalnost MINOR, nekompatibilna
promena MAJOR).

Krug se zatvara ovako: kod → PR → CI provere → merge → image/chart na registru → podigneš verziju
u kube-state → ArgoCD to primeni na klaster. **Niko nigde ništa ručno ne instalira.**

---

## 14. Kratka pitanja

Brzi odgovori od 2-4 rečenice — za zagrevanje pred odbranu. (Duge verzije su u glavama iznad.)

**Šta je operator?** Program koji proširuje Kubernetes novim tipom resursa (kod nas Shop) i u
petlji dovodi stvarnost u stanje opisano u tom resursu — termostat za prodavnice.

**Šta je razlika spec i status?** Spec je želja i piše je korisnik; status je stvarnost i piše je
operator. Operator ne sme da menja spec — čak i tehnički ne može, jer status ide kroz poseban
endpoint sa posebnim dozvolama.

**Šta znači da je reconciler idempotentan i zašto mora?** Više uzastopnih poziva = isti efekat kao
jedan (f(f(x))=f(x)). Mora jer se petlja poziva mnogo puta za isti objekat; bez toga bi dupli
poziv pravio duple resurse (npr. dva Discord kanala).

**Šta se desi kad obrišem Shop?** Finalizer prvo počisti spoljne stvari (Grafana tabla), zatim
Garbage Collector preko vlasničkih veza obriše sve u klasteru (Deployment, Service, bazu...), a
Discord kanal obriše njegov kontroler kroz svoj finalizer — kanal se briše i NA Discord serveru.

**Zašto MongoDB a ne Redis?** Redis operatori nisu upotrebljivi (jedan mrtav — image mu ne
postoji, drugi traži licencu), a spec izričito dozvoljava zamenu uz uslov da se koristi operator
te baze — MongoDB Community Operator je živ i besplatan. Odluka je u decision logu.

**Zašto standard=2 a high=3 replike?** Zahtev iz specifikacije; sprovodi ga operator pri kreiranju
Deploymenta. Uz to smo dodali opcioni replicas override pa `kubectl scale shop` i HPA rade preko
standardnog scale priključka.

**Kako znate da je uplata stvarno stigla?** Backend nađe receipt transakcije po tx hashu, proveri
da je uspešna, pa u eventima traži Transfer koji je ispisao naš token ugovor, ka našoj adresi, sa
dovoljnim iznosom. Polje "primalac transakcije" je kod tokena adresa UGOVORA, pa se mora gledati
event — česta zamka.

**Šta ako pod umre usred plaćanja?** Ništa strašno: stanje porudžbina je u bazi, a provera je
petlja koja svakih 15s pita bazu šta je pending — novi pod nastavlja gde je stari stao. Potvrda je
uslovna izmena pa ni dve replike ne mogu duplo da potvrde.

**Kako korisnik ne vidi tuđe prodavnice?** JWT nosi njegov namespace; middleware ga izvuče iz
potpisanog tokena i svaki upit ka Kubernetesu je ograničen na taj namespace. Tuđi namespace se u
toku zahteva nigde ne pojavljuje, a token se ne može falsifikovati jer bi potpis pukao.

**Kako Prometheus sazna za novu prodavnicu?** Operator uz svaku prodavnicu napravi ServiceMonitor;
prometheus-operator iz njih generiše listu koga da obilazi. Nula ručne konfiguracije.

**Zašto readiness proverava bazu a liveness ne?** Restart ne popravlja pokvarenu bazu — samo bi
napravio lančano restartovanje. Ovako pod bez baze samo privremeno izađe iz saobraćaja i sam se
vrati.

**Čemu Helm kad postoji operator?** Helm je za STATIČKI deo (instalacija operatora, monitoringa,
panela — jednom, u talasima); operator je za DINAMIČKI deo (prodavnice nastaju i nestaju
klikovima). Helm šalje fajlove i ne proverava ništa posle; operator neprestano proverava.

**Šta vam radi ArgoCD?** Drži klaster jednakim git-u: automatski primenjuje izmene, vraća ručne
izmene (selfHeal), briše uklonjeno (prune). App-of-apps: ceo sistem se podiže jednim manifestom.

**Gde su vam tajne?** Nigde u git-u. Tri se prave ručno kao Secreti (Grafana admin, Discord token,
JWT ključ), sve ostale generišu operatori u klasteru. Fajlovi u git-u pominju samo imena Secreta.

---

## 15. Rečnik

Abecedno ne — po srodnosti, da se lakše pamti:

**Kontejneri:** *image* = paket programa sa svime što mu treba; *kontejner* = pokrenut image;
*registry/DockerHub* = server za image-e; *tag* = verzija image-a; *Dockerfile* = recept;
*multi-stage* = build u radionici, u finalni paket ide samo rezultat; *non-root* = program ne radi
kao administrator; *exec forma* = program je glavni proces pa mu signali stižu; *SIGTERM* = "molim
te ugasi se"; *graceful shutdown* = uredno gašenje; *hadolint* = lektor za Dockerfile.

**Kubernetes osnovno:** *klaster* = skup računara pod jednom upravom; *nod* = jedan računar;
*k3d* = mini klaster na laptopu; *kubectl* = komanda za razgovor sa klasterom; *YAML* = format
opisa; *pod* = omotač kontejnera, smrtan i zamenjiv; *Deployment* = "drži N kopija"; *replika* =
jedna kopija; *Service* = stalna adresa + raspodela saobraćaja; *Ingress* = ulaz spolja po imenu
sajta; *namespace* = fioka; *ConfigMap* = podešavanja; *Secret* = tajne; *env varijabla* =
vrednost koju program pročita pri pokretanju; *liveness/readiness* = "jesi li živ / jesi li
spreman"; *RBAC* = ko šta sme; *ServiceAccount* = lična karta programa.

**Kubernetes iznutra:** *API server* = jedini koji piše u bazu stanja; *etcd* = sveska stanja;
*kontroler* = termostat; *watch* = pretplata na promene; *keš* = lokalna kopija; *work queue* =
red čekanja bez duplikata; *level-based* = poravnavaj se sa stanjem, ne jurcaj za događajima;
*409 Conflict* = "imaš zastarelu verziju, probaj opet"; *predicate* = filter "tiče li me se";
*FieldIndexer* = indeks za brzu pretragu.

**Operator:** *CRD* = definicija novog tipa resursa (obrazac); *CR* = konkretan zapis (popunjen
obrazac); *Kubebuilder* = alat koji generiše kostur operatora; *marker* = komentar iz kog se
generiše validacija; *Reconcile* = petlja pomirenja želje i stvarnosti; *idempotentnost* =
ponavljanje bez dodatnog efekta; *RequeueAfter* = "probaj opet za N sekundi"; *conditions* =
semafori u statusu (Available/Progressing/Degraded); *OwnerReference* = cedulja "moj vlasnik je...";
*Garbage Collector* = đubretar koji briše sve što je pripadalo obrisanom; *finalizer* = kočnica za
brisanje dok se ne počisti spoljni svet; *scale subresource* = standardni priključak za skaliranje;
*leader election* = kad ima više kopija operatora, samo vođa radi.

**Baze:** *CNPG* = operator za PostgreSQL; *MongoDB Community Operator* = operator za MongoDB;
*connection string* = ceo pristup bazi u jednom URL-u; *`<ime>-app` Secret* = tu operator baze
objavi pristup; *Store interfejs* = ugovor u kodu, dve implementacije (postgres/mongo).

**Web3:** *blockchain* = javna neizbrisiva knjiga; *Ethereum/Sepolia* = mreža / njen testnet;
*wallet* = par ključeva; *privatni ključ* = tajna koja potpisuje; *adresa* = broj računa;
*MetaMask* = browser novčanik; *smart contract* = program na blockchainu; *ERC-20* = standard za
tokene; *USDT* = dolar-token (naš je self-deployed sa otvorenim mintom); *decimals* = gde stoji
zamišljeni zarez; *gas* = naknada mreži; *tx hash* = broj potvrde; *receipt* = potvrda izvršenja
sa eventima; *Transfer event* = zapisnik "ko→kome→koliko"; *faucet* = slavina za testne pare.

**ShopHub:** *bcrypt* = spor jednosmerni hash za lozinke; *JWT* = potpisana propusnica sa rokom;
*claims* = podaci u propusnici (kod nas i namespace); *middleware* = kontrola na ulazu svakog
zahteva; *multi-tenancy* = više korisnika bezbedno na istoj platformi; *tenant* = jedan korisnik.

**Observability:** *metrike/logovi/trace-ovi* = brojke / dnevnik / put jednog zahteva;
*Prometheus* = sakuplja metrike (scrape); *ServiceMonitor* = "obilazi i ovoga"; *PromQL* = jezik
upita; *Grafana* = ekran za sve; *dashboard* = kontrolna tabla; *Grafana organizacija* = odvojen
svet u Grafani (naša izolacija po korisniku); *Loki/Promtail* = logovi; *Tempo/OpenTelemetry* =
trace-ovi; *PrometheusRule* = pravila alarma; *fire* = alarm se upalio; *Alertmanager* = "kome ovo
javiti"; *AlertmanagerConfig* = uputstvo rutiranja po fioci; *webhook* = URL na koji se šalju
poruke u kanal.

**Helm/GitOps/CI:** *chart* = paket za instalaciju; *values.yaml* = podesive vrednosti;
*template* = YAML šablon; *crds/ folder* = CRD-ovi koji se instaliraju prvi i ne diraju;
*OCI artefakt* = chart okačen na docker registar; *kube-state* = repo sa opisom celog sistema;
*GitOps* = git je istina, agent poravnava; *ArgoCD* = naš agent; *Application* = "prati ovaj
chart"; *app-of-apps* = koren koji pokazuje na sve ostale; *selfHeal/prune* = vrati ručne izmene /
obriši uklonjeno; *PR* = zahtev za prijem izmene; *branch protection* = zaštita glavne grane;
*squash* = ceo PR postane jedan commit; *Conventional Commits* = format poruka `tip(oblast): opis`;
*commitlint* = automat koji format proverava; *CI/CD* = automatske provere / automatska isporuka;
*Testcontainers* = test sam digne pravu bazu u kontejneru; *SemVer* = MAJOR.MINOR.PATCH;
*required check* = provera bez koje se PR ne prima.

---

*Predlog kako da učiš iz ovoga: pređi glave 0–4 dok ne budu "pričljive" (da možeš naglas da ih
prepričaš) — to je 70% odbrane. Onda 6, 8 i 10 (baze, web3, observability). Glave 11–13 su
najlakše za na kraju. Pa tek onda otvori `odbrana.md` — videćeš da ti je odjednom čitljiv — i
`asistent_beleske_odbrana.md` za završni sloj. I obavezno bar dvaput provežbaj demo iz odbrana.md
sekcije 17.*
