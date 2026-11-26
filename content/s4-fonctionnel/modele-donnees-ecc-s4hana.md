---
title: "Impacts S/4HANA : adéquation du modèle de données ECC ↔ S/4HANA"
description: "Tables ECC remplacées, conservées ou modifiées en S/4HANA (ACDOCA, MATDOC, PRCD_ELEMENTS, VBUK/VBUP, CVI, MATNR 40), vues de compatibilité, sources SAP (SIC, Simplification List, notes S4TWL) et pièges de la réplication."
tags: ["s4hana", "migration", "ecc", "modele-de-donnees", "acdoca", "matdoc", "replication"]
date: "2026-11-26"
draft: false
---

> [!abstract] En bref
> Pour une migration ECC → S/4HANA, quelles tables sont remplacées par des vues de compatibilité, lesquelles sont conservées ou modifiées, et ce que cela implique pour la réplication vers un entrepôt.
> Typologie A–F, mapping par domaine (FI/CO, MM-IM, SD, MD) et sources SAP de référence.

**Projet** : migration SAP ECC → S/4HANA + datawarehouse Snowflake (remplacement de l'alimentation BO depuis les Calculation Views HANA d'ECC)
**Objet du document** : identifier les sources, le processus SAP de référence et les gaps de modèle de données à traiter avant de définir (a) les tables à répliquer S/4HANA → Snowflake et (b) les règles de reproduction des 44 vues HANA.
**Version** : 0.1 — cadrage, à valider
**Statut des informations** : chaque affirmation porte un niveau de confiance (🟢 source SAP officielle recoupée / 🟡 source SAP + sources secondaires concordantes / 🔴 non vérifié sur source officielle — à confirmer sur le système cible)

---

## 1. Sources fournissant la matière pour le mapping ECC → S/4HANA

### 1.1 Sources SAP officielles (référentiel prioritaire)

| # | Source | Contenu utile pour le mapping | Accès | Confiance |
|---|--------|-------------------------------|-------|-----------|
| S1 | **Simplification Item Catalog (SIC)** — `https://launchpad.support.sap.com/#sic` (SAP for Me) | Base *machine-readable* de tous les *simplification items* (S4TWL-…). Filtrable par release source (ECC) / release cible S/4HANA, par domaine applicatif. Chaque item référence sa *Business Impact Note*, les tables/objets concernés, et les checks de pertinence. Export ZIP possible (réutilisé par le SI-Check). | S-User | 🟢 |
| S2 | **Simplification List (PDF par release)** — help.sap.com, ex. `SIMPL_OP2023.pdf` (2023), et édition 2025 | Version consolidée et lisible du SIC. Chaque S4TWL décrit : description, impact business, procédure requise, custom code, tables et vues de compatibilité. Indispensable pour lire les changements FIN, MM-IM, SD, MD. | Public | 🟢 |
| S3 | **SAP Notes « Business Impact » (S4TWL)** — cf. liste §1.4 | Détail table par table, champ par champ, cookbooks en pièces jointes (ex. note 2220005 pricing, note 2206980 MM-IM). | S-User | 🟢 |
| S4 | **SAP Readiness Check for SAP S/4HANA** (rapport propre au système ECC du client) | Liste des simplification items **pertinents pour ce système** (relevance), volumétrie des tables FI classiques et estimation du nombre de lignes ACDOCA, analyse du custom code, add-ons. C'est la seule source qui restreint le catalogue générique au périmètre réellement utilisé. | Équipe conversion | 🟢 |
| S5 | **Simplification Item Check (SI-Check)** — report `/SDF/RC_START_CHECK` (note 2399707) | Relevance check + consistency check exécutés sur le système source. Confirme quels items s'appliquent (ex. Business Function actives, contenu de tables). | Équipe conversion | 🟢 |
| S6 | **Custom Code Migration (ATC, variante `S4HANA_READINESS`)** | Identifie les développements ABAP qui lisent/écrivent les tables modifiées. **Limite majeure : ne couvre pas les Calculation Views HANA natives** (objets du repository HANA, pas du dictionnaire ABAP). Utile uniquement si des CDS/ABAP alimentent les vues. | Équipe conversion | 🟢 |
| S7 | **Conversion Guide for SAP S/4HANA** (help.sap.com, par release) | Phases, pré-checks, CVI, migration FIN/MM-IM, ordre des activités. | Public | 🟢 |
| S8 | **KBA 3523400** — *Overview of the compatibility view topic in S/4HANA* + KBA 2713495 (perf. ancien modèle stock) | Liste des tables devenues vues de compatibilité (ANEA, ANEK, ANEP, ANLC, BSAD, BSID, BSAK, BSIK, BSAS, BSIS, COEP, COSP, COSS, COVP, FAGLFLEXT, GLT0, MARC, MARD, MARDH, MBEW, MBEWH, MCHB, MCHBH, MKPF, MSEG…) et mécanisme de redirection (proxy object). | S-User | 🟢 |
| S9 | **SAP LT Replication Server – « Compatibility Views and SAP LT Replication Server »** (PDF help.sap.com) | Décrit précisément ce qui se passe quand on réplique une table redirigée vers une vue de compatibilité : le réplicateur crée la table physique (vide), pas la vue ; il faut répliquer les tables sous-jacentes (ACDOCA, etc.). Transposable à Fivetran. | Public | 🟢 |
| S10 | **Dictionnaire ABAP du système S/4HANA cible** (SE11 → *Proxy Object* ; DDL sources des vues de compat ; SE16N) | Vérité terrain : pour chaque table source d'une vue HANA, vérifier si elle est (i) une table transparente encore alimentée, (ii) une table redirigée vers une vue de compatibilité, (iii) une table conservée mais vide. À exécuter sur le bac à sable S/4 dès qu'il existe. | Projet | 🟢 |

