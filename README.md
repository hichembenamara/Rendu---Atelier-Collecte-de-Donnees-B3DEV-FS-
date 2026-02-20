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
| `ventes.csv` | CSV (`;`) | 100 lignes | Blob Storage `data/raw/` |
| `catalog.json` | JSON | 20 produits | Blob Storage `data/raw/` |
| `regions.xml` | XML | 8 regions | Blob Storage `data/raw/` |
| `clients.sql` | SQL | 50 clients | Azure SQL Database `dbo.Clients` |

## 4-10. Configuration ADF, Pipeline, Data Flow, Execution, OLAP, Bonus

Voir README complet pour Linked Services, Datasets, Pipeline ETL, Data Flow, runs, Data Lake RAW/SILVER, ReportingVentes, schema etoile, Power BI, Trigger.

---

## Arborescence du rendu

```
rendu/
├── README.md
├── sources/
│   ├── ventes.csv
│   ├── catalog.json
│   ├── regions.xml
│   └── clients.sql
├── json/
│   ├── pipeline_ETL_ventes.json
│   └── df_transform_ventes.json
├── 01-infrastructure/ ... 12-trigger/ (screenshots)
```
