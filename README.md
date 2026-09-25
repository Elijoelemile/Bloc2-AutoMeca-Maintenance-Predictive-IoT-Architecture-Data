# AutoMeca Systems — Maintenance prédictive IoT

![Cloud](https://img.shields.io/badge/cloud-souverain%20UE-000E9C?style=flat-square)
![PostgreSQL](https://img.shields.io/badge/datamart-PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![ClickHouse](https://img.shields.io/badge/t%C3%A9l%C3%A9metrie-ClickHouse-FFCC01?style=flat-square&logo=clickhouse&logoColor=black)
![Kafka](https://img.shields.io/badge/ingestion-Kafka-231F20?style=flat-square&logo=apachekafka&logoColor=white)
![Kaggle](https://img.shields.io/badge/dataset-Kaggle-20BEFF?style=flat-square&logo=kaggle&logoColor=white)

AutoMeca Systems conçoit des équipements de freinage pour l'industrie
automobile. Ce projet met en place une plateforme de données pour la
maintenance prédictive de son parc de machines de production : capteurs
IoT en atelier, modélisation et stockage des données, pipelines
d'ingestion, et déploiement d'un modèle prédictif de panne.

Le projet est organisé en dépôts indépendants, un par domaine :

| Dépôt | Contenu |
|---|---|
| [Bloc2-AutoMeca-Maintenance-Predictive-IoT-Architecture-Data](https://github.com/<user>/Bloc2-AutoMeca-Maintenance-Predictive-IoT-Architecture-Data) | Architecture de données : diagramme Edge/Cloud, modèle conceptuel, star schema, dictionnaire de données |
| [Bloc3-AutoMeca-Maintenance-Predictive-IoT-Pipeline-Data](https://github.com/<user>/Bloc3-AutoMeca-Maintenance-Predictive-IoT-Pipeline-Data) | Pipelines d'ingestion et de transformation des données (ELT) |
| [Bloc4-AutoMeca-Maintenance-Predictive-IoT-Solution-IA](https://github.com/<user>/Bloc4-AutoMeca-Maintenance-Predictive-IoT-Solution-IA) | Modèles de maintenance prédictive (entraînement) |
| [Bloc4-AutoMeca-Maintenance-Predictive-IoT-CICD](https://github.com/<user>/Bloc4-AutoMeca-Maintenance-Predictive-IoT-CICD) | Intégration et déploiement continus |

---

## Ce dépôt : Bloc2-AutoMeca-Maintenance-Predictive-IoT-Architecture-Data

Conception de l'architecture de données : comment les données du parc
de machines (capteurs, erreurs, pannes, maintenances) sont modélisées
et stockées, depuis leur origine (Edge industriel / systèmes SI)
jusqu'à leur mise à disposition pour l'analyse et le futur modèle
prédictif. Basé sur le dataset réel *Microsoft Azure Predictive
Maintenance* (Kaggle).

Deux niveaux de modélisation sont documentés séparément : le modèle
conceptuel (entités métier et associations, indépendant de toute
optimisation technique) et le modèle physique qui en découle (star
schema avec table de faits unifiée, choisi pour servir efficacement
la reconstitution de l'historique chronologique d'une machine).

L'architecture adresse explicitement les 3V (volume via le data lake,
vélocité via un broker d'ingestion, variété via la coexistence data
lake / data warehouse) ainsi que la sécurisation (chiffrement,
segmentation IT/OT, IAM, VPN dédié) et la supervision de
l'infrastructure — détaillées dans la bande transverse du diagramme
d'architecture.

> [!NOTE]
> Le star schema modélise aussi une dimension `opérateur` (qui a réalisé une intervention), anticipée pour une future intégration GMAO : le dataset source ne contient aucune donnée opérateur, la dimension est donc présente mais non peuplée dans cette itération.

## 🗂️ Structure

```
Bloc2-AutoMeca-Maintenance-Predictive-IoT-Architecture-Data/
├── data/                                   # dataset source (non versionné, voir .gitignore)
│   ├── PdM_machines.csv
│   ├── PdM_errors.csv
│   ├── PdM_failures.csv
│   ├── PdM_maint.csv
│   └── PdM_telemetry.csv
├── diagram/
│   ├── architecture-edge-cloud.pptx        # source éditable, 3 pages
│   ├── architecture-edge-cloud.pdf         # les 3 pages en un seul fichier
│   ├── 01_architecture-edge-cloud.png      # page 1 : architecture Edge/Cloud
│   ├── 02_modele-conceptuel-mcd.png        # page 2 : MCD (notation Merise)
│   └── 03_schema-etoile-physique.png       # page 3 : star schema physique (fait unifié + dimensions)
├── database_scripts/
│   ├── 01_staging.sql                      # couche ER, fidèle aux sources
│   ├── 02_datamart.sql                     # star schema (PostgreSQL)
│   └── 03_telemetrie_clickhouse.sql        # télémétrie (ClickHouse)
└── data_dictionary/
    └── dictionnaire_donnees.md             # dictionnaire complet des tables/colonnes
```

> [!NOTE]
> `data/` n'est pas versionné (voir `.gitignore`) : le dataset s'obtient sur Kaggle, [arnabbiswas1/microsoft-azure-predictive-maintenance](https://www.kaggle.com/datasets/arnabbiswas1/microsoft-azure-predictive-maintenance).

## 🛠️ Stack technique

- ☁️ **Cloud souverain européen** (région UE) — Object Storage (S3-compatible, multi-format), PostgreSQL managé, instances Compute (Kafka et ClickHouse auto-hébergés), IAM, réseau privé
- 📡 **Kafka** (auto-hébergé, petite instance Compute) — broker d'ingestion (pont MQTT → Kafka), absorbe la vélocité des flux capteurs
- 🐘 **PostgreSQL managé** — couche staging (ER) + datamart (star schema)
- ⚡ **ClickHouse** (auto-hébergé, petite instance Compute) — télémétrie haute fréquence (série temporelle)
- 🔒 **Sécurité** — chiffrement en transit (TLS) et au repos, segmentation IT/OT (IEC 62443) côté Edge, IAM, réseau privé, VPN site-à-site (données) et VPN dédié aux sous-traitants de maintenance

## 💶 Contraintes de coût et choix d'infrastructure

Le sujet impose un **budget limité** (section 1.2) et une **infrastructure Cloud européenne** garantissant la souveraineté des données (RGPD). Ces deux contraintes ne sont pas restées théoriques : les choix d'infrastructure ci-dessous en découlent directement, et sont documentés ici pour que le raisonnement soit traçable — référencé depuis les README des dépôts Bloc 3 et Bloc 4 (CI/CD), qui en héritent sans le dupliquer.

**Cloud souverain européen, facturation à l'usage réel** — l'infrastructure est hébergée chez un fournisseur cloud européen, avec facturation horaire (pas d'abonnement mensuel fixe) : seul le temps réellement consommé est payé.

**Kafka et ClickHouse auto-hébergés, pas managés** — les offres managées de broker de messages et de base colonnaire sont pensées pour de la haute disponibilité multi-zone en production réelle (plusieurs nœuds redondants) — disproportionné pour ce cas fictif de certification, et nettement plus coûteux qu'une instance simple. Les deux tournent donc à la place sur une petite instance Compute auto-gérée, mêmes principes de sécurité (réseau privé, chiffrement) mais sans le coût du multi-nœud managé. PostgreSQL, lui, reste géré (offre d'entrée de gamme accessible) et n'a pas cette contrainte.

**Une seule instance Compute partagée, pas plusieurs** — Kafka, ClickHouse, Grafana **et** les conteneurs du Bloc 4 (API de prédiction + interface de supervision) tournent sur la **même** petite instance Compute (au lieu d'une instance dédiée par service), pour diviser le coût par autant de services consolidés plutôt que de le multiplier.

**Pas de Kubernetes managé, pas de registre de conteneurs** — Kubernetes orchestre des flottes de conteneurs répartis sur plusieurs machines avec mise à l'échelle automatique ; un registre sert à faire voyager une image Docker entre la machine qui la construit et celle qui l'exécute. Aucun des deux besoins ne se pose ici : les conteneurs applicatifs (+ Kafka, ClickHouse, Grafana) tournent tous sur une seule instance, construits directement dessus (`git pull` + `docker compose build`, comme en local) — pas de flotte à orchestrer, pas d'image à faire voyager entre machines.

**Conséquence opérationnelle** — une instance Compute facturée à l'heure reste abordable seulement si elle n'est allumée que pendant les fenêtres de test/démonstration actives, pas laissée tourner en permanence. Même discipline que celle déjà appliquée en local pendant le développement (conteneurs Docker jetables, vérifiés puis détruits), transposée à l'infrastructure cloud réelle.

**Déploiement réel effectué** — cette architecture n'est pas restée théorique : Kafka, ClickHouse et Grafana (Bloc 3) ainsi que l'API et l'interface de supervision (Bloc 4) ont été réellement déployés ensemble sur une seule instance Compute **Scaleway** (le fournisseur cloud souverain européen retenu pour ce projet), avec une vraie prédiction de bout en bout testée avec succès (voir les README des dépôts Bloc 3 et Bloc 4 pour le détail).

## 📦 Contenu

- **`diagram/`** — architecture Edge/Cloud (sources, broker d'ingestion, stockage, bande transverse sécurité/supervision), modèle conceptuel (MCD, notation Merise) et schéma en étoile physique (table de faits unifiée + dimensions)
- **`database_scripts/`** — DDL des 3 couches : staging (ER), datamart (star schema, table de faits unifiée `fait_evenement`), télémétrie (ClickHouse)
- **`data_dictionary/`** — dictionnaire de données : chaque table, colonne, type, contrainte, description, exemple
