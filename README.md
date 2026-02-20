# Rendu - Atelier Collecte de Donnees (B3DEV FS)

## Pipeline ETL Azure Data Factory

**Auteur** : Hicham Benamara (IFAG Paris)
**Date** : 18/02/2026

---

## 1. Contexte du projet

Construction d'un pipeline ETL cloud avec **Azure Data Factory** pour consolider 4 sources de donnees heterogenes (CSV, JSON, XML, SQL) dans un Data Lake organise en zones (RAW / SILVER) et une base SQL de reporting.

## 2. Infrastructure Azure

**Resource Group** : `rg-b3dev-etl` (France Central)

| Ressource | Type | Nom |
|-----------|------|-----|
| Storage Account | Blob + ADLS Gen2 (HNS) | `stb3devetl` |
| SQL Server | Azure SQL | `sqlb3devetl` |
| SQL Database | Azure SQL Database Hyperscale | `dbb3devetl` |
| Data Factory | ADF V2 | `adfb3devetl` |

> Screenshot : `01-infrastructure/01-resource-group.png`

## 3. Sources de donnees

| Fichier | Format | Contenu | Stockage |
|---------|--------|---------|----------|
| `ventes.csv` | CSV (`;`) | 100 lignes - id, date, produit_id, montant, client_id, quantite | Blob Storage `data/raw/` |
| `catalog.json` | JSON | 20 produits - produit_id, nom, categorie, prix_catalogue | Blob Storage `data/raw/` |
| `regions.xml` | XML | 8 regions - pays, zone | Blob Storage `data/raw/` |
| `clients.sql` | SQL | 50 clients - client_id, nom, prenom, age, pays, date_inscription | Azure SQL Database `dbo.Clients` |

> Screenshot : `02-sources-blob/02-blob-raw-files.png`

## 4. Configuration ADF

### 4.1 Linked Services (3)

| Nom | Type | Cible |
|-----|------|-------|
| `ls_blob_storage` | Azure Blob Storage | `stb3devetl` |
| `ls_adls_gen2` | Azure Data Lake Storage Gen2 | `stb3devetl` (HNS) |
| `ls_sql_database` | Azure SQL Database | `dbb3devetl` |

> Screenshot : `03-linked-services/03-linked-services.png`

### 4.2 Datasets (11)

**Sources (4)** :
- `ventes_csv` : DelimitedText, delimiteur `;`, UTF-8 -> Blob Storage
- `catalog_json` : JSON, single document -> Blob Storage
- `regions_xml` : XML -> Blob Storage (+ `regions_sql` : table `dbo.Regions` -> SQL)
- `clients_sql` : table `dbo.Clients` -> Azure SQL Database

**Sinks RAW (4)** :
- `sink_raw_ventes` : CSV -> ADLS Gen2 `data/raw/`
- `sink_raw_catalog` : JSON -> ADLS Gen2 `data/raw/`
- `sink_raw_regions` : JSON -> ADLS Gen2 `data/raw/`
- `sink_raw_clients` : CSV -> ADLS Gen2 `data/raw/`

**Sinks Silver/Gold (2)** :
- `sink_silver_parquet` : Parquet (Snappy) -> ADLS Gen2 `data/silver/ventes/`
- `sink_sql_reportingventes` : table `dbo.ReportingVentes` -> Azure SQL Database

> Screenshot : `04-datasets/04-datasets-all.png`

## 5. Pipeline ETL : `pipeline_ETL_ventes`

### 5.1 Activites (6)

```
GetMetadata_CheckFiles
    |
    +---> Copy_ventes_to_RAW     (CSV -> Data Lake RAW)
    +---> Copy_catalog_to_RAW    (JSON -> Data Lake RAW)
    +---> Copy_regions_to_RAW    (XML -> Data Lake RAW)
    +---> Copy_clients_to_RAW    (SQL -> Data Lake RAW)

df_transform_ventes (Data Flow)
```

