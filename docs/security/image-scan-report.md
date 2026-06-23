# Sigurnosno izvješće skeniranja kontejnerskih slika

**Projekt:** Secure Event Ticketing Platform
**Datum skeniranja:** 2026-06-23
**Alat:** [Trivy](https://trivy.dev/) v0.71.2 (baza ranjivosti povučena na dan skeniranja)
**Skenirane slike:** `ticketing-api:latest`, `ticketing-worker:latest`, `ticketing-frontend:latest`

> Ovo izvješće zadovoljava zahtjev *"skeniranje slika prije deploya"* (2. dio, sigurnosni minimum)
> i isporuku *"Sigurnosno izvješće skeniranja slika"* (Očekivani artefakti).

---

## 1. Metodologija

Svaka slika izgrađena je lokalno (multi‑stage build, `node:22-alpine` baza, non‑root korisnik)
i skenirana naredbom:

```bash
trivy image ticketing-api:latest
trivy image ticketing-worker:latest
trivy image ticketing-frontend:latest
```

Trivy provjerava dvije razine:
- **OS paketi** (Alpine `apk` baza),
- **Aplikacijske ovisnosti** (Node.js `node-pkg`, čita `package.json` datoteke u slici).

Skenira se **gotova slika prije deploya**, čime se ranjivost otkriva prije nego dođe u produkciju.

---

## 2. Sažetak rezultata

| Slika | OS (Alpine 3.24.1) | Node.js | CRITICAL | HIGH | MEDIUM | LOW |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `ticketing-api:latest` | 0 | 6 | 0 | 1 | 5 | 0 |
| `ticketing-worker:latest` | 0 | 5 | 0 | 1 | 4 | 0 |
| `ticketing-frontend:latest` | 0 | 5 | 0 | 1 | 4 | 0 |

**Zapažanja:**
- **Operativni sustav (Alpine): 0 ranjivosti** u sve tri slike — posljedica minimalne Alpine baze i multi‑stage builda.
- **CRITICAL: 0** u sve tri slike.
- Sve Node.js ranjivosti su statusa `fixed` (postoji zakrpana verzija).

---

## 3. Analiza nalaza

Nalazi se dijele u dvije kategorije prema **putanji** u kojoj ih je Trivy pronašao:

### 3.1. Ranjivost u vlastitim ovisnostima aplikacije

| Slika | Paket | Verzija | CVE | Severity | Zakrpa |
|---|---|---|---|:---:|---|
| api | `uuid` (`app/node_modules/uuid`) | 10.0.0 | CVE-2026-41907 | MEDIUM | 11.1.1 |

`uuid` je **direktna ovisnost** API servisa (`api/package.json`). Ovo je jedina ranjivost koju
projekt izravno kontrolira i koju treba popraviti bumpom verzije.

### 3.2. Ranjivosti iz bazne slike (npm CLI ugrađen u `node:22-alpine`)

Sljedeći paketi pronađeni su pod `usr/local/lib/node_modules/npm/...`, tj. dio su **npm CLI alata
koji dolazi ugrađen u službenu `node:22-alpine` baznu sliku**. Aplikacija ih ne instalira niti ih
poziva u runtimeu (runtime CMD je `node src/server.js`, izvršava se kao non‑root `appuser`).

| Paket | Verzija | CVE | Severity | Zakrpa | Tip |
|---|---|---|---|:---:|---|
| `picomatch` | 4.0.3 | CVE-2026-33671 | **HIGH** | 4.0.4 | ReDoS |
| `picomatch` | 4.0.3 | CVE-2026-33672 | MEDIUM | 4.0.4 | Method injection |
| `brace-expansion` | 2.0.2 | CVE-2026-33750 | MEDIUM | 2.0.3 | DoS |
| `ip-address` | 10.1.0 | CVE-2026-42338 | MEDIUM | 10.1.1 | XSS |
| `tar` | 7.5.11 | CVE-2026-53655 | MEDIUM | 7.5.16 | node-tar |

Ovih 5 ranjivosti pojavljuje se u **sve tri** slike (jednaka bazna slika). Stvarna izloženost je
niska jer se npm CLI ne izvršava u produkcijskom runtimeu.

---

## 4. Korektivne mjere

| # | Mjera | Prioritet | Status |
|---|---|:---:|:---:|
| 1 | **Bump `uuid` `^10.0.0` → `^11.1.1`** u `api/package.json`, zatim `npm install` + rebuild slike. | Visok | ⬜ planirano |
| 2 | Periodički **rebuild s `--pull`** da se povuče novija `node:22-alpine` (kad bazna slika dobije zakrpani npm). | Srednji | ⬜ planirano |
| 3 | (Opcionalno) Ukloniti/skratiti npm iz runtime sloja jer se ne koristi nakon builda. | Nizak | razmotreno |
| 4 | **Zadržati postojeće prakse:** minimalna Alpine baza, multi‑stage build, non‑root korisnik (već daju 0 OS ranjivosti). | — | ✅ u primjeni |

---

## 5. Politika objave slika (image policy)

- **Tagging:** slike se grade s eksplicitnim tagom; `latest` se koristi za lokalni razvoj/minikube.
  U CI/CD-u (planirano) tag bi bio `<git-sha>` radi sljedivosti.
- **Quality gate (planirano u CI/CD):** Trivy korak s `--severity HIGH,CRITICAL --exit-code 1`
  zaustavlja objavu slike ako se pojavi nova HIGH/CRITICAL ranjivost.
- **Evidencija:** ovo izvješće je evidencija skeniranja; obnavlja se pri svakoj značajnoj promjeni
  ovisnosti ili bazne slike.

---

## 6. Zaključak

Sve tri slike imaju **čist OS sloj (0)** i **nula CRITICAL** ranjivosti. Jedina ranjivost u vlastitom
kodu je `uuid` u API servisu (MEDIUM, postoji zakrpa) i adresira se korektivnom mjerom #1. Preostale
ranjivosti potječu iz npm CLI-ja ugrađenog u baznu sliku, niske su stvarne izloženosti i prate se
kroz redovit rebuild bazne slike. Sigurnosna pozicija slika ocjenjuje se **prihvatljivom za deploy**
uz provedbu navedenih korektivnih mjera.