### 1.2 Sources de l'outil de réplication (Fivetran)

| # | Source | Point clé | Confiance |
|---|--------|-----------|-----------|
| F1 | **Fivetran – SAP ERP on HANA connector** (`fivetran.com/docs/connectors/databases/sap/sap-erp-hana`) | Supporte tables transparentes (column/row store) et vues. **Avertissement explicite : ne pas répliquer les tables de base qui pointent vers des proxy objects (vues de compatibilité) — elles retournent des données incomplètes ou vides ; répliquer directement la vue de compatibilité.** Pas de vues paramétrées ni de projection views. Pas de CDC sur vues et tables row-store (mode « no-trigger » seulement). | 🟢 |
| F2 | **Fivetran – SAP OData connector** (ODP / CDS) | Alternative si l'on souhaite exposer des CDS standards S/4 (ex. `I_JournalEntryItem`) plutôt que des tables brutes. Hors périmètre actuel mais à connaître. | 🟢 |
| F3 | **fivetran/dbt_sap** (GitHub) | Modèles dbt qui reconstituent BSID/COEP/COSP/COSS/FAGLFLEXA « compat » à partir d'ACDOCA côté entrepôt. Bon indicateur des règles de dérivation ACDOCA → anciennes structures. Non officiel SAP. | 🟡 |

### 1.3 Sources secondaires (validation croisée uniquement)

- SAP Learning Journeys (learning.sap.com) : *Identifying Data Model Simplifications in SAP S/4HANA (Sales)*, *Simplifications for the Material Master*, *Business Partners in SAP S/4HANA* — contenu SAP, pédagogique, non contractuel. 🟡
- Blogs SAP Community rédigés par SAP (« SAP S/4HANA: Inventory Management Tables NSDM », « Data volume problems during an S/4HANA conversion », « SI-Check – how to do it right »). 🟡
- SAP PRESS blog (Jotev, *Obsolete and New Transaction Codes and Tables in S/4HANA Finance*). 🟡
- Blogs communautaires / tiers (queryviz, mps4hana, etc.) : utilisés seulement pour repérer des pistes, jamais comme preuve. 🔴

### 1.4 SAP Notes de référence à récupérer (S-User requis)

