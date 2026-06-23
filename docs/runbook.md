# Runbook — troubleshooting incidenata (Secure Event Ticketing Platform)

**Okruženje:** Kubernetes (minikube), namespace `ticketing`
**Datum vježbe:** 2026-06-23
**Servisi:** `frontend`, `api` (2 replike), `worker`, `postgres` (PVC), `redis`

> Runbook pokriva tri tipična produkcijska incidenta: **pad baze**, **loš image tag** i
> **neispravan secret**. Svi su **stvarno reproducirani** na clusteru, pa su outputi u nastavku pravi.
> Svaki scenarij ima: simptom → dijagnostiku → analizu uzroka → korektivnu mjeru → validaciju.

---

## 0. Opći troubleshooting postupak

Sistematičan redoslijed pri svakom incidentu:

1. **Pregled stanja:** `kubectl -n ticketing get pods` — tko nije `Running`/`Ready`.
2. **Detalji i događaji:** `kubectl -n ticketing describe pod <pod>` — sekcija **`Events`** i **`Last State`** (Exit Code, Reason).
3. **Logovi aplikacije:** `kubectl -n ticketing logs <pod> --tail=N` (+ `--previous` ako se pod restartao).
4. **Prepoznaj uzorak** (vidi tablicu dolje) → poveži simptom s uzrokom.
5. **Korektivna mjera** → primijeni najmanju nužnu promjenu.
6. **Validacija** → potvrdi `Running`/`Ready` i funkcionalnost.

### Brza tablica prepoznavanja

| Simptom | Vjerojatni uzrok | Scenarij |
|---|---|---|
| Pod `0/1`, `Exit Code 1`, restarta se, log `ECONNREFUSED` | ovisni servis (baza) nedostupan | 1 |
| Pod `ImagePullBackOff` / `ErrImagePull`, kontejner se nikad ne pokrene | kriva/nepostojeća slika ili tag | 2 |
| Pod `0/1`, **bez** restarta, readiness `503`, baza dostupna | kriva konfiguracija/kredencijali (Secret) | 3 |
| Pod `ContainerCreating`, `FailedMount` u Events | kriv naziv ConfigMap/Secret/volumena | (vidi Dodatak) |

---

## 1. Incident: pad baze (PostgreSQL nedostupan)

**Reprodukcija (simulacija pada):**
```bash
kubectl -n ticketing scale deployment/postgres --replicas=0
```

### Simptom
```text
api-7645fb9574-xlb5l   0/1   Running   1 (81s ago)   ...
# postgres pod nestao s popisa
```
API podovi padaju na `0/1` (NotReady) i restartaju se.

### Dijagnostika
`kubectl -n ticketing describe pod <api-pod>`:
```text
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
Restart Count:  1
Events:
  Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 503
```
`kubectl -n ticketing logs <api-pod> --previous` pokazuje grešku `pg` drivera (ispis konekcijskog
objekta) i pad procesa (`Node.js v22...`).

### Analiza uzroka
Baze nema → API ne može otvoriti konekciju na `postgres:5432`. Readiness probe `/readyz` vraća **503**
(pod izlazi iz Service endpointa), a neuhvaćena greška na kraju ruši proces (**Exit Code 1**), pa ga
Kubernetes restarta. Sustav se ne oporavlja dok baza ne vrati.

### Korektivna mjera
```bash
kubectl -n ticketing scale deployment/postgres --replicas=1
```

### Validacija
```bash
kubectl -n ticketing get pods
# postgres ... 1/1 Running   (novi pod)
# api ...      1/1 Running   (oba)
```
API se **sam** oporavi čim baza vrati (readiness opet prolazi). **Podaci nisu izgubljeni** — Postgres
koristi PVC, pa nova instanca montira isti volumen sa svim narudžbama.

---

## 2. Incident: loš image tag (ImagePullBackOff)

**Reprodukcija (deploy nepostojeće slike):**
```bash
kubectl -n ticketing set image deployment/api api=ticketing-api:nepostojeci-v999
```

### Simptom
```text
api-55886bbdf8-s9f77   0/1   ErrImagePull   0   36s     <- novi pod
api-7645fb9574-prvr5   1/1   Running        1   ...     <- stari, i dalje radi
api-7645fb9574-xlb5l   1/1   Running        1   ...     <- stari, i dalje radi
```
Novi pod ne može povući sliku; **stari podovi i dalje rade** → aplikacija nije pala.

### Dijagnostika
`kubectl -n ticketing describe pod <novi-pod>`:
```text
State:   Waiting
  Reason: ImagePullBackOff
Events:
  Warning  Failed  Failed to pull image "ticketing-api:nepostojeci-v999":
           ... repository does not exist or may require 'docker login': denied
  Warning  Failed  Error: ErrImagePull
  Warning  Failed  Error: ImagePullBackOff
```

