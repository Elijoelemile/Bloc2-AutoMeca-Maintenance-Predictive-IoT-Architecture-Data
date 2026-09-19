# AutoMeca Systems — Maintenance prédictive IoT

![OVHcloud](https://img.shields.io/badge/cloud-OVHcloud%20(UE)-000E9C?style=flat-square)
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
| [bloc3-pipeline-donnees](https://github.com/<user>/bloc3-pipeline-donnees) | Pipelines d'ingestion et de transformation des données |
| [bloc4-solution-ia](https://github.com/<user>/bloc4-solution-ia) | Modèle de maintenance prédictive et déploiement |
| [bloc4-cicd](https://github.com/<user>/bloc4-cicd) | Intégration et déploiement continus |

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
│   ├── architecture-edge-cloud.pptx        # source éditable, 2 pages
│   ├── architecture-edge-cloud.pdf         # les 2 pages en un seul fichier
│   ├── 01_architecture-edge-cloud.png      # page 1 : architecture Edge/Cloud
│   └── 02_explication-detaillee-mcd-schema-etoile.png  # page 2 : MCD
├── database_scripts/
│   ├── 01_staging.sql                      # couche ER, fidèle aux sources
│   ├── 02_datamart.sql                     # star schema (PostgreSQL)
│   └── 03_telemetrie_clickhouse.sql        # télémétrie (ClickHouse)
└── data_dictionary/
    └── dictionnaire_donnees.md             # dictionnaire complet des tables/colonnes
```

> [!TIP]
> `data/` n'est pas versionné (voir `.gitignore`) : le dataset s'obtient sur Kaggle, [arnabbiswas1/microsoft-azure-predictive-maintenance](https://www.kaggle.com/datasets/arnabbiswas1/microsoft-azure-predictive-maintenance).

## 🛠️ Stack technique

- ☁️ **OVHcloud** (région UE) — Object Storage (S3-compatible, multi-format), Kafka managé, PostgreSQL managé, ClickHouse managé, IAM, vRack
- 📡 **Kafka managé** — broker d'ingestion (pont MQTT → Kafka), absorbe la vélocité des flux capteurs
- 🐘 **PostgreSQL** — couche staging (ER) + datamart (star schema)
- ⚡ **ClickHouse** — télémétrie haute fréquence (série temporelle)
- 🔒 **Sécurité** — chiffrement en transit (TLS) et au repos, segmentation IT/OT (IEC 62443) côté Edge, IAM, vRack, VPN site-à-site (données) et VPN dédié aux sous-traitants de maintenance

## 📦 Contenu

- **`diagram/`** — architecture Edge/Cloud (sources, broker d'ingestion, stockage, bande transverse sécurité/supervision) et modèle conceptuel de données (MCD, notation Merise : entités, associations, cardinalités)
- **`database_scripts/`** — DDL des 3 couches : staging (ER), datamart (star schema, table de faits unifiée `fait_evenement`), télémétrie (ClickHouse)
- **`data_dictionary/`** — dictionnaire de données : chaque table, colonne, type, contrainte, description, exemple