| Note | Sujet | Domaine |
|------|-------|---------|
| 2270333 | S4TWL – Data model changes in FIN (Universal Journal / ACDOCA) | FI/CO/AA/ML |
| 1976487 | Compatibility views FIN – read only, write access supprimé | FIN |
| 2206980 | S4TWL – Material Inventory Management: change of data model (MATDOC) | MM-IM |
| 2713495 | Performance des accès custom à l'ancien modèle de stock | MM-IM |
| 2267306 | S4TWL – SD data model: elimination of status tables VBUK/VBUP | SD |
| 2198647 | Détail des champs statut déplacés vers VBAK/VBAP/LIKP/LIPS/VBRK | SD |
| 2220005 | S4TWL – Data model changes in pricing & condition technique (KONV → PRCD_ELEMENTS) + cookbook | SD/MM |
| 2200691 | SD rebate simplifications (VBOX) | SD |
| 2267140 | S4TWL – Material number field length extension (18 → 40) | MD |
| 2713963 | FAQ Business Partner / CVI en conversion | MD |
| 3523400 | KBA overview compatibility views | Transverse |
| 2399707 | Simplification Item Check | Outillage |

> ⚠️ Les numéros de notes ci-dessus proviennent de sources SAP ou concordantes, mais **le contenu intégral des notes n'a pas pu être consulté** (accès S-User). À vérifier une par une avant utilisation contractuelle.

### 1.5 Ce qui n'a pas été trouvé / limites

- **Aucun document SAP officiel ne traite du mapping des Calculation Views HANA natives (ECC on HANA) vers S/4HANA.** Les outils SAP (SIC, SI-Check, ATC) raisonnent sur le custom code ABAP et les tables du dictionnaire ; les vues du repository HANA (`_SYS_BIC`) sont invisibles pour eux. → Le mapping devra être **produit manuellement** à partir des sources des 44 vues (cf. §2.3).
- Pas trouvé de liste officielle consolidée « table ECC → table S/4 » couvrant tous les domaines en un seul document : l'information est éclatée par simplification item. La synthèse §3 est donc une consolidation projet, pas un document SAP.
- La release S/4HANA cible du projet n'est pas connue à ce stade : certains items diffèrent entre 1909, 2020, 2022, 2023, 2025 (ex. adaptation progressive des transactions CO à ACDOCA, dépréciation CKMLCT en 2025).

---

## 2. Processus proposé / recommandé par SAP pour le mapping

### 2.1 Le processus SAP standard (conversion système)

SAP structure la conversion (« brownfield ») autour de la méthode SAP Activate et de trois familles d'outils, dans cet ordre :

| Phase | Outil SAP | Rôle vis-à-vis du mapping de données |
|-------|-----------|---------------------------------------|
| **Discover / Prepare** | **SAP Readiness Check** (données collectées par `RC_COLLECT_ANALYSIS_DATA`, rapport dans SAP for Me) | Vue macro : quels simplification items sont pertinents, volumétrie des tables à migrer (FI, MM-IM), custom code impacté, sizing. |
| **Prepare** | **Simplification Item Check** (`/SDF/RC_START_CHECK`, client 000, catalogue SIC uploadé, release cible sélectionnée) | *Relevance check* : quels items s'appliquent. *Consistency check* : cohérence des données préalable à la conversion (obligatoire avant SUM). |
| **Prepare / Explore** | **Simplification Item Catalog** + **Business Impact Notes** | Pour chaque item pertinent, lire l'impact sur le modèle de données et la procédure requise. |
| **Explore / Realize** | **Custom Code Migration** (ATC remote, variante `S4HANA_READINESS`, ou app Custom Code Migration sur BTP) | Détecte les objets ABAP custom qui accèdent aux tables modifiées (lecture des tables remplacées, écriture sur KONV, etc.). |
| **Realize** | **SUM** avec migration FIN (ACDOCA) et MM-IM (MATDOC), CVI (Business Partner) | Exécute la migration des données dans les nouvelles structures ; tables anciennes vidées / remplacées par vues de compat. |
| **Realize / Deploy** | **Data Transition Validation** (DTV) | Compare les données métier avant/après conversion (contrôles standard). |

