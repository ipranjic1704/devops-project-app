# Deployment u produkciju (Kubernetes / minikube)

Upute za podizanje cijele aplikacije na Kubernetes okruženje. Koristi se **minikube** (radi
identično na Windowsu i macOS-u). Svi manifesti su u mapi `k8s/`, namespace je `ticketing`.

> Za lokalni razvoj kroz Docker Compose vidi `README.md`. Ovaj dokument pokriva orkestrirani
> (produkcijski) deployment.

---

## 1. Preduvjeti

- **Docker Desktop** (pokrenut — minikube vrti cluster unutar Dockera)
- **minikube** i **kubectl** instalirani
- Slobodan port `80` (za Ingress preko `minikube tunnel`)

---

## 2. Pokretanje clustera i Ingress addona

```bash
minikube start
minikube addons enable ingress
```
`minikube start` diže jednočvorni cluster; addon `ingress` instalira ingress-nginx controller koji
nam treba za vanjski pristup.

---

## 3. Izgradnja i učitavanje slika

Aplikacijske slike grade se lokalno i učitavaju u minikube (bez vanjskog registryja). Slike baze i
Redisa (`postgres:16-alpine`, `redis:7-alpine`) povlače se automatski s Docker Huba.

```bash
# build (iz root direktorija projekta)
docker build -t ticketing-api:latest ./api
docker build -t ticketing-worker:latest ./worker
docker build -t ticketing-frontend:latest ./frontend

# učitaj u minikube (interni Docker minikubea)
minikube image load ticketing-api:latest
minikube image load ticketing-worker:latest
minikube image load ticketing-frontend:latest
```

> Manifesti koriste `imagePullPolicy: IfNotPresent`, pa Kubernetes koristi učitane lokalne slike i
> ne pokušava ih povući s registryja.

---

## 4. Namespace, Secret i init ConfigMap

```bash
# 4.1 namespace
kubectl apply -f k8s/namespace.yaml

# 4.2 Secret s lozinkom baze (NE commita se u git — kreira se imperativno)
kubectl -n ticketing create secret generic app-secret \
  --from-literal=POSTGRES_PASSWORD='PromijeniLozinku123!'

# 4.3 init.sql kao ConfigMap (Postgres ga izvodi pri prvom startu — kreira tablicu ticket_orders)
kubectl -n ticketing create configmap postgres-init \
  --from-file=init.sql=infra/postgres/init.sql
```

> **Sigurnost:** lozinka se nikad ne sprema u repozitorij. Verzioniran je samo `.env.example` s
> placeholderom; pravi Secret nastaje ovom naredbom. Vidi `docs/security/image-scan-report.md` za
> sigurnosne prakse oko slika.

---

## 5. Primjena manifesta

```bash
# konfiguracija i sigurnost
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/rbac.yaml
kubectl apply -f k8s/networkpolicy.yaml

# stateful servisi
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/redis.yaml

# aplikacijski servisi
kubectl apply -f k8s/api.yaml
kubectl apply -f k8s/worker.yaml
kubectl apply -f k8s/frontend.yaml

# vanjski pristup
kubectl apply -f k8s/ingress.yaml
```

Provjera da je sve podignuto:
```bash
kubectl -n ticketing get pods
# svih 6 podova (api ×2, worker, frontend, postgres, redis) -> Running, READY 1/1
```

---

## 6. Vanjski pristup (Ingress)

Ingress je **host-less** (bez `host:` polja), s rutama `/api` → `api:8080` i `/` → `frontend:3000`.
Za pristup s host stroja pokreni tunnel (drži ga otvorenim u zasebnom terminalu):

```bash
minikube tunnel
```
Aplikacija je zatim dostupna na **http://127.0.0.1** (frontend), a API na **http://127.0.0.1/api**.

> Frontend je konfiguriran s `API_BASE_URL=http://127.0.0.1/api` (ConfigMap), jer preglednik radi
> izvan clustera pa API zove kroz Ingress.

---

## 7. Validacija

```bash
# health
curl http://127.0.0.1/api/healthz      # -> {"status":"ok","service":"api"}
curl http://127.0.0.1/api/readyz        # -> ready (provjerava i bazu)

# kupnja karte (queue -> worker -> baza)
curl -X POST http://127.0.0.1/api/tickets/purchase \
  -H "Content-Type: application/json" \
  -d "{\"eventId\":\"evt-1001\",\"customerEmail\":\"student@example.com\",\"quantity\":2}"

# obrađene narudžbe (status "processed")
curl http://127.0.0.1/api/tickets/orders
```
U pregledniku: otvori **http://127.0.0.1**, odaberi event i klikni *Purchase* — narudžba prolazi
kroz Redis queue, worker je obradi i zapiše u Postgres kao `processed`.

---

## 8. Operativni postupci

```bash
# rolling update na novu verziju slike + rollback
kubectl -n ticketing set image deployment/api api=ticketing-api:v2
kubectl -n ticketing rollout status deployment/api
kubectl -n ticketing rollout undo deployment/api

# pregled logova / dijagnostika
kubectl -n ticketing logs deployment/api --tail=50
kubectl -n ticketing describe pod <pod>
```
Za incidentne scenarije (pad baze, loš image tag, neispravan secret) vidi **`docs/runbook.md`**.

---

## 9. Gašenje / čišćenje

```bash
# obriši sve resurse aplikacije (zadrži cluster)
kubectl delete namespace ticketing

# ili ugasi cijeli cluster
minikube stop          # zaustavi (stanje ostaje)
minikube delete        # potpuno obriši cluster
```

> Brisanjem namespacea briše se i PVC baze (podaci). Za čuvanje podataka zaustavi samo cluster
> (`minikube stop`) umjesto brisanja namespacea.
