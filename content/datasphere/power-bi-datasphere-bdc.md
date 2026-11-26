---
title: "Power BI sur SAP Datasphere et SAP Business Data Cloud : solutions de connexion"
description: "Connecter Power BI à SAP Datasphere ou BDC : SQL (connecteur SAP HANA, ODBC, DirectQuery), API OData, Mirroring Fabric, BDC Connect (Databricks, Fabric), SAP Databricks ; sémantique, coûts, limites, état au 20/11/2026."
tags: ["datasphere", "bdc", "power-bi", "fabric", "odata", "databricks", "sac"]
date: "2026-11-20"
draft: false
---

> [!abstract] En bref
> Aucune connexion live Power BI ↔ Datasphere n'équivaut à la connexion live SAC : chaque voie est un compromis entre sémantique SAP, absence de copie et coût.
> État de l'art au 20/11/2026 (statuts produit à revérifier) : voies SQL et OData, réplication vers Fabric, options BDC.

> **Document de cadrage — version 1.0**
> Date de vérification des informations : **20 novembre 2026**
> Contexte : cadrage d'une plateforme data destinée à remplacer un socle SAP BW + SAP BusinessObjects par une restitution plus moderne et interactive sous Power BI, les données restant stockées dans SAP Datasphere / SAP Business Data Cloud (BDC).
> Critères prioritaires : (1) conservation de la sémantique SAP, (2) minimisation des coûts, (3) minimisation de la charge de maintenance et d'exploitation. Contrainte souhaitée : **pas de duplication** des données de Datasphere/BDC vers un autre data lake / lakehouse.

---

## Conventions de lecture

| Marqueur | Signification |
|---|---|
| **[Vérifié]** | Issu d'une source consultée le 20/11/2026 (référence `[Sx]` renvoyant à l'annexe Sources). |
| **[Connaissance]** | Connaissance de l'auteur non revérifiée pendant la rédaction ; à confirmer, peut être datée. |
| **[Déduction]** | Raisonnement à partir d'éléments vérifiés ; la base du raisonnement est indiquée. |
| **[Hypothèse]** | Supposition faute d'élément suffisant ; à tester en POC. |

Les marqueurs ne sont utilisés que lorsque l'information n'est pas évidente. Les statuts produit (GA / preview / annoncé) sont donnés **à la date du 20/11/2026** ; ce domaine évolue très vite (cf. § 8).

**Limite méthodologique** : le portail `help.sap.com` n'a pas pu être lu intégralement par l'outil de collecte (accès robot refusé). Les éléments SAP Help cités proviennent des extraits indexés des pages officielles, du dépôt GitHub officiel `SAP-docs/sap-datasphere`, et des KBA SAP publiques. Les notes SAP soumises à authentification (S-user) n'ont pas été lues en intégralité.

---

## Table des matières