Points de doctrine SAP à retenir pour la BI :
1. **Compatibility views** : les tables totaux/index/lignes anciennes (FIN, CO, AA, ML, MM-IM) sont remplacées par des vues de compatibilité **de même nom** ; l'accès en lecture ABAP est redirigé de façon transparente, l'écriture est interdite. Ces vues sont des CDS générées, souvent complexes, avec risque de performance.
2. **Nouvelles persistances de référence** : ACDOCA (journal universel), MATDOC / MATDOC_EXTRACT (mouvements et stocks), PRCD_ELEMENTS (conditions de prix), tables BUT* (Business Partner). SAP recommande de développer les nouveaux rapports sur ces tables (ou sur les CDS standards), pas sur les vues de compat.
3. **Denormalisation SD** : les statuts sont portés par les tables d'en-tête/poste (VBAK, VBAP, LIKP, LIPS, VBRK) ; VBUK/VBUP existent mais ne sont plus alimentées ; pas de vue de compatibilité, mais des modules fonction de lecture (`SD_VBUK_READ_FROM_DOC`).

### 2.2 Ce que le processus SAP ne couvre pas pour ce projet

- Les 44 Calculation Views HANA ne sont **pas analysées** par Readiness Check / ATC.
- Le réplicateur (Fivetran) travaille au niveau **base de données / dictionnaire**, sans redirection ABAP : répliquer `COEP`, `MARD`, `BSIS`, `MKPF`… sur S/4 remonte une table physique **vide ou partielle** (données legacy uniquement). Le mécanisme de redirection qui protège l'ABAP ne protège pas la réplication.
- Les CDS de compatibilité peuvent, elles, être répliquées comme vues (Fivetran le recommande) mais **sans CDC**, avec un coût de calcul reporté sur HANA à chaque extraction complète, et une dépendance à une couche « legacy » que SAP démonte progressivement (transactions CO ré-écrites sur ACDOCA release après release). Ce n'est pas une cible durable.

### 2.3 Processus proposé pour le projet (adaptation)

| Étape | Livrable | Responsable |
|-------|----------|-------------|
| **P1. Inventaire des sources des 44 vues** — extraction depuis le repository HANA d'ECC (`_SYS_REPO`/`OBJECT_DEPENDENCIES`, ou export XML des vues via Eclipse) : tables, colonnes, jointures, filtres, paramètres, analytic privileges. | Matrice *vue → table → colonne* | Consultant data |
| **P2. Qualification de chaque table source** selon le statut S/4 (cf. §3 typologie A–F), croisée avec le Readiness Check et le SIC filtré sur la release cible. | Matrice enrichie *table ECC → statut S/4 → table(s) cible* | Consultant data + équipe conversion |
| **P3. Mapping colonne à colonne** pour les tables remplacées : ACDOCA vs (BSEG, FAGLFLEXA, FAGLFLEXT, GLT0, COEP, COSP, COSS, ANEP, ANLC, MLIT…) ; MATDOC vs (MKPF, MSEG) ; MATDOC_EXTRACT / NSDM vs (MARD, MCHB, MSKA…) ; PRCD_ELEMENTS vs KONV ; VBAK/VBAP/LIKP/LIPS/VBRK vs VBUK/VBUP. Point d'appui : lire les DDL sources des vues de compat dans le système cible (elles contiennent la formule exacte), et les modèles dbt_sap comme contre-vérification. | Fiches de mapping par table | Consultant data |
| **P4. Vérification terrain sur le bac à sable S/4HANA** : SE11 (*proxy object*), SE16N (comptages), comparaison ECC vs S/4 sur un même périmètre après conversion test. | PV de vérification | Consultant data + Basis |
| **P5. Liste des tables à répliquer** (SPEC 1) et règles de reconstruction des vues (SPEC 2/3), en excluant toute table redirigée. | SPEC 1, 2, 3 | Consultant data |

