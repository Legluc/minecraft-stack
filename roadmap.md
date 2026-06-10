# Roadmap — Stack Minecraft Docker

## Phase 1 — Setup Git & arborescence du projet

> Soin du rendu — **1 pt**

### Créer le repo Git en local

Initialiser le projet et créer toute l'arborescence attendue par le correcteur.

```powershell
# En local
mkdir minecraft-stack; cd minecraft-stack
git init
New-Item -ItemType Directory -Force landing, prometheus, grafana/provisioning/datasources, grafana/provisioning/dashboards
```

### Créer les fichiers vides de base

Créer les fichiers nécessaires pour que le repo soit complet dès le départ.

```powershell
New-Item docker-compose.yml, README.md
New-Item prometheus/prometheus.yml
New-Item grafana/provisioning/datasources/prometheus.yml
New-Item grafana/provisioning/dashboards/dashboards.yml
```

### Créer le repo distant et le lier

```powershell
# Sur GitHub / GitLab
git remote add origin git@github.com:<user>/minecraft-stack.git
git add .; git commit -m "init: arborescence du projet"
git push -u origin main
```

### Cloner sur le VPS

```bash
# Sur le VPS
git clone git@github.com:<user>/minecraft-stack.git
cd minecraft-stack
```

---

## Phase 2 — Serveur Minecraft

> **2 pts**

### Rédiger le service `minecraft` dans `docker-compose.yml`

Utiliser l'image `itzg/minecraft-server`, mode PAPER, 2 Go de RAM, port public 25566, volume nommé pour la persistance.

```yaml
# docker-compose.yml — service minecraft
minecraft:
  image: itzg/minecraft-server
  environment:
    EULA: "TRUE"
    TYPE: "PAPER"
    MEMORY: "2G"
  ports:
    - "25566:25565"
  volumes:
    - minecraft-data:/data
  networks:
    - internal
  restart: unless-stopped
```

### Déclarer le volume nommé

```yaml
# Section volumes du docker-compose.yml
volumes:
  minecraft-data:
```

### Tester que le serveur démarre

```bash
# Sur le VPS, après git pull
docker compose up -d minecraft
docker compose logs -f minecraft
```

Attendre le message `Done! For help, type "help"` dans les logs.

---

## Phase 3 — Plugin Minecraft Prometheus

> **1 pt**

### Télécharger le .jar du plugin

Le plugin de sladkoff expose les métriques Minecraft sur un endpoint HTTP.
Le placer dans un dossier `plugins/` qui sera monté dans le conteneur.

```powershell
# En local — récupérer la dernière release depuis GitHub
New-Item -ItemType Directory -Force plugins
# Télécharger depuis https://github.com/sladkoff/minecraft-prometheus-exporter/releases
# Placer le .jar dans ./plugins/
```

### Monter le dossier plugins dans le service minecraft

```yaml
# Ajout dans le service minecraft du docker-compose.yml
volumes:
  - minecraft-data:/data
  - ./plugins:/data/plugins
```

### Vérifier l'endpoint de métriques

Le plugin expose les métriques sur le port **9225** du conteneur Minecraft par défaut.

```powershell
# Sur le VPS, depuis un autre conteneur du réseau interne
docker compose exec prometheus wget -qO- http://minecraft:9225/metrics | Select-Object -First 20
```

---

## Phase 4 — Landing page Node.js / Express / EJS / Tailwind

> **2 pts + 1 pt visuel**

### Initialiser le projet Node.js

```powershell
# En local, dans le dossier landing/
cd landing
npm init -y
npm install express ejs
npm install -D tailwindcss
npx tailwindcss init
```

### Créer la structure de la landing

```powershell
New-Item -ItemType Directory -Force views, public/css
New-Item views/index.ejs, public/css/input.css, server.js
```

- `server.js` — serveur Express + rendu EJS
- `views/index.ejs` — template de la page (nom, adresse, règles…)
- `public/css/input.css` — directives Tailwind (`@tailwind base/components/utilities`)

### Configurer le build Tailwind dans `package.json`

```json
"scripts": {
  "build:css": "tailwindcss -i ./public/css/input.css -o ./public/css/style.css",
  "start": "node server.js"
}
```

### Écrire le Dockerfile de la landing

```dockerfile
# landing/Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build:css
EXPOSE 3000
CMD ["node", "server.js"]
```

