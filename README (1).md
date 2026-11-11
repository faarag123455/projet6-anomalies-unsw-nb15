# Détection d'anomalies sans étiquettes dans le trafic réseau (UNSW-NB15)

**Projet 6 : détecteurs entraînés uniquement sur du trafic normal, évalués avec les étiquettes seulement à la fin.**

Binôme : Farah Chebbi, Firas Ben Hmida

---

## 1. Objectif

Dans un réseau réel, on dispose de beaucoup de trafic normal et de très peu d'attaques étiquetées. Le projet apprend donc **ce qu'est un trafic normal**, puis signale tout ce qui s'en écarte. Il répond à trois questions :

1. Cette approche détecte-t-elle des attaques, et avec combien de fausses alertes ?
2. Les résultats tiennent-ils sur un autre fichier du jeu, et avec un seuil fixé sans étiquettes de test ?
3. Que se passe-t-il si l'entraînement est contaminé par quelques attaques ?

## 2. Contenu du dépôt

| Fichier | Rôle |
|---|---|
| `projet6_anomalies.ipynb` | Code, résultats, analyses et conclusions (s'exécute de haut en bas) |
| `requirements.txt` | Bibliothèques Python utilisées |
| `README.md` | Ce fichier |

Les données ne sont **pas** versionnées (voir section 3).

## 3. Données

- **Jeu** : UNSW-NB15, version partitionnée (Moustafa et Slay, *UNSW-NB15 : a comprehensive data set for network intrusion detection systems*, MilCIS 2015).
- **Fichiers utilisés** : `UNSW_NB15_training-set.csv` (82 332 flux) et `UNSW_NB15_testing-set.csv` (175 341 flux), 42 variables par flux, colonne `label` (0 = normal, 1 = attaque) et colonne `attack_cat` (famille d'attaque : Generic, Exploits, Fuzzers, DoS, Reconnaissance, Analysis, Backdoor, Shellcode, Worms).
- **Source** : Kaggle, `mrwellsdavid/unsw-nb15`, fichier `archive.zip`. Les quatre gros fichiers bruts du zip (`UNSW-NB15_1.csv` à `_4.csv`) ne sont pas utilisés.

### Après nettoyage

| Partie | Flux | Part d'attaques |
|---|---|---|
| Entraînement (trafic normal uniquement) | 23 944 | 0 % |
| Validation | 30 008 | 65,8 % |
| Test (fichier testing) | 99 869 | 48,4 % |

Le nettoyage supprime la colonne `id`, les doublons (28 380 dans le training, soit 34 %, et 74 072 dans le testing, soit 42 %) et les 1 400 flux du test identiques à un flux du training.

## 4. Exécution

### Dans Google Colab (recommandé)

1. Placer `archive.zip` dans `projet6/data/` de Google Drive.
2. Déposer `projet6_anomalies.ipynb` dans `projet6/` et l'ouvrir avec Google Colaboratory.
3. Exécuter les cellules **dans l'ordre**. Le notebook monte Drive, extrait les deux fichiers utiles et écrit ses résultats dans `projet6/data/UNSW/`.

**Durée** : environ 40 à 60 minutes pour les sections 1 à 10, plus 10 à 15 minutes pour la section 6.2. Les cellules les plus longues sont le réglage de l'autoencodeur (section 4), la contamination (section 6) et la borne supérieure supervisée (section 7).

**Conseils**
- Garder l'onglet ouvert pendant les longues cellules : une session Colab coupée perd toutes ses variables.
- Exécuter les sections lourdes une par une, et télécharger le `.ipynb` après chacune (*Fichier > Télécharger > Télécharger .ipynb*).
- Un seul onglet et une seule personne à la fois par notebook : sinon Colab affiche « Échec de l'enregistrement automatique : ce fichier a été modifié à distance ou dans un autre onglet ».
- Pour accélérer : `INCLURE_OCSVM = False` (section 6) et `INCLURE_OCSVM_R = False` (section 6.2, valeur par défaut).

### Hors Colab

Définir la variable d'environnement `UNSW_DATA_DIR` vers le dossier contenant `archive.zip`, puis exécuter le notebook avec Jupyter.

### Installation locale

```
pip install -r requirements.txt
```

### Fichiers produits (dans `projet6/data/UNSW/`)

`prepared.npz`, `params_final.json`, `scores_test.npz`, `resultats_test.csv`, `contamination_resultats.csv`, `contamination_elagage_resultats.csv`, `resultats_combinaison.csv`.

## 5. Méthode

**Règles d'évaluation** : le test n'est utilisé qu'après figement des hyperparamètres et des seuils ; les seuils sont toujours fixés sur le trafic **normal de validation**, jamais avec les étiquettes du test ; toute performance est comparée au hasard (PR-AUC égale à la part d'attaques) ; graine fixée (`SEED = 42`).

| Section | Contenu |
|---|---|
| 1 | Configuration, extraction des données, versions des bibliothèques |
| 2 | Nettoyage, découpage sans fuite, prétraitement (log signé, standardisation, one-hot, appris sur l'entraînement seul) |
| 3 | Baseline : score z et distance de Mahalanobis |
| 4 | Détecteurs et réglage sur la validation : Isolation Forest, One-Class SVM, DBSCAN, kNN, autoencodeur |
| 5 | Évaluation unique sur le test : résultats, rappel par famille, charge d'alertes |
| 5.3 | Métriques opérationnelles : rappel à faible taux de faux positifs, Precision@k réaliste (1 % d'attaques), alertes à traiter |
| 5.4 | Intervalles de confiance et comparaisons appariées (bootstrap, 500 tirages) |
| 5.5 | Calibration du seuil sur un petit échantillon normal du fichier testing |
| 6 | Robustesse : contamination de l'entraînement par 1 % et 5 % d'attaques |
| 6.2 | Entraînement robuste : élagage des 10 % de flux les plus suspects |
| 7 | Borne supérieure supervisée (forêt aléatoire, XGBoost) |
| 8 | Analyse critique des familles difficiles |
| 9 | Combinaison : détecteur non supervisé puis classifieur supervisé |
| 10 | Conclusions et limites |
| Annexes A à E | Analyses supplémentaires, non requises (voir section 7 de ce README) |

**Détecteurs.** Tous sont entraînés sur les mêmes 23 944 flux normaux.
- *Isolation Forest* : isole chaque flux par des découpages aléatoires.
- *One-Class SVM* (noyau RBF) : frontière autour du trafic normal.
- *DBSCAN* : DBSCAN n'ayant pas de `predict`, le score d'un flux est sa distance au point central le plus proche.
- *kNN* (témoin) : distance au plus proche voisin normal.
- *Autoencodeur* (scikit-learn, réseau 58-32-b-32-58) : score égal à l'erreur de reconstruction.

## 6. Résultats principaux

PR-AUC sur le test (hasard : 0,484).

| Détecteur | PR-AUC test | PR-AUC avec 5 % de contamination | Rappel à 5 % de FP visé (FP observé) |
|---|---|---|---|
| Mahalanobis (baseline) | 0,680 | 0,629 | 0,232 (6,4 %) |
| Isolation Forest | 0,701 | 0,597 | 0,343 (7,2 %) |
| One-Class SVM | 0,732 | 0,552 | 0,499 (16,5 %) |
| DBSCAN | 0,737 | non répété | 0,479 (14,8 %) |
| kNN | 0,739 | 0,560 | 0,480 (14,7 %) |
| **Autoencodeur** | **0,757** | **0,646** | 0,485 (8,0 %) |
| Supervisé (XGBoost, borne supérieure) | 0,968 | - | 0,777 (3,9 %) |

**Constats**
- **Les performances de validation ne se transfèrent pas au test** (PR-AUC 0,94 contre 0,74 pour le kNN) : les deux fichiers du jeu ne suivent pas la même distribution.
- **L'autoencodeur est meilleur que chacun des cinq autres détecteurs** (bootstrap, 500 tirages : PR-AUC 0,757 [0,753 ; 0,760] ; différence avec le kNN : +0,018 [+0,015 ; +0,020]).
- **DBSCAN équivaut au kNN (k = 1)** : mêmes scores et mêmes rappels par famille.
- **Seuil** : un seuil fixé sur la validation se transfère mal (pour 5 % visés, le kNN observe 14,7 %). Un seuil calibré sur 500 flux normaux du fichier testing ramène l'écart moyen entre le FP observé et la cible de 0,063 à 0,001. Hypothèse : un échantillon fiable de trafic normal du réseau cible est disponible.
- **Charge d'alertes** : si les attaques représentent 1 % du trafic, au moins 94 % des alertes sont fausses, quel que soit le détecteur.
- **Contamination** : 5 % d'attaques dans l'entraînement dégradent fortement tous les détecteurs et divisent leur rappel (sauf Mahalanobis, le moins touché).
- **Élagage de 10 %** : aide le kNN (+0,033 de PR-AUC à 5 %) mais n'est pas une solution générale (il dégrade Isolation Forest et Mahalanobis).
- **Familles** : Generic, Exploits, DoS et Worms sont détectables sans étiquettes ; Reconnaissance, Shellcode et Fuzzers sont différentes du trafic normal mais pas atypiques (AUC non supervisée de 0,37 à 0,61, contre 0,92 à 0,997 en supervisé) ; Analysis relève surtout de la dérive entre fichiers (AUC du kNN de 0,992 en validation contre 0,606 en test).
- **Combinaison** One-Class SVM + XGBoost : rappel 0,597 pour 5,9 % de FP, contre 0,499 pour 16,5 % pour le détecteur seul, soit 61 % d'alertes en moins à 1 % de prévalence.

Détails, tableaux complets et résultats des annexes : section 10 du notebook.

## 7. Annexes (analyses supplémentaires)

| Annexe | Contenu | Résultat |
|---|---|---|
| A | Sélection de variables | Aucune amélioration constante ; la sélection manuelle dégrade la détection |
| B | Ensemble de détecteurs (vote, fusion de scores) | N'améliore pas le meilleur détecteur seul |
| C | Explicabilité (décompositions exactes, occlusion, SHAP) | Décompositions exactes pour Mahalanobis et kNN ; test de fidélité concluant seulement pour SHAP sur Isolation Forest |
| D | Score de risque de 0 à 100 | Met en tête les attaques faciles (Generic) ; non recommandé tel quel |
| E | Évaluation sur 500 attaques synthétiques complémentaires | Optionnelle : exige `UNSW_NB15_complementary_synthetic.csv` dans `projet6/data/` |

Les annexes relisent le test à titre d'analyse et ne modifient aucun réglage ni aucun résultat des sections 5 à 9. L'annexe E n'est **pas** fusionnée avec le jeu principal : le fichier ne contient que des attaques synthétiques, sans flux normaux, et ses variables sont incohérentes entre elles.

## 8. Limites

**Liées au jeu de données**
- Jeu ancien (2015), généré en laboratoire : il ne représente pas nécessairement les menaces et le trafic actuels.
- Menaces non représentées : phishing, rançongiciels modernes, vol d'identifiants, mouvements latéraux.
- Familles rares : Worms compte 44 flux dans le training et 130 dans le testing d'origine (43 en validation et 120 en test après nettoyage).
- Jeu très riche en attaques (48 % du test) : non représentatif d'un réseau réel.
- Dérive entre les deux fichiers.

**Liées à la méthode**
- Hyperparamètres choisis sur la validation avec ses étiquettes d'attaque ; le réglage de l'autoencodeur est au bord de la grille testée (goulot 16).
- Les chiffres « à 1 % d'attaques » supposent que rappel et taux de faux positifs restent valables à une autre prévalence.
- Les intervalles de confiance supposent des flux indépendants, sans mesurer la dérive entre fichiers ni la variabilité des graines : ils sont probablement trop étroits.
- La contamination utilise des attaques tirées au hasard et 3 graines seulement.
- Plusieurs explications restent des hypothèses non testées (sensibilité de Generic à la contamination, rôle de `sttl`).

**Portée** : les conclusions valent pour UNSW-NB15 et des détecteurs sur statistiques de flux ; elles ne se généralisent pas telles quelles à un réseau actuel. Une validation sur un jeu plus récent (par exemple CIC-IDS2017) est une piste.

## 9. Reproductibilité

- Graine fixée (`SEED = 42`) pour les tirages et les modèles ; versions des bibliothèques affichées à la section 1.
- Les hyperparamètres gelés sont enregistrés dans `params_final.json`.
- L'autoencodeur est réglé à chaque exécution (aucune valeur pré-calculée) ; avec les graines fixées, les valeurs retenues lors de notre exécution étaient : goulot 16, `alpha` = 1e-4.

## 10. Référence

N. Moustafa et J. Slay, « UNSW-NB15 : a comprehensive data set for network intrusion detection systems (UNSW-NB15 network data set) », *Military Communications and Information Systems Conference (MilCIS)*, 2015.