- **GetMetadata_CheckFiles** : Verifie l'existence du fichier `ventes.csv` dans Blob Storage
- **Copy_ventes_to_RAW** : Copie CSV avec gestion des erreurs (Continue on Error -> `data/raw/errors`)
- **Copy_catalog_to_RAW** : Copie JSON
- **Copy_regions_to_RAW** : Copie XML
- **Copy_clients_to_RAW** : Extraction SQL `SELECT * FROM Clients`
- **df_transform_ventes** : Data Flow de transformation (voir section 6)

> Screenshot : `05-pipeline/05-pipeline-canvas.png`
> JSON : `json/pipeline_ETL_ventes.json`

## 6. Data Flow : `df_transform_ventes`

### 6.1 Sources (4)

| Nom | Dataset | Type |
|-----|---------|------|
| `ventesSource` | `ventes_csv` | CSV |
| `clientsSource` | `clients_sql` | SQL Table |
| `catalogSource` | `catalog_json` | JSON |
| `regionsSource` | `regions_sql` | SQL Table |

### 6.2 Transformations (7)

1. **flattenCatalog** : `foldDown(unroll(catalogue))` - Aplatit le tableau JSON imbrique
2. **joinVentesClients** : `INNER JOIN` ventes <-> clients sur `client_id`
3. **joinVentesCatalog** : `LEFT JOIN` ventes <-> catalogue sur `produit_id`
4. **joinClientsRegions** : `LEFT JOIN` clients <-> regions sur `pays`
5. **selectColumns** : Desambiguation des colonnes dupliquees apres les 3 jointures
6. **filterQuantite** : `quantite > 0` - Filtre les lignes invalides
7. **derivedColumns** : Colonnes calculees :
   - `date` : conversion en type Date (`yyyy-MM-dd`)
   - `montant` : conversion en Decimal(10,2)
   - `quantite` : conversion en Integer
   - `ca_total` : `montant * quantite`
   - `is_senior` : `iif(age >= 60, true(), false())`
   - `annee` : `year(date)`
   - `mois` : `month(date)`
   - `produit_in_catalogue` : `iif(isNull(produit_id), 'NON', 'OUI')`

### 6.3 Sinks (2)

| Nom | Destination | Format | Details |
|-----|-------------|--------|---------|
| `sinkSilver` | ADLS Gen2 `data/silver/ventes/` | Parquet | Compression Snappy |
| `sinkSQL` | `dbo.ReportingVentes` | SQL Table | `TRUNCATE TABLE` avant insertion |

> Screenshot : `06-dataflow/06-dataflow-canvas.png`
> JSON : `json/df_transform_ventes.json`

## 7. Execution et validation

### 7.1 Pipeline runs

- **Statut** : 6/6 activites **Succeeded**
- Plusieurs executions de debug reussies

> Screenshot : `07-execution/07-pipeline-runs.png`

### 7.2 Data Lake - Zone RAW

Fichiers dans `data/raw/` :
- `ventes.csv`
- `catalog.json`
- `regions.xml`
- `clients.csv` (export de la table SQL)

> Screenshot : `08-datalake-raw/08-datalake-raw.png`

### 7.3 Data Lake - Zone SILVER

Fichiers dans `data/silver/ventes/` :
- `_SUCCESS` (marqueur de succes)
- 22 fichiers `part-00000-*.snappy.parquet` (~1.6 - 7.1 KiB chacun)

> Screenshot : `09-datalake-silver/09-datalake-silver.png`

### 7.4 Resultats SQL - Table ReportingVentes

**Nombre de lignes** : `SELECT COUNT(*) FROM ReportingVentes` = **100**

**Colonnes** (11) :
`id`, `date`, `produit_id`, `client_id`, `montant`, `quantite`, `categorie`, `pays`, `zone`, `ca_total`, `produit_in_catalogue`

**Repartition par zone** :

| Zone | Nombre de ventes |
|------|-----------------|
| EU-Ouest | 37 |
| Afrique-Nord | 19 |
| EU-Sud | 18 |
| EU-Centre | 13 |
| Amerique-Nord | 13 |