---

## 3. Synthèse des gaps de modèle de données ECC → S/4HANA

### 3.1 Typologie retenue

| Type | Définition | Conséquence réplication |
|------|-----------|-------------------------|
| **A – Remplacée par vue de compatibilité** | La table n'est plus alimentée ; un CDS de même nom la remplace (proxy object). | **Ne pas répliquer la table**. Répliquer la table cible (ACDOCA, MATDOC…) et reconstruire la logique dans Snowflake. |
| **B – Conservée mais plus alimentée (sans vue de compat)** | La table existe, contient au mieux des données legacy. | **Ne pas répliquer**. Basculer sur la nouvelle persistance. |
| **C – Hybride** | Table master data toujours persistée pour ses attributs, mais dont les champs de quantité/valeur sont calculés à la volée via CDS. | Répliquer la table **pour ses attributs uniquement** ; les champs de stock proviennent de MATDOC_EXTRACT / MATDOC. |
| **D – Conservée, rôle ou contenu modifié** | Table toujours écrite mais dont le périmètre ou la sémantique change. | Répliquer, mais **revoir les règles métier** qui la consomment. |
| **E – Nouvelle table** | N'existe pas dans ECC. | À ajouter au périmètre de réplication. |
| **F – Changement de champ** | Longueur, type, nouveau champ, champ obsolète. | Impact sur types Snowflake, clés de jointure, filtres. |

### 3.2 Finance (FI / CO / AA / ML) — journal universel ACDOCA

| Table(s) ECC | Type | Cible S/4HANA | Remarques / règles à prévoir | Confiance |
|--------------|------|---------------|-------------------------------|-----------|
| GLT0, FAGLFLEXT (totaux GL classique / new GL) | A | ACDOCA (agrégation) | Totaux calculés à la volée ; reproduire l'agrégation par ledger (RLDNR), période, compte (RACCT), centre de profit… | 🟢 |
| FAGLFLEXA (lignes new GL), FAGLBSIS, FAGLBSAS | A | ACDOCA | La vue compat FAGLFLEXA expose ACDOCA hors lignes de migration (`BSTAT = 'C'`, périodes > 0) — filtre à reproduire. Champs `MIG_SOURCE`, `BSTAT` à connaître. | 🟢 |
| BSIS, BSAS (index GL) ; BSID, BSAD (clients) ; BSIK, BSAK (fournisseurs) | A | ACDOCA (+ BSEG pour certains champs) | Statut ouvert/soldé dérivé (clearing) ; attention aux champs conditions de paiement (ZTERM, ZBD1T…) absents d'ACDOCA, toujours dans BSEG. | 🟢 |
| KNC1, KNC3, LFC1, LFC3 (totaux AR/AP) | A | ACDOCA | Idem totaux. | 🟢 |
| COEP (lignes CO, value types 04 et 11), COSP, COSS (totaux CO), COVP | A | ACDOCA | Autres value types restent dans COEP / COSP_BAK / COSS_BAK ; accès données historiques via `V_COEP_ORI`. Mapping objet CO (OBJNR) → champs ACDOCA (KOSTL, AUFNR, PS_PSP_PNR…). | 🟢 |
| ANEK, ANEP, ANEA, ANLP, ANLC (FI-AA) | A | ACDOCA (+ nouvelles tables FI-AA) | Évaluation des immobilisations dans le journal universel ; ledger/depreciation area à mapper. | 🟢 |
| MLIT, MLPP, MLPPF, MLCR, MLCD, CKMI1, BSIM (Material Ledger) | A | ACDOCA / ML nouveau modèle | Material Ledger **obligatoire** en S/4 (valorisation stock) ; impact sur les vues de valorisation. CKMLCT déprécié en 2025 (→ `FMLV_CKMLCT`). | 🟢 |
| BSEG | D | BSEG (conservée) + ACDOCA | Toujours alimentée pour les documents FI (vue « entry view », gestion des postes ouverts), en lecture seule ; limite 999 lignes maintenue alors qu'ACDOCA a 6 positions (DOCLN). Les écritures purement CO ne sont pas dans BSEG. Reporting analytique attendu sur ACDOCA. | 🟢 |
| BKPF | D | BKPF (conservée) | Toujours l'en-tête ; champs analytiques également présents dans ACDOCA. | 🟢 |
| GLPCA, GLPCP, GLPCT (PCA classique) | D | ACDOCA (PRCTR) | PCA classique conservé techniquement ; SAP recommande le centre de profit dans ACDOCA ; pas de migration obligatoire. Décision projet à prendre. | 🟡 |
| CE1xxxx–CE4xxxx (CO-PA costing-based) | D | ACDOCA (Margin Analysis) | CO-PA costing-based fonctionne encore ; SAP ne développe plus que Margin Analysis (account-based, dans ACDOCA). Si des vues HANA lisent CE1/CE4 : décision fonctionnelle. | 🟡 |
| — | E | **ACDOCA**, ACDOCP (plan), FINSC_LEDGER, FINSC_LD_CMP, (ACDOCT vue totaux) | Nouvelles tables à répliquer. Nature exacte d'ACDOCT (table vs vue) à confirmer sur le système cible. | 🟢 / 🔴 pour ACDOCT |
| Champs | F | — | Montants ACDOCA 23 chiffres ; DOCLN 6 car. ; ledger obligatoire (RLDNR) ; multi-devises (HSL, KSL, OSL…) ; champs `MIG_SOURCE`/`BSTAT` sur données migrées ; CI_COBL repris automatiquement. | 🟢 |

