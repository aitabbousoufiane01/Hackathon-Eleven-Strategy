# Hackathon-Eleven-Strategy
Un hackathon organisé par ElevenStrategy à l'Ecole des Ponts et chaussées. Le but est d'implementer un système de recommendation de produit pour les clients, à partir d'un ensemble de données, à savoir: transactions, produits, clients, stores, stocks. Pour chaque client on recommande 5 identifiants de produits différents. 

Le dépôt contient le système  recommandation de produits que j'ai  développé. L'algorithme propose une approche en deux étapes (Génération de candidats + Re-ranking) optimisée pour gérer les historiques d'achats clairsemés et le problème du "Cold Start".

## Données utilisées:
* `transactions.csv` : Historique des achats.
* `clients.csv` : Données démographiques des utilisateurs.
* `products.csv` : Catalogue et hiérarchie des produits.
* `stocks.csv` : Disponibilité locale.
* `test_clients_validation.csv` : Liste des clients cibles.

##  Architecture du Modèle
Le système repose sur un pipeline de Machine Learning structuré en trois grandes phases :

### 1. Data Processing & Nettoyage
* **Filtrage des anomalies :** Exclusion des comptes "grossistes" (achats massifs de produits distincts) et des valeurs aberrantes pour se concentrer sur le comportement retail authentique (Un seul client ayant un nombre non réaliste d'achats)
* **Profilage Client :** Calcul de la marque et de la catégorie préférées pour chaque client.
* **Encodage :** Encodage des variables catégorielles (Pays, Genre, Segment, Catégorie,Marque,  Catégorie préférée, Marque préférée)

### 2. Génération de Candidats & Feature Engineering
Pour éviter de calculer des prédictions sur l'intégralité du catalogue produit, ce qui serait impossible avec la RAM de collab : 2 billions de lignes en totale, le système génère une *short-list* ciblée plafonnée à `MAX_CANDIDATES` (100 produits) pour chaque client. Cette sélection repose sur la combinaison de plusieurs signaux heuristiques.

Le cœur du mécanisme de scoring s'appuie sur une **décroissance temporelle (Time-Decay)**. Chaque interaction passée est pondérée par son ancienneté via une fonction de demi-vie (Half-life), donnant mathématiquement plus d'importance aux achats récents. 
Le poids de récence w(Δt) d'un achat effectué il y a Δt jours est défini par :

> w(Δt) = 0.5 ^ (Δt / H)
> *(où H est la constante HALF_LIFE_DAYS fixée à 30 jours)*

Pour un client donné, chaque produit candidat *p* reçoit quatre scores distincts basés sur l'historique d'achat *q* :

* **A. Score de Récence (Rachat) :** Favorise les produits que le client a l'habitude de consommer récemment.
  `S_recence(p) = W_repeat * Σ w(Δt_q)`

* **B. Score de Co-occurrence Conditionnelle :** Évalue la probabilité qu'un candidat *p* soit acheté sachant que le client a acheté *q* dans le passé. Le score est obtenue en sommant les probabilités sur l'ensemble de l'historique du client, pondérés par les scores de récences et un poids choisi pour ce score.
  `S_cooc(p) = Σ [ W_cooc * w(Δt_q) * P(p|q) ]`

* **C. Score de Transition Séquentielle :** Capte l'ordre d'achat. Si *p* est fréquemment acheté juste après *q* dans les données globales. Le calcul est similaire au score de co-occurence.
  `S_trans_hist(p) = Σ [ W_trans * w(Δt_q) * P_trans(p|q) ]`

* **D. Score de Transition (Dernier Achat) :** Un boost spécifique appliqué uniquement à partir du tout dernier produit acheté, indépendamment du temps écoulé.
  `S_trans_last(p) = P_trans(p|q_last)`

L'ensemble de ces signaux sont sommés pour obtenir un score de base. Si le candidat est disponible dans le stock local, un multiplicateur est appliqué :
> S_final(p) = ( S_recence + S_cooc + S_trans_hist + S_trans_last ) * B_stock

Les 100 meilleurs candidats sont conservés et enrichis de leurs features catégorielles (vélocité à 30 jours, marque préférée) avant d'être envoyés au modèle.

### 3. Stratégie de Repli (Cold Start & Fallback)
Dans le cas des nouveaux clients ou de ceux ayant un historique trop pauvre (moins de 5 candidats générés), le système déploie une cascade de popularité calculée sur une fenêtre glissante stricte de 90 jours :
1. **Niveau 1 (Hyper-ciblé) :** Top 50 par profil complet (Pays × Genre × Segment).
2. **Niveau 2 (Ciblé) :** Top 50 par Genre.
3. **Niveau 3 (Régional) :** Top 50 par Pays.
4. **Niveau 4 (Global) :** Top 50 global du catalogue.

### 4. Re-ranking avec LightGBM (LambdaMART)
Les candidats sont classés à l'aide d'un modèle **LightGBM Ranker** optimisé pour la métrique **MAP@5** (Mean Average Precision).
* Le modèle n'apprend pas à prédire une probabilité d'achat absolue, mais à **ordonner** la pertinence des candidats au sein du groupe (panier) d'un même client.
* Entraînement basé sur une stratégie *Leave-One-Out* (la dernière visite du client est cachée et sert de cible `Label=1`).

## Résultats 
Les prédictions générées font un score de 0.2767 sur le test de validation (Le premier score du hackathon est 0.284), et un score de 0.22 sur le test finale  (Premier score est 0.25)

## Prérequis et Données

* `transactions.csv` : Historique des achats.
* `clients.csv` : Données démographiques des utilisateurs.
* `products.csv` : Catalogue et hiérarchie des produits.
* `stocks.csv` : Disponibilité locale.
* `test_clients_validation.csv` : Liste des clients cibles.
