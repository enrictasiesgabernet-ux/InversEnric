# InversEnric — Pràctica Docker Compose amb Proxy Invers Nginx

Desplegament d'una infraestructura web amb Docker Compose que compleix els requisits següents:

- Únic punt d'entrada a `http://localhost` (port 80)
- Dos servidors web darrere d'un **proxy invers Nginx** amb balanceig round-robin
- Xarxa interna que **aïlla els backends** de l'exterior
- **Volum compartit** amb el contingut web entre els dos backends
- Capçaleres HTTP que permeten **identificar quin backend** respon cada petició
- Contingut multimèdia **cachejat** al proxy
- Només **imatges oficials** de Docker Hub (`nginx:alpine`)

---

## 📐 Arquitectura

```
                    ┌────────────────────────┐
                    │   Client (navegador)   │
                    └──────────┬─────────────┘
                               │  HTTP :80
                               ▼
             ╔═════════════════════════════════════╗
             ║        Xarxa: frontend (bridge)     ║
             ║   ┌─────────────────────────────┐   ║
             ║   │    proxy (nginx:alpine)     │   ║
             ║   │    - Reverse proxy          │   ║
             ║   │    - Balanceig round-robin  │   ║
             ║   │    - Cache de /media/       │   ║
             ║   │    - X-Backend, X-Cache-... │   ║
             ║   └──────────────┬──────────────┘   ║
             ╚══════════════════│══════════════════╝
                                │
             ╔══════════════════│══════════════════╗
             ║   Xarxa: backend (internal: true)  ║
             ║                  │                  ║
             ║         ┌────────┴────────┐         ║
             ║         ▼                 ▼         ║
             ║   ┌──────────┐      ┌──────────┐    ║
             ║   │   web1   │      │   web2   │    ║
             ║   │ (nginx)  │      │ (nginx)  │    ║
             ║   └─────┬────┘      └────┬─────┘    ║
             ║         │                │          ║
             ║         └────────┬───────┘          ║
             ║                  ▼                  ║
             ║       ./html  (bind mount, :ro)     ║
             ╚═════════════════════════════════════╝
```

### Flux d'una petició

1. El client fa `GET http://localhost/` al host.
2. El port 80 del host està mapejat al port 80 del contenidor `proxy`.
3. El `proxy` rep la petició i, gràcies a la directiva `upstream`, la reenvia alternativament a `web1:80` o `web2:80` (round-robin).
4. El backend seleccionat llegeix l'arxiu del volum compartit `./html` i respon, afegint la capçalera `X-Backend: webN`.
5. El proxy reenvia la resposta al client afegint `X-Upstream-Addr` i, per a `/media/`, `X-Cache-Status`.

---

## 🗂️ Estructura del projecte

```
.
├── docker-compose.yml       # Definició de la infraestructura
├── nginx/
│   ├── nginx.conf           # Config del proxy invers (upstream + cache)
│   └── backend.conf         # Config dels backends (afegeix X-Backend)
├── html/                    # Contingut web (volum compartit)
│   ├── index.html
│   ├── style.css
│   └── media/
│       ├── mico.jpg         # (a afegir manualment)
│       └── gatets.mp4       # (a afegir manualment)
└── README.md
```

---

## 🎯 Decisions de disseny

### Imatge escollida: `nginx:alpine`

S'ha triat **`nginx:alpine`** (imatge oficial) tant per al proxy com per als backends perquè:

- És una imatge oficial de Docker Hub mantinguda pel mateix projecte Nginx.
- La variant `alpine` és molt lleugera (~50 MB) i redueix la superfície d'atac.
- Nginx és idoni tant per servir contingut estàtic com per fer de reverse proxy, de manera que només cal una imatge per a tota la infraestructura (menys descàrregues, menys manteniment).
- Permet mostrar el funcionament de `upstream`, `proxy_cache`, capçaleres personalitzades i balanceig amb una sola eina.

