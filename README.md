# Secure Event Ticketing Platform (Sample DevSecOps Project)

Ovaj repozitorij je referentni uzorak aplikacije za kolegij **Uvod u DevOps - DevSecOps**.
Prikazuje cijeli tok: lokalni razvoj kroz Compose i produkcijski deployment kroz Kubernetes manifeste.

## Arhitektura

- `frontend` - web UI za pregled evenata i kupnju karata
- `api` - REST API za evente, narudzbe i health provjere
- `worker` - pozadinska obrada queue poruka
- `postgres` - trajna pohrana narudzbi
- `redis` - queue/cache sloj

## Preduvjeti

- Docker Desktop (ili Podman) s Docker Compose v2+
- Slobodni portovi: `3000` (frontend) i `8080` (API)

## Pokretanje (startup)

1. Kopiraj primjer okolišnih varijabli u stvarni `.env`:
   ```bash
   cp .env.example .env
   ```
2. Podigni cijeli stack jednom naredbom:
   ```bash
   docker compose up --build
   ```
   Compose poštuje `depends_on` + healthcheck: postgres i redis se prvo dignu kao healthy, tek onda kreću api i worker.
3. (Opcionalno) razvoj s hot-reloadom — sinkronizira izmjene koda uživo:
   ```bash
   docker compose watch
   ```

## Zaustavljanje

- Zaustavi i ukloni kontejnere (podaci u bazi ostaju sačuvani u volumenu):
  docker compose down
- Zaustavi i obriši i podatke (čisti reset baze):
  docker compose down -v

### Brza validacija funkcionalnosti

1. Health API:
   ```bash
   curl http://localhost:8080/healthz
   curl http://localhost:8080/readyz
   ```
2. Dohvati evente:
   ```bash
   curl http://localhost:8080/events
   ```
3. Posalji narudzbu:
   ```bash
   curl -X POST http://localhost:8080/tickets/purchase \
     -H "Content-Type: application/json" \
     -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
   ```
4. Provjeri obradene narudzbe:
   ```bash
   curl http://localhost:8080/tickets/orders
   ```
5. UI:
   - Otvori `http://localhost:3000`

## Sigurnosni elementi

- Multi-stage Docker build i non-root runtime korisnik
- Secret + ConfigMap odvojena konfiguracija
- Liveness/Readiness probe
- Resource requests/limits
- ServiceAccount + RBAC
- NetworkPolicy segmentacija
- Trivy skeniranje slika u CI pipelineu

Detalji skeniranja: `docs/security/image-scan-report.md`

## Dokumentacija

- Arhitektura i obrazloženje (kontejner vs VM, servisi, komunikacija): `docs/architecture.md`
- Deployment u produkciju (Kubernetes / minikube, korak-po-korak): `docs/deployment.md`
- Sigurnosno izvješće skeniranja slika (Trivy): `docs/security/image-scan-report.md`
- Runbook za troubleshooting (pad baze, loš image tag, neispravan secret): `docs/runbook.md`
