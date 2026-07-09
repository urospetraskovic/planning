# Asistentove vežbe, objašnjene od nule — verzija za razumevanje

> Ovo je "spora" verzija fajla `asistent_beleske_odbrana.md`. Ideja je ista: da na odbrani
> pokažem da sam slušao vežbe, tako što ubacim tačno ono što je ON naglašavao. Ali ovde svaku
> njegovu poentu prvo raspakujem do kraja — šta je rekao, šta to ZNAČI ljudskim jezikom, zašto je
> to pametno, i tek onda gotova rečenica za odbranu.
>
> Zašto ovo uopšte radi: profesori i asistenti boduju znanje, ali NAJVIŠE veruju studentu za kog
> vide da je pratio proces. Kad u odgovoru prepoznaju svoju rečenicu sa vežbi — i vide da je
> razumeš, ne samo ponavljaš — dobijaš kredibilitet koji vredi više od naštrebane definicije.
>
> **Kako se koristi (isto pravilo kao ranije):** ne recituješ napamet. Čekaš pitanje → odgovoriš
> svojim rečima → pa TEK ONDA zakačiš "to ste i na vežbama naglasili kad ste..." + gde je to kod
> nas. Jedna referenca po temi. I nikad ne pominji ono što ne umeš da odbraniš dubinski — svaka
> referenca poziva potpitanje.

---

## Sadržaj