### Estructura de xarxa: dues xarxes separades

Es defineixen **dues xarxes bridge** per implementar una segmentació clàssica *DMZ / interna*:

| Xarxa | Driver | `internal` | Serveis | Propòsit |
|-------|--------|-----------|---------|----------|
| `frontend` | bridge | no | proxy | Connecta el proxy amb l'exterior a través del port publicat |
| `backend` | bridge | **sí** | proxy, web1, web2 | Comunicació interna entre proxy i backends |

L'opció `internal: true` a la xarxa `backend` és clau: impedeix que els contenidors d'aquesta xarxa tinguin sortida a internet i, combinat amb el fet que `web1` i `web2` **no publiquen cap port**, garanteix que siguin completament inaccessibles des del host.

El proxy és l'únic servei connectat a **ambdues** xarxes, actuant com a pont controlat entre l'exterior i els backends.

### Volum compartit: bind mount de `./html`

Els dos backends munten la mateixa carpeta del host (`./html`) a `/usr/share/nginx/html` en **mode read-only** (`:ro`). Això garanteix:

- Els dos backends serveixen exactament el **mateix contingut** (coherència).
- Actualitzar el contingut és tan senzill com editar els arxius del host, sense reiniciar contenidors.
- El `:ro` impedeix que un backend comprometiu pugui modificar contingut que afecti l'altre.

### Balanceig i identificació del backend

- **Balanceig**: round-robin per defecte del bloc `upstream` de Nginx (no cal configurar res extra).
- **Identificació**: cada backend té una config `backend.conf` que afegeix `add_header X-Backend $hostname always;`. Com que els noms de contenidor són `web1` i `web2`, el client pot veure en la resposta HTTP quin ha respost.
- A més, el proxy afegeix `X-Upstream-Addr $upstream_addr` (IP:port del backend).

### Cache del contingut multimèdia

El proxy defineix una `proxy_cache_path` i aplica `proxy_cache` **només** a la ruta `/media/` (on viuen la imatge i el vídeo). Cada resposta cachejada inclou la capçalera `X-Cache-Status` amb valor `MISS` (primera petició), `HIT` (servida des de cache) o `EXPIRED`. El TTL és de **10 minuts**.

---

## 🚀 Com aixecar l'entorn

### Requisits

- Docker Engine 20.10+
- Docker Compose v2 (integrat com a `docker compose`)

### Passos

```bash
# 1. Clonar el repositori
git clone https://github.com/enrictasiesgabernet/InversEnric.git
cd InversEnric

# 2. Col·locar els arxius multimèdia (si no venen al repo)
#    html/media/mico.jpg
#    html/media/gatets.mp4

# 3. Aixecar tota la infraestructura
docker compose up -d

# 4. Obrir el navegador
#    http://localhost
```

Per aturar-ho:

```bash
docker compose down
```

---

## ✅ Comandes de verificació

### 1. Comprovar que els serveis estan en marxa

```bash
docker compose ps
```

Hauries de veure 3 serveis en estat `running`: `proxy`, `web1`, `web2`.

### 2. Verificar el balanceig (X-Backend alterna)

```bash
for i in 1 2 3 4; do
  curl -s -I http://localhost/ | grep -i x-backend
done
```

Sortida esperada (alternant):

```
X-Backend: web1
X-Backend: web2
X-Backend: web1
X-Backend: web2
```

### 3. Verificar les capçaleres completes

```bash
curl -I http://localhost/
```

Busca:
- `X-Backend: web1` o `web2`
- `X-Upstream-Addr: 172.x.x.x:80`

### 4. Verificar el cache del vídeo

```bash
# Primera petició → MISS
curl -s -I http://localhost/media/gatets.mp4 | grep -i x-cache-status

# Segona petició → HIT
curl -s -I http://localhost/media/gatets.mp4 | grep -i x-cache-status
```

Sortida esperada:

