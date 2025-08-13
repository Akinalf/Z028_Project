# Z028_Project : Documentation Technique

## Vue d'ensemble métier
- **Objectif** : Trouver la pire/meilleur route de process du debut jusqu'au PT intermediare
- **Techno** : [Z028]
- **Mesures de succès** : Validation metier + reproductibilitée 

## Architecture technique
- **Technologies** : Spotfire, python, Databricks, pySpark, SQL
- **Ressources cluster** : [Voir Section Driver](#DriverNode)




# État de l'Art : Optimisation des Routes de Process

## 1. Problématique et Contexte 

### 1.1 Définition du problème
L'optimisation des routes de processus (de semi-conducteurs) vise à identifier les séquences d'équipements et d'étapes qui maximisent le rendement (yield) et et par consequent minimisent les défauts. Une "route" représente le chemin exact qu'un lot/wafer suit à travers les différentes etapes de fabrication.

### 1.2 Enjeux industriels
Dans le cadre de la technologie Z028 au cours de la route jusqu'au PT intermediaire on retrouve:
- **Complexité des fabs** : 134 opérations, 222 étapes, 1 a 5 équipements par étape
- **Impact économique** : 1% à 2% d'amélioration du yield serait non negligable
- **Variabilité équipements** : Performances différentes selon l'équipement choisi pour chaque étape
- **Contraintes temps réel** : Décisions de routage nécessaires en continu

## 2. Approches Algorithmiques pour l'Optimisation de Routes

### 2.1 Approches de Data Mining

#### 2.1.1 Règles d'Association (Approche retenue)
**Principe** : Identifier les patterns fréquents entre équipements et résultats qualité.

APRIORI : "Si ETCH-XXX01 ET LITHO-XXX2 alors 89% chance défaut"
→ Explication claire et actionnable

FP-Growth : Résultat identique mais processus "boîte noire"  
→ Plus difficile à expliquer aux experts métier

Volume Données vs Performance Réelle
l'approche fenêtre glissante 12 semaines change la donne :
|Algorithme|Dataset Complet (50M)|Fenêtre 12 sem (~4M)|Verdict|
|----------|---------------------|--------------------|-------|
|APRIORI|Lent (~45+ min)|Acceptable (~4.2 min)|Optimal|
|FP-Growth|Rapide (~12 min)|Très rapide (~1.8 min)|Excessif|

**Algorithmes principaux** :
- **APRIORI** (Agrawal & Srikant, 1994)
  - ↗️ Avantages : Simplicité, interprétabilité des règles
  - ↗️ Adapté : Données catégoriques (équipements), relation claire cause-effet
  - ❌ Limitations : Performance sur gros volumes, scan multiple base de données

- **FP-Growth** (Jiawei Han, Jian Pei, and Yiwen Yin, 2000)
  - ↗️ Avantages : Performance supérieure, un seul scan base de données
  - ↗️ Adapté : Gros volumes manufacturing data
  - ❌ Limitations : Consommation mémoire, moins intuitif

- **Prefix Span** (Jian Pei, Jiawei Han ,Behzad Mortazavi-Asl et Helen Pinto 2001)
  - ↗️ Avantages : Prise en compte séquences temporelles
  - ↗️ Adapté : Routes séquentielles dans manufacturing
  - ❌ Limitations : Complexité algorithmique élevée


#### 2.1.2 Market Basket Analysis Adapté
**Principe** : Transposition des techniques retail vers manufacturing.
- "Produits" = Équipements utilisés
- "Transactions" = Lots de wafers
- "Règles" = Si équipement A alors probabilité Wafer BAD élevée

### 2.2 Approches Machine Learning

#### 2.2.1 Apprentissage Supervisé sur Données Continues
**Algorithmes testés** :
- **Random Forest** : Robuste aux outliers, importance variables
- **Gradient Boosting** : Performance élevée, gestion de la non-linéarités
- **SVM** : Efficace pour dimensions élevées

**Limitations identifiées** :
- Variables continues (températures, pressions) moins discriminantes que choix équipements
- Perte d'information causale directe équipement→défaut
- Complexité interprétation pour experts manufacturing
- Dimensionalité trop élevée

#### 2.2.2 Apprentissage Supervisé sur Données Catégoriques
**Algorithmes testés** :
- **Régression Logistique** : Baseline interpretable
- **Régression Binomiale Négative** : Gestion de la surdispersion
- **Classification Naive Bayes** : Rapide, requiert peu de parametres

**Avantages** :
- Probabilités prédiction directement utilisables
- Variables catégoriques = équipements (mapping naturel)

**Limitations** :
- Hypothèses statistiques forte (indépendance, distribution)
- Difficulté capture interactions complexes équipements
- Dimensionalité trop élevée

#### 2.2.3 Deep Learning / Réseaux de Neurones
**Approches envisagées** :
- **LSTM** : Séquences temporelles de routes
- **CNN** : Patterns spatiaux sur wafers

**Raisons rejet** :
- "Trop de combinaisons pour l'algorithme" → Explosion combinatoire
- 134 opérations × moyens 3-4 équipements/opération = ~10^200 routes possibles
- Manque données étiquetées pour entraînement deep learning
- Complexité déploiement et maintenance en production

### 2.3 Méthodes d'Optimisation Combinatoire

#### 2.3.1 Algorithmes Génétiques
- **Principe** : Évolution d'une population de routes complètes vers l'optimum yield par sélection, croisement et mutation successives. Chaque route (chromosome) représente une séquence complète d'équipements pour les 134 opérations.
- **Applications** : Optimisation simultanée de séquences complètes d'équipements (Kumar et al., 2006).
- **Limitations** : Explosion combinatoire, Solutions optimales sans justification causale

## 3. Techniques de Validation Routes Manufacturing

### 3.1 Validation (Approche retenue)
**Principe** : Comparaison performance routes identifiées sur données historiques.

**Méthode** :
- Récupération wafers ayant suivi exactement la "golden/worst route"
- Comparaison taux GOOD/BAD vs wafers routes alternatives
- Test significativité statistique (Chi-2, Fisher exact test)

**Avantages** :
- Données réelles, représentatives dans les conditions de production
- Validation robuste si volume suffisant
- Interprétation directe pour experts métier

### 3.3 Validation Expérimentale
- **Principe** : Test contrôlé routes identifiées sur lots dédiés.
- **Avantages** : Validation définitive, conditions réelles
- **Limitations** : Coût élevé, risque production, délais longs

### 3.4 Cross-Validation Temporelle
- **Principe** : Entraînement sur période N, validation période N+1.
- **Application** : Fenêtre glissante 12 semaines avec validation 4 semaines suivantes
- **Robustesse** : Détection dérive performance équipements

## 4. Comparaison et Justification Choix Techniques

### 4.1 Matrice de Décision

| Critère | Règles Association | ML Supervisé | Deep Learning | Optimisation Combinatoire |
|:---------:|:-------------------:|:--------------:|:---------------:|:--------------:|
| **Interprétabilité** |  🟩Excellente | 🟨Moyenne |  🟥Faible | 🟨Moyenne |
| **Performance** | 🟨Moyenne |🟩Bonne |🟩Excellente | 🟩Bonne |
| **Temps développement** |🟩Court | 🟨Moyen |  🟥Long | 🟥Long |
| **Robustesse données** |🟩Bonne | 🟨Moyenne |  🟥Faible |🟩 Bonne |
| **Déploiement prod** |🟩Simple | 🟨Moyen | 🟥Complexe | 🟨Moyen |
| **Expertise requise** | 🟨Moyenne |🟩Standard | 🟥Élevée | 🟥Élevée |

### 4.2 Justification Choix Final

**Règles d'Association (APRIORI) retenues pour** :
- **Interprétabilité essentielle** : Les experts métier doivent pouvoir comprendre facilement les recommandations d’équipements.
- **Données adaptées** : Les variables catégoriques (types d’équipements) sont naturellement compatibles avec cet algorithme.
- **Validation simple** : Les règels générées sont facilement vérifiable à partir des données historiques 
- **Déploiement rapide** : L’algorithme est simple à mettre en œuvre et à maintenir.

**Machine Learning écarté pour** :
- La complexité d'interprétation est trop élevée pour une prise de décision en production.
- Le gain de performance est insuffisant face au coût de développement et de maintenance.
- Risque important de surapprentissage (overfitting) dû au bruit présent dans les données de fabrication.

**Deep Learning écarté pour** :
- Explosion combinatoire liée au nombre très élevé de routes possibles.
- Manque données (wafer) en production pour un apprentissage efficace.
- Complexité importante en termes d’infrastructure et d’expertise nécessaire.

## 6. Perspectives et Améliorations Futures

### 6.1 Hybridation des approches
- Combinaison des règles d'association avec des méthodes ML pour améliorer la précision et la robustesse des recommandations.

### 6.2 Temps Réel et Streaming
- Adaptation Kafka pour données équipements en temps réel


## Implémentation code détaillé

---
Pipeline Unity Catalog
---
<img width="1615" height="765" alt="image" src="https://github.com/user-attachments/assets/dddd34db-a18e-42af-b4d7-e13d26724eeb" />


---

#  Job et Pipeline Databricks

## **Vue d'ensemble du Job**

###  **Principe clé :**
- **Un seul notebook** → Deux comportements différents
- **Paramètre `type_route`** → Détermine le flux de données
- **Logique conditionnelle** → `"good"` vs `"bad"` routes
- **Planification** → Execution par période.

---


| Phase | Processus | Durée |
|-------|-----------|--------|
| **Phase 1** | Data Prep | 6 min |
| **Phase 2** | APRIORI | 56 min |
| **Phase 3** | Validation | 30 sec |


# **Job Principal : `PT_Z028_Worst_Route`**

## Cluster Z028_Job_Cluster
<a id="DriverNode"></a>
## Configuration des Nodes

<table>
<tr>
<td width="50%" align="center">


### **Driver Node**
**Type :** Standard_D5s_v2
**RAM :** 112 GB
**Cores :** 16
**Quantité :** 1

</td>
<td width="50%" align="center">

### **Worker Nodes**
**Type :** Standard_D4ds_v5  
**RAM :** 64 GB par node
**Cores :** 4 par node
**Quantité :** 8 (Spot instances)

*Les Spot Instances dans Databricks sont des machines virtuelles à prix réduit, mais avec moins de garanties de disponibilité.*
</td>
</tr>
</table>

## Ressources Totales

<div align="center">

| **RAM Total** | **Cores Total** | **Nodes Total** |
|:----------------:|:------------------:|:-------------------:|
| **128-384 GB** | **32-96 Cores** | **6 Nodes** |

---

### **Performance**

| Tâche | Durée | Notes |
|:-------:|:-------:|:-------:|
| **Data_Preparation** | 10 min | Traitement Unity Catalog |
| **AR_BAD/GOOD** | 46 min | APRIORI fenêtre glissante |
| **Validation** | 6 min | Tests statistiques |
| **Total Pipeline** | **Auto-scaling** |  |

### **Orchestration & Dépendances**

```mermaid
graph TD
    A[Data_Preparation] --> B[AR_BAD]
    A --> C[AR_GOOD] 
    B --> D[AR_BAD_VALIDATION]
    C --> E[AR_GOOD_VALIDATION]
    
    style A fill:#667eea
    style B fill:#f093fb  
    style C fill:#f093fb
    style D fill:#4facfe
    style E fill:#4facfe
```
</div>

### Planification
|**Fréquence** |**Déclencheur**|**Timeout**|
|---------------------------------------|------------------------|---------------------|
| Hebdomadaire (chaque lundi 02:00 UTC) | Cron `58 30 4 ? * Mon` | 2h maximum par job  |


### Avantages Architecture Databricks

####  Réutilisabilité maximale

- **Même notebook** `AR_FAMILIES_EQPT`  
- **Paramètre** `type_route` change le comportement  
- **DRY Principle** respecté  

####  Parallélisation intelligente

- **AR_BAD** et **AR_GOOD** en parallèle  
- **Validation** en séquence pour cohérence  
- **Optimisation ressources** automatique  


####  Monitoring intégré

- **Métriques temps réel** par tâche  
- **Notifications email** en cas d'échec  
- **Logs centralisés** pour debug  

---
####  Scalabilité


|**Auto-scaling cluster** |**Photon engine**|**Adaptive Query Execution**|
|:---------------------:|:-----------------------:|:--------------:|
|  selon la charge  | pour performance SQL | optimisé |


---


## **Principe de Paramétrage**

### **Notebook unique : `AR_FAMILIES_EQPT`**

| Paramètre | Valeur | Comportement |
|:------------:|:--------:|:-------------------------------:|
| `type_route` | `"good"` | Lit/Écrit dans **Golden Route** |
| `type_route` | `"bad"` | Lit/Écrit dans **Worst Route** |

### **Logique dynamique dans le code :**
```python
# Dans le notebook AR_FAMILIES_EQPT
type_route = dbutils.widgets.get("type_route")

# Pour import
df_EQPT_spark = spark.sql(f"""
    SELECT *
    FROM mds_prod_gold_experiment.datasciences_dev.pt_z028_input_{type_route.lower()}_for_ar
    WHERE T84_TEST_DATE BETWEEN DATE_SUB(CURRENT_DATE(), 365) AND CURRENT_DATE()
""")
# Et export
    create_or_update_table(spark.createDataFrame(sparkfinal_merged_df),
    f"mds_prod_gold_experiment.datasciences_dev.pt_z028_ar_{type_route.lower()}_route_results")
```

---

## **Avantages de cette approche**

### **Réutilisabilité**

- **Un seul code source** à maintenir
- **Logique métier identique** pour les deux cas
- **Évite la duplication** de code

### **Maintenabilité**

- **Changements centralisés** dans un seul notebook
- **Cohérence garantie** entre les deux flux
- **Tests simplifiés**

### **Flexibilité**
- **Facilement extensible** (ajout de nouveaux types)
- **Configuration par paramètres**
- **Orchestration Databricks native**

---



</div>


# Phase 1 : DATA_PREP.py - Rapport d'état des lieux

## Vue d'ensemble

Le script `DATA_PREP.py` constitue la phase de préparation des données du projet Z028. Il traite les données brutes du PT intermediaire pour produire les tables nécessaires à l'analyse des règles d'association.

*Note technique : Une version parallèle de ce script existe avec des clauses WHERE étendues à 365 jours (au lieu des filtres standards). Cette version a servi à l'initialisation initiale des tables de référence avec un historique complet d'une année de données de production.*

### 1. Extraction SQL depuis Unity Catalog

#### LOT_LIST_INFORMATION_DF - Liste des lots éligibles
```sql
SELECT pt_kdf_lot_number AS LOT, 
       pt_kdf_cam_location AS LOCATION, 
       pt_kdf_product_code AS PRODUCT,
       pt_kdf_test_start_datetime AS START_DATE,
       pt_kdf_test_end_datetime AS END_DATE,
       pt_kdf_spec_name AS SPEC_NAME,
       pt_kdf_spec_version AS SPEC_VERSION

FROM sem1.pt_kdf_lot_normalized 
WHERE pt_kdf_cam_location = 'M2SICN-ADT02' 
  AND fab_name = 'CROLLES 300'
  AND LEFT(pt_kdf_product_code, 5) = 'KVB98'
  AND pt_kdf_test_end_datetime >= DATE_SUB(CURRENT_DATE(), 7)
  AND length(pt_kdf_lot_number) <= 7
```
**Filtres appliqués** :
- **Localisation** : M2SICN-ADT02 (équipement de test spécifique)
- **Fab** : CROLLES 300 uniquement
- **Produit** : Code produit commençant par 'KVB98' (technologie Z028)
- **Période** : 365 derniers jours
- **Format lot** : Maximum 7 caractères

#### df_parameter_info - Données des paramètres
```sql
SELECT
    pt_kdf_lot_number AS LOT,
    pt_kdf_cam_location AS T84_LOCATION,
    pt_kdf_parameter_id AS PARAMETER,
    pt_kdf_parameter_value AS PARAMETER_VALUE,
    pt_kdf_test_start_datetime AS START_DATE,
    pt_kdf_test_end_datetime AS END_DATE,
    pt_kdf_wafer_number AS WAFER,
    tech_lz_folder_source_id AS FAB,
    pt_kdf_spec_name AS SPEC_NAME,
    pt_kdf_spec_version AS SPEC_VERSION,
    pt_kdf_site_name AS SITE_NUMBER,
    pt_kdf_site_is_last_pattern AS IS_LAST_PATTERN,
    MAX(pt_kdf_test_end_datetime) OVER (PARTITION BY pt_kdf_lot_number) AS MAX_TEST_DATE

FROM sem1.pt_kdf_parameter_normalized

WHERE pt_kdf_cam_location = 'M2SICN-ADT02' 
    AND fab_name = 'CROLLES 300' 
    AND pt_kdf_test_end_datetime >= DATE_SUB(CURRENT_DATE(), 7)
    AND length(pt_kdf_lot_number) <= 7
    AND pt_kdf_lot_number IN (SELECT LOT FROM LOT_LIST_INFORMATION_DF)
    AND pt_kdf_parameter_id IN (SELECT PARAMETER FROM df_PTM2_param_family_Parameter)
```
**Optimisations SQL** :
- **Window function** : MAX(test_end_datetime) OVER (PARTITION BY lot) pour identifier le dernier test
- **Jointure implicite** : IN clause avec LOT_LIST_INFORMATION_DF 
- **Filtrage paramètres** : Seulement les paramètres définis dans df_PTM2_param_family_Parameter

####  df_Value_info - Limites et spécifications
```sql
SELECT
    pt_klf_spec_name AS SPEC_NAME,
    pt_klf_spec_version AS SPEC_VERSION,
    pt_klf_parameter_id AS PARAMETER,
    pt_klf_validity_limit_lower_bound AS LVL,
    pt_klf_validity_limit_upper_bound AS UVL,
    pt_klf_spec_limit_lower_bound AS LSL,
    pt_klf_spec_limit_upper_bound AS USL,
    pt_klf_parameter_category AS PARAMETER_TYPE,
    pt_klf_parameter_target AS PARAMETER_TARGET

FROM sem1.pt_klf_normalized
    
WHERE
    pt_klf_parameter_id in (SELECT PARAMETER FROM df_PTM2_param_family_Parameter)
    AND (pt_klf_parameter_category in ('GY', 'GR', 'KR', 'KY'))
    AND pt_klf_spec_name = "C28SOIM2SICN"
```
**Limites définies** :
- **LVL/UVL** : limites de validité
- **LSL/USL** :limites de spécification
- **Catégories** : GY, GR, KR, KY (types de paramètres qualité)
- **Spec** : C28SOIM2SICN 

#### df_lot_history - Historique des équipements
```sql
select 
  leh_fe_std_lot_id as LOT
  ,leh_fe_std_operation_name as OPERATION
  ,leh_fe_std_operation_description as OPERATION_DESCRIPTION
  ,leh_fe_std_step_name as STEP
  ,leh_fe_std_equipment_id as EQUIPMENT
  ,leh_fe_std_lot_step_start_datetime as STEP_START_DATE
  ,leh_fe_std_lot_step_end_datetime as STEP_END_DATE
  ,leh_fe_std_mes_product_name as PRODUCT
  ,leh_fe_std_route_name as ROUTE
  ,leh_fe_std_recipe_id as RECIPE_ID
from publication.v_leh_fe_std_normalized
where 
  leh_fe_std_route_name = 'Z028RR_14KLR_34LS_10M'
  AND leh_fe_std_lot_id IN (SELECT LOT FROM LOT_LIST_INFORMATION_DF)
  AND SUBSTR(leh_fe_std_equipment_id, 1, 1) <> 'X'
  AND INSTR(leh_fe_std_step_name, 'MEAS') = 0
```
**Filtres route** :
- **Route spécifique** : Z028RR_14KLR_34LS_10M (route de référence Z028)
- **Équipements valides** : Exclusion des équipements commençant par 'X'
- **Étapes process** : Exclusion des étapes de mesure ('MEAS')

### 2. Fichiers CSV de référence
- **Z028_DEFAULT_PROCESS_ROUTE.csv** : Route de processus par défaut (séparateur `;`)
- **Df_Family_Ref.csv** : Référentiel des familles de paramètres (séparateur `;`)

## Transformations principales

### 1. Calcul des rendements
- Jointure données test + spécifications
- Calcul OOV/OOS par paramètre → Yield par famille
- Pivot des familles de paramètres en colonnes

### 2. Traitement des équipements
- Filtrage des étapes avec équipements variables uniquement
- Calcul de l'ordre des étapes et rang des opérations
- Encodage catégoriel des combinaisons STEP-EQUIPMENT

### 3. Classification GOOD/BAD
- Application des seuils métier par famille (ex: CONTACT < 0.98 → BAD)
- Agrégation au niveau LOGICAL_ID + OPERATION + STEP

## Outputs
Deux tables stocker dans :
  ***mds_prod_gold_experiment.datasciences_dev***
- **pt_z028_input_bad_for_ar** : Table avec indicateur BAD
- **pt_z028_input_good_for_ar** : Table avec indicateur GOOD (inverse)

**Structure finale** : 1 ligne = 1 lot + 1 étape + métriques yield + classification + équipements encodés

→  **Prêt pour l'analyse des règles d'association** dans AR_BAD.py


# Phase 2 : AR_FAMMILLIES_EQPT.py - Analyse des Règles d'Association

##  Objectif
Identifier les équipements problématiques par famille de paramètres en utilisant l'algorithme APRIORI sur des fenêtres glissantes de 12 semaines.

## Données d'entrée
- **pt_z028_input_bad_for_ar** : Table de sortie de Data_Prep.py, dataset avec encodage équipements + indicateur BAD
- **Df_Family_Ref.csv** : Référentiel familles de paramètres

## Pipeline principal

### 1. Préparation des données
- **Optimisation mémoire** : Conversion dtypes (downcast integers/floats, catégorisation strings)
- **Fenêtres glissantes** : Découpage en périodes de 12 semaines avec décalage hebdomadaire
- **Filtrage par famille** : Sélection des étapes pertinentes selon le référentiel

### 2. Analyse APRIORI par étape
Pour chaque (OPERATION, STEP) :
- **Encodage** : Colonnes `STEP_EQPT_[STEP]-[EQUIPMENT]` + `BAD`
- **Filtrage fréquence** : Garder équipements présents >5% du temps
- **APRIORI** : Recherche itemsets fréquents (min_support=0.1)
- **Règles d'association** : Calcul confidence, lift, support
- **Filtrage ciblé** : Règles avec conséquent = `BAD` uniquement

### 3. Scoring et ranking
- **Score composite** : `confidence × lift × support`
- **Ranking** : Classement des équipements par score décroissant
- **Fenêtre glissante** : Application sur toutes les semaines

### 4. Agrégation temporelle
- **Pivot** : Semaines en colonnes, scores en valeurs
- **Ranking final** : Rang de chaque équipement par semaine et par étape


## Output
Deux tables stocker dans :
***mds_prod_gold_experiment.datasciences_dev***

- **pt_z028_ar_bad_route_results** : Resultats des regles d'associations pour chaque famille a travers les semaines.
- **pt_z028_ar_good_route_results** : Resultats des regles d'associations pour chaque famille a travers les semaines.
- **Structure** : FAMILY | OPERATION | STEP | EQPT | week_0 | week_1 | ... | week_n
- **Valeurs** : Rang de l'équipement (1 = le plus problématique)

##  Résultat métier
**Identification des équipements les plus corrélés aux défauts par famille**, avec une évolution temporelle pour détecter les dérives.

→ **Prêt pour la validation** dans VALIDATION_AR_FAMILLIES.py

# Phase 3 : VALIDATION_AR_FAMILIES - Validation des Routes avec Scoring Pondéré

## Objectif
Valider statistiquement les routes identifiées comme problématiques ou bonnes dans la Phase 2 en utilisant un **système de scoring pondéré** qui privilégie la récence et la continuité des problèmes, puis comparer les performances réelles des wafers ayant suivi ces équipements.

##  Évolution majeure : Scoring pondéré temporel

### Principe du nouveau système
Au lieu d'une simple somme des occurrences, le système calcule un **score composite** basé sur :
- **50% Récence** : Les problèmes récents sont prioritaires (décroissance exponentielle)
- **30% Continuité** : Les séquences consécutives sont valorisées
- **20% Persistance** : Le nombre total d'apparitions compte aussi

### Justification métier
- **Problème résolu** : Un équipement problématique il y a 6 mois (tres certainement corrigé depuis) ne doit pas être prioritaire sur un equipement problématique récent
- **Problème émergent** : Un équipement devenant problématique récemment doit être détecté rapidement
- **Pattern sporadique vs continu** : Un problème continu est plus critique qu'un problème intermittent (Drift)

## Paramétrage dynamique

### Paramètres de scoring
```python
use_weighted_scoring = True     # Active le scoring pondéré
recency_weight = 0.5           # 50% d'importance pour la récence
continuity_weight = 0.3        # 30% d'importance pour la continuité
# 20% restant automatiquement alloué à la persistance
```

## Données d'entrée

### 1. Données équipements (Fenêtre 365 jours)
```python
df_EQPT_spark = spark.sql(f"""
    SELECT *
    FROM mds_prod_gold_experiment.datasciences_dev.pt_z028_input_{type_route.lower()}_for_ar
    WHERE T84_TEST_DATE BETWEEN DATE_SUB(CURRENT_DATE(), 365) AND CURRENT_DATE()
""")
```

### 2. Résultats AR les plus récents
```python
df_ar_result = spark.sql(f"""
    SELECT * 
    FROM mds_prod_gold_experiment.datasciences_dev.pt_z028_ar_{type_route.lower()}_route_results 
    WHERE Processed_Date_Job = ( 
        SELECT MAX(Processed_Date_Job) FROM ... 
    )
""")
```

## Pipeline de validation

### 1. Fonctions de scoring

#### `calculate_weighted_score()` - Calcul du score pondéré
```python
def calculate_weighted_score(row, week_columns, recency_weight=0.7, continuity_weight=0.2):
    """
    Composantes du score :
    1. Score de récence : Décroissance exponentielle (demi-vie = 10 semaines)
    2. Score de continuité : Bonus quadratique pour séquences consécutives
    3. Score de persistance : Nombre total d'apparitions normalisé
    4. Pénalité gaps : Réduction pour patterns sporadiques
    """
```

**Décroissance temporelle** :
- Semaine 40 (actuelle) : poids = 100%
- Semaine 30 (-10 sem) : poids ≈ 50%
- Semaine 20 (-20 sem) : poids ≈ 25%
- Semaine 10 (-30 sem) : poids ≈ 12.5%
- Semaine 1 (-39 sem) : poids ≈ 6%

#### `analyze_continuity_patterns()` - Analyse des patterns
```python
def analyze_continuity_patterns(row, week_columns):
    """
    Classification des patterns :
    - 'strong_continuous' : ≥6 semaines consécutives
    - 'moderate_continuous' : ≥3 semaines consécutives
    - 'sporadic' : "Grandes" interruptions (>4 semaines)
    - 'intermittent' : Apparitions espacées régulières
    """
```

### 2. Fonction `validate_routes`

#### Paramètres principaux
```python
df_validation = validate_routes(
    df_ar_result,           # Résultats AR de la phase 2
    df_EQPT_spark,         # Données équipements sur 365j
    type_route,            # "BAD" ou "GOOD"
    min_sum=6,             # Seuil minimum du score
    use_weighted_scoring=True,  # Active le scoring pondéré
    recency_weight=0.5,         # 50% pour la récence
    continuity_weight=0.3       # 30% pour la continuité
)
```

### 3. Processus de validation détaillé

#### Étape 1 : Calcul des scores pondérés
```python
# Pour chaque équipement, calcul du score composite
df_ar['WEIGHTED_SCORE'] = df_ar.apply(
    lambda row: calculate_weighted_score(row, week_columns, 0.5, 0.3), 
    axis=1
)

# Ajout des métriques de continuité
df_ar['continuity_pattern'] = ...      # Type de pattern détecté
df_ar['max_consecutive_weeks'] = ...   # Plus longue séquence
```

#### Étape 2 : Sélection des worst/best routes

**Nouveau critère** : Utilisation de `WEIGHTED_SCORE`

1. **Filtrage score minimum** : `df[df['WEIGHTED_SCORE'] >= min_sum]`
2. **Sélection par OP_STEP** : Équipement avec le score pondéré maximum
3. **Ajustement par taille de famille** :
   - Petite (≤8 étapes) : 100% conservé
   - Moyenne (≤15 étapes) : 60% conservé
   - Grande (>15 étapes) : 30% conservé (max 10)

#### Étape 3 : Affichage enrichi des routes

```
Traitement famille : CONTACT
Utilisation du scoring pondéré avec bonus de continuité
   Poids récence: 50%, continuité: 30%, persistance: 20%
   Score moyen (simple) : 8.43
   Score moyen (pondéré) : 24.67

Top 5 équipements par score pondéré :
  • O_STRIP|STEP_DRY -> SGAMA04
    Score: 45.2 (simple: 12)
    Pattern: strong_continuous, Max consécutif: 8 semaines
  • O_DEP_HK|DEP_HK -> TCENT11
    Score: 38.7 (simple: 7)
    Pattern: moderate_continuous, Max consécutif: 3 semaines
```

### 4. Tests statistiques

#### Sélection automatique du test selon la taille d'échantillon
- **n ≥ 30** : Test Z (proportions)
- **10 ≤ n < 30** : Test exact de Fisher
- **5 ≤ n < 10** : Test binomial exact
- **n < 5** : Pas de test (échantillon insuffisant)

#### Nouvelle colonne dans les résultats
- **SCORING_METHOD** : 'weighted' ou 'simple' pour traçabilité

## Output

### Table de sortie
**Nom** : `pt_z028_ar_{type_route}_validation_summary`  
**Localisation** : `mds_prod_gold_experiment.datasciences_dev`

### Structure DataFrame enrichie
| Colonne | Type | Description |
|:-------:|:----:|:------------|
| FAMILY | String | Famille de paramètres analysée |
| WORST_ROUTE_STEPS | Integer | Nombre d'étapes dans la route |
| MATCHES | Integer | Nombre d'équipements problématiques utilisés |
| WAFER_COUNT | Integer | Volume de wafers dans ce segment |
| BAD_RATE_PCT | Float | Taux de défaut observé (%) |
| BASELINE_PCT | Float | Taux moyen famille (%) |
| DELTA_PCT | Float | Impact relatif (+/-%) |
| P_VALUE | Float | Significativité statistique |
| **SCORING_METHOD** | String |'weighted' ou 'simple'|
| Processed_Date_Job | Date | Date d'exécution |

## Résultats et interprétation

### Exemple de sortie console
```
Traitement famille : NMOS-GO1-LVT
Utilisation du scoring pondéré (50% récence, 30% continuité, 20% persistance)

Worst route identifiée : 4 étapes retenues
  O_STRIP|STRIP_DRY -> SGAMA04 (Score pondéré: 42.3, Pattern: strong_continuous)
  O_DEP_HK|DEP_HK -> TCENT11 (Score pondéré: 38.1, Pattern: moderate_continuous)
  O_IMPL2|IMPL2_NWELL -> IVISM07 (Score pondéré: 31.5, Pattern: sporadic)
  O_STRIP_ADJ|STRIP_DRY -> SGAMA14 (Score pondéré: 28.9, Pattern: intermittent)

Baseline : 15.2%
  0/4 matches: 12.1% (-3.1%) [1234 wafers] p=0.023(Z)
  1/4 matches: 14.8% (-0.4%) [856 wafers] p=0.412(Z)
  2/4 matches: 18.7% (+3.5%) [423 wafers] p=0.008(Z)
  3/4 matches: 24.3% (+9.1%) [187 wafers] p<0.001(Fisher)
  4/4 matches: 31.2% (+16.0%) [52 wafers] p<0.001(Binomial)
```

### Comparaison weighted vs simple

| Aspect | simple | weighted |
|:------:|:-------------:|:---------------:|
| **Métrique principale** | Somme simple (WEEK_SUM) | Score pondéré (WEIGHTED_SCORE) |
| **Récence** | Non considérée | 70% du score (décroissance exp.) |
| **Continuité** | Non évaluée | 20% du score (bonus quadratique) |
| **Équipement ancien corrigé** | Reste prioritaire | Score fortement réduit |
| **Problème récent émergent** | Peut être ignoré | Détecté rapidement |
| **Pattern analysis** | Absent | Classification automatique |

### Critères de validation réussie

#### Validation quantitative
**Corrélation positive** : Plus de matches → taux défaut plus élevé  
**Significativité statistique** : p-value < 0.05 pour segments critiques  
**Volume suffisant** : >50 wafers pour robustesse  
**Pattern cohérent** : Équipements avec patterns 'continuous' priorisés  

#### Validation qualitative
**Récence confirmée** : Les problèmes identifiés sont actuels  
**Continuité vérifiée** : Pas de fausses alertes sur pics isolés  
**Actionnable** : Focus sur ce qui peut être corrigé maintenant  

## Impact

### 1. **Priorisation optimisée**
- Focus sur les problèmes **actuels** vs historiques
- Détection rapide des **dérives émergentes**
- Élimination du **bruit** (pics isolés non significatifs)

### 2. **ROI maintenance**
- Interventions sur équipements **vraiment problématiques**
- Évite les actions sur problèmes **déjà résolus**

### 3. **Traçabilité complète**
- Historique des scores et patterns
- Évolution temporelle des problèmes trackée

### 4. **Adaptabilité**
Les poids peuvent être ajustés selon le contexte :
```python
# Process très stable → Focus continuité
validate_routes(..., recency_weight=0.3, continuity_weight=0.6)

# Process en évolution → Focus récence
validate_routes(..., recency_weight=0.8, continuity_weight=0.1)
```

# Conclusion Global