0. [Synthèse exécutive](#0-synthèse-exécutive)
1. [Contexte, périmètre et versions](#1-contexte-périmètre-et-versions)
2. [Rappels d'architecture](#2-rappels-darchitecture)
3. [Partie A — Datasphere seul](#3-partie-a--sap-datasphere-seul)
   - A1. Connexion SQL directe (connecteur SAP HANA / ODBC) — configuration détaillée
   - A2. API OData de consommation — configuration détaillée
   - A3. Réplication sortante vers Fabric (Mirroring / Copy job)
   - A4. SAP Analytics Cloud en complément
4. [Partie B — SAP Business Data Cloud](#4-partie-b--sap-business-data-cloud)
5. [Matrice comparative](#5-matrice-comparative)
6. [Scénarios d'architecture et recommandations pour le cadrage BW/BO](#6-scénarios-darchitecture-et-recommandations)
7. [Points de vigilance transverses](#7-points-de-vigilance-transverses)
8. [Feuille de route et éléments à surveiller](#8-feuille-de-route-et-éléments-à-surveiller)
- [Annexe A — Accès direct S/4HANA → Fabric/Power BI (hors Datasphere)](#annexe-a--accès-direct-s4hana--fabricpower-bi-hors-datasphere)
- [Annexe B — Solutions tierces](#annexe-b--solutions-tierces)
- [Annexe C — Check-lists de configuration](#annexe-c--check-lists-de-configuration)
- [Annexe D — Points non vérifiés et moyens de vérification](#annexe-d--points-non-vérifiés-et-moyens-de-vérification)
- [Annexe E — Sources](#annexe-e--sources)

---

## 0. Synthèse exécutive

**Constat principal.** Il n'existe pas, au 20/11/2026, de connexion « live » Power BI ↔ Datasphere équivalente à la connexion live SAP Analytics Cloud, qui exploite l'intégralité de la sémantique du *modèle analytique* (Analytic Model) via le moteur multidimensionnel SAP (MDS). Toute solution Power BI implique un **compromis entre conservation de la sémantique, absence de copie et coût**.

**Les deux voies natives « sans duplication » (Datasphere seul) :**

| Voie | Ce qui est consommable | Mode Power BI | Sémantique conservée | Verdict |
|---|---|---|---|---|
| **A1. SQL (connecteur SAP HANA / ODBC)** sur un utilisateur de base de données Datasphere | Vues (et perspectives) **exposées pour consommation** — **pas les modèles analytiques** [Vérifié S4, S9] | Import ou **DirectQuery** (connecteur SAP HANA) ; Import seul (connecteur ODBC) | Faible à moyenne : la logique des vues est conservée, mais la couche « modèle analytique » (mesures restreintes/calculées, hiérarchies, textes, variables) est perdue et doit être reconstruite dans le modèle sémantique Power BI | **Voie la plus robuste et la plus performante aujourd'hui**, au prix d'une double couche sémantique |
| **A2. API OData de consommation** | Vues exposées **et modèles analytiques** (seule voie supportée par SAP pour un client tiers sur un modèle analytique) [Vérifié S9, S6] | Via connecteur personnalisé ou requête vide (M) [Vérifié S6, S10] ; Import uniquement [secondaire S54 ; Déduction] | Meilleure sur les agrégations/mesures du modèle analytique, mais pas de hiérarchies ni de variables multi-valeurs [Vérifié S14 ; secondaire S35] | Adaptée aux volumes faibles/moyens ; outillage côté Power BI fragile (pas de connecteur Microsoft certifié) |

**Les voies BDC (zéro copie au sens « partage Delta »)** :

- **BDC Connect for Azure Databricks** — GA depuis le 15/10/2025 [Vérifié S28] : les *data products* SAP sont partagés en zéro copie (Delta Sharing) dans le Unity Catalog d'un Azure Databricks client, d'où Power BI peut les consommer (Import/DirectQuery).
- **SAP Databricks** (composant de BDC) : les data products y sont disponibles automatiquement via Delta Sharing [Vérifié S30] ; Power BI Desktop et le service Power BI se connectent à SAP Databricks [Vérifié S29].
- **BDC Connect for Microsoft Fabric** — annoncé le 18/11/2025, GA initialement prévue au T3 2026 [Vérifié S26] ; **aucune confirmation officielle de GA trouvée au 20/11/2026** ; une réponse sur SAP Community (sept. 2026, non officielle) évoque un décalage à fin T1 2027 [Vérifié S27]. **C'est la cible la plus alignée avec la demande (zéro copie + Direct Lake), mais elle ne peut pas être positionnée comme disponible à date.**

**La voie Fabric disponible mais avec copie** : *Mirroring for SAP Datasphere* (GA depuis mars 2026 [Vérifié S23]) — elle duplique les données dans OneLake et consomme l'option payante *Premium Outbound Integration* de Datasphere [Vérifié S22, S17]. Contraire à la contrainte « pas de duplication », elle reste utile pour des cas ciblés (gros volumes, Direct Lake).

**Recommandation de cadrage (résumé, détail au § 6)** [Déduction, fondée sur les éléments ci-dessus] :
1. **Socle immédiat** : Datasphere comme couche de modélisation et de stockage unique ; exposition de **vues de consommation « à plat »** (faits + dimensions dénormalisés, textes, conversions de devises matérialisées dans les vues) et consommation Power BI via **connecteur SAP HANA** (DirectQuery pour les gros volumes / Import pour les modèles compacts), avec un **modèle sémantique Power BI partagé** (« golden semantic model ») pour remplacer les univers BO.
2. **OData** réservé aux cas où le modèle analytique Datasphere doit faire foi (indicateurs complexes) et où les volumes restent modestes.
3. **Trajectoire BDC** : préparer l'architecture « data products » (BW PCE + *Data Product Generator*, data products SAP) pour basculer vers **BDC Connect for Fabric** dès sa GA confirmée, et éventuellement vers les capacités *lakehouse* annoncées à TechEd 2026 (GA prévue T1 2027 [Vérifié S32]).
4. **Conserver SAP Analytics Cloud** (inclus dans BDC) pour les usages où la sémantique SAP complète est indispensable (reporting financier hiérarchique, planification), si le client accepte deux outils — à arbitrer en cadrage.

---

## 1. Contexte, périmètre et versions

### 1.1 Contexte client

- Existant : SAP BW (version à préciser : BW 7.5 / BW/4HANA) + SAP BusinessObjects (Web Intelligence, univers, éventuellement Analysis for Office / Lumira).
- Cible : restitution **Power BI** (interactivité, self-service), données **stockées et modélisées dans Datasphere / BDC**.
- Critères prioritaires : sémantique SAP, coûts, run.

### 1.2 Périmètre et validité

| Élément | Version / date de référence |
|---|---|
| SAP Datasphere | Cloud, release continue ; vague **2026.22** citée dans les nouveautés d'octobre 2026 [Vérifié S37] |
| SAP Business Data Cloud | État au 20/11/2026 (post-TechEd Berlin, 27–29/10/2026) |
| Power BI / Microsoft Fabric | Service cloud, état au 20/11/2026 |
| Contexte juridique | France / UE — non spécifiquement traité ; points contractuels SAP (note 3255746) en Annexe A |

**Hors périmètre** : SAP BW Bridge en détail, Power BI Report Server, Excel/Analysis for Office, connecteurs SAP BW de Power BI (traités seulement en Annexe A).

---

## 2. Rappels d'architecture

### 2.1 SAP Datasphere — objets et canaux d'exposition

Datasphere expose trois types d'objets aux outils de restitution [Vérifié S5] :

| Objet | SAP Analytics Cloud | Microsoft Excel | Autres clients (Power BI…) |
|---|---|---|---|
| **Modèles analytiques** (exposés automatiquement) | Connexion live | Connexion live (add-in SAP) | **OData** |
| **Perspectives** (exposées automatiquement) | Connexion live | Connexion live (add-in SAP) | — (cf. remarque) |
| **Vues** (exposées si le commutateur *Expose for Consumption* est activé) | OData | — | **OData, ODBC/JDBC** |

*Remarque* : la page SAP Help consacrée à ODBC/JDBC indique que Power BI peut consommer « *perspectives and views that are exposed for consumption* » via un Open SQL schema [Vérifié S4], alors que le tableau de synthèse ne mentionne pas les perspectives pour les clients tiers [Vérifié S5]. **Divergence de la documentation SAP à lever en POC.**

**Point structurant** : l'accès SQL (ODBC/JDBC) aux **modèles analytiques** n'est **pas supporté**. La KBA 3674874 précise que leur visibilité passée en ODBC résultait d'un bug corrigé, que les modèles analytiques redéployés deviennent invisibles pour les utilisateurs Open SQL, et que l'accès supporté pour des clients tiers est l'**API OData** ; une alternative limitée consiste à exposer la vue de faits sous-jacente [Vérifié S9]. Raison invoquée : des résultats corrects dépendent du moteur multidimensionnel (MDS) [Vérifié S9].

### 2.2 SAP Business Data Cloud — composants utiles au sujet

| Composant | Rôle dans le sujet |
|---|---|
| **SAP Datasphere** | Couche de modélisation (vues, modèles analytiques, contrôles d'accès) ; *object store* (HANA Data Lake Files) pour les data products |
| **SAP Analytics Cloud** | Restitution SAP native (live sur modèles analytiques) |
| **SAP BW, private cloud edition (PCE)** + **Data Product Generator** | Publication des InfoProviders BW comme data products dans l'object store de Datasphere [Vérifié S31] |
| **Data products SAP / custom** | Jeux de données gouvernés ; installables dans un espace Datasphere [Vérifié S31] |
| **SAP Databricks** | Moteur Databricks intégré à BDC ; data products disponibles automatiquement via Delta Sharing [Vérifié S30] |
| **BDC Connect** | Partage zéro copie bidirectionnel (Delta Sharing) avec des plateformes partenaires : Databricks (GA), Google BigQuery (GA juillet 2026, d'après une source secondaire), Microsoft Fabric (annoncé), Snowflake (annoncé) [Vérifié S28, S26 ; secondaire S38] |
| **SAP HANA Cloud, SAP MDG** | Présentés par SAP comme composants cœur de BDC depuis mai 2026 (source secondaire) [S39] |

Points à retenir :
- **Installer un data product dans un espace Datasphere n'utilise pas Delta Sharing** ; l'installation génère des objets dans l'espace (par exemple des tables locales et des flux de réplication) [Vérifié S31]. *[Déduction]* Une fois installés, les data products se consomment depuis Power BI **par les mêmes canaux que le reste de Datasphere** (A1/A2).
- Le **Data Product Generator** réplique les données des InfoProviders BW (InfoObject, aDSO, CompositeProvider, Query as InfoProvider…) vers l'object store ; **les hiérarchies ne sont pas encore supportées** [Vérifié S31]. Il est réservé à BW PCE [Vérifié S31 ; secondaire S40].

### 2.3 Power BI / Microsoft Fabric — modes de stockage utiles

| Mode | Principe | Pertinence ici |
|---|---|---|
| **Import** | Données chargées dans le modèle sémantique (VertiPaq), rafraîchissement planifié | Copie dans Power BI (cache de restitution, pas un lakehouse) — limite 1 Go/modèle en Pro [Connaissance] |
| **DirectQuery** | Requêtes envoyées à la source à chaque interaction | Seul mode « sans copie » ; limite d'1 million de lignes par requête intermédiaire hors réglage Premium [Connaissance] |
| **Composite** | Mélange Import + DirectQuery (agrégats importés + détail en DirectQuery) | Bon compromis performance / fraîcheur [Connaissance] |
| **Direct Lake** | Lecture directe de tables Delta dans OneLake (Fabric) | Nécessite que les données soient dans OneLake (miroir, shortcut ou futur BDC Connect for Fabric) [Connaissance] |

*Remarque* : le mode Import crée une copie des données dans le modèle sémantique Power BI. **[Déduction]** Ce n'est pas une duplication vers un data lake / lakehouse au sens de la contrainte exprimée, mais il faut l'expliciter au client (gouvernance, fraîcheur, RLS à recréer).

---

## 3. Partie A — SAP Datasphere seul

### A1. Connexion SQL directe (connecteur SAP HANA ou ODBC) — configuration détaillée

#### A1.1 Principe

Datasphere s'exécute sur une base SAP HANA Cloud. Un **utilisateur de base de données** (database user / Open SQL schema) créé dans un espace permet à un outil tiers de lire, en SQL, les **vues exposées pour consommation** de cet espace [Vérifié S4, S36]. Power BI s'y connecte :
- soit par le **connecteur « Base de données SAP HANA »** (recommandé) : Import et **DirectQuery** [Vérifié S2] ;
- soit par le **connecteur ODBC** générique : **Import uniquement** (la documentation ne mentionne que l'Import) [Vérifié S3].

#### A1.2 Objets consommables

- **Vues** avec *Expose for Consumption* activé [Vérifié S5].
- **Perspectives** : divergence documentaire, à tester (cf. § 2.1).
- **Modèles analytiques : non supportés** (KBA 3674874) [Vérifié S9]. Contournement SAP : exposer la vue de faits (sans mesures restreintes, textes ni agrégations propres au modèle analytique) [Vérifié S9].

#### A1.3 Configuration côté SAP Datasphere

| # | Étape | Détail | Rôle requis |
|---|---|---|---|
| 1 | **Créer un utilisateur de base de données** dans l'espace | *Space Management → (espace) → Database Access → Database Users → Create*. Le nom est de la forme `<ESPACE>#<SUFFIXE>`. Cocher **Enable Read Access (SQL)** pour lire les vues exposées. Déployer l'espace. Noter **hôte, port, utilisateur, mot de passe** (affichés dans les détails de l'utilisateur) | Administrateur d'espace ou intégrateur [Vérifié S4] ; l'étape d'activation de la lecture est décrite dans des guides partenaires [secondaire S41] |
| 2 | **Ajouter les IP clientes à l'IP allowlist** | *System → Configuration → IP Allowlist → Trusted IPs* : IP publique **de chaque poste Power BI Desktop** (ou de la sortie NAT de l'entreprise) **et de la passerelle de données** | Administrateur [Vérifié S4 ; chemin de menu : secondaire S42] |
| 3 | **Exposer les vues** | Activer *Expose for Consumption* sur chaque vue à publier (option possible au niveau de l'espace pour les nouvelles vues — [secondaire S43]) | Modélisateur [Vérifié S5] |
| 4 | **Sécurité des lignes** | Les *Data Access Controls* (DAC) peuvent s'appliquer à une consommation ODBC (contenu de support SAP dédié) [Vérifié S36]. **[Déduction]** L'identité vue par Datasphere est l'utilisateur de base de données technique, **pas l'utilisateur final Power BI** : pas de filtrage par utilisateur final sans un utilisateur de base par population | Modélisateur / sécurité |
| 5 | **Concevoir des vues « de consommation »** | **[Déduction]** Puisque la couche modèle analytique n'est pas accessible, prévoir des vues dédiées : jointures faits/dimensions, **libellés (textes) résolus**, conversions de devises/unités calculées dans la vue, colonnes de filtre (année, société…) adaptées au *query folding* | Modélisateur |

*Paramètres de connexion* : hôte du type `<id>.hana.prod-<région>.hanacloud.ondemand.com`, **port 443**, chiffrement TLS obligatoire [Connaissance ; port et hôte cohérents avec S44].

#### A1.4 Configuration côté postes et passerelle

| # | Étape | Détail |
|---|---|---|
| 1 | **Installer le client SAP HANA 2.0** (pilote ODBC `HDBODBC`, 64 bits) sur **chaque poste Power BI Desktop** et **chaque nœud de passerelle** | Prérequis SAP [Vérifié S4] ; le connecteur Power BI s'appuie sur le pilote ODBC SAP [Vérifié S2]. Téléchargement sur `tools.hana.ondemand.com` (compte SAP Universal ID requis — [secondaire S45]) |
| 2 | **Power BI Desktop** — connecteur SAP HANA | *Obtenir les données → Base de données SAP HANA* ; serveur `<hôte>:443` ; mode **Importer** ou **DirectQuery** ; authentification **Base de données** (utilisateur/mot de passe du database user). L'option *Valider le certificat du serveur* est activée par défaut [Vérifié S2] |
| 3 | **Choix du comportement DirectQuery** | *Fichier → Options → DirectQuery* : « traiter SAP HANA comme source **multidimensionnelle** » (défaut) ou « **relationnelle** ». Le choix est global, s'applique aux **nouveaux** rapports et n'est pas modifiable sur un rapport existant [Vérifié S1] |
| 4 | **Power BI Desktop** — alternative ODBC | Créer un DSN `HDBODBC` (hôte, port 443, TLS) puis *Obtenir les données → ODBC* — Import uniquement [Vérifié S3] |
| 5 | **Service Power BI / Fabric** | **Passerelle de données locale obligatoire** pour le connecteur SAP HANA, « que les données soient locales ou en ligne » [Vérifié S2]. Déclarer la source dans la passerelle (authentification Basic), y installer le client HANA, et ajouter l'IP de sortie de la passerelle à l'allowlist Datasphere [Déduction, à partir de S2 et S4] |
| 6 | **Passerelle réseau virtuel (VNet) gérée** | **[Hypothèse]** Option à étudier pour supprimer une VM de passerelle à exploiter ; compatibilité du connecteur SAP HANA et IP de sortie fixe à vérifier |

#### A1.5 Comportement DirectQuery : multidimensionnel vs relationnel

| Aspect | Multidimensionnel (défaut) | Relationnel |
|---|---|---|
| Granularité | Une seule vue analytique/de calcul par rapport ; pas de relations ni de combinaison de vues [Vérifié S1] | Plusieurs vues, relations, colonnes calculées [Vérifié S1] |
| Agrégation | Calculée par SAP HANA ; mesures non additives correctes [Vérifié S1] | Mesures non additives (distinct count, moyennes) potentiellement fausses ou lentes [Vérifié S1] |
| Hiérarchies HANA | Visibles (sauf parent-enfant) [Vérifié S1] | **Indisponibles** [Vérifié S1] |
| DAX | Restreint ; pas de vue Table ; colonnes figées [Vérifié S1] | Complet |
| TLS | Supporté | **Les connexions relationnelles ne peuvent pas se connecter à des instances configurées en TLS** [Vérifié S1] |

> ⚠️ **Point d'attention majeur [Déduction]** : Datasphere (HANA Cloud) impose TLS. Si la limitation « relationnel + TLS » documentée par Microsoft (page mise à jour le 10/12/2024) s'applique à Datasphere, **seul le mode multidimensionnel serait utilisable en DirectQuery**, avec ses restrictions (une vue par rapport). Par ailleurs, les vues Datasphere exposées sont des vues SQL et non des vues de calcul HANA classiques ; le comportement réel du mode multidimensionnel sur ces vues est **à tester en POC** [Hypothèse]. Des retours de terrain mentionnent un usage du DirectQuery sur des vues de calcul déployées dans HANA Cloud sous Datasphere [secondaire S46].

**Variables / paramètres** : le connecteur supporte les variables et paramètres d'entrée HANA ; ils sont demandés à la connexion, modifiables dans *Transformer les données → Modifier les paramètres*, mais **non modifiables dans le service Power BI après publication** [Vérifié S1, S2]. Un fil communautaire signale des difficultés à voir des vues Datasphere à invites dans le navigateur Power BI [secondaire S47] → **à tester en POC**.

#### A1.6 Sémantique conservée / perdue

| Élément SAP | Conservé ? | Commentaire |
|---|---|---|
| Logique de jointure / filtres / calculs des vues | ✅ | Exécutée dans HANA Cloud |
| Mesures du modèle analytique (restreintes, calculées, agrégation d'exception) | ❌ | Modèles analytiques non accessibles en SQL [Vérifié S9] → **à recréer en DAX** |
| Hiérarchies (niveaux) | ⚠️ | Mode multidimensionnel uniquement ; **parent-enfant non visibles** [Vérifié S1] |
| Textes / associations | ⚠️ | À résoudre dans la vue (jointure explicite) [Déduction] |
| Conversion de devises / unités | ⚠️ | Seulement si matérialisée dans la vue [Déduction] |
| Sécurité DAC | ⚠️ | Applicable à l'utilisateur de base, pas à l'utilisateur final [Vérifié S36 ; Déduction] |
| Variables / invites | ⚠️ | Fixées à la publication [Vérifié S1] |

#### A1.7 Coûts et exploitation

- **Licences SAP** : **[Déduction]** aucun poste de licence spécifique documenté pour l'accès ODBC ; la charge de requêtes est absorbée par la capacité de calcul de Datasphere (unités de capacité). Un usage DirectQuery intensif peut nécessiter un dimensionnement mémoire/calcul supérieur → **à chiffrer avec l'estimateur de capacité SAP**.
- **Microsoft** : licences Power BI (Pro / PPU / capacité) + **VM(s) de passerelle** (haute disponibilité conseillée en cluster) [Connaissance].
- **Run** : maintenance du client HANA sur les postes et passerelles, rotation des mots de passe des utilisateurs de base, gestion de l'allowlist IP, double maintenance de la sémantique (vues Datasphere + modèle Power BI).

#### A1.8 Avantages / inconvénients

| ✅ Avantages | ❌ Inconvénients / limites |
|---|---|
| Voie standard documentée par SAP et Microsoft [Vérifié S4, S2] | Pas d'accès aux modèles analytiques [Vérifié S9] |
| **DirectQuery** possible → aucune copie hors Datasphere | Utilisateur technique : pas de SSO ni de propagation de l'identité de l'utilisateur final [Déduction] |
| Performance HANA ; *query folding* | Passerelle obligatoire pour le service [Vérifié S2] ; client HANA à déployer |
| Fonctionne aussi pour Excel/Power Query, Fabric Dataflows | Limitation TLS en mode relationnel à clarifier [Vérifié S1] |
| Pas de coût de sortie de données SAP spécifique documenté [Déduction] | Sémantique à reconstruire dans Power BI (double couche) |

---

### A2. API OData de consommation — configuration détaillée

#### A2.1 Principe et points de terminaison

L'API OData (v4) de Datasphere comprend un **service catalogue** (liste des espaces et objets exposés) et un **service de consommation** (données des vues et modèles analytiques) ; elle accepte `$select`, `$filter`, `$orderby`, `$top`, `$skip` [secondaire S48]. Formats d'URL documentés [Vérifié S7] :

```text
# Vues (relationnel)
GET https://<tenant_url>/api/v1/datasphere/consumption/relational/<space_id>/<asset_id>

# Modèles analytiques (analytique) — seuls les appels analytiques sont supportés
GET https://<tenant_url>/api/v1/datasphere/consumption/analytical/<space_id>/<asset_id>/<asset_id>[(<asset_params>)/Set][?<query_params>]

# Exemple
GET https://<tenant_url>/api/v1/datasphere/consumption/analytical/<space_id>/<asset_id>/<asset_id>?$select=Sales,Country
```

Les modèles analytiques ne supportent que les requêtes `consumption/analytical` [Vérifié S7]. Des formes d'URL historiques (`/api/v1/dwc/...`, `/dwaas-core/odata/v4/...`) existent encore dans la littérature [secondaire S49].

**Seule voie supportée par SAP pour accéder à un modèle analytique depuis un client tiers** (hors SAC et add-in Excel SAP) [Vérifié S9].

#### A2.2 Prérequis SAP Datasphere

| # | Élément | Détail |
|---|---|---|
| 1 | **Rôle à portée (scoped role)** de l'utilisateur consommateur | Privilèges *Data Warehouse Consumption* (`-R------`) et *Space Files* (`-R------`) ; le modèle de rôle **DW Consumer** les fournit [Vérifié S6] |
| 2 | **Client OAuth 2.0** créé par un administrateur | *System → Administration → App Integration → OAuth Clients* ; prérequis admin : *Data Warehouse General*, *System Information*, *User (CRUD)* — inclus dans **DW Administrator** [Vérifié S8] |
| 3 | **Finalité (Purpose) du client OAuth** | **Interactive Usage** ou **Technical User** — les deux permettent de consommer l'API OData [Vérifié S6, S8] |
| 4 | Paramètres à récupérer | Client ID, Secret, Authorization URL (**non requis** pour un client *Technical User*), Token URL [Vérifié S6] |
| 5 | **Exposition** | Vues : *Expose for Consumption* ; modèles analytiques : exposés automatiquement [Vérifié S5] |

**Choix de la finalité du client OAuth — point clé pour le run :**

| | Interactive Usage | Technical User |
|---|---|---|
| Flux | OAuth 2.0 *authorization code* (3 jambes, l'utilisateur s'authentifie auprès de l'IdP) ou *SAML Bearer* [Vérifié S6, S12] | Accès direct sans authentification interactive [Vérifié S6] |
| Identité vue par Datasphere | Utilisateur final → **[Déduction]** DAC appliqués à l'utilisateur | Utilisateur technique et ses rôles propres [Vérifié S6, S50] |
| Durée du jeton de rafraîchissement | **180 jours max** (KBA 3392890 citée par la KBA 3718383) [Vérifié S11] | **Jusqu'à 30 ans** [Vérifié S11] |
| Disponibilité | Historique | Pour SAC : supporté **depuis T3–T4 2025** [Vérifié S11] ; documenté pour Power BI et autres clients [Vérifié S6] |
| Adapté à un rafraîchissement planifié Power BI | Peu (ré-authentification périodique) | **Oui** [Déduction] |

> ⚠️ Des KBA plus anciennes (3383701, 3576221) indiquent que seuls les flux 3 jambes sont supportés et que `client_credentials` ne l'est pas [Vérifié S12]. La documentation actuelle (finalité *Technical User*) les rend **partiellement obsolètes** [Déduction, fondée sur S6 et S11] — à confirmer sur le tenant du client (fonctionnalité dépendante de la version).

#### A2.3 Configuration côté Power BI — trois options, une seule à proscrire

| Option | Statut | Détail |
|---|---|---|
| ❌ Connecteurs standards **OData** ou **Web** | **Non supporté** | Erreur typique « *more than 20 redirects* » ; SAP qualifie cette approche de non supportée [Vérifié S10] |
| ✅ **Connecteur personnalisé SAP (fichier `.mez`)** | Exemple SAP sur GitHub (`SAP-samples/data-warehouse-cloud-connectors`), **sans SLA, « à vos risques »** [secondaire S51] ; **développement actif arrêté**, correctifs éventuels seulement [Vérifié S19] | Copier le `.mez` dans `Documents\Power BI Desktop\Custom Connectors`, autoriser les extensions non certifiées dans les options de sécurité de Power BI Desktop [Connaissance] ; supporte vues et modèles analytiques [Vérifié S48]. Service : rafraîchissement via **passerelle** configurée avec le dossier des connecteurs personnalisés [Connaissance ; cohérent avec S52] |
| ✅ **Requête vide (Blank Query, langage M)** | Approche citée par SAP [Vérifié S10] | Le script M obtient un jeton via `Web.Contents` sur le Token URL, puis appelle `OData.Feed(url, [Headers=[Authorization="Bearer …"]])` [Vérifié S20]. Variante historique : jeton de rafraîchissement (validité 30 jours glissants dans l'article de 2022), secrets stockés dans le code M [Vérifié S20]. **[Déduction]** Avec un client *Technical User*, le jeton peut être obtenu sans étape interactive, ce qui rend le rafraîchissement planifié possible ; stocker secret et ID dans des paramètres protégés et vérifier la compatibilité du rafraîchissement dans le service (sources dynamiques `Web.Contents`) [Connaissance] |

**Exemple de structure M (à adapter, non testé)** [Déduction, à partir de S20 et S6] :

```powerquery
let
    TokenUrl     = "https://<tenant>.authentication.<region>.hana.ondemand.com/oauth/token",
    ClientId     = pClientId,          // paramètre Power BI
    ClientSecret = pClientSecret,      // paramètre Power BI
    Auth         = "Basic " & Binary.ToText(Text.ToBinary(ClientId & ":" & ClientSecret), BinaryEncoding.Base64),
    TokenResp    = Json.Document(
                     Web.Contents(TokenUrl,
                       [ Headers = [#"Authorization" = Auth,
                                    #"Content-Type"  = "application/x-www-form-urlencoded"],
                         Content = Text.ToBinary("grant_type=client_credentials") ])),
    AccessToken  = TokenResp[access_token],
    Source       = OData.Feed(
                     "https://<tenant_url>/api/v1/datasphere/consumption/relational/<space_id>/<view_id>/<view_id>",
                     [ #"Authorization" = "Bearer " & AccessToken ],
                     [ Implementation = "2.0" ])
in
    Source
```

> Le format exact du Token URL et le type de *grant* à utiliser avec un client *Technical User* sont à relever sur la page *App Integration* du tenant ; une question SAP Community décrit un essai réussi avec `client_credentials` et l'URL de jeton *OAuth2SAML* [secondaire S53].

#### A2.4 Limites techniques de l'API

| Limite | Source |
|---|---|
| **Pagination serveur imposée** : `@odata.nextLink` / `$skiptoken` ; impossible de forcer une page unique | [Vérifié S13a] |
| Taille de page : **50 000 enregistrements max** et **50 Mo max** par page (décrit dans le contexte SAC) | [Vérifié S13] |
| Modèle analytique avec **variable non mono-valeur** (plage, multiple, intervalle) : **non supporté** | [Vérifié S14] |
| `$apply` **non supporté** sur l'API des modèles analytiques, et non planifié | [Vérifié S15] |
| Power BI affiche parfois les **noms techniques** au lieu des descriptions de champs | [Vérifié S16] |
| Hiérarchies **non supportées** ; associations du modèle analytique **supportées** ; agrégation du modèle analytique **supportée** | [secondaire S35 — à confirmer en POC] |
| Mode **Import uniquement** (pas de DirectQuery) | [secondaire S54, S20 ; Déduction : `OData.Feed` ne supporte pas DirectQuery] |

#### A2.5 Sémantique, coûts, run

- **Sémantique** : meilleure que SQL sur les **mesures et agrégations** des modèles analytiques (calculées côté Datasphere) ; perte des hiérarchies et des variables multi-valeurs [Vérifié S14 ; secondaire S35].
- **Coûts** : **[Déduction]** pas de licence additionnelle documentée ; consommation de calcul Datasphere ; passerelle si connecteur personnalisé.
- **Run** : dépendance à un connecteur non maintenu activement [Vérifié S19] ou à du code M maison ; gestion des secrets OAuth ; temps de rafraîchissement élevés sur gros volumes (pagination).

#### A2.6 Avantages / inconvénients

| ✅ Avantages | ❌ Inconvénients / limites |
|---|---|
| **Seul accès supporté aux modèles analytiques** pour un client tiers [Vérifié S9] | Pas de connecteur Microsoft certifié ; connecteurs Power BI standards non supportés [Vérifié S10] |
| Mesures/agrégations calculées par Datasphere | Import uniquement → copie dans le modèle Power BI |
| Pas d'allowlist IP de base de données ni de client HANA à déployer [Déduction] | Pagination 50 000 lignes / 50 Mo → lent sur gros volumes [Vérifié S13] |
| Client *Technical User* : jeton longue durée [Vérifié S11] | Hiérarchies et variables multi-valeurs non supportées [Vérifié S14 ; secondaire S35] |
| Avec *Interactive Usage* : identité utilisateur possible [Déduction] | Connecteur `.mez` d'exemple sans SLA ni développement actif [Vérifié S19] |

---

### A3. Réplication sortante vers Microsoft Fabric (avec copie — pour mémoire)

Présentée pour comparaison car **elle duplique les données** (contraire à la contrainte exprimée).

| Élément | Détail |
|---|---|
| **Mirroring for SAP Datasphere** | GA depuis **mars 2026** [Vérifié S23]. Deux étapes : un *replication flow* Datasphere dépose l'instantané initial et les deltas dans **ADLS Gen2**, puis le moteur de mirroring Fabric fusionne dans une base miroir OneLake (Delta), avec un point de terminaison SQL en lecture seule [Vérifié S22] |
| **Copy job (SAP Datasphere Outbound)** | CDC vers différentes destinations ; nécessite *Premium Outbound Integration* [Vérifié S24]. Un **connecteur SAP Datasphere** (Copy job) est aussi annoncé GA dans Fabric Data Factory [secondaire S55] |
| Coûts Fabric | Calcul de réplication gratuit ; stockage miroir gratuit jusqu'à un seuil lié à la capacité ; requêtes facturées normalement [Vérifié S22] |
| Coûts SAP | **Premium Outbound Integration** obligatoire pour toute cible non SAP (blocs à allouer dans la configuration du tenant) [Vérifié S17, S22] ; unité de facturation par **20 Go** [Vérifié S34] |
| Contraintes | Paramètres du flux : *Group Delta = None*, *File Type = Parquet* ; types de chargement *Initial and Delta* ou *Initial Only* [Vérifié S22b] |
| Datasphere comme source | Une **table locale** Datasphere en source n'autorise que *Initial Only* (pas de delta) [Vérifié S18] ; **[Connaissance, à vérifier]** les vues ne sont pas des sources de replication flow → la sémantique des vues et modèles analytiques **n'est pas répliquée** (à persister au préalable, par exemple via un *transformation flow*) |
| Conformité | Présenté par Microsoft comme non concerné par la note SAP 3255746 [Vérifié S25] |

**Intérêt** : Direct Lake dans Power BI, découplage des charges de restitution. **Inconvénients** : copie, coût de sortie SAP, perte de sémantique (tables), double gouvernance.

### A4. SAP Analytics Cloud en complément (rappel)

SAC accède en **connexion live** aux modèles analytiques, perspectives et datasets Datasphere [Vérifié S5, S56]. **[Déduction]** C'est la seule restitution qui exploite toute la sémantique Datasphere (MDS) sans reconstruction. SAC étant inclus dans BDC [Vérifié S38], un **modèle hybride** (Power BI pour le self-service et les tableaux de bord, SAC pour le reporting financier hiérarchique et la planification) est une option de cadrage légitime, à arbitrer selon la volonté du client de rationaliser ses outils.

---

## 4. Partie B — SAP Business Data Cloud

### B1. Ce que BDC change pour Power BI

1. **Data products** : jeux de données gouvernés (SAP-managed ou custom) stockés dans l'object store, partageables en Delta Sharing [Vérifié S31, S28].
2. **Zéro copie inter-plateformes** via **BDC Connect** (Delta Sharing) — c'est le seul mécanisme natif qui réponde à « pas de duplication » **et** permette une consommation hors SAP à grande échelle [Vérifié S28, S26].
3. **Modernisation de BW** : BW PCE + Data Product Generator [Vérifié S31].

### B2. Data products consommés via Datasphere → Power BI

- Installation du data product dans un espace Datasphere → modélisation (vues, modèles analytiques) → consommation Power BI via **A1** ou **A2**.
- **[Vérifié S31]** L'installation peut générer des tables locales et des flux de réplication : vérifier, data product par data product, si l'installation crée une copie physique dans l'espace (impact stockage Datasphere, pas une sortie vers un lakehouse tiers).
- **Avantages / limites** : identiques à A1/A2, avec en plus le bénéfice des modèles SAP livrés (*Intelligent Applications* : data products + modèles analytiques Datasphere + stories SAC [Vérifié S31]). Les modèles analytiques livrés restent soumis à la contrainte « pas d'accès SQL » [Vérifié S9].

### B3. SAP Databricks (dans BDC) comme couche de service pour Power BI

| Élément | Détail |
|---|---|
| Principe | Les data products de BDC sont **automatiquement disponibles** dans SAP Databricks via Delta Sharing, sans copie physique [Vérifié S30] |
| Connexion Power BI | Power BI Desktop (Partner Connect ou configuration manuelle) vers clusters et **SQL warehouses** SAP Databricks ; publication vers le service Power BI ; authentification par **principal de service (OAuth M2M)** ; pilote ADBC ou ODBC [Vérifié S29 — page mise à jour le 11/11/2026] |
| Modes | **[Connaissance]** Import et DirectQuery sont supportés par le connecteur Azure Databricks de Power BI ; la page SAP Databricks ne détaille pas les modes [Vérifié S29] → à confirmer |
| Sémantique | **[Déduction]** Tables Delta (data products) avec métadonnées ; les modèles analytiques Datasphere ne sont pas transportés → mesures à reconstruire (DAX ou couche SQL Databricks) |
| Coûts | Calcul SAP Databricks (capacité BDC) + Power BI ; SAP Databricks fait partie des *intelligent packages* tarifés en unités de capacité BDC [Vérifié S34] |
| Intérêt | DirectQuery sur moteur SQL élastique ; pas de passerelle requise pour une source cloud publique [Connaissance] |
| Limites | Périmètre fonctionnel de SAP Databricks par rapport à un Databricks « entreprise » à vérifier [Hypothèse] ; double modélisation sémantique |

### B4. BDC Connect for (Azure) Databricks → Power BI et/ou Fabric

| Élément | Détail |
|---|---|
| Statut | **GA depuis le 15/10/2025** [Vérifié S28] |
| Principe | Connexion de confiance BDC ↔ Azure Databricks approuvée par les deux administrateurs ; les data products sont montés dans **Unity Catalog** via Delta Sharing ; les données restent en place, non répliquées [Vérifié S28] |
| Prérequis | Workspace Unity Catalog, Delta Sharing configuré, privilèges `CREATE PROVIDER` / `CREATE RECIPIENT` [Vérifié S28] |
| Power BI | Connecteur Azure Databricks (Import / DirectQuery) [Connaissance] ; ou exposition dans Fabric via la mise en miroir du catalogue Azure Databricks (Unity Catalog) dans OneLake [Connaissance — à vérifier] puis Direct Lake |
| Points d'attention | Azure Databricks peut communiquer à SAP certaines données d'usage liées aux données partagées [Vérifié S28] ; licence Azure Databricks en plus de BDC ; termes d'usage SAP des données en aval à vérifier [Vérifié S25] |

**[Déduction]** Pour un client **déjà équipé d'Azure Databricks**, c'est aujourd'hui la voie zéro copie la plus mature pour amener des data products BDC vers Power BI.

### B5. BDC Connect for Microsoft Fabric — la cible annoncée

| Élément | Détail |
|---|---|
| Annonce | 18/11/2025 (Microsoft Ignite) : partage **bidirectionnel zéro copie** ; data products SAP intégrés dans OneLake, jeux de données OneLake visibles dans BDC [Vérifié S26] |
| GA annoncée | **T3 2026** [Vérifié S26] ; Microsoft évoquait « *second half of 2026* » en mai 2026 [Vérifié S23b] |
| **Statut au 20/11/2026** | **Aucune annonce officielle de GA trouvée.** Une réponse SAP Community du 03/09/2026 indique une GA « *planned for end of Q1 2027* », renvoyant au Road Map Explorer BDC [Vérifié S27 — **source non officielle**] ; une analyse tierce au 27/09/2026 ne trouvait pas de confirmation de livraison [secondaire S57] ; la page Microsoft Learn sur SAP CDC (mise à jour le 27/05/2026) l'indiquait encore « *available later this year* » [Vérifié S25] |
| Intérêt attendu | **[Déduction]** Data products lisibles dans OneLake sans copie → **Direct Lake** dans Power BI, sans passerelle, avec gouvernance OneLake |
| Inconnues | Tarification (capacité BDC + capacité Fabric F-SKU, sans offre groupée d'après une source tiers [secondaire S57]), conservation des métadonnées sémantiques, sécurité (propagation des DAC ?), objets partageables (data products uniquement ? vues Datasphere ?) |

> **Recommandation** : **ne pas fonder le planning du projet sur cette brique** tant que la GA n'est pas confirmée sur le **SAP Road Map Explorer** et par Microsoft ; concevoir l'architecture pour pouvoir y basculer (data products, modèle sémantique Power BI découplé de la connexion).

### B6. BW PCE + Data Product Generator (trajectoire de remplacement BW)

| Élément | Détail |
|---|---|
| Principe | Abonnements BW → génération de tables Delta dans l'object store de Datasphere à partir d'InfoProviders/InfoObjects ; définition de data products dans Datasphere [Vérifié S31] |
| Sources supportées | InfoObject (**hiérarchies non supportées à date**), DSO, aDSO, InfoCube, CompositeProvider, MultiProvider, Query as InfoProvider [Vérifié S31] |
| Prérequis | **BW en Private Cloud Edition** dans BDC [Vérifié S31 ; secondaire S40] |
| Restitution Power BI | Via Datasphere (A1/A2) ou via Delta Sharing (B3/B4/B5) [Déduction] |
| Intérêt | **[Déduction]** Réutiliser les modèles BW existants pendant la transition, éviter un *big bang*, alimenter Power BI depuis un socle unique |

### B7. Combinaisons BDC « pour en tirer le maximum »

| Combinaison | Description | Copie hors BDC ? | Statut |
|---|---|---|---|
| **BDC-1** | Data products → Datasphere (modèles) → Power BI via SQL/DirectQuery | Non | Disponible |
| **BDC-2** | Data products → Datasphere modèles analytiques → Power BI via OData (indicateurs certifiés) | Non (Import Power BI) | Disponible |
| **BDC-3** | Data products → SAP Databricks (SQL warehouse) → Power BI DirectQuery | Non | Disponible |
| **BDC-4** | Data products → BDC Connect → Azure Databricks → Power BI (ou Fabric) | Non (Delta Sharing) | Disponible (GA 10/2025) |
| **BDC-5** | Data products → BDC Connect for Fabric → OneLake → Power BI Direct Lake | Non (annoncé zéro copie) | **Non GA à date** |
| **BDC-6** | SAC live sur modèles analytiques + Power BI sur vues/data products | Non | Disponible |

---

## 5. Matrice comparative

Échelle : ●●● favorable — ●●○ moyen — ●○○ défavorable. Évaluation **[Déduction]** fondée sur les sections précédentes ; à pondérer avec le client.

| Critère (poids proposé) | A1 SQL Import | A1 SQL DirectQuery | A2 OData | A3 Mirroring Fabric | B3 SAP Databricks | B4 BDC Connect Az. Databricks | B5 BDC Connect Fabric | A4/BDC-6 SAC live |
|---|---|---|---|---|---|---|---|---|
| **Sémantique SAP (30 %)** | ●○○ | ●○○ / ●●○ (multidim.) | ●●○ | ●○○ | ●○○ | ●○○ | ●●○ ? | ●●● |
| **Coût (25 %)** | ●●● | ●●○ (charge HANA) | ●●● | ●○○ (premium outbound + Fabric) | ●●○ | ●○○ (licence Databricks) | ? | ●●○ (si SAC déjà inclus) |
| **Run / maintenance (25 %)** | ●●○ (passerelle, client HANA) | ●●○ | ●○○ (connecteur non maintenu / code M) | ●●○ | ●●○ | ●●○ | ●●● ? | ●●● |
| **Absence de duplication (contrainte)** | ●●○ (cache Power BI) | ●●● | ●●○ (cache Power BI) | ●○○ | ●●● | ●●● | ●●● | ●●● |
| **Volumétrie / performance** | ●●○ | ●●○ | ●○○ | ●●● | ●●● | ●●● | ●●● ? | ●●○ |
| **Sécurité / identité utilisateur final** | ●○○ (utilisateur technique) | ●○○ | ●●○ (interactif) / ●○○ (technique) | ●○○ | ●●○ | ●●○ | ? | ●●● |
| **Maturité / statut** | ●●● | ●●○ (TLS relationnel à clarifier) | ●●○ | ●●● (GA 03/2026) | ●●○ | ●●● (GA 10/2025) | ●○○ (non GA) | ●●● |

`?` = information non disponible à date.

---

## 6. Scénarios d'architecture et recommandations

### 6.1 Scénarios proposés pour le cadrage

**Scénario 1 — « Datasphere pivot, Power BI frontal » (disponible, sans copie hors Datasphere)**
- Datasphere : couches d'acquisition/harmonisation (remplacement des flux BW), **couche de consommation dédiée Power BI** (vues exposées dénormalisées, textes, devises, colonnes de partition).
- Power BI : **modèle sémantique partagé** par domaine (remplaçant les univers BO), en **composite** (agrégats en Import + détail en DirectQuery SAP HANA) ; RLS Power BI.
- OData pour quelques modèles analytiques « certifiés » à faible volume.
- *Pour* : disponible, coût maîtrisé. *Contre* : double sémantique (Datasphere + DAX), sécurité à redéfinir.

**Scénario 2 — « BDC data products + moteur SQL Databricks » (disponible)**
- Data products (SAP-managed + BW via Data Product Generator) → SAP Databricks ou Azure Databricks via BDC Connect → Power BI DirectQuery/Import.
- *Pour* : zéro copie, performance, ouverture IA/ML. *Contre* : licences Databricks, compétences, sémantique à reconstruire hors Datasphere.

**Scénario 3 — « BDC + Fabric » (cible, conditionnée à la GA de BDC Connect for Fabric)**
- Data products partagés dans OneLake → Power BI Direct Lake.
- Phase transitoire possible via Scénario 1 (sans copie) ; **le Mirroring (A3) n'est à retenir que si le client lève la contrainte de non-duplication**.

**Scénario 4 — « Hybride SAC + Power BI »**
- SAC live pour les restitutions à forte sémantique SAP (finance, hiérarchies, planification) ; Power BI pour le self-service et les tableaux de bord transverses.
- *Pour* : sémantique intégrale sans reconstruction. *Contre* : deux outils, deux compétences, risque de recouvrement.

### 6.2 Arbre de décision simplifié

```text
Le client exige-t-il la sémantique complète des modèles analytiques (hiérarchies, variables, mesures restreintes) ?
├─ Oui → Scénario 4 (SAC pour ces usages) et/ou OData (A2) si volumes faibles et pas de hiérarchie
└─ Non / partiellement
   ├─ Le client dispose-t-il (ou veut-il) d'un moteur Databricks ?
   │   ├─ Oui → Scénario 2 (BDC Connect Azure Databricks ou SAP Databricks)
   │   └─ Non
   │       ├─ BDC Connect for Fabric est-il GA à la date de décision ?
   │       │   ├─ Oui → Scénario 3 (après POC)
   │       │   └─ Non → Scénario 1 maintenant, architecture prête pour le Scénario 3
   └─ Contrainte de non-duplication levée pour certains domaines volumineux ? → A3 Mirroring ciblé
```

### 6.3 POC recommandés (avant engagement)

| # | Test | Objectif |
|---|---|---|
| 1 | DirectQuery SAP HANA sur des vues Datasphere en mode **relationnel** et **multidimensionnel** | Lever le doute TLS / mode relationnel (§ A1.5) |
| 2 | Vues Datasphere **avec paramètres d'entrée** dans Power BI | Comportement des invites |
| 3 | OData avec client **Technical User** + requête M + rafraîchissement planifié dans le service | Viabilité opérationnelle (A2) |
| 4 | OData sur un modèle analytique avec associations et hiérarchie | Confirmer ce qui est restitué |
| 5 | Volumétrie : rafraîchissement OData vs SQL sur un fait représentatif (10–50 M lignes) | Performance |
| 6 | DAC Datasphere + utilisateur de base de données ; RLS Power BI | Modèle de sécurité cible |
| 7 | (Si BDC) Data product → SAP Databricks → Power BI DirectQuery | Valider le Scénario 2 |

### 6.4 Questions à poser au client (cadrage)

1. Contrat SAP : Datasphere seul ou BDC ? BW actuel éligible à BW PCE ? SAC déjà inclus ?
2. Socle Microsoft : licences Power BI (Pro / PPU / capacité Fabric F-SKU) ? Azure Databricks existant ?
3. Volumétrie et fraîcheur attendues par domaine (temps réel, J-1…) ?
4. Exigences de sécurité : filtrage par utilisateur final obligatoire ? SSO ?
5. Patrimoine BO : nombre de rapports WebI, univers, usages de hiérarchies et de variables ?
6. Acceptabilité d'une copie dans OneLake pour certains domaines (exception à la règle) ?

---

## 7. Points de vigilance transverses

1. **Double couche sémantique** : quelle que soit la voie (hors SAC), une partie de la sémantique Datasphere doit être reconstruite dans Power BI (DAX) [Déduction]. Prévoir une **gouvernance des indicateurs** (référentiel, tests de non-régression Datasphere vs Power BI).
2. **Sécurité** : les accès SQL utilisent un **utilisateur technique** ; la RLS Power BI devient la référence pour le filtrage par utilisateur final [Déduction]. En OData *Interactive Usage*, l'identité utilisateur est transmise, mais le rafraîchissement planifié est contraint (jeton 180 jours) [Vérifié S11].
3. **Réseau** : allowlist IP Datasphere (postes + passerelles) [Vérifié S4] ; passerelle obligatoire pour le connecteur SAP HANA [Vérifié S2].
4. **Outillage non maintenu** : connecteur OData d'exemple SAP sans SLA et sans développement actif [Vérifié S19 ; secondaire S51].
5. **Coûts SAP de sortie** : toute réplication vers une cible non SAP déclenche *Premium Outbound Integration* [Vérifié S17].
6. **Feuille de route** : BDC Connect for Fabric non GA à date [Vérifié S27 — non officiel ; S25].
7. **Conformité ODP** : ne pas extraire directement d'un système ABAP SAP avec un outil tiers utilisant ODP-RFC (cf. Annexe A).
8. **Modèles analytiques et SQL** : ne pas bâtir de rapports sur un accès SQL aux modèles analytiques, même s'il semble fonctionner : il est non supporté et disparaît au redéploiement [Vérifié S9].

---

## 8. Feuille de route et éléments à surveiller

| Élément | Statut au 20/11/2026 | Source |
|---|---|---|
| BDC Connect for Microsoft Fabric | Annoncé ; GA prévue T3 2026 non confirmée ; possible glissement à fin T1 2027 (non officiel) | [S26, S27, S25] |
| BDC Connect for Snowflake | Annoncé, sans date de GA | [secondaire S38] |
| BDC Connect for Google BigQuery | GA depuis le 27/07/2026 (BDC sur Google Cloud et AWS ; Azure « coming soon ») | [secondaire S38] |
| SAP Lakehouse (Dremio : Apache Iceberg/Polaris, moteur SQL fédéré) | Annoncé à TechEd 2026 ; **GA prévue T1 2027** | [Vérifié S32] |
| **[Hypothèse]** Moteur SQL fédéré Dremio comme futur point d'accès Power BI | Non annoncé par SAP pour Power BI ; à surveiller | — |
| Mirroring for SAP Datasphere (Fabric) | GA (mars 2026) | [Vérifié S23] |
| Copy job SAP ABAP add-on (Fabric) | Annoncé « *publicly available at Microsoft Build 2026* » | [Vérifié S25] |

---

## Annexe A — Accès direct S/4HANA → Fabric/Power BI (hors Datasphere)

*Synthèse d'une page demandée pour comparaison.*

| Option | Principe | Statut contractuel / technique | Commentaire |
|---|---|---|---|
| **Outils tiers via ODP-RFC** (ancien connecteur SAP CDC d'ADF, etc.) | Extraction delta via les modules RFC ODP | **Interdit pour les scénarios SAP → non-SAP** : la note 3255746 (v4, 02/2024) qualifie l'usage de « non permis » ; la version 11 (21/04/2026) restreint les modules RFC ODP aux transferts entre applications SAP [secondaire S33, S58] ; correctif de sécurité bloquant les appels tiers [Vérifié S25] | À exclure |
| **Copy job Fabric avec add-on ABAP** | Extraction sans ODP-RFC | Annoncé disponible à Microsoft Build 2026 [Vérifié S25] | Vérifier certification SAP et termes contractuels |
| **OData S/4HANA (CDS)** | API OData exposées par S/4HANA | Option citée par SAP (ODP-OData) comme alternative [secondaire S59] | Performance limitée pour la masse ; sémantique CDS partielle |
| **Connecteur SAP HANA sur la base S/4HANA** | SQL direct sur la base HANA de S/4 | **[Connaissance, à vérifier contractuellement]** Une licence HANA *runtime* restreint l'accès direct à la base par des outils tiers ; contournement des autorisations ABAP | Non recommandé |
| **Connecteurs SAP BW de Power BI** (serveur d'applications / de messages) | Requêtes BEx/BW via MDX | **[Connaissance]** Disponible pour BW on-premise ; non pertinent pour la cible Datasphere | Utile en transition seulement |
| **Mirroring / Copy job via Datasphere** | Datasphere comme intermédiaire supporté | Présenté comme non concerné par la note 3255746 [Vérifié S25] | Copie + premium outbound |

**Conclusion [Déduction]** : pour un client SAP, **le passage par Datasphere/BDC est la voie conforme recommandée par SAP** pour alimenter des outils tiers ; un accès direct S/4HANA → Fabric n'apporte ni conservation de la sémantique BW/BO ni simplicité contractuelle.

---

## Annexe B — Solutions tierces

*Non natives ; citées pour information, sans évaluation en environnement réel. Toute solution tierce doit être vérifiée au regard de la note SAP 3255746 et des conditions de support SAP/Microsoft.*

| Solution | Apport | Inconvénients / points d'attention |
|---|---|---|
| **CData Power BI Connector for SAP HANA** (utilisable sur Datasphere) | Connecteur Power BI s'appuyant sur un DSN ; documenté pour Datasphere (hôte, port 443, TLS, utilisateur de base) [secondaire S60] ; l'éditeur revendique de meilleures performances que le connecteur natif (benchmark publié par l'éditeur lui-même) [secondaire S61] | Licence supplémentaire ; mêmes limites fonctionnelles qu'A1 (pas de modèles analytiques) [Déduction] ; benchmark non indépendant |
| **Connecteurs « SAP → Power BI » d'éditeurs** (ex. Metrica, référencé sur le site partenaires SAP [secondaire S62]) | Accès simplifié à S/4HANA | Ciblent S/4HANA, pas Datasphere ; vérifier la méthode d'extraction (ODP) |
| **Theobald Xtract Universal** | Réplication SAP vers de nombreuses cibles, dont Datasphere ou des environnements tiers [secondaire S33] | Hors périmètre « données dans Datasphere » ; conformité ODP à vérifier selon le mode d'extraction |
| **Solutions certifiées SAP via Open Mirroring Fabric** (SNP Glue, etc.) | CDC à faible latence vers OneLake [Vérifié S25 pour le principe] | Copie dans OneLake ; abonnement supplémentaire |
| **Fivetran / HVR, Qlik, Matillion…** | Plateformes d'intégration multi-sources | Copie ; impact de la note 3255746 selon les connecteurs (ex. Matillion indique un connecteur ODP bloqué, OData autorisé [secondaire S63]) |

---

## Annexe C — Check-lists de configuration

### C1. Connexion SQL (A1)

- [ ] Espace Datasphere identifié ; vues de consommation conçues et **exposées**
- [ ] Utilisateur de base de données créé (`ESPACE#SUFFIXE`), lecture activée, espace déployé
- [ ] Hôte / port 443 / mot de passe stockés dans un coffre ; politique de rotation définie
- [ ] IP publiques des postes et des passerelles ajoutées à l'allowlist Datasphere
- [ ] Client SAP HANA 2.0 (64 bits, version récente) installé sur les postes et passerelles
- [ ] Connecteur **SAP HANA** (et non ODBC) pour DirectQuery ; mode relationnel/multidimensionnel choisi **avant** création du rapport
- [ ] Passerelle : source SAP HANA déclarée, authentification Basic, test de connexion
- [ ] RLS Power BI définie ; DAC Datasphere éventuels testés avec l'utilisateur de base
- [ ] Modèle sémantique partagé publié, certifié, documenté

### C2. OData (A2)

- [ ] Objets exposés (vues) / modèles analytiques déployés
- [ ] Rôle à portée avec *Data Warehouse Consumption* + *Space Files* (ou modèle DW Consumer)
- [ ] Client OAuth créé — finalité **Technical User** (rafraîchissement planifié) ou **Interactive Usage** (identité utilisateur)
- [ ] Client ID / Secret / Token URL (+ Authorization URL si interactif) stockés de manière sécurisée
- [ ] Méthode Power BI : connecteur `.mez` d'exemple **ou** requête M ; **pas** les connecteurs OData/Web standards
- [ ] Gestion de la pagination validée (`nextLink`) et durée de rafraîchissement mesurée
- [ ] Variables du modèle analytique **mono-valeur** uniquement
- [ ] Passerelle configurée si connecteur personnalisé

---

## Annexe D — Points non vérifiés et moyens de vérification

| Point | Nature | Comment vérifier |
|---|---|---|
| Limitation « mode relationnel + TLS » appliquée à HANA Cloud / Datasphere | Doc Microsoft potentiellement datée | POC 1 ; ticket Microsoft si nécessaire |
| Perspectives consommables en ODBC | Divergence SAP Help | POC ; SAP Help « Consume Data in Power BI… via ODBC/JDBC » |
| `client_credentials` avec client *Technical User* pour Power BI | Documentation récente vs KBA anciennes | POC 3 ; page *App Integration* du tenant |
| Hiérarchies et associations en OData analytique | Source secondaire (NTT Data) | POC 4 |
| Vues Datasphere non sources de replication flow | Connaissance | SAP Help « Add the Source for a Replication Flow » |
| Statut GA de BDC Connect for Fabric | Non confirmé | SAP Road Map Explorer (BDC) ; annonces Microsoft Fabric / SAP News |
| Restrictions de la licence HANA *runtime* | Connaissance | Contrat SAP du client ; account manager SAP |
| Modes Import/DirectQuery sur SAP Databricks | Non détaillé par la doc | Doc Databricks / POC 7 |
| Limites Power BI (1 Go Pro, 1 M lignes DirectQuery) | Connaissance | Microsoft Learn « Power BI capacity and SKUs » / « DirectQuery » |
| Tarifs (unités de capacité, premium outbound, F-SKU) | Non chiffrés | Estimateur de capacité SAP ; calculateur Fabric ; offres commerciales |

---

## Annexe E — Sources

*Consultées le 20/11/2026. « Officielle » = éditeur ; « secondaire » = partenaire, communauté ou presse (signalée comme telle dans le texte).*

**Microsoft (officielles)**
- [S1] Microsoft Learn — *Connect to SAP HANA data sources by using DirectQuery in Power BI* — mise à jour 10/12/2024 — https://learn.microsoft.com/en-gb/power-BI/connect-data/desktop-directquery-sap-hana
- [S2] Microsoft Learn — *Power Query SAP HANA database connector* — mise à jour 01/12/2025 — https://learn.microsoft.com/en-us/power-query/connectors/sap-hana/overview
- [S3] Microsoft Learn — *ODBC connector* — mise à jour 29/09/2026 — https://learn.microsoft.com/en-us/power-query/connectors/odbc
- [S22] Microsoft Learn — *Mirroring SAP* — mise à jour 27/06/2026 — https://learn.microsoft.com/en-us/fabric/mirroring/sap
- [S22b] Microsoft Learn — *Limitations in Microsoft Fabric mirrored databases from SAP* — mise à jour 26/02/2026 — https://learn.microsoft.com/en-us/fabric/mirroring/sap-limitations
- [S23] Microsoft Tech Community — *SAP on Azure Product Announcements Summary – SAP Sapphire 2026* (Mirroring for SAP Datasphere GA depuis mars 2026) — https://techcommunity.microsoft.com/blog/sapapplications/sap-on-azure-product-announcements-summary-%E2%80%93-sap-sapphire-2026/4517634 ; Fabric Updates Blog — FabCon 2026 — https://community.fabric.microsoft.com/t5/Fabric-Updates-Blog/Fabric-Data-Factory-at-FabCon-Atlanta-Built-for-modern-data/ba-p/5172225
- [S23b] Azure Blog — *Advancing enterprise AI: New SAP on Azure announcements from SAP Sapphire 2026* — https://azure.microsoft.com/en-us/blog/advancing-enterprise-ai-new-sap-on-azure-announcements-from-sap-sapphire-2026/
- [S24] Microsoft Learn — *Change Data Capture from SAP via SAP Datasphere Outbound in Copy job* — https://learn.microsoft.com/en-us/fabric/data-factory/copy-job-tutorial-sap-datasphere
- [S25] Microsoft Learn — *Overview and architecture of the SAP CDC capabilities* (note 3255746, alternatives) — mise à jour 27/05/2026 — https://learn.microsoft.com/en-us/azure/data-factory/sap-change-data-capture-introduction-architecture
- [S28] Microsoft Tech Community — *SAP Business Data Cloud Connect with Azure Databricks is now generally available* — 15/10/2025 — https://techcommunity.microsoft.com/blog/analyticsonazure/-/4459490 ; Microsoft Learn — *SAP BDC connector for Azure Databricks* — https://learn.microsoft.com/azure/databricks/delta-sharing/sap-bdc

**SAP (officielles)**
- [S4] SAP Help — *Consume Data in Power BI and Other Clients, Tools, and Apps via ODBC/JDBC* — https://help.sap.com/docs/SAP_DATASPHERE/d4f3c5a0bb074d09ae9b42b2b9bd7a08/4db6f5a329af44509ae422ad707877b2.html
- [S5] SAP Help — *Consuming Data Exposed by SAP Datasphere* — https://help.sap.com/docs/SAP_DATASPHERE/d4f3c5a0bb074d09ae9b42b2b9bd7a08/ff47998f19154d159b89da50980f98b2.html
- [S6] SAP Help — *Consume Data in Power BI and Other Clients, Tools, and Apps via an OData Service* — https://help.sap.com/docs/SAP_DATASPHERE/c8a54ee704e94e15926551293243fd1d/add771abf6f54c9d8de4c7e470a0e6f0.html
- [S7] SAP Help — *Consuming Data via the OData API* — https://help.sap.com/docs/SAP_DATASPHERE/43509d67b8b84e66a30851e832f66911/7a453609c8694b029493e7d87e0de60a.html
- [S8] SAP-docs (GitHub officiel) — *Create OAuth2.0 Clients to Authenticate Against SAP Datasphere* — https://github.com/SAP-docs/sap-datasphere/blob/main/docs/Administering/Creating-and-Configuring-Your-Tenant/create-oauth2-0-clients-to-authenticate-against-sap-datasphere-3f92b46.md
- [S9] KBA 3674874 — *Analytic Model will not be visible anymore to users accessing via ODBC / JDBC in SAP Datasphere* — https://userapps.support.sap.com/sap/support/knowledge/en/3674874
- [S10] KBA 3532736 — connexion OData Datasphere depuis Power BI (méthode non supportée) — https://userapps.support.sap.com/sap/support/knowledge/en/3532736
- [S11] KBA 3718383 — *Using a Technical Usage OAuth Client for SAP Datasphere OData Connections in SAP Analytics Cloud* — https://userapps.support.sap.com/sap/support/knowledge/en/3718383
- [S12] KBA 3318090 — https://userapps.support.sap.com/sap/support/knowledge/en/3318090 ; KBA 3383701 — https://userapps.support.sap.com/sap/support/knowledge/en/3383701 ; KBA 3576221 — https://userapps.support.sap.com/sap/support/knowledge/en/3576221
- [S13] KBA 3707272 — pagination serveur (50 000 enregistrements / 50 Mo) — https://userapps.support.sap.com/sap/support/knowledge/en/3707272
- [S13a] KBA 3337495 — pagination `nextLink` — https://userapps.support.sap.com/sap/support/knowledge/en/3337495
- [S14] KBA 3501298 — variables non mono-valeur non supportées en OData — https://userapps.support.sap.com/sap/support/knowledge/en/3501298
- [S15] KBA 3746273 — `$apply` non supporté sur l'API des modèles analytiques — https://userapps.support.sap.com/sap/support/knowledge/en/3746273
- [S16] KBA 3570774 — descriptions de champs en OData dans Power BI — https://userapps.support.sap.com/sap/support/knowledge/en/3570774
- [S17] KBA 3456481 — Premium Outbound requis pour les cibles non SAP — https://userapps.support.sap.com/sap/support/knowledge/en/3456481
- [S18] KBA 3467070 — table locale Datasphere en source : *Initial Only* — https://userapps.support.sap.com/sap/support/knowledge/en/3467070
- [S19] SAP-samples — dépôt *data-warehouse-cloud-connectors* (statut du connecteur Power BI) — https://github.com/sap-samples/data-warehouse-cloud-connectors (via l'index https://awesome.ecosyste.ms/projects/github.com%2Fsap-samples%2Fdata-warehouse-cloud-connectors)
- [S20] SAP Blog — *Connecting SAP Data Warehouse Cloud OData API with Power BI via a Blank Query* (2022) — https://blogs.sap.com/2022/09/23/connecting-sap-data-warehouse-cloud-odata-api-with-powerbi-via-a-blank-query-2/
- [S26] SAP News — *SAP and Microsoft Accelerate Business Insights and AI Innovation with SAP BDC Connect for Microsoft Fabric* — 18/11/2025 — https://news.sap.com/2025/11/sap-bdc-connect-for-microsoft-fabric-business-insights-ai-innovation/
- [S27] SAP Community Q&A — *SAP BDC Connect for Microsoft Fabric – GA?* — réponse du 03/09/2026 (**non officielle**) — https://community.sap.com/t5/data-and-ai-professionals-q-a/sap-bdc-connect-for-microsoft-fabric-ga/qaq-p/14469254
- [S31] SAP Learning — *Data Product Generator for SAP Business Data Cloud* — https://learning.sap.com/courses/transforming-sap-bw-with-sap-business-data-cloud/introducing-the-data-product-generator ; *Installing a Data Product to the Foundation Services* — https://learning.sap.com/courses/developing-detailed-knowledge-of-sap-business-data-cloud/installing-a-data-product-to-the-foundation-services ; *Extending Data Products* — https://learning.sap.com/courses/developing-detailed-knowledge-of-sap-business-data-cloud/extending-data-products_d60954ca-c983-4da3-b98d-460b1fe82820 ; *Illustrating the Role of SAP Datasphere in SAP BDC* — https://learning.sap.com/courses/introducing-sap-business-data-cloud/exploring-the-role-of-sap-datasphere-in-sap-business-data-cloud
- [S32] SAP News — *Advancing SAP Lakehouse and New Data Fabric Innovations in SAP BDC* — 29/10/2026 — https://news.sap.com/2026/10/sap-lakehouse-new-data-fabric-innovations-sap-bdc-autonomous-enterprise/ ; *SAP TechEd Innovation News Guide 2026* — https://www.sap.com/topics/events/teched/innovation-news-guide-2026
- [S34] SAP User Groups — *Breaking down the SAP BDC pricing model* — 16/04/2026 — https://assets.dm.ux.sap.com/sap-user-groups/pdfs/260416_breaking_down_the_sap_bdc_pricing_model.pdf
- [S36] SAP Help Support Content — *How to use Data Access Control Consuming SAP Datasphere via ODBC in Power BI…* — https://help.sap.com/docs/SUPPORT_CONTENT/datasphere/5416291142.html
- [S37] SAP Community — *SAP Business Data Cloud and Datasphere News in October* (2026) — https://community.sap.com/t5/data-and-ai-professionals-blog-posts/sap-business-data-cloud-and-datasphere-news-in-october/ba-p/14524405
- [S56] SAP Help (SAC) — *Live Data Connections to SAP Datasphere* — https://help.sap.com/doc/00f68c2e08b941f081002fd3691d86a7/2023.20/en-US/ad4281e2875949f0b4d45d1072ff4c38.html

**Databricks (officielles)**
- [S29] Databricks Docs — *Power BI (SAP Databricks)* — mise à jour 11/11/2026 — https://docs.databricks.com/sap/en/partners/bi/power-bi
- [S30] Databricks Blog — *SAP Databricks now available on AWS* (data products disponibles via Delta Sharing) — https://www.databricks.com/blog/sap-databricks-now-available-aws

**Secondaires (partenaires, communauté, presse) — à considérer avec prudence**
- [S33] Theobald Software — *SAP Note 3255746: How to Achieve Data Integration Without ODP-RFC* — 23/04/2026 — https://theobald-software.com/en/blog/sap-note-325574
- [S35] NTT DATA Business Solutions — *Connecting Power BI to SAP Datasphere* — https://nttdata-solutions.com/uk/blog/connecting-power-bi-to-sap-datasphere/
- [S38] Kanerika — *SAP Business Data Cloud: Architecture and How It Fits* (état novembre 2026) — https://kanerika.com/blogs/sap-business-data-cloud/
- [S39] Altivate — *SAP Business Data Cloud for Enterprise Data and AI* — https://www.altivate.com/blog/sap-business-data-cloud-enterprise-data-ai-2/
- [S40] zpartner — Data Product Generator (BW PCE) — https://www.zpartner.eu/?p=14822
- [S41] McCoy Partners — *Datasphere and Power BI, best of both worlds! Part 2* — https://mccoy-partners.com/en/updates/datasphere-and-power-bi-best-of-both-worlds-part-2
- [S42] SAP Community (membre) — *Consuming data from SAP Datasphere into Power BI via ODBC connector* — https://community.sap.com/t5/technology-blog-posts-by-members/consuming-data-from-sap-datasphere-into-power-bi-via-odbc-connector/ba-p/13577346
- [S43] McCoy Partners — *Datasphere and Power BI, best of both worlds! Part 1* — https://mccoy-partners.com/en/updates/datasphere-and-power-bi
- [S44] Microsoft Fabric Community — fil d'erreur ODBC Datasphere (hôte `…hanacloud.ondemand.com:443`) — https://community.fabric.microsoft.com/t5/Power-Query/SAP-Datasphere-to-PBI-via-ODCB/m-p/4830412
- [S45] CData — *Power BI SAP HANA Connector Performance Benchmark* (prérequis client HANA) — https://www.cdata.com/kb/articles/sap-hana-powerbi-benchmark.rst
- [S46] SAP Blog — *SAP Datasphere – Power BI DirectQuery via HANA Cloud Calculation Views* (2023) — https://blogs.sap.com/2023/11/03/sap-datasphere-powerbi-direct-query-via-hana-cloud-calculation-views/
- [S47] Microsoft Fabric Community — *Connect SAP Datasphere view with Prompts to Power BI using Direct Query* — https://community.fabric.microsoft.com/t5/Power-Query/Need-help-to-Connect-SAP-Datasphere-view-with-Prompts-to-Power/m-p/5005603
- [S48] SAP Community (blog SAP) — *SAP Datasphere Analytical and Relational OData APIs* — https://answers.sap.com/t5/technology-blogs-by-sap/sap-datasphere-analytical-and-relational-odata-apis/ba-p/13573797
- [S49] SAP Community (blog SAP) — *Using the Datasphere OData API with SAP Analytics Cloud* — https://community.sap.com/t5/technology-blogs-by-sap/using-the-datasphere-odata-api-with-sap-analytics-cloud/ba-p/13525315
- [S50] KBA 3662442 — erreur 403 avec utilisateur technique (rôles à portée) — https://userapps.support.sap.com/sap/support/knowledge/en/3662442
- [S51] NTT DATA (cf. S35) — mention « *not covered by any SLA* » du connecteur GitHub
- [S52] Microsoft Fabric Community — *Custom OData SAP DataSphere connector without Gateway* — https://community.fabric.microsoft.com/t5/Service/Custom-OData-SAP-DataSphere-connector-without-Gateway/m-p/4401111
- [S53] SAP Community Q&A — *Retrieve data from SAP Datasphere using OData API* — https://community.sap.com/t5/technology-q-a/retrieve-data-from-sap-datasphere-using-odata-api-for-system-integration/qaq-p/14213530
- [S54] McCoy Partners (cf. S43) — « *OData is always importing data* »
- [S55] Microsoft Fabric Community — *Best-in-class connectivity for data movement in Fabric Data Factory* (connecteur SAP Datasphere GA) — https://community.fabric.microsoft.com/t5/Fabric-Updates-Blog/Best-in-class-connectivity-for-data-movement-in-Fabric-Data/ba-p/5172141
- [S57] Analytics Legends — *BDC Connect for Microsoft Fabric* (revue au 27/09/2026) — https://analyticslegends.ai/concepts/bdc-connect-for-microsoft-fabric/
- [S58] SAPinsider — note 3255746 v11 — https://sapinsider.org/?p=95135
- [S59] E3 Magazine — *Use of SAP interfaces prohibited* — https://e3mag.com/en/use-of-sap-interfaces-prohibited/
- [S60] CData Community — *Integrate CData Power BI Connector for SAP HANA with SAP Datasphere* (06/2024) — https://community.cdata.com/getting-started-60/integrate-cdata-power-bi-connector-for-sap-hana-with-sap-datasphere-909
- [S61] CData — benchmark (cf. S45)
- [S62] SAP — page partenaire Metrica *Power BI Connector for SAP* — https://www.sap.com/products/data-cloud/partners/metrica-software-inc-power-bi-connector-for-sap.html
- [S63] Matillion Docs — *SAP Note 3255746 and its impact on SAP ODP data extraction* — https://docs.matillion.com/metl/docs/tech-note-sap-3255746/

---

*Fin du document — v1.0 du 20/11/2026. Les statuts de disponibilité (GA/preview/annoncé) doivent être revérifiés à chaque diffusion client.*

## Voir aussi

- [[modele-donnees-ecc-s4hana]]
