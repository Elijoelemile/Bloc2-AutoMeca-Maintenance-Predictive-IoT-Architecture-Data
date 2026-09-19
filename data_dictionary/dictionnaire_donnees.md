# Dictionnaire de données — AutoMeca Systems

Dataset source : Microsoft Azure Predictive Maintenance (Kaggle,
`arnabbiswas1/microsoft-azure-predictive-maintenance`), 5 fichiers,
100 machines, période 2014-06-01 à 2016-01-01.

## 1. Couche staging (PostgreSQL, schéma `staging`)

Reflète fidèlement les fichiers sources. Clé naturelle `machine_id`
utilisée comme clé étrangère entre les tables.

### staging.machines — source `PdM_machines.csv`
Grain : 1 ligne / machine. 100 lignes.

| Colonne | Type | Contrainte | Description | Exemple |
|---|---|---|---|---|
| machine_id | SMALLINT | PK | Identifiant machine | 1 |
| model | VARCHAR(20) | NOT NULL | Modèle de la machine | model3 |
| age | SMALLINT | NOT NULL, 0-40 | Âge en années | 18 |

### staging.erreurs — source `PdM_errors.csv`
Grain : 1 ligne / événement d'erreur. ~3 900 lignes.

| Colonne | Type | Contrainte | Description | Exemple |
|---|---|---|---|---|
| id_erreur | BIGSERIAL | PK | Clé technique | 1 |
| datetime_evt | TIMESTAMP | NOT NULL | Horodatage de l'erreur | 2015-01-03 07:00:00 |
| machine_id | SMALLINT | FK → machines | Machine concernée | 1 |
| error_id | VARCHAR(10) | NOT NULL | Code d'erreur (error1-error5) | error1 |

### staging.pannes — source `PdM_failures.csv`
Grain : 1 ligne / panne. ~760 lignes. **Table cible du futur modèle
prédictif (Bloc 3/4)**.

| Colonne | Type | Contrainte | Description | Exemple |
|---|---|---|---|---|
| id_panne | BIGSERIAL | PK | Clé technique | 1 |
| datetime_evt | TIMESTAMP | NOT NULL | Horodatage de la panne | 2015-01-05 06:00:00 |
| machine_id | SMALLINT | FK → machines | Machine concernée | 1 |
| failure_comp | VARCHAR(10) | NOT NULL | Composant en panne (comp1-comp4) | comp4 |

### staging.maintenances — source `PdM_maint.csv`
Grain : 1 ligne / intervention planifiée. ~3 300 lignes.

| Colonne | Type | Contrainte | Description | Exemple |
|---|---|---|---|---|
| id_maintenance | BIGSERIAL | PK | Clé technique | 1 |
| datetime_evt | TIMESTAMP | NOT NULL | Horodatage de la maintenance | 2014-06-01 06:00:00 |
| machine_id | SMALLINT | FK → machines | Machine concernée | 1 |
| comp | VARCHAR(10) | NOT NULL | Composant remplacé (comp1-comp4) | comp2 |

---

## 2. Couche datamart — star schema (PostgreSQL, schéma `datamart`)

Construite à partir du staging par le pipeline du Bloc 3. Table de
faits **unifiée** (erreurs + pannes + maintenances) : mêmes grain et
structure minimale dans les 3 sources, ce qui évite un UNION ALL
répété lors de la reconstitution de l'historique chronologique d'une
machine (besoin central pour le feature engineering du modèle
prédictif).

### datamart.dim_date
| Colonne | Type | Description |
|---|---|---|
| id_date | INT PK | Format AAAAMMJJ |
| date_complete | DATE | Date calendaire |
| annee, trimestre, mois, jour | SMALLINT | Attributs calendaires |
| jour_semaine | VARCHAR(10) | Lundi, mardi, ... |
| est_weekend | BOOLEAN | Vrai si samedi/dimanche |