```
X-Cache-Status: MISS
X-Cache-Status: HIT
```

### 5. Comprovar l'aïllament dels backends

```bash
# Cap backend hauria de publicar ports al host
docker compose ps --format "table {{.Name}}\t{{.Ports}}"

# Intentar arribar directament als backends ha de fallar
docker inspect web1 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# Des del host, qualsevol intent de curl a aquesta IP fallarà perquè
# la xarxa backend és internal.

# Llistar les xarxes i confirmar internal
docker network inspect inversenric_backend --format '{{.Internal}}'
# → true
```

### 6. Inspeccionar el volum compartit

```bash
# Confirmar que els dos backends veuen el mateix contingut
docker compose exec web1 ls /usr/share/nginx/html
docker compose exec web2 ls /usr/share/nginx/html
```

### 7. Veure logs del proxy amb info de balanceig

```bash
docker compose logs -f proxy
```

Cada línia mostrarà `upstream=web1:80` o `upstream=web2:80` i l'estat del cache.

---

## 📸 Captures de pantalla

Totes les captures estan disponibles a la carpeta `screenshots/`.

### 1. Funcionament del proxy invers

La pàgina web és accessible a `http://localhost` gràcies al proxy Nginx, que rep totes les peticions i les reenvia als backends interns. El client només veu un únic punt d'entrada (port 80).

![Funcionament del proxy invers](screenshots/01-proxy-funcionament.png)

### 2. Balanceig round-robin entre backends

El proxy alterna automàticament entre `web1` i `web2` a cada petició. La capçalera `X-Backend` que retorna cada backend permet verificar-ho des del client:

```bash
for i in 1 2 3 4; do curl -s -I http://localhost/ | grep -i x-backend; done
```

![Balanceig round-robin](screenshots/02-balanceig.png)

### 3. Xarxa interna aïllada

Els contenidors `web1` i `web2` **no publiquen ports** al host, i la xarxa `backend` està definida com a `internal: true`. Això garanteix que només s'hi pugui accedir des del proxy, mai directament des de l'exterior.

![Xarxa aïllada](screenshots/03-xarxa-aillada.png)

---

## 🧪 Prova ràpida (tot en una línia)

```bash
docker compose up -d && sleep 3 && \
echo "== Balanceig ==" && \
for i in 1 2 3 4; do curl -s -I http://localhost/ | grep -i x-backend; done && \
echo "== Cache (1a petició MISS, 2a HIT) ==" && \
curl -s -I http://localhost/media/gatets.mp4 | grep -i x-cache-status && \
curl -s -I http://localhost/media/gatets.mp4 | grep -i x-cache-status
```

---

## 🧹 Neteja

```bash
docker compose down          # Atura i elimina contenidors + xarxes
docker compose down -v       # A més, elimina volums anònims
```

---

## 📋 Resum de compliment dels requisits

| Requisit | Compliment |
|----------|-----------|
| Punt d'entrada únic al port 80 | ✅ Només `proxy` exposa `80:80` |
| Dos servidors web + reverse proxy | ✅ `web1`, `web2`, `proxy` (nginx:alpine) |
| Balanceig | ✅ `upstream` de Nginx (round-robin) |
| Capçalera identificadora del backend | ✅ `X-Backend`, `X-Upstream-Addr` |
| Xarxa interna aïllada | ✅ Xarxa `backend` amb `internal: true` |
| Backends no accessibles des del host | ✅ Sense `ports:` publicats |
| Xarxa Docker personalitzada | ✅ `frontend` + `backend` definides al compose |
| Volum compartit entre backends | ✅ Bind mount `./html` als dos webs |
| Contingut multimèdia cachejat | ✅ `proxy_cache` a `/media/` + `X-Cache-Status` |
| Només imatges oficials de Docker Hub | ✅ Només `nginx:alpine` |
| Desplegament amb docker compose | ✅ `docker compose up` aixeca tot l'entorn |
