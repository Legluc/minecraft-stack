# Minecraft Stack

Stack complète pour héberger un serveur Minecraft **Paper** avec monitoring en temps réel (Prometheus + Grafana) et une page d'accueil publique, le tout orchestré via Docker Compose.

## Architecture

```
                        ┌─────────────────────────────┐
                        │          VPS / Hôte         │
                        │                             │
  :80  ───────────────► │  landing (Node.js/Express)  │
  :8081 ──────────────► │  grafana                    │
  :8083 ──────────────► │  prometheus                 │
  :12000 (TCP/UDP) ───► │  minecraft (Paper)          │
                        │                             │
                        │  node-exporter  (interne)   │
                        │  cadvisor       (interne)   │
                        └─────────────────────────────┘
```

| Service         | Image / Source             | Port exposé | Rôle                             |
| --------------- | -------------------------- | ----------- | -------------------------------- |
| `minecraft`     | `itzg/minecraft-server`    | `12000`     | Serveur Paper 1.x                |
| `landing`       | build `./landing`          | `80`        | Page d'accueil publique          |
| `prometheus`    | `prom/prometheus`          | `8083`      | Collecte des métriques           |
| `node-exporter` | `prom/node-exporter`       | —           | Métriques système (CPU/RAM/disk) |
| `cadvisor`      | `gcr.io/cadvisor/cadvisor` | —           | Métriques conteneurs Docker      |
| `grafana`       | `grafana/grafana`          | `8081`      | Dashboards de visualisation      |

---

## Prérequis

- [Docker](https://docs.docker.com/engine/install/) ≥ 24
- [Docker Compose](https://docs.docker.com/compose/install/) ≥ 2.20 (plugin intégré à Docker Desktop)
- Git

---

## Installation

### 1. Cloner le dépôt

```bash
git clone https://github.com/Legluc/minecraft-stack.git
cd minecraft-stack
```

### 2. Vérifier le plugin Prometheus Exporter

Le fichier JAR du plugin doit être présent dans `plugins/` :

```
plugins/
├── minecraft-prometheus-exporter-3.1.2.jar
└── PrometheusExporter/
    └── config.yml          # host: 0.0.0.0 / port: 9940
```

> Le fichier `config.yml` est déjà versionné. Sans lui, le plugin écoute sur `localhost` et Prometheus ne peut pas le scraper depuis un autre conteneur.

### 3. Lancer la stack

```bash
docker compose up -d --build
```

Le flag `--build` est nécessaire au premier lancement pour construire l'image de la page d'accueil.

### 4. Vérifier que tous les services sont up

```bash
docker compose ps
```

Résultat attendu : tous les conteneurs en état `running` ou `healthy`.

---

## Accès aux services

| Service      | URL                                        | Identifiants      |
| ------------ | ------------------------------------------ | ----------------- |
| Landing page | http://\<votre-ip\>                        | —                 |
| Grafana      | http://\<votre-ip\>:8081                   | `admin` / `admin` |
| Prometheus   | http://\<votre-ip\>:8083                   | —                 |
| Minecraft    | `\<votre-ip\>:12000` (client Java Edition) | —                 |

> Changez le mot de passe Grafana à la première connexion.

---

## Grafana — Dashboards

Un dashboard Minecraft est provisionné automatiquement au démarrage depuis `grafana/provisioning/dashboards/`.

Pour ajouter les dashboards infrastructure :

1. Connectez-vous à Grafana
2. **Dashboards → Import**
3. Importez les IDs suivants :
   - **1860** — Node Exporter Full (CPU, RAM, disque)
   - **14282** — cAdvisor (métriques conteneurs)

---

## Structure du projet

```
minecraft-stack/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml              # Configuration des scrape jobs
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   └── prometheus.yml      # Datasource Prometheus auto-provisionnée
│       └── dashboards/
│           ├── dashboards.yml      # Provider de dashboards
│           └── minecraft-general-dashboard.json
├── landing/
│   ├── Dockerfile
│   ├── server.js                   # Serveur Express/EJS
│   ├── views/
│   │   └── index.ejs
│   ├── public/css/
│   │   └── input.css               # Tailwind v4
│   └── package.json
└── plugins/
    ├── minecraft-prometheus-exporter-3.1.2.jar
    └── PrometheusExporter/
        └── config.yml
```

---

## Commandes utiles

```bash
# Voir les logs d'un service en temps réel
docker compose logs -f minecraft

# Redémarrer un seul service
docker compose restart grafana

# Mettre à jour les images et relancer
docker compose pull
docker compose up -d

# Arrêter la stack (les volumes sont conservés)
docker compose down

# Arrêter ET supprimer les volumes (⚠ perte des données Minecraft)
docker compose down -v
```

---

## Dépannage

| Problème                                     | Solution                                                                                                    |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Métriques Minecraft absentes dans Prometheus | Vérifier que `plugins/PrometheusExporter/config.yml` contient `host: 0.0.0.0`                               |
| Page d'accueil non accessible sur :80        | `docker compose logs landing` — vérifier que le build Tailwind a réussi                                     |
| Grafana affiche "datasource not found"       | La datasource Prometheus est auto-provisionnée ; vérifier `grafana/provisioning/datasources/prometheus.yml` |
| Minecraft ne démarre pas                     | Vérifier `EULA: "TRUE"` dans `docker-compose.yml` et les logs : `docker compose logs minecraft`             |

---

## Licence

MIT