- [1. Vežbe 3 — Docker](#1-vezbe-3--docker)
- [2. Vežbe 4 — kako se piše operator](#2-vezbe-4--operator)
- [3. Vežbe 5 — brisanje, skaliranje, baze, Helm, finalizeri](#3-vezbe-5)
- [4. Njegove rečenice — svaka objašnjena](#4-njegove-recenice)
- [5. Tabela: on je rekao → mi smo uradili](#5-tabela)
- [6. Tri najjača aduta + kako da ne preteram](#6-aduti)

---

## 1. Vežbe 3 — Docker

### 1.1 "Mora build faza, i mora da se razdvaja" — multi-stage

**Šta je rekao:** Dockerfile mora da ima više faza: posebna faza za GRADNJU aplikacije, posebna
finalna faza u kojoj je samo ono što se IZVRŠAVA.

**Šta to znači, od nule:** kad praviš program, treba ti gomila alata — kompajler, biblioteke,
pomoćni fajlovi. Ali kad program RADI, ništa od toga mu više ne treba. Multi-stage build znači:
prva faza je **radionica** (svi alati, prljavo, veliko), a u finalni paket (image) se prenese
SAMO gotov proizvod. Kao stolar: u radionici ima testere, lepak, strugotinu — ali tebi u stan
stigne samo sto.

**Zašto je to pametno:** (1) finalni paket je 10x manji → brže se skida i pokreće; (2) bezbednije
je — u paketu nema alata kojima bi haker mogao nešto da prepravi i ponovo sagradi.

**Gotova rečenica:**
> "Sva tri naša Dockerfile-a su multi-stage kao što ste tražili — kod shophub-a su čak tri faze:
> Node faza sagradi sajt, Go faza iskompajlira backend, a u finalnu alpine sliku ide samo izvršni
> fajl i gotovi fajlovi sajta. Ni kompajler ni izvorni kod ne postoje u finalnoj slici."

### 1.2 "Kompajlirani jezici su u prednosti" — jedan fajl vs ceo kod

**Šta je rekao:** kad se Go/Rust program sagradi, dobiješ JEDAN binarni fajl i samo njega nosiš i
štitiš. Kod Pythona/Node-a moraš da nosiš SAV izvorni kod i sve biblioteke — i sve to moraš da
zaštitiš, jer ako neko u kontejneru sme da izmeni neki JS fajl, može da podmetne malicioznu
skriptu. Njegov zaključak: po pitanju bezbednosti u kontejneru, Python i Node NISU ništa bolji od
kompajliranih — naprotiv.

**Šta to znači, od nule:** interpretirani jezik (Python, JavaScript) se izvršava tako što poseban
program (interpreter) ČITA tvoj kod red po red — dakle kod mora fizički da bude prisutan. A gde
ima koda, ima i šta da se menja. Kompajlirani jezik se unapred prevede u mašinski jezik — jedan
zaključan fajl, nema šta da se čita ni prepravlja.

**Gotova rečenica:**
> "To što ste pričali o prednosti kompajliranih jezika u kontejnerima je bio jedan od razloga da
> ceo backend bude u Go-u — statički binary, `CGO_ENABLED=0`, i u release slici se štiti jedan
> fajl umesto celog source stabla."

### 1.3 "Root je vlasnik, a app korisnik samo izvršava — i root NIKAD ne pokreće kontejner"

**Šta je rekao:** nije dovoljno samo da aplikacija ne radi kao root. Treba i da fajlovi budu
podešeni tako: vlasnik binarnog fajla je root, a novi korisnik (`app`) sme samo da ga ČITA i
POKREĆE — ne i da ga menja ili briše. I kontejner se pokreće kao taj `app`, pogotovo u produkciji.

**Šta to znači, od nule:** u Linuxu svaki fajl ima vlasnika i dozvole (ko sme da čita/piše/
izvršava). Zamisli hotelsku sobu: gost (app) ima ključ i sme da koristi sobu, ali ne sme da ruši
zidove — to sme samo vlasnik hotela (root). Ako provalnik ukrade gostu ključ, i dalje ne može da
renovira. Prevod: i da haker preuzme našu aplikaciju, ne može da prepiše njen izvršni fajl niti da
instalira svoje alate.

**Gotova rečenica:**
> "Ispoštovana je tačno ona šema sa vežbi: `chown root:root` pa `chmod 0755` pa tek onda
> `USER app` — app korisnik može read i execute, write ne može. Praktična posledica je i da
> slušamo na portu 8080, jer običan korisnik ne sme na portove ispod 1024."

### 1.4 "Uvek slim; alpine ako proradi iz prve"

**Šta je rekao:** koristi smanjene (slim/alpine) verzije osnovnih slika — manje nepotrebnih paketa
znači manje ranjivosti, manju sliku, brže pokretanje. Ali budi pragmatičan: alpine ponekad traži da
ručno doinstaliraš gomilu stvari, pa se prednost istopi — ako radi iz prve, super; ako moraš da
"budžiš", razmisli.

**Šta to znači, od nule:** "bazna slika" je polazni mini operativni sistem na koji dodaješ svoj
program. Puna verzija (npr. ceo Debian) nosi stotine programa koje ne koristiš — a svaki od njih je
potencijalna rupa. Slim/alpine su iste stvari na dijeti.

**Gotova rečenica:**
> "Alpine baza svuda, i prošla je iz prve — jer statičkom Go binarnom fajlu od sistema treba
> praktično samo ca-certificates paket (sertifikati da može HTTPS)."

### 1.5 Shell vs exec forma — "root procesnog stabla je shell, i signal ne stigne"

**Šta je rekao:** ENTRYPOINT/CMD imaju dve varijante zapisa. Kod shell varijante, glavni proces u
kontejneru je shell, a tvoja aplikacija je njegovo "dete" — i kad sistem pošalje signal za gašenje,
shell ga NE prosledi detetu. Aplikacija ne zna da treba da se ugasi, pa je posle ~10 sekundi sistem
ubije na silu. Test: ako `docker stop` traje 10 sekundi — imaš shell formu. (Objasnio je i podelu
uloga: ENTRYPOINT je osnova — ŠTA se pokreće; CMD su podrazumevani dodaci/argumenti.)

**Šta to znači, od nule:** kad Kubernetes gasi kontejner, on glavnom procesu kaže "SIGTERM" —
učtivo "završi šta radiš i izađi". To je prilika da aplikacija zatvori konekcije ka bazi, dovrši
započeto, i izađe čisto. Ako poruka stigne posredniku (shell-u) koji je ne prenese, aplikacija
nastavi da radi kao da ništa nije — dok je ne ubiju usred posla. Kao da recepcija primi poruku
"evakuacija" i ne javi gostima.

**Gotova rečenica:**
> "Exec forma svuda — i to nam nije samo forma radi forme: naš backend hvata SIGTERM i uredno gasi
> HTTP server i pozadinsku petlju za plaćanja. Da je shell forma, signal ne bi ni stigao, kao što
> ste pokazali sa onim testom da `docker stop` visi 10 sekundi."

### 1.6 "Jako je loša praksa da staviš tačku kao build kontekst"

**Šta je rekao:** ne šalji CEO folder projekta Dockeru kao materijal za gradnju — ili napravi čist
poseban folder, ili `.dockerignore`-om isključi nepotrebno. Usput je objasnio da Docker radi kao
klijent-server, pa build može da se izvršava i na udaljenoj mašini (primer: jača mašina na AWS-u
sa GPU-om).

**Šta to znači, od nule:** "build kontekst" je paket fajlova koji se preda Dockeru pre gradnje.
Ako mu daš sve (uključujući node_modules, .git, dokumentaciju), gradnja je sporija, keš se kvari
na svaku sitnicu, i rizikuješ da ti neka tajna ili smeće uleti u sliku. Analogija: seliš se na more
na nedelju dana — ne nosiš ceo stan.

**Kod nas:** `.dockerignore` u svim repoima + redosled kopiranja u Dockerfile-u složen tako da se
keš čuva (prvo lista zavisnosti, pa tek onda kod — jer se kod menja često a zavisnosti retko).

### 1.7 Hadolint — "ne pušta te da merguješ dok ne rešiš"

**Šta je rekao:** postoji alat koji skenira Dockerfile i prijavi propuste; veže se u automatiku
tako da izmena ne može da se primi dok se problemi ne isprave.

**Gotova rečenica:**
> "Hadolint nam je obavezan check na svakom PR-u — bukvalno ona priča sa vežbi da te ne pusti da
> merguješ: dok Dockerfile ne prođe, dugme za merge je zaključano."

### 1.8 JDK vs JRE — fabrika vs prodavnica

**Šta je rekao:** JDK (Java Development Kit) ima kompajler i alate za GRADNJU; JRE (Java Runtime
Environment) ima samo ono za IZVRŠAVANJE. Build faza koristi JDK sliku, finalna JRE.

**Šta to znači za nas:** mi nemamo Javu, ali je princip identičan našem: `golang` slika (fabrika)
u fazi gradnje, `alpine` (prodavnica — samo gotov proizvod) u finalnoj. Ako pomene Javu, imam
paralelu spremnu.

---

## 2. Vežbe 4 — operator

Ovo su vežbe gde je UŽIVO pisao operator (njegov primer se zvao "Dojo"). Naš Shop operator je
rađen po tom šablonu, pa je svaka poenta odavde direktno naša.

### 2.1 "Možemo i ručno da pišemo CRD, ali što bismo"

**Šta je rekao:** CRD (definicija novog tipa resursa) je u suštini šema koja se registruje na API
serveru. Može da se piše ručno kao YAML — ali je to mnogo posla i mnogo prostora za grešku. Umesto
toga: napišeš Go strukturu, dodaš "notacije" (markere — specijalne komentare sa pravilima), i
komanda `make manifests generate` sve izgeneriše, uključujući validaciju podataka.

**Šta to znači, od nule:** validacija na nivou šeme znači da API server ODMAH odbije besmislen
zapis. Ako naš marker kaže da availability sme biti samo `standard` ili `high`, i neko pošalje
`turbo` — greška stiže istog trena, i naš operator ne mora ni da zna da se to desilo. Kao obrazac
u banci gde šalter ne prima nepopunjen formular — ne stigne do obrade.

**Gotova rečenica:**
> "Nijednu liniju CRD YAML-a nismo pisali ručno — sve tri šeme su generisane iz Go tipova
> markerima, sa enum validacijom i default vrednostima. Ono vaše 'možemo od nule, ali što bismo'."

### 2.2 RBAC iz notacija — i zašto `make run` laže

**Šta je rekao:** kubebuilder automatski generiše dozvole (RBAC) za NAŠ resurs. Ali čim operator
počne da upravlja TUĐIM resursima (Deploymentima, Secretima, bazama), moramo ručno da dodamo
markere za te dozvole — inače API server vrati 403 Forbidden. I bitna zamka: `make run` pokreće
operator na tvom laptopu SA TVOJIM (admin) pravima — pa sve prolazi i rupe u dozvolama se ne vide.
Tek kad operator radi u klasteru sa sopstvenim identitetom, istina ispliva.

**Šta to znači, od nule:** to je razlika između "probam aplikaciju ulogovan kao direktor" i "probam
je kao običan zaposleni". Kao direktor sve radi; problemi se vide tek sa pravim nalogom.

**Gotova rečenica:**
> "Naš spisak RBAC markera je rastao tačno onako kako ste opisali — svaki novi resurs koji operator
> dira (CNPG cluster, ServiceMonitor, AlertmanagerConfig...) dodat je onda kad nam je in-cluster
> deploy vratio 403. `make run` to maskira jer koristi admin kubeconfig — zato na odbrani operator
> radi iz klastera, instaliran iz Helm charta."

### 2.3 "Šta ako imaš 2 replike operatora? 1000 bugova" — stateless + status

**Šta je rekao:** reconciler (petlja operatora) NE SME da pamti stanje u svojim varijablama između
prolaza. Ako pamti — šta biva kad se operator restartuje (pamćenje nestane)? Šta kad ima dve
kopije (svaka pamti svoje)? "1000 bugova, 1000 problema." Sve što treba zapamtiti ide u **status**
objekta: petlja pročita spec (želju), pročita status (dokle se stiglo), i inkrementalno — možda
kroz više prolaza — dovodi stvarnost u red.

**Šta to znači, od nule:** zamisli smenskog radnika koji sve drži u glavi — kad ode kući, sledeća
smena ne zna ništa. Rešenje iz stvarnog sveta je isto kao ovde: **sveska primopredaje**. Sve se
piše u svesku (status), pa je svejedno ko nastavlja posao i koliko puta se smena menjala.

**Gotova rečenica:**
> "Naš reconciler ne pamti ništa između prolaza — sveska je status. Najlepši primer je Discord
> kontroler: čim Discord vrati ID kanala, upišemo ga u status PRE nego što idemo dalje. Ako
> sledeći korak pukne, ponovni prolaz iz sveske vidi 'kanal već postoji' i ne pravi dupli."

### 2.4 Idempotentnost: f(f(x)) = f(x)

**Šta je rekao:** petlja mora biti idempotentna — koliko god puta se ista akcija ponovi, efekat je
kao da se desila jednom. Dao je i matematiku: f(f(x)) = f(x); i onu sa množenjem — A×A=A važi samo
za 0 i 1. To rešava problem dupliranih događaja (isti event stigne dvaput — ne sme dvaput da
napravi resurs).

**Šta to znači, od nule:** dugme za lift. Pritisneš ga 7 puta — lift dođe JEDNOM. Dugme je
idempotentno. Da nije, došlo bi 7 liftova. Naše "dugme": svaka ensure funkcija prvo POGLEDA da li
stvar već postoji, pa tek onda pravi. Ponovljen poziv = "aha, već postoji, ništa ne radim".

**Gotova rečenica:**
> "Zapamtio sam f(f(x))=f(x) sa vežbi — kod nas se to zove ensure-pattern: svaka funkcija prvo
> proveri postojeće stanje, pa pravi samo ako fali. Zato petlja sme da se okine 50 puta za isti
> Shop i ništa se ne duplira."

### 2.5 Level-based: "od 5 izmena zanima nas samo poslednja"

**Šta je rekao:** Kubernetes je "level-based", ne "edge-based". Ako se objekat promeni 5 puta dok
je petlja zauzeta, red čekanja te izmene DEDUPLIKUJE — petlja se okine jednom, i gleda samo
poslednje stanje. Ne jurimo svaki događaj; poravnavamo se sa trenutnim stanjem.

**Šta to znači, od nule:** kao provera mejlova naspram provere temperature. Mejlove (događaje)
moraš SVE da pročitaš redom — to je edge-based. Temperaturu (stanje) samo pogledaš SADA — nebitno
koliko se puta menjala dok nisi gledao — to je level-based. Termostat ne čita istoriju temperature;
gleda trenutnu i reaguje.

**Zašto je to moćno kod nas:** operator sme da padne, da se restartuje, da odspava — kad se vrati,
pogleda TRENUTNO stanje svega i poravna se. Nema "propuštenih poruka" jer poruke i nisu bitne —
bitno je stanje.

### 2.6 Conditions: tri semafora — "da se ne bismo patili na odbrani"

**Šta je rekao:** konvencija su TRI conditiona (semafora) u statusu: **Available** (radi),
**Progressing** (radi se na tome), **Degraded** (puklo, ne mogu sam da popravim). Degraded i
Progressing se međusobno isključuju; Available može uz Progressing (radi, ali se npr. skalira).
Preporučio je pomoćne funkcije za postavljanje (jedna za "nepopravljivo", jedna za "napredujem").
I rekao je ZAŠTO to sve: (1) da na odbrani odmah vidimo šta se dešava ("da se ne bismo patili"),
i (2) jer su conditions JAVNI JEZIK operatora — drugi alati ih čitaju (pomenuo je da ArgoCD gleda
te statuse i čeka da Available postane true). Zato ne smeš da ih menjaš kako ti padne na pamet —
backward compatibility.

**Šta to znači, od nule:** semafor na raskrsnici ne postoji zbog tebe lično — postoji da bi se SVI
učesnici (i pešaci i autobusi i hitna) razumeli bez reči. Isto conditions: standardizovan signal
koji razume i čovek sa kubectl-om i ArgoCD i bilo koji alat.

**Gotova rečenica:**
> "Imamo tačno tu trojku sa pomoćnim helperima, plus Reason za svaki prelaz — i pošto vozimo
> ArgoCD, ona vaša priča da drugi alati čitaju conditione kod nas nije teorija: mogu uživo da
> pustim `kubectl get shop -w` i vidi se put DatabaseProvisioning → Deploying → Ready."

### 2.7 Stalled vs Failed — "ti si kriv" vs "klaster je kriv"

**Šta je rekao:** kad postavljaš Degraded (crveno), Reason treba da kaže KO treba da reaguje.
**Stalled** = greška u konfiguraciji koju je uneo KORISNIK (npr. tekst umesto broja) — ne rešava
se dok korisnik ne ispravi svoj zapis. **Failed** = operator je sve uradio kako treba, ali KLASTER
nije uspeo (nema resursa, interni problem) — treba da reaguje onaj ko održava klaster.

**Šta to znači, od nule:** kad ti se pokvari veš mašina, bitno je da li piše "začepljen filter —
očistite ga" (ti si kriv, ti rešavaš) ili "kvar motora — zovite servis" (majstor rešava). Ista
greška "ne radi", potpuno različit primalac poruke.

**Gotova rečenica:**
> "Naš DatabaseFailed je 'failed' kategorija iz vaše podele — operator je zahtev poslao ispravno a
> provisioning baze pukao. A 'stalled' greške kod nas uglavnom ni ne stignu do petlje, jer ih enum
> validacija iz markera odbije već na API serveru."

### 2.8 409 Conflict — "to će vas najviše nervirati" (bio je u pravu)

**Šta je rekao:** svaki objekat u Kubernetesu ima broj verzije koji se menja na svaku izmenu. Ako
pošalješ izmenu a kod sebe (u kešu) imaš zastarelu verziju — dobiješ grešku 409 Conflict. Pravila
da se sa tim živi: radi JEDNU izmenu po prolazu petlje; posle svakog update-a proveri da li je
greška baš konflikt; ako jeste — NIJE greška, samo odustani od tog prolaza i pusti petlju da krene
iznova sa svežim stanjem. Najavio je da će nas baš to najviše nervirati i da je jako error-prone.

**Šta to znači, od nule:** dvoje uređuju isti dokument. Ti si otvorio verziju od 14h, kolega u
međuvremenu sačuvao svoju u 14:05. Kad ti klikneš "sačuvaj", sistem kaže: "stop — radiš na staroj
verziji, osveži pa ponovi". To je CEO 409: ne kvar, nego zaštita od gaženja tuđih izmena. A pošto
po našim objektima legitimno piše više programa istovremeno (mi, Kubernetes, operator baze),
sudari su svakodnevica — i zato se tretiraju kao normalna pojava, ne kao greška.

**Gotova rečenica:**
> "Ono vaše 'ovo će vas najviše nervirati' — potpisujem. Dok nisam ispoštovao jednu-izmenu-po-
> prolazu, petlja mi se vrtela. Sad svaki status update proguta IsConflict, a na vrhu Reconcile
> imamo wrapper koji 409 pretvara u čisto ponovno zakazivanje."

### 2.9 Happy path sa table — do zelenog kroz 4-5 prolaza

**Šta je rekao (crtao na tabli):** od `kubectl apply` do Available=True se NE stiže u jednom
prolazu, nego kroz više malih: (1) upiši "Progressing, počinjem"; (2) napravi Deployment i zalepi
vlasničku cedulju, upiši "Creating"; (3) Deployment postoji ali kopije još nisu spremne — samo
proveri i izađi; (4) kopije spremne → Available=True; (5) neko promeni broj replika → vidiš
neslaganje → ispravi Deployment → nazad na (3). Svaki prolaz: jedna izmena, pa izlaz.

**Šta to znači, od nule:** kao zidanje po fazama sa nadzorom: svaki dan uradiš jednu stvar, upišeš
u građevinski dnevnik dokle se stiglo, odeš. Sutra pročitaš dnevnik i nastaviš. Nikad ne pokušavaš
sve odjednom — jer između tvojih poteza i drugi rade (beton se suši = baza se diže).

**Gotova rečenica:**
> "Naš Shop prolazi istu sekvencu sa vaše table, samo sa korakom više jer čekamo bazu: prvo
> DatabaseProvisioning dok CNPG ne objavi secret, pa Deploying, pa Ready — svaki prolaz jedna
> izmena i izlaz iz petlje."

### 2.10 "Nije puno filozofirao — ukrao je koncepte iz Kubernetesa"

**Šta je rekao:** za status svog resursa je prepisao polja od Deployment statusa (readyReplicas i
društvo) — namerno. Ne izmišljaj svoje konvencije kad postoji ustaljeni rečnik koji svi razumeju.

**Gotova rečenica:**
> "I mi smo 'krali' kao što ste preporučili: Shop.status.readyReplicas je ogledalo Deployment
> statusa, conditions su standardna trojka — nula originalnosti, maksimalna čitljivost za svakoga
> ko već zna Kubernetes."

### 2.11 "Jedino API server sme da dira bazu"

**Šta je rekao:** etcd (bazu stanja klastera) piše ISKLJUČIVO API server. Sve ostale komponente —
kubelet, scheduler, controller manager, operatori — slušaju obaveštenja o izmenama i drže sebi
lokalnu kopiju (keš), da ne bi zatrpavali API server pitanjima. Naš operator preko biblioteke ima:
reflektor (prima obaveštenja) → keš (kopija objekata) → red čekanja (samo "ime+fioka", bez
duplikata) → petlja. Čitanje ide iz keša; pisanje direktno na API server. I pazi da ne uvedeš
kružnu petlju (da tvoj sopstveni upis ne okida tebe u nedogled — zato izmenu radiš samo kad stvarno
ima šta da se menja).

**Šta to znači, od nule:** firma sa jednom sveskom i jednim pisarom. Ko god hoće nešto da upiše,
ide kod pisara (API server). Ko hoće da ČITA, ne stoji u redu kod pisara — drži svoju fotokopiju
koju pisar ažurira cirkularnim mejlom (watch). Zato je čitanje brzo i jeftino, a pisanje uređeno i
bez sudara... osim kad ti je fotokopija ustajala — i eto 409 iz tačke 2.8. Sve se uklapa.

Ovo je dijagram za koji je na vežbama 5 rekao **"morate ga znati usred noći"** — pa ga ja na
odbrani crtam SAM, bez čekanja da me pitaju, čim tema dođe blizu.

---

## 3. Vežbe 5

### 3.1 "Sleep infinity → čekaš 10 sekundi na KILL" — graceful shutdown

**Šta je rekao:** proces koji ignoriše signal za gašenje čeka da istekne timeout pa ga sistem
ubije na silu — zato brisanje "visi". Ispravno: uhvati signal, završi posao, zatvori konekcije,
izađi sam.

**Veza sa ranijim:** ovo je druga polovina priče o exec formi (1.5) — exec forma OMOGUĆI da signal
stigne; hvatanje signala u kodu ga ISKORISTI. Kod nas: backend na SIGTERM uredno gasi server i
petlju za plaćanja.

### 3.2 OwnerReferences i blockOwnerDeletion — "tu može da dođe do deadlocka"

**Šta je rekao:** tri stvari na "vlasničkoj cedulji" kontrolišu đubretara (Garbage Collector): ko
je vlasnik, ko je kontroler (`controller: true` — sme SAMO JEDAN), i `blockOwnerDeletion` (vlasnik
ne može da se obriše dok dete postoji). Brisanje ide kao lanac: obrišeš vlasnika → đubretar nađe
svu decu → briše od dna ka vrhu; dok se dete ne obriše, roditelj visi u "Terminating". UPOZORENJE:
ako VIŠE vlasnika drži blockOwnerDeletion nad istim objektima, možeš da napraviš situaciju gde
niko nikoga ne može da obriše — **deadlock**. Zato: ta opcija samo za veze jedan-na-jedan; za
jedan-na-više koristi labele i sam vodi logiku.

**Šta to znači, od nule:** deadlock = uzajamno blokiranje. Dva čoveka na vratima: "samo posle
tebe" — "ne, posle TEBE" — i niko ne prođe. U našem kontekstu: objekat A ne sme da se obriše dok
postoji B, a B ne sme dok postoji A. Zauvek.

**Gotova rečenica:**
> "Kod nas je svaki objekat u vlasništvu tačno jednog CR-a, pa smo bezbedni — ali znam za taj
> deadlock koji ste pominjali, i to je razlog što webhook Secret ne kačimo vlasničkim vezama na
> više Shop-ova: Shop-ovi ga referenciraju kroz spec, a mi ih nalazimo indeksom."

### 3.3 Scale subresource — priča o booking-u i broju kreveta

**Šta je rekao:** počeo je od REST API sveta: imaš resurs `booking` (rezervacija) i pod-resurs
`booking/{id}/bed_number` (samo broj kreveta) — jer kad te zanima JEDAN podatak, ne vučeš ceo
ogromni objekat. E, Kubernetes CRD-ovi imaju ugrađene pod-resurse: `/status`, `/scale`,
`/finalizers`. I tu je sve kliknulo: **ZATO reconciler ne može da menja spec** — `r.Status().
Update()` bukvalno gađa DRUGI endpoint (`dojo/status`), a dozvole se daju po endpointu; pokušaj
pisanja u spec iz operatora je namerno onemogućen. Za `/scale` treba: gde piše željeni broj
(specpath), gde piše trenutni (statuspath), i selektor da automatika nađe podove. I poenta zašto
sve to: `kubectl scale` i SVI autoscaleri (pomenuo HPA, Karpenter, KEDA) rade preko tog istog
standardnog priključka — implementiraš jedan interfejs, dobiješ ceo ekosistem.

**Šta to znači, od nule:** pod-resurs je kao šalter za jednu stvar. Umesto "dajte mi ceo dosije da
promenim broj telefona", postoji šalter samo za broj telefona — brže, i može da mu se da posebna
dozvola (šalterski radnik za telefone ne sme da menja ime i prezime).

**Gotova rečenica:**
> "Ta priča sa booking/bed_number mi je razjasnila zašto operator 'ne sme' u spec — nije bonton
> nego drugi endpoint sa drugim dozvolama. Scale smo implementirali do kraja: pošto je availability
> string a scale traži broj, uveli smo opcioni spec.replicas kao override — `kubectl scale shop
> --replicas=4` radi i ostane 4, a HPA bi mogao da se zakači ne znajući uopšte šta je Shop."

### 3.4 CNPG: "pisanje kontrolera za postgres je jako teško — i ne bi bilo primenljivo u industriji"

**Šta je rekao:** zadatak NAMERNO traži gotove operatore za baze — poenta je da istražimo
ekosistem, jer se tako radi u praksi. CNPG operator: u zapisu Cluster definišeš bootstrap/initdb
(koja baza, koji vlasnik), operator sve digne i OBJAVI Secret sa kredencijalima — a naš operator
taj Secret pročita i prosledi aplikaciji kao env varijable. Njegova krilatica: "to je prava moć
Kubernetesa — ono što bi inače bile skripte i migracije, operator rešava kroz YAML". I princip
raspodele: naš operator NE PRAVI bazu; on napravi ZAHTEV, a bazin operator ga ispuni — "oslanjamo
se na druge operatore, zato je bitan redosled".

**Šta to znači, od nule:** ti si izvođač radova (naš operator). Ne kopaš sam bunar — pozoveš
firmu za bunare (CNPG) i kažeš "ovde bunar". Oni iskopaju i ostave ti cedulju gde je slavina i
šifra (Secret). Tvoj posao je koordinacija, njihov ekspertiza.

**Gotova rečenica:**
> "Naš operator je taj 'dirigent' sa vežbi — ni jedan StatefulSet ne pravi sam: napravi CNPG
> Cluster ili MongoDBCommunity zahtev i čeka cedulju sa kredencijalima. A pošto Redis operatori
> nisu prošli — Spotahome je mrtav projekat, image mu ne postoji, a REDB traži licencu — istu
> filozofiju smo primenili na MongoDB Community operator, što spec izričito dozvoljava. Sve piše
> u decision logu sa datumima."

### 3.5 Watches i predicate: "gledaš sve secrete → memory leak"

**Šta je rekao:** kad tvoj operator zavisi od TUĐEG objekta (Secret koji pravi CNPG), dodaješ
"posmatranje" (Watches). ALI: ako posmatraš SVE secrete u klasteru, reflektor ih SVE trpa u keš —
memorija operatora raste i raste ("posle nekog vremena imaš memory leak, jer nismo lepo
konfigurisali watches"). Rešenje ima dva dela: **predicate** = filter NA ULAZU ("tiče li me se
ovaj objekat uopšte?") — što ne prođe filter, ne ulazi ni u keš; **event handler** = mapiranje NA
IZLAZU ("ovaj secret se tiče ovih 5 aplikacija — okini petlju za svaku"). Plus **indeksi** za brzo
pronalaženje ("ne morate, ali je preporuka" — O(1) umesto prolaska kroz sve). Plus filtriranje po
TIPU događaja: za bazin secret zanima te samo IZMENA (Update) — kreiranje stigne pre svega, a za
BRISANJE je rekao nešto zlatno: **"nema adekvatnog rešenja — napišite u dokumentaciji da se taj
secret ne dira, i to je na odbrani sasvim okej odgovor"**.

**Šta to znači, od nule:** pretplata na vesti. Ne pretplaćuješ se na SVE novine sveta pa da ti
poštansko sanduče eksplodira (keš raste) — pretplatiš se samo na rubriku koja te se tiče
(predicate). A kad vest stigne, razdeliš je ukućanima koje se tiče (event handler). Indeks je
imenik: umesto da zoveš sve redom i pitaš "je l' se tebe tiče?", pogledaš u imenik direktno.

**Gotova rečenica:**
> "Oba naša Secret watch-a imaju predicate baš zbog te priče o memory leaku — filtriramo po
> sufiksu imena. Za pronalaženje pogođenih Shop-ova koristimo FieldIndexer, ono vaše 'ne morate
> ali je preporuka'. A za slučaj da neko obriše bazin secret imamo doslovno vaš odgovor: nema
> elegantnog rešenja, dokumentovano je da se ne dira."

### 3.6 Helm: "nije namenjeno da sve radi u jednom prolazu"

**Šta je rekao:** u values.yaml se prave prekidači po komponenti (`pg.enabled`, `mongo.enabled`,
`app.enabled`). Ako sve uključiš odjednom → **race condition** (trka): aplikacija krene pre nego
što je baza objavila lozinku → podovi pucaju → a Helm kaže "uspešno"! Jer Helm samo ŠALJE fajlove
— ne proverava da li sistem posle radi. "Ljudi to zovu najvećim problemom Helma, ali to nije
njegova nadležnost — Helm služi za template-ovanje." Ispravno: u iteracijama — prvo baza, sačekaš,
pa aplikacija. Preporučio je bitnami chartove kao školu (njihovi values imaju po 1000 linija — "za
vas je 20-30 dovoljno"). I NAJVAŽNIJA rečenica za naš projekat: **"ono što ja radim helmom u više
iteracija, vi u admin panelu morate da izhendlujete — panel dinamički šalje zahteve, kreira
resurse, a operatori rešavaju ostalo."**

**Šta to znači, od nule:** Helm je dostavljač nameštaja: donese sve kutije i ode — "uspešno
isporučeno". Da li si sastavio krevet pre nego što si legao — tvoja stvar. Ako legneš na kutije
(sve odjednom), Helm nije kriv.

**Gotova rečenica (ovu bih obavezno rekao):**
> "Ona vaša rečenica 'što ja radim helmom u iteracijama, vi u panelu morate da izhendlujete' je
> bukvalno arhitektura našeg projekta: statički deo ide helmom u talasima kroz kube-state, a taj
> race condition između baze i aplikacije kod nas rešava operator — prvo zahtev za bazu, pa
> čekanje na secret kroz requeue, pa tek onda Deployment. Automatizovali smo tačno ono što se kod
> vas radilo ručno u više prolaza."

### 3.7 make run vs make deploy — i šta očekuje na odbrani

**Šta je rekao:** `make run` = operator kao lokalni proces (pretplaćen na evente, radi sa tvojim
pravima) — za razvoj i debug. `make deploy` = operator kao pod u klasteru. I očekivanja za
odbranu, otvoreno: da postoji Helm chart koji instalira operator, da u README-u piše kako se
instalira i konfiguriše, i da se kroz to na odbrani prođe i pokaže da radi.

**Gotova rečenica:**
> "Otišli smo korak dalje: operator se instalira iz Helm charta koji CI objavljuje na DockerHub
> kao OCI paket, a kube-state/SETUP.md je to uputstvo — od praznog laptopa do celog sistema,
> testirano od nule. ArgoCD pattern app-of-apps podigne sve jednim manifestom."

### 3.8 Finalizeri: "kad god je resurs VAN Kubernetesa"

**Šta je rekao:** Discord kanal ne živi u Kubernetesu — đubretar ga ne vidi. Finalizer je marker
koji kaže "ne završavaj brisanje dok ja ne potvrdim da je spoljna stvar počišćena". Pravilo, bez
izuzetka: **čim tvoj operator upravlja nečim van klastera — finalizer.**

**Šta to znači, od nule:** odjava iz stana. Ne možeš samo da nestaneš — prvo vratiš ključeve,
odjaviš struju (spoljne stvari), pa TEK ONDA ugovor prestaje. Finalizer je stavka "vrati ključeve"
u ugovoru: dok nije čekirana, ugovor (objekat) formalno još postoji.

**Gotova rečenica:**
> "Imamo dva finalizera: na DiscordChannel-u (kanal se briše NA Discord serveru pre nego što CR
> nestane) i na Shop-u (tenant Grafana dashboard ide preko HTTP-a pa ga đubretar ne vidi). A
> Wallet NAMERNO nema finalizer — blockchain adresa se ne može obrisati, a Secret sa ključem
> počisti đubretar preko vlasničke veze. Znati gde finalizer NE treba mi deluje jednako bitno."

### 3.9 Najava alarma preko Discorda

**Šta je rekao:** sledeća iteracija je: PrometheusRule pravila (primer: CPU preko 80% → alarm),
i CRD-ovi za Discord (server/kanali, warning/critical), pa alarmi stižu u kanale.

**Gotova rečenica:**
> "Tu smo ideju razradili do kraja: operator za svaki Shop pravi AlertmanagerConfig sa izolacijom
> po namespace-u — alarmi prodavnice idu SAMO u njen kanal; klasterski alarmi imaju poseban
> maintainers kanal, napravljen kroz naš sopstveni DiscordChannel CRD. A webhook adresa nikad
> nije u git-u — ide kao referenca na Secret."

### 3.10 Testovi: "ručno okineš reconcile petlju" + zamka sa namespace-ima

**Šta je rekao:** unit test operatora = u testu RUČNO pozoveš Reconcile funkciju (ono što je na
tabli bio jedan prolaz petlje — u testu je jedan poziv) i proveriš efekat: da li je nastao
Deployment, da li je status dobar. I upozorenje na ograničenje test-okruženja: **brisanje
namespace-a ne radi** — pa se ne čisti za sobom, nego svaki test dobije SVEŽ namespace sa
nasumičnim imenom.

**Gotova rečenica:**
> "Testovi rade tačno to — ručno okidanje Reconcile sa lažnim klijentom (uz caku da mu se uključi
> status subresource, inače status update ne radi), pa provera da je Deployment nastao sa 2
> odnosno 3 replike. I da — naleteli smo na to ograničenje sa namespace-ima koje ste najavili,
> rešeno svežim namespace-om po testu."

---

## 4. Njegove rečenice

Svaka objašnjena jednom rečenicom — da znam i ŠTA znači, ne samo da je citiram:

- **"Dijagram morate znati usred noći"** — API server → reflektor → keš → red → petlja; jedini
  pisar je API server, svi ostali čitaju svoje kopije. (Crtam ga sam, ne čekam pitanje.)
- **"f(f(x)) = f(x)"** — idempotentnost: ponovljen poziv nema novi efekat; naše dugme za lift.
- **"Šta ako imaš 2 replike operatora? 1000 bugova"** — zato ništa u pamćenju, sve u statusu
  (sveska primopredaje); i zato postoji leader election ako se ikad ide na više kopija.
- **"Stalled = ispravi svoj YAML; Failed = klaster ima problem"** — greška mora da kaže KO reaguje.
- **"409 će vas najviše nervirati"** — sudar verzija je normalna pojava, gutaš ga i puštaš petlju
  iznova; NE prijavljuje se kao greška.
- **"Helm nije kriv — njegov posao je templating"** + **"nije namenjeno da sve radi u jednom
  prolazu"** — dostavljač kutija, ne monter; zato talasi/iteracije.
- **"Ne pišite svoj operator za postgres — ne bi bio primenljiv u industriji"** — koristi CNPG;
  naš operator je dirigent, ne svira svaki instrument.
- **"Za obrisan tuđi secret nema adekvatnog rešenja — napišite u dokumentaciji da se ne dira"** —
  NJEGOVE reči, smem da ih upotrebim kao legitiman odgovor.
- **"Da se ne bismo patili na odbrani"** — statusi i conditions moraju biti čitljivi; naši jesu.
- **"Ono što ja radim helmom u iteracijama, vi u panelu morate da izhendlujete"** — opis našeg
  projekta u jednoj rečenici; panel piše želje, operator sprovodi redosled.
- **"Morate obraditi terminate signal"** — graceful shutdown; veže se na exec formu.
- **Bitnami kao škola: "njima 1000 linija values-a, vama 20-30"** — naši values su baš toliki.
- **"Nije clean code nego clear code"** — usputna fora o imenovanju; ako padne priča o kvalitetu
  koda, znam referencu (koristiti samo ako se sam našali prvi).
- **Očekivanje za demo:** "neće tražiti 100 aplikacija — bitno da korektno radi i kad se skalira"
  → demo `kubectl scale shop` je tačno ono što želi da vidi.

---

## 5. Tabela

Brza mapa "on je rekao → mi smo uradili" (za ponavljanje pred sam ulazak):

| On na vežbama | Mi u projektu |
|---|---|
| multi-stage, 3 faze | node → go → alpine u sva tri Dockerfile-a |
| jedan binary lakše se štiti | ceo backend Go, statički binary |
| root vlasnik + app izvršava, nikad root | chown root / chmod 0755 / USER app, port 8080 |
| slim/alpine "ako proradi iz prve" | alpine svuda, prošlo iz prve |
| exec forma + obradi SIGTERM | PID 1 + graceful shutdown servera i sweep petlje |
| hadolint blokira merge | required check na svakom PR-u |
| CRD iz markera, ne ručno | sve 3 šeme generisane, enum validacije |
| RBAC markeri za tuđe resurse; make run laže | spisak markera rastao kroz 403-ke; demo iz klastera |
| stateless + status sveska | ensure-pattern; Discord channelID u status odmah |
| f(f(x))=f(x) | ponovljeni prolazi ne dupliraju ništa |
| level-based, dedup u redu | operator sme da padne i da se poravna |
| 3 conditiona + isključivost | Available/Progressing/Degraded + Reason-i |
| Stalled vs Failed | DatabaseFailed = failed kategorija |
| 409: jedna izmena po prolazu | IsConflict gutanje + requeue wrapper |
| status prepisan od Deploymenta | readyReplicas ogledalo |
| samo API server piše; keš + red | crtam dijagram "usred noći" |
| blockOwnerDeletion deadlock | 1-na-1 vlasništvo svuda |
| /scale + selektor; svi autoscaleri na isti priključak | kubectl scale shop radi; replicas override |
| CNPG umesto svog DB operatora | CNPG + MongoDB Community (Redis otpao — decision log) |
| watches bez filtera = memory leak | predicate po sufiksu + FieldIndexer |
| samo Update event; delete → dokumentuj | isto: "ne dirati secret" u dokumentaciji |
| helm u iteracijama; race condition | kube-state talasi + operator sprovodi redosled |
| "helm radnje → vaš panel" | ShopHub piše CR, operator orkestrira |
| helm chart za operator + README | OCI chart na DockerHub + SETUP.md od nule + ArgoCD |
| finalizer za sve van klastera | DiscordChannel + Shop (Grafana); Wallet namerno bez |
| PrometheusRule + Discord alarmi | per-Shop kanali + maintainers kanal, sve kroz naš CRD |
| test = ručno okini Reconcile; namespace zamka | fake client + svež namespace po testu |

---

## 6. Aduti

Ako mogu da izaberem samo TRI stvari koje ću sigurno ubaciti (jer nose najviše i najlakše se
vezuju za pitanja):

1. **Dijagram "usred noći"** (2.11) — pokriva keš, watch, 409, level-based... pola operatorskih
   pitanja se svodi na njega. Crtam ga sam.
2. **f(f(x))=f(x) + status kao sveska primopredaje** (2.3, 2.4) — srce reconcile logike, i imam
   konkretan primer (Discord channelID).
3. **"Oslanjamo se na druge operatore" + "ono što on radi helmom, naš panel automatizuje"**
   (3.4, 3.6) — to dvoje zajedno objašnjava CELU arhitekturu projekta njegovim rečima.

I tri pravila da ne preteram:
- Referencu kačim POSLE svog odgovora, nikad umesto njega. Redosled: znanje → primena kod nas →
  "to ste i vi naglašavali".
- Maksimum jedna referenca po temi razgovora — dve u istom odgovoru zvuče kao snimak.
- Ako mi neka poenta odavde nije 100% jasna, ne pominjem je — svaka povlači potpitanje, a
  potpitanje na sopstvenu referencu koje ne znaš da odbraniš je gore nego da je nisi ni rekao.
