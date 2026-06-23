# Arhitektura i obrazloženje (Secure Event Ticketing Platform)

Ovaj dokument opisuje kako je sustav građen — redoslijedom kojim je nastajao: od izbora
kontejnerskog pristupa, preko lokalnog razvoja, do produkcijskog deploymenta na Kubernetes i
sigurnosnih mjera kroz cijeli tok.

---

## Polazište: zašto kontejneri

Aplikacija se sastoji od pet manjih, neovisnih servisa koje treba moći brzo podizati, skalirati i
isporučiti identično na svačijem računalu i u produkciji. Zbog toga je odabran **kontejnerski**
pristup umjesto virtualnih mašina.

Kontejneri dijele kernel domaćina i izoliraju samo proces (namespaces/cgroups), pa su mnogo lakši i
brži od VM-ova: slike su reda veličine megabajta (koristimo Alpine baze), pokreću se u sekundi, i na
jednom hostu ih može raditi puno. Najvažnije, `Dockerfile` je kod — ista slika vrti se lokalno, u
CI-ju i u produkciji, što daje reproducibilnost koju je s VM-ovima teško postići.

Cijena tog pristupa je slabija izolacija (dijeljeni kernel). Tu cijenu pokrivamo sigurnosnim
mjerama opisanima niže (non-root, minimalne slike, skeniranje, RBAC, segmentacija mreže). Da je
projekt tražio jaču izolaciju ili različite kernele, VM bi imao smisla — ovdje nema.

---

## Servisi i njihove uloge

| Servis | Tehnologija | Uloga |
|---|---|---|
| **frontend** | Node, port 3000 | Web UI za pregled evenata i kupnju karata; API zove kroz Ingress |
| **api** | Node/Express, port 8080 | REST API: eventi, narudžbe, health (`/healthz`, `/readyz`); stavlja narudžbe u queue |
| **worker** | Node | Pozadinska obrada: čita poruke iz Redis queuea i upisuje narudžbe u Postgres |
| **postgres** | PostgreSQL 16 | Trajna pohrana narudžbi (tablica `ticket_orders`) |
| **redis** | Redis 7 | Queue/cache sloj između API-ja i workera |

Podjela prati granice odgovornosti: prikaz (frontend), poslovna logika i ulaz (api), pozadinska
obrada (worker), pohrana (postgres) i razmjena poruka (redis). Ključna odluka je odvajanje
**sinkronog** dijela (API korisniku odgovori odmah) od **asinkronog** (worker obrađuje u pozadini):
nagli val kupnji puni queue, a worker ga obrađuje svojim tempom bez rušenja API-ja.

---

## 1. dio — lokalni razvoj (Docker Compose)

Prvi korak bila je kontejnerizacija svih servisa. Za svaki je napisan **multi-stage `Dockerfile`**:
prvi stage instalira ovisnosti, drugi kopira samo potrebno u minimalnu runtime sliku i pokreće je kao
**non-root korisnik** (`appuser`). Time je slika i manja i sigurnija.

Cijeli stack se za razvoj diže jednom naredbom kroz `compose.yaml`:
- postgres i redis imaju **healthcheck**, a api/worker/frontend `depends_on: service_healthy` — baza i
  queue se prvo dignu kao zdravi, tek onda kreću aplikacijski servisi;
- **volume** čuva podatke Postgresa između pokretanja;
- **hot-reload** (`develop.watch`) sinkronizira izmjene koda uživo.

Tok kupnje validiran je end-to-end: `frontend → api → (Redis queue) → worker → Postgres (processed)`.

---

## 2. dio — produkcija na Kubernetesu (minikube)

Ista aplikacija prenesena je na orkestrirano okruženje. Manifesti su u `k8s/`, sve u namespaceu
`ticketing`. Građeno je ovim redom:

1. **Namespace** — izolirani prostor za sve resurse.
2. **ConfigMap + Secret** — ne-tajne postavke u `app-config` (ConfigMap), lozinka baze u `app-secret`
   (Secret koji se kreira izvan repozitorija). Nema hardkodiranih kredencijala u kodu.
3. **Postgres + Redis** — Postgres ima **PVC** (trajni disk) pa podaci preživljavaju zamjenu poda;
   init skripta kreira tablicu `ticket_orders`. Oba imaju resource requests/limits.
4. **api / worker / frontend** — api ima **2 replike** i **liveness/readiness probe** (`/healthz`,
   `/readyz`); worker nema HTTP probe jer nema HTTP sučelje. Slike se grade lokalno i učitavaju u
   minikube (`minikube image load`), uz `imagePullPolicy: IfNotPresent`.
5. **Ingress** — jedna vanjska ulazna točka: `/` → frontend, `/api` → api. Interni servisi (postgres,
   redis) nisu izloženi prema van.

### Kako servisi komuniciraju

```
   korisnik          Ingress (nginx)
  (browser) ───►  /     ──────────►  frontend:3000
              └─  /api  ──────────►  api:8080 ──(LPUSH)──► redis:6379
                                        │                     │
                                        │ (SQL)               ▼
                                        └──────► postgres:5432 ◄──(SQL)── worker
                                                    │ (PVC)
```

Interno se servisi pronalaze preko **ClusterIP Service** objekata i DNS imena (`postgres`, `redis`,
`api`). Vanjski promet uvijek ulazi kroz Ingress.

Na kraju je demonstriran **rolling update** (postupna zamjena slike bez downtimea, jer Service ne
šalje promet u novi pod dok readiness probe ne prođe) i **rollback** na prethodnu verziju.

---

## Sigurnost kroz cijeli tok

Sigurnost nije zaseban korak nego je ugrađena u svaku fazu:

- **Slike:** multi-stage build, minimalna Alpine baza, **non-root** korisnik.
- **Skeniranje:** sve slike skenirane **Trivyjem** prije deploya; nalazi i mjere u
  `docs/security/image-scan-report.md`.
- **Tajne:** lozinka baze samo u Kubernetes Secretu, nikad u repozitoriju (verzioniran je samo
  `.env.example` s placeholderom).
- **Pristup (RBAC):** aplikacija vozi pod zasebnim `app-sa` ServiceAccountom s minimalnim pravima
  (least-privilege) umjesto pod default računom.
- **Mreža:** NetworkPolicy dopuštaju da postgres i redis primaju promet samo od api/worker.

---

## Zašto ovakva arhitektura

Granice servisa prate granice odgovornosti, što omogućuje neovisan razvoj, skaliranje i sigurnosnu
segmentaciju. Bezstanjeni api i worker lako se repliciraju; queue apsorbira navale prometa; Postgres
na PVC-u čuva podatke; probe i rolling update daju isporuku bez prekida rada. Isti se sustav
deklarativno opisuje lokalno (Compose) i u produkciji (k8s manifesti), pa je okruženje reproducibilno
i spremno za automatiziranu isporuku.