### 3.3 Logistique — gestion des stocks (MM-IM), nouveau modèle NSDM

| Table(s) ECC | Type | Cible S/4HANA | Remarques / règles | Confiance |
|--------------|------|---------------|--------------------|-----------|
| MKPF, MSEG (documents matière) | A | **MATDOC** (dénormalisée en-tête + poste + attributs) | Vues compat `NSDM_V_MKPF`, `NSDM_V_MSEG`. Filtre `RECORD_TYPE = 'MDOC'` pour les lignes document. Champs facilitateurs (`STOCK_QTY`, `CONSUMPTION_QTY`, `MAT_KDAUF`…). MKPF/MSEG ne contiennent que d'éventuelles données legacy. | 🟢 |
| MARC, MARD, MCHB, MKOL, MSKA, MSKU, MSLB, MSPR (hybrides) | C | Table conservée (attributs) + **MATDOC_EXTRACT** (quantités) | Ex. `MARD-LABST` calculé via `NSDM_V_MARD` = somme MATDOC_EXTRACT (+ MIG_DELTA pour compenser les documents archivés). Répliquer MARC/MARD pour les attributs, recalculer les stocks. | 🟢 |
| MSSA, MSSQ, MSTB, MSTE, MSTQ (agrégats purs) | A | MATDOC_EXTRACT | Plus persistées. | 🟢 |
| MARCH, MARDH, MCHBH, MKOLH, MSKAH, MSKUH, MSLBH, MSPRH, MSSAH… (historiques) | A | MATDOC / MATDOC_EXTRACT | Stocks « à date » recalculés à partir des documents ; règle de reconstitution à spécifier si les vues HANA font du stock historique. | 🟢 |
| MBEW, MBEWH (valorisation) | C | MBEW (attributs prix) + ML / ACDOCA (quantités-valeurs) | Hybride : quantité et valeur de stock calculées ; prix standard/moyen toujours en MBEW. | 🟢 |
| — | E | **MATDOC**, **MATDOC_EXTRACT**, MIG_DELTA | À répliquer. | 🟢 |

### 3.4 Ventes (SD)