### datamart.dim_machine
| Colonne | Type | Description |
|---|---|---|
| id_machine | SERIAL PK | Clé de substitution |
| machine_id_nat | SMALLINT | Clé naturelle (= staging.machines.machine_id) |
| model | VARCHAR(20) | Modèle |
| age | SMALLINT | Âge en années |
| tranche_age | VARCHAR(20) | Regroupement (ex. "0-5 ans") |

### datamart.dim_type_evenement
| Colonne | Type | Description |
|---|---|---|
| id_type_evenement | SMALLSERIAL PK | Clé de substitution |
| code_type | VARCHAR(20) | ERREUR \| PANNE \| MAINTENANCE |
| libelle | VARCHAR(60) | Libellé lisible |

### datamart.dim_code_evenement
Dimension pont regroupant les deux référentiels de code observés
(error1-5 pour les erreurs, comp1-4 pour les pannes/maintenances)
sous une clé de substitution commune.

| Colonne | Type | Description |
|---|---|---|
| id_code | SERIAL PK | Clé de substitution |
| id_type_evenement | SMALLINT FK | Type d'événement associé |
| code_brut | VARCHAR(10) | error1-5 ou comp1-4 |
| libelle | VARCHAR(60) | Libellé lisible |

### datamart.dim_operateur
Opérateur/technicien ayant réalisé une intervention. **Non peuplée** :
le dataset source (Kaggle) ne contient aucune donnée opérateur dans
`PdM_errors.csv`, `PdM_failures.csv` ni `PdM_maint.csv`. Table
modélisée pour anticiper une future intégration GMAO, où cette
information existerait.

| Colonne | Type | Description |
|---|---|---|
| id_operateur | SERIAL PK | Clé de substitution |
| matricule | VARCHAR(20) | Matricule (GMAO) |
| nom | VARCHAR(100) | Nom de l'opérateur |
| equipe | VARCHAR(50) | Équipe/atelier |

### datamart.fait_evenement
Grain : 1 ligne = 1 événement machine (erreur, panne ou maintenance)
à un instant donné. ~8 000 lignes au total.

| Colonne | Type | Description |
|---|---|---|
| id_evenement | BIGSERIAL PK | Clé technique |
| id_date | INT FK → dim_date | Jour de l'événement |
| datetime_evt | TIMESTAMP | Horodatage précis |
| id_machine | INT FK → dim_machine | Machine concernée |
| id_type_evenement | SMALLINT FK → dim_type_evenement | Nature de l'événement |
| id_code | INT FK → dim_code_evenement | Code précis (error*/comp*) |
| id_operateur | INT FK → dim_operateur, nullable | NULL pour ERREUR (pas d'intervention) ; non renseigné pour PANNE/MAINTENANCE (absent du dataset source) |
| source_fichier | VARCHAR(40) | Traçabilité (nom du CSV source) |

### datamart.v_interventions (vue)
Sous-ensemble de `fait_evenement` restreint aux interventions réelles
— `code_type IN ('PANNE', 'MAINTENANCE')`. Une alerte seule (ERREUR)
n'est pas une intervention : ce filtre répond au besoin de reporting
maintenance sans dupliquer le fait unifié.

---

## 3. Télémétrie — série temporelle (ClickHouse)

### automeca.telemetrie — source `PdM_telemetry.csv`
Grain : 1 ligne / machine / heure. 876 100 lignes/an.
Hors star schema : volume et cadence trop élevés pour PostgreSQL,
table dédiée triée par (machine_id, datetime_mes).

| Colonne | Type | Description | Exemple |
|---|---|---|---|
| machine_id | UInt16 | Identifiant machine | 1 |
| datetime_mes | DateTime | Horodatage de la mesure (horaire) | 2015-01-01 06:00:00 |
| volt | Float64 | Tension mesurée | 176.22 |
| rotate | Float64 | Vitesse de rotation | 418.50 |
| pressure | Float64 | Pression | 113.08 |
| vibration | Float64 | Vibration | 45.09 |