**Verification P1005** :
- P1005 est present dans `catalog.json` (prix_catalogue = 196.11, categorie = Informatique)
- `produit_in_catalogue = 'OUI'` pour les 4 ventes concernees (V0002, V0004, V0066, V0069)

> Screenshot : `10-sql-reporting/10-sql-results.png`

## 8. Table SQL ReportingVentes - Schema

```sql
CREATE TABLE ReportingVentes (
    id VARCHAR(10),
    date DATE,
    produit_id VARCHAR(10),
    client_id VARCHAR(10),
    montant DECIMAL(10,2),
    quantite INT,
    categorie VARCHAR(50),
    pays VARCHAR(50),
    zone VARCHAR(50),
    ca_total DECIMAL(12,2),
    produit_in_catalogue VARCHAR(5)
);
```

> Screenshot : `10-sql-reporting/10-table-creation.png`

## 9. Introduction OLAP

### Schema en etoile propose

**Table de faits** : `FactVentes`
- id (PK), date_id (FK), produit_id (FK), client_id (FK), region_id (FK)
- montant, quantite, ca_total

**Tables de dimensions** :
- `DimProduits` : produit_id (PK), nom, categorie, prix_catalogue
- `DimClients` : client_id (PK), nom, prenom, age, is_senior, date_inscription
- `DimRegions` : region_id (PK), pays, zone
- `DimTemps` : date_id (PK), date, annee, mois, trimestre, jour_semaine

**Hierarchies** :
- Geographique : Zone -> Pays
- Temporelle : Annee -> Trimestre -> Mois
- Produit : Categorie -> Produit

**Mesures** :
- CA total : `SUM(ca_total)`
- Nombre de ventes : `COUNT(id)`
- Montant moyen : `AVG(montant)`
- Quantite totale : `SUM(quantite)`
- Panier moyen : `SUM(ca_total) / COUNT(DISTINCT client_id)`

**Alimentation** : La table `ReportingVentes` serait importee dans Power BI via DirectQuery ou Import depuis Azure SQL Database.

## 10. Bonus

### 10.1 Dashboard Power BI

Rapport `Dashboard_CA_ETL_B3DEV` sur Power BI Service avec :
- **Graphique barres horizontales** : CA total par pays (Maroc en tete)
- **Graphique camembert** : CA total par categorie (Informatique 31.15%, Accessoires 23.46%, Maison 22.8%, Gaming 22.51%)

> Screenshot : `11-power-bi/11-dashboard-powerbi.png`

### 10.2 Trigger horaire

Trigger `trigger_horaire` configure dans ADF :
- Type : Schedule
- Recurrence : Toutes les heures
- Statut : Stopped (en attente d'activation)

> Screenshot : `12-trigger/12-trigger-horaire.png`

---

## Arborescence du rendu

```
rendu/
├── README.md
├── sources/
│   ├── ventes.csv          (100 lignes, CSV ;)
│   ├── catalog.json        (20 produits, JSON)
│   ├── regions.xml         (8 regions, XML)
│   └── clients.sql         (50 clients, SQL)
├── json/
│   ├── pipeline_ETL_ventes.json
│   └── df_transform_ventes.json
├── 01-infrastructure/
│   └── 01-resource-group.png
├── 02-sources-blob/
│   └── 02-blob-raw-files.png
├── 03-linked-services/
│   └── 03-linked-services.png
├── 04-datasets/
│   └── 04-datasets-all.png
├── 05-pipeline/
│   └── 05-pipeline-canvas.png
├── 06-dataflow/
│   ├── 06-dataflow-canvas.png
│   └── 06-dataflow-zoomed.png
├── 07-execution/
│   └── 07-pipeline-runs.png
├── 08-datalake-raw/
│   └── 08-datalake-raw.png
├── 09-datalake-silver/
│   └── 09-datalake-silver.png
├── 10-sql-reporting/
│   ├── 10-sql-results.png
│   ├── 10-sql-validation.png
│   └── 10-table-creation.png
├── 11-power-bi/
│   └── 11-dashboard-powerbi.png
└── 12-trigger/
    └── 12-trigger-horaire.png
```