### Analiza uzroka
Traženi tag `nepostojeci-v999` ne postoji (ni lokalno u minikubeu ni u registryju). Zbog **RollingUpdate**
strategije Kubernetes prvo digne novi pod i čeka da postane `Ready` prije gašenja starih — pošto novi
nikad ne postane Ready, stari ostaju i **nema downtimea**. Loš deploy se "zaglavi", ne ruši produkciju.

### Korektivna mjera
```bash
kubectl -n ticketing rollout undo deployment/api
kubectl -n ticketing rollout status deployment/api   # -> successfully rolled out
```

### Validacija
```bash
kubectl -n ticketing get pods
# loš pod (55886bbdf8) nestao; ostaju zdravi api podovi 1/1 na :latest
```

### Prevencija
- U CI/CD koristiti **eksplicitan, sljediv tag** (`<git-sha>`) umjesto ručnog upisa.
- **Trivy quality gate** prije objave (vidi `docs/security/image-scan-report.md`).

---

## 3. Incident: neispravan secret (kriva lozinka baze)

**Reprodukcija (kriva lozinka + restart da je pod pokupi):**
```bash
# 1) snimi ISPRAVNU vrijednost (za siguran povratak)
kubectl -n ticketing get secret app-secret -o jsonpath="{.data.POSTGRES_PASSWORD}"
# 2) postavi krivu lozinku
kubectl -n ticketing create secret generic app-secret \
  --from-literal=POSTGRES_PASSWORD=KrivaLozinka999 --dry-run=client -o yaml | kubectl apply -f -
# 3) restart (Secret se NE primjenjuje dok se pod ne restarta!)
kubectl -n ticketing rollout restart deployment/api
```

### Simptom
```text
api-6db8fb8fb-chg2z   0/1   Running   0   2m29s
```
Pod je `0/1` (NotReady), ali **`Restart Count: 0`** — ne ruši se. `logs` pokazuje samo
`API listening on port 8080` (greška autentikacije se ne loga na stdout).

### Dijagnostika
`kubectl -n ticketing describe pod <api-pod>`:
```text
Restart Count: 0
Events:
  Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 503
```
Provjera endpointa iznutra:
```bash
kubectl -n ticketing exec <api-pod> -- wget -qO- http://localhost:8080/readyz
# wget: server returned error: HTTP/1.1 503 Service Unavailable
```

### Analiza uzroka
Baza **radi** (TCP veza i `/healthz` prolaze → nema crasha), ali Postgres **odbija autentikaciju** jer
je lozinka u Secretu kriva → `/readyz` (koji izvodi DB upit) vraća **503**.

**Ključni kontrast s incidentom 1:** ovdje **nema** `Exit Code 1`/restarta (samo readiness pada), dok je
kod pada baze pod i crashao. Uzorak "readyz 503 bez crasha + baza dostupna" upućuje na **konfiguraciju/
kredencijale**, ne na dostupnost baze.

> Napomena: promjena Secreta **ne** restarta podove automatski — nove vrijednosti se učitavaju tek pri
> sljedećem startu poda (`kubectl rollout restart`).

### Korektivna mjera
```bash
kubectl -n ticketing create secret generic app-secret \
  --from-literal=POSTGRES_PASSWORD='PromijeniLozinku123!' --dry-run=client -o yaml | kubectl apply -f -
kubectl -n ticketing rollout restart deployment/api
```

### Validacija
```bash
kubectl -n ticketing get pods
# api-7cbf6d8d8-...  1/1  Running  0   <- oba poda Ready
kubectl -n ticketing get secret app-secret -o jsonpath="{.data.POSTGRES_PASSWORD}"
# -> J1Byb21pamVuaUxvemlua3UxMjMhJw==  (identično originalu)
```
Pod je opet `Ready` jer `/readyz` ponovno uspješno upituje bazu → kredencijali ispravni.

---

## Dodatak: stvarni incident iz izgradnje (FailedMount)

Tijekom postavljanja baze dogodio se stvaran incident vrijedan zabilježbe: ConfigMap s init skriptom
kreiran je s tipfelerom u imenu (`postres-init` umjesto `postgres-init`), pa je pod ostao u
`ContainerCreating`:
```text
Events:
  Warning  FailedMount  configmap "postgres-init" not found
```
- **Dijagnostika:** `kubectl describe pod` → `FailedMount`.
- **Uzrok:** referencirano ime ConfigMap-a ne postoji (tipfeler).
- **Mjera:** obrisati krivi ConfigMap, kreirati ispravan (`postgres-init`), obrisati pod da se ponovno stvori.
- **Pouka:** kad je pod u `ContainerCreating`, uzrok je gotovo uvijek u `Events` (mount/volume/secret/configmap).