### Ajouter le service landing dans `docker-compose.yml`

```yaml
landing:
  build: ./landing
  ports:
    - "3000:3000"
  networks:
    - public
  restart: unless-stopped
```

---

## Phase 5 — Stack monitoring — Prometheus, node-exporter, cAdvisor

> Grafana **4 pts** + réseau **2 pts**

### Ajouter les services dans `docker-compose.yml`

```yaml
prometheus:
  image: prom/prometheus
  volumes:
    - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    - prometheus-data:/prometheus
  networks:
    - internal
  restart: unless-stopped

node-exporter:
  image: prom/node-exporter
  networks:
    - internal
  restart: unless-stopped

cadvisor:
  image: gcr.io/cadvisor/cadvisor
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
  networks:
    - internal
  restart: unless-stopped
```

### Rédiger `prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100"]
  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]
  - job_name: minecraft
    static_configs:
      - targets: ["minecraft:9225"]
```

### Déclarer les réseaux Docker

Réseau `internal` pour monitoring + Minecraft, réseau `public` pour la landing page et Grafana.

```yaml
# Section networks du docker-compose.yml
networks:
  internal:
  public:
```

---

## Phase 6 — Grafana — service, provisioning, dashboards

> **4 pts** + volumes **1 pt**

### Ajouter le service Grafana dans `docker-compose.yml`

```yaml
grafana:
  image: grafana/grafana
  ports:
    - "3001:3000"
  volumes:
    - grafana-data:/var/lib/grafana
    - ./grafana/provisioning:/etc/grafana/provisioning
  networks:
    - internal
    - public
  restart: unless-stopped
```

### Configurer la datasource Prometheus

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
```

### Configurer le chargement automatique des dashboards

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1
providers:
  - name: default
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

### Récupérer le dashboard Minecraft du plugin

Le concepteur du plugin fournit un dashboard JSON prêt à l'emploi sur son repo GitHub.
Le télécharger et le placer dans `grafana/provisioning/dashboards/`.

```bash
# Récupérer le dashboard depuis le repo sladkoff sur GitHub
# https://github.com/sladkoff/minecraft-prometheus-exporter
# Placer le fichier .json dans grafana/provisioning/dashboards/
```

### Créer le dashboard infra (CPU, RAM, disque, conteneurs)

Construire le dashboard dans l'interface Grafana, puis l'exporter en JSON et le placer dans le dossier de provisioning.

```bash
# Dans Grafana — Menu Dashboard → Export → Save to file
# Exporter le dashboard infra en JSON
# Le placer dans grafana/provisioning/dashboards/infra-dashboard.json
```

### Déclarer les volumes Grafana et Prometheus

```yaml
# Section volumes du docker-compose.yml
volumes:
  minecraft-data:
  prometheus-data:
  grafana-data:
```

---

## Phase 7 — Vérification finale & déploiement

> Docker Compose — **4 pts**

### Push final et déploiement sur le VPS

```powershell
# En local
git add .; git commit -m "feat: stack complète"
git push origin main

# Sur le VPS
git pull
docker compose up -d --build
```

### Vérifier que tous les conteneurs tournent

```bash
docker compose ps
```

### Vérifier les cibles Prometheus

Ouvrir `http://<IP_VPS>:9090/targets` — toutes les cibles doivent être en état `UP`.

### Vérifier Grafana et les dashboards

Ouvrir `http://<IP_VPS>:3001` — les deux dashboards doivent être présents et afficher des données.

### Rédiger le `README.md`

Décrire : les prérequis, la commande de lancement, les ports exposés, les credentials Grafana par défaut, et l'adresse du serveur Minecraft.

---

## Bonus — Reverse proxy Nginx

> **+2 pts bonus**

### Créer la config Nginx

```nginx
# nginx/nginx.conf
server {
  listen 80;
  location / {
    proxy_pass http://landing:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

### Ajouter le service Nginx dans `docker-compose.yml`

Nginx écoute sur le port 80 public, la landing n'expose plus son port directement.

```yaml
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
  depends_on:
    - landing
  networks:
    - public
  restart: unless-stopped
```

### Retirer le port exposé de la landing

Supprimer la ligne `ports: - "3000:3000"` du service landing — elle n'est plus accessible que via Nginx.