| Table(s) ECC | Type | Cible S/4HANA | Remarques / règles | Confiance |
|--------------|------|---------------|--------------------|-----------|
| VBUK (statuts en-tête) | B | VBAK, LIKP, VBRK (champs statut intégrés) | Pas de vue de compat. Reconstruire les jointures statut depuis les tables en-tête. | 🟢 |
| VBUP (statuts poste) | B | VBAP, LIPS | Idem. | 🟢 |
| VAKPA, VAPMA, VLKPA, VLPMA, VRKPA, VRPMA (index) | B | — (requêtes directes sur VBAK/VBAP…) | Si utilisés en vue HANA pour la sélection par client/matière : remplacer par jointures directes. | 🟢 |
| VBOX (index rabais) | B | Settlement Management (condition contracts) | FI-AR rebate remplacé. | 🟢 |
| KONV (conditions de prix) | A/B | **PRCD_ELEMENTS** (+ vue lecture `V_KONV_CDS`) | KONV subsiste comme structure ; persistance dans PRCD_ELEMENTS ; devise document obligatoire ; champs KOLNR/ZAEHK allongés. Toute vue HANA qui lit KONV est impactée. | 🟢 |
| VBFA (flux documents) | D | VBFA (simplifiée) | Table conservée, modèle simplifié ; `VBTYP` (CHAR1) → `VBTYPL` (CHAR4) : impacte les filtres par catégorie de document. | 🟢 |
| VBAK, VBAP, LIKP, LIPS, VBRK, VBRP | D | Enrichies | Nouveaux champs statut ; VBTYPL. | 🟢 |
| KNKK / FI-AR-CR (credit management classique) | D | FIN-FSCM-CR (UKM_*) | Si vues de crédit client. | 🟡 |

### 3.5 Données de base (MD)

| Table(s) ECC | Type | Cible S/4HANA | Remarques / règles | Confiance |
|--------------|------|---------------|--------------------|-----------|
| KNA1, KNB1, KNVV, LFA1, LFB1, LFM1… | D | Conservées, synchronisées par **CVI** depuis le Business Partner | Toujours alimentées ; mais la maintenance passe par BP. Numérotation client/fournisseur potentiellement modifiée par la stratégie CVI (same number vs nouveaux numéros) → **risque sur les clés**. Nouvelles tables BUT000, BUT020, BUT050, CVI_CUST_LINK, CVI_VEND_LINK à répliquer si l'on veut la vue BP. | 🟢 |
| MARA, MAKT, MVKE… | F | Conservées | `MATNR` CHAR18 → CHAR40 dans **toutes** les tables ; contenu inchangé sans activation de l'extension, mais type Snowflake à dimensionner à 40. `MAKTX` 40 car. | 🟢 |
| Classification / variantes (AUSP, INOB…) | 🔴 | — | Non analysé ; à vérifier si des vues HANA les utilisent. | 🔴 |

### 3.6 Autres domaines à qualifier (non couverts par cette première passe)

| Domaine | Point d'attention | Confiance |
|---------|-------------------|-----------|
| MM-PUR (EKKO, EKPO, EKBE, EKET) | Peu de changements de persistance connus ; vérifier les items S4TWL achats (ex. remplacement de certaines fonctions, MM-PUR Analytics). | 🟡 |
| LIS / SIS / PURCHIS (tables S0xx, S5xx) | Toujours présentes, mais SAP positionne les CDS d'Embedded Analytics en remplacement fonctionnel ; item SIC à lire pour savoir si les structures d'info utilisées sont encore mises à jour. | 🟡 |
| PP, QM, PM, HR/HCM | Non analysés — à instruire si des vues HANA en dépendent. | 🔴 |
| Extensions client (Z-tables, appends CI_*) | Reprises telles quelles par la conversion, sauf appends sur VBUK/VBUP (à redéplacer sur VBAK/VBAP/LIKP/LIPS/VBRK) et CI_COBL (repris dans ACDOCA). | 🟢 |

### 3.7 Vue d'ensemble

```
ECC (source des 44 vues HANA)                 S/4HANA (source de la réplication Fivetran)
──────────────────────────────                ─────────────────────────────────────────
GLT0 / FAGLFLEXT / FAGLFLEXA  ─┐
BSIS BSAS BSID BSAD BSIK BSAK ─┤
COEP COSP COSS / ANEP ANLC …  ─┼──► ACDOCA (+ BSEG, BKPF conservées)
MLIT MLCR … / KNC1 LFC1       ─┘

MKPF + MSEG                    ───► MATDOC
MARD/MCHB/MSKA… (quantités)    ───► MATDOC_EXTRACT (attributs : tables conservées)
MARDH/MCHBH… (historique)      ───► recalcul depuis MATDOC

VBUK / VBUP                    ───► VBAK VBAP LIKP LIPS VBRK (champs statut)
KONV                           ───► PRCD_ELEMENTS
VAKPA…VRPMA, VBOX              ───► (supprimés)

KNA1 / LFA1                    ───► conservées (CVI) + BUT000…
MATNR CHAR18                   ───► CHAR40 partout
```

---

## 4. Conséquences immédiates pour la spécification de réplication

1. **Règle d'exclusion** : aucune table de type A ou B n'entre dans la liste de réplication Fivetran ; la vérification « proxy object » (SE11) sur le système S/4 cible est le critère d'acceptation de chaque table de SPEC 1.
2. **Règle de substitution** : chaque table exclue est remplacée par sa (ses) table(s) cible(s) + une fiche de mapping colonne à colonne (SPEC 2) ; la DDL source de la vue de compatibilité SAP sert de référence de dérivation.
3. **Tables hybrides** : documenter explicitement, pour MARC/MARD/MCHB/MBEW, quelles colonnes sont « persistées » et quelles colonnes doivent être recalculées côté Snowflake.
4. **Historique** : la conversion migre les données ECC dans ACDOCA / MATDOC ; les lignes migrées portent des marqueurs (`MIG_SOURCE`, `BSTAT = 'C'`, `MIG_DELTA`) qu'il faudra traiter dans les règles de calcul (SPEC 3) pour éviter doubles comptes ou ruptures de série.
5. **Typage** : prévoir MATNR VARCHAR(40), VBTYPL VARCHAR(4), montants ACDOCA DECIMAL(23,2), DOCLN VARCHAR(6).
6. **Option de repli** : répliquer une vue de compatibilité **en vue** (pas en table) reste techniquement possible avec Fivetran, mais sans CDC et avec dépendance à une couche que SAP fait évoluer ; à réserver à des cas marginaux et documentés.

---

## 5. Points ouverts

| # | Question | Impact |
|---|----------|--------|
| Q1 | Release S/4HANA cible (2023 ? 2025 ?) et périmètre fonctionnel activé (Margin Analysis, ML, Settlement Management, FSCM-CR) ? | Filtrage du SIC, tables cibles CO-PA / crédit / rabais. |
| Q2 | Le rapport SAP Readiness Check du système ECC est-il disponible ? | Réduit le catalogue générique aux items réellement pertinents. |
| Q3 | Un bac à sable S/4HANA converti (copie de PRD) sera-t-il accessible en lecture (SE11/SE16N/HANA Studio) et à quelle date ? | Étape P4 de vérification terrain. |
| Q4 | Stratégie CVI : conservation des numéros clients/fournisseurs ou renumérotation ? | Clés de jointure des vues clients/fournisseurs. |
| Q5 | Extension MATNR 40 activée ou non ? | Typage et éventuelles règles de formatage. |
| Q6 | Mode de connexion Fivetran retenu (NetWeaver/RFC vs accès base HANA direct) et licence SAP associée ? | Objets visibles, CDC disponible, performances. |
| Q7 | Les 44 vues HANA lisent-elles des tables CO-PA (CE1/CE4), PCA classique (GLPCA/GLPCT) ou LIS (S0xx) ? | Décisions fonctionnelles à trancher avant mapping. |
| Q8 | Périmètre historique attendu dans Snowflake (données migrées ECC via conversion, ou chargement initial séparé depuis ECC) ? | Traitement des marqueurs de migration, stocks historiques. |

## Voir aussi

- [[versions-commandes-achat-s4hana-2023]]
- [[power-bi-datasphere-bdc]]
