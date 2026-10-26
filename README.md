# Prévision de la consommation électrique française à J+1

Ce projet porte sur la prévision de la consommation d'électricité en France métropolitaine, heure par heure, un jour à l'avance. Je voulais savoir ce que la météo et le calendrier apportent à la prévision, comment les modèles se comparent à la prévision officielle de RTE, et ce qui se passe quand la consommation sort de son régime habituel, avec le Covid en 2020 et la sobriété énergétique en 2022 et 2023. Les données vont de 2012 à la fin de 2024.

## Question de recherche

La question centrale est la suivante. Dans quelle mesure les variables météorologiques et calendaires améliorent-elles la prévision de la consommation électrique française à J+1, par rapport à des modèles de référence et à la prévision officielle de RTE, et comment leurs performances évoluent-elles lors de périodes de rupture ?

Je l'ai découpée en cinq sous-questions.

1. Comment la sensibilité de la consommation à la température varie-t-elle selon la saison et l'heure de la journée ?
2. Quels modèles améliorent le plus la prévision à J+1 par rapport aux références naïves et à la prévision de RTE ?
3. Les erreurs de prévision changent-elles significativement en 2020 et en 2022 et 2023 ?
4. Quel gain prédictif apporte chaque famille de variables : calendrier, historique de consommation et météo ?
5. À quelles heures et pendant quelles saisons la consommation nette de la production éolienne et solaire est-elle la plus élevée ?

## Données

La consommation et les productions viennent de l'export éCO2mix de RTE (données consolidées et définitives), publié sur le portail Open Data Réseaux Énergies. Le fichier contient une ligne tous les quarts d'heure, soit 514 272 lignes et 37 colonnes pour la période de janvier 2012 à août 2026. Je n'ai gardé que les années 2012 à 2024. Les années 2025 et 2026 sont volontairement mises de côté.

La météo vient de l'archive Open-Meteo, qui est gratuite. Je télécharge la température à 2 mètres, le vent à 100 mètres et le rayonnement solaire pour huit grandes villes (Paris, Lyon, Marseille, Lille, Toulouse, Nantes, Strasbourg et Bordeaux), puis je fais une moyenne pondérée par un poids proche de leur population. Ce sont des données de réanalyse, c'est-à-dire la météo réellement observée et non des prévisions.

Les jours fériés viennent du paquet Python `holidays`. J'ai défini un pont comme un jour ouvré collé à un férié et à un week-end.

## Organisation du dépôt

```
projet_electricite.ipynb   notebook complet, commenté
README.md                  ce fichier
requirements.txt           paquets Python utilisés
figures/                   graphiques enregistrés par le notebook
resultats/                 tableaux enregistrés par le notebook (csv)
```

Les fichiers de données ne sont pas dans le dépôt. Le fichier RTE se télécharge sur le portail, la météo est récupérée automatiquement par le notebook.

## Méthode

### Nettoyage des données RTE

Le nettoyage m'a pris du temps à cause du changement d'heure. La colonne `Date et Heure` est en UTC donc continue, mais RTE a construit son fichier avec 96 créneaux locaux par jour, tous les jours. Au passage à l'heure d'été, l'heure de 02:00 à 02:45 n'existe pas, mais le fichier la remplit quand même avec 4 lignes qui reprennent des valeurs voisines. Au passage à l'heure d'hiver, la première occurrence de cette heure manque. Sur treize ans, cela donne 52 lignes fictives à supprimer (je les repère parce que l'heure locale reconstruite à partir de l'UTC ne correspond pas aux colonnes `Date` et `Heure`) et 13 heures à interpoler.

Il reste 455 852 quarts d'heure, que je passe au pas horaire par une moyenne, ce qui donne 113 976 heures. Les 13 heures manquantes d'octobre sont comblées par interpolation, soit 0,01 % des données. Enfin, 18 656 heures ont une production solaire légèrement négative la nuit, que je remets à zéro.

### Variables

J'ai construit trois familles de variables.

Le calendrier regroupe l'heure et le jour de l'année en sinus et cosinus, le jour de la semaine, le mois, les jours fériés, les ponts, les veilles et lendemains de férié, la période du 23 décembre au 2 janvier, et le fait que le jour d'un retard de consommation soit férié.

L'historique de consommation regroupe la consommation 48 heures et 168 heures plus tôt, et les moyennes sur 7 jours et sur 28 jours, toutes décalées de 48 heures. Une prévision à J+1 est émise la veille, donc je ne connais pas la consommation de la veille au moment de prévoir. C'est pourquoi aucun retard de 24 heures ne figure dans les modèles principaux.

La météo regroupe la température, sa moyenne lissée sur 24 heures et sur 48 heures, les degrés sous le seuil de chauffage de 15,5 °C trouvé à la question 1, les degrés au-dessus de 23 °C, la température minimale et maximale du jour, la variation sur 24 heures, le vent et le rayonnement.

### Protocole d'évaluation

L'entraînement va jusqu'en 2022, la validation porte sur 2023 et le test sur 2024. Tous les choix (jeu de variables, algorithme, réglages, correction du biais) sont faits sur la validation de 2023. Le modèle retenu est ensuite réentraîné sur 2012 à 2023, puis il prédit 2024.

Je mesure la MAE et le RMSE en MW, le biais moyen (prévision moins réel) et le MAPE en complément. Pour savoir si l'écart entre deux modèles est significatif, j'utilise un test de Diebold-Mariano avec des écarts types robustes à l'autocorrélation, parce que les erreurs d'heures voisines se ressemblent.

### Modèles comparés

La référence naïve reprend la consommation de la même heure une semaine plus tôt. J'ai ensuite une régression Ridge, deux modèles de boosting (XGBoost et HistGradientBoosting), SARIMAX et Prophet au pas journalier, et la prévision J-1 de RTE. Le modèle retenu sur la validation est un XGBoost de 600 arbres et de profondeur 6, avec un poids triple pour les années depuis 2018, car le niveau de consommation a baissé après 2020.

## Résultats

### Question 1. Sensibilité à la température

Sur les jours ouvrés, la consommation est à peu près constante au-dessus de 15,5 °C, puis elle augmente de 2 221 MW pour chaque degré de moins. Le niveau de base est de 46 128 MW et le R² vaut 0,90 sur 3 237 jours. Un froid de 5 °C sous zéro double à peu près la consommation par rapport au plateau.

Voici les résultats par période, avec le seuil fixé à 15,5 °C.

| Période | Pente (MW par °C) | Niveau de base (MW) | R² | Jours |
|---|---|---|---|---|
| 2012 à 2016 | 2 228 | 47 328 | 0,93 | 1 241 |
| 2017 à 2019 | 2 247 | 46 992 | 0,93 | 744 |
| 2020 à 2024 | 2 153 | 44 592 | 0,89 | 1 252 |

Entre 2012 à 2016 et 2020 à 2024, le niveau de base baisse de 2 736 MW, soit environ 5,8 %. Ce résultat est très net. La pente baisse de 74 MW par °C, soit 3,3 %. Je suis plus prudent sur ce point : le test avec des écarts types classiques concluait à une baisse significative, mais le Durbin-Watson du modèle est de 0,53, ce qui montre que les résidus sont très autocorrélés et que ces p-values étaient trop optimistes. Le notebook refait le test avec des écarts types robustes, et je ne retiens pas la baisse de pente comme établie tant que ce test ne la confirme pas.

L'effet de la climatisation en été est mal déterminé. La pente au-dessus du seuil varie de 396 à 817 MW par °C selon le seuil choisi entre 21 et 24 °C, avec un R² de 0,05 à 0,07, donc je ne cite pas de chiffre unique.

La variation selon l'heure et la saison est traitée dans les sections 1.3 et 1.4 du notebook, avec une régression par heure et par saison puis une courbe de sensibilité par splines. Les pentes sont enregistrées dans `resultats/q1_pente_heure_saison.csv` et `resultats/q1_pente_spline_par_saison.csv`, et illustrées par les figures `q1_pente_selon_heure.png`, `q1_pente_heure_saison.png` et `q1_courbe_sensibilite_saison.png`. Une association entre température et consommation ne suffit pas à démontrer un effet causal.

### Question 2. Comparaison des modèles

Voici le résultat sur le test 2024, au pas horaire (8 784 heures). Un biais positif signifie que la prévision est trop haute.

| Modèle | MAE (MW) | RMSE (MW) | MAPE | Biais (MW) |
|---|---|---|---|---|
| Modèle final (XGBoost) | 1 152 | 1 542 | 2,26 % | +530 |
| Modèle final corrigé du biais | 1 047 | 1 396 | 2,07 % | +2 |
| Prévision J-1 de RTE | 1 428 | 1 766 | 2,86 % | -1 146 |
| RTE corrigé du biais | 1 038 | 1 345 | 2,09 % | +3 |
| Ridge, mêmes variables | 1 942 | 2 459 | 4,04 % | +1 022 |
| Naïve à 168 heures | 3 246 | 4 857 | 6,1 % | -163 |

Par rapport à la prévision de RTE telle qu'elle est publiée, le modèle final réduit la MAE de 276 MW (19 %) et le RMSE de 13 %, avec un écart significatif (p inférieur à 0,001 pour la MAE). Il y a cependant plusieurs raisons de relativiser ce résultat.

La première est le biais de RTE. En 2024 sa prévision est en moyenne 1 146 MW trop basse. J'ai donc appliqué la même correction aux deux : je retire à chaque prévision l'erreur moyenne des 7 derniers jours connus (avec 48 heures de décalage), et la fenêtre de 7 jours a été choisie sur 2023. À biais égal, le modèle et RTE sont proches à l'heure (MAE de 1 047 et 1 038 MW).

Au pas journalier, sur 366 jours, le résultat est moins favorable.

| Modèle | MAE (MW) | RMSE (MW) | MAPE |
|---|---|---|---|
| RTE corrigé du biais | 660 | 866 | 1,30 % |
| Modèle final corrigé du biais | 886 | 1 176 | 1,74 % |
| Modèle final | 1 018 | 1 349 | 1,98 % |
| Prévision J-1 de RTE | 1 197 | 1 435 | 2,41 % |
| SARIMAX | 1 249 | 1 677 | 2,43 % |
| Ridge | 1 500 | 1 899 | 3,00 % |
| Prophet | 2 645 | 3 096 | 5,62 % |
| Naïve à 168 heures | 3 116 | 4 666 | 5,81 % |

Le modèle fait mieux que RTE publié en MAE (écart de 179 MW, p = 0,03) mais pas de façon significative en erreur quadratique (p = 0,37). À biais égal, RTE corrigé est nettement meilleur que le modèle corrigé (p inférieur à 0,001). SARIMAX est équivalent à RTE publié (p = 0,56) et le modèle final le bat de façon significative.

La deuxième raison de relativiser est la météo. Elle est observée et non prévue, ce qui avantage mon modèle face à RTE. Pour mesurer cet effet, j'ai ajouté à la température un bruit de 1,5 °C par jour (c'est une hypothèse, je n'ai pas mesuré l'erreur réelle des prévisions de température). Avec ce bruit, la MAE du modèle monte à 1 432 MW, ce qui revient au niveau de RTE (1 428 MW), et son RMSE reste plus élevé (1 934 contre 1 766 MW). L'avance sur RTE n'est donc pas robuste à une météo imparfaite.

La troisième raison apparaît à la question 3 : en année normale, RTE est meilleur que mon modèle.

### Question 3. Périodes de rupture

Pour chaque année de 2016 à 2024, un modèle est entraîné sur toutes les années précédentes puis prédit l'année. Le tableau donne l'indice de la MAE, avec 100 pour la moyenne des années 2016 à 2019.

| Année | Indice du modèle | Indice de RTE | Biais du modèle (MW) | Biais de RTE (MW) |
|---|---|---|---|---|
| 2020 | 170 | 99 | +1 372 | -115 |
| 2021 | 115 | 107 | +652 | -227 |
| 2022 | 132 | 120 | +685 | -575 |
| 2023 | 158 | 158 | +1 269 | -1 131 |
| 2024 | 123 | 168 | +530 | -1 146 |

Le Covid a surtout touché mon modèle : sa MAE augmente de 70 % en 2020 alors que celle de RTE ne bouge pas. En mars, avril et mai 2020, le MAPE mensuel du modèle est de 6,3 %, 7,7 % et 5,8 %, contre 2,8 %, 1,9 % et 1,7 % pour RTE. De même, d'octobre 2022 à février 2023, l'erreur du modèle est 1,6 à 2,2 fois celle de RTE. Mon explication est qu'un modèle à arbres ne sait pas prédire des niveaux qu'il n'a jamais vus dans des conditions comparables, mais je ne l'ai pas testée. Les retards de consommation limitent les dégâts : sans eux, l'indice du modèle est de 259 en 2020 et de 251 en 2024, contre 170 et 123 avec eux.

En année normale (de 2016 à 2019), la MAE de RTE est inférieure à celle de mon modèle de 3 à 20 % (rapport entre 1,03 et 1,20). Si le modèle fait mieux que RTE en 2024, c'est parce que RTE se dégrade à partir de 2022, avec un biais qui passe de -115 MW en 2020 à -1 146 MW en 2024. Je ne connais pas la cause de cette dérive de RTE.

Pour mesurer de combien la consommation s'est écartée de ce que la météo et le calendrier laissaient attendre, j'utilise un modèle sans historique entraîné sur 2012 à 2019, calibré par des années placebo (2016 à 2019) sans rupture. Le bruit de ces placebos est de 2,0 % par jour et de 1,15 % par mois.

| Période | Écart à l'attendu | Effet après retrait des placebos | En MW |
|---|---|---|---|
| Premier confinement (17 mars au 10 mai 2020) | -13,6 % | -12,6 % | -5 525 |
| Crise sanitaire, année 2020 | -5,0 % | -4,5 % | -2 296 |
| Année 2021 | -2,5 % | -2,1 % | -1 100 |
| Année 2022 | -4,6 % | -4,1 % | -2 121 |
| Hiver de sobriété (octobre 2022 à mars 2023) | -9,3 % | -9,2 % | -5 151 |
| Année 2023 | -7,5 % | -7,1 % | -3 508 |
| Année 2024 | -6,6 % | -6,1 % | -3 044 |

Le premier confinement et l'hiver 2022 à 2023 sortent très nettement de la bande de bruit des placebos. L'écart n'est jamais revenu à zéro après 2020, et environ 87 % de l'écart de 2023 persiste en 2024. En revanche, ces écarts ne prouvent pas que le Covid ou la sobriété en sont la cause. L'écart de 2022 à 2024 contient aussi la tendance de fond (efficacité énergétique, industrie, autoconsommation solaire), car l'écart moyen des années placebo glisse déjà vers le bas, de -0,11 % en 2016 à -1,08 % en 2019.

Le notebook compare aussi les périodes définies à l'avance (crise sanitaire, premier confinement, sobriété et période récente) aux mêmes dates des années de référence, avec des intervalles de confiance par bootstrap par blocs de 7 jours. Les rapports de MAE et de RMSE sont dans `resultats/q3_periodes_bootstrap.csv` et la figure `q3_degradation_par_periode.png`.

### Question 4. Gain apporté par chaque famille de variables

J'ai entraîné le même modèle sur différentes combinaisons de familles (entraînement jusqu'en 2023, test sur 2024).

| Variables utilisées | MAE (MW) | MAPE | Gain de MAE par rapport au calendrier seul |
|---|---|---|---|
| Calendrier seul | 4 214 | 8,16 % | |
| Calendrier et historique | 2 069 | 3,94 % | 50,9 % |
| Calendrier et météo | 2 604 | 5,06 % | 38,2 % |
| Calendrier, historique et météo | 1 154 | 2,26 % | 72,6 % |

Quand on retire la météo au modèle complet, la MAE passe de 1 154 à 2 069 MW (79 % de plus). Quand on retire l'historique de consommation, elle passe à 2 604 MW (126 % de plus). Le gain de la météo est donc très net, et celui de l'historique encore plus. Sans l'historique, le modèle ne connaît pas le niveau de consommation du moment et son biais atteint +2 476 MW.

L'importance par permutation du modèle final place en tête les deux retards de consommation (168 heures et 48 heures), puis le jour de la semaine, la température lissée sur 24 heures et la température maximale du jour. Les variables de fêtes, de férié et de pont comptent très peu. Les variables de météo sont corrélées entre elles, donc leur importance individuelle est sous-estimée.

Le notebook calcule aussi une décomposition de type Shapley sur les trois familles (gain moyen de chaque famille sur tous les ordres d'ajout) et les valeurs de SHAP du modèle complet. Les résultats sont dans `resultats/q4_marginal_shapley.csv` et `resultats/q4_importance_shap.csv`. SHAP explique le modèle et non le phénomène réel.

### Question 5. Demande résiduelle

La demande résiduelle est la consommation moins la production éolienne et solaire. En moyenne sur 2012 à 2024, elle est la plus élevée à 19 h (53 155 MW) et la plus basse à 4 h (41 129 MW). Elle a un second sommet à 9 h (51 891 MW) et un creux en milieu d'après-midi à 16 h (46 858 MW), donc une montée de 6 297 MW (13 %) entre 16 h et 19 h.

La forme de la journée change avec la saison.

| Saison | Heure du maximum | Maximum (MW) | Montée de 16 h à 19 h |
|---|---|---|---|
| Hiver | 19 h | 68 236 | +8 658 MW (14,5 %) |
| Printemps | 9 h | 50 835 | +6 255 MW (14,4 %) |
| Été | 23 h | 44 175 | +3 263 MW (8,3 %) |
| Automne | 19 h | 52 201 | +7 063 MW (15,6 %) |

L'hiver, de décembre à février, est la saison la plus sollicitée. La combinaison la plus élevée est l'hiver à 19 h (68 236 MW). Le sommet de 19 h n'est que 4 % au-dessus du plateau de 9 h, donc la tension s'étale sur toute la journée, de 8 h à 13 h puis de 18 h à 20 h. Par mois, janvier est le plus chargé (63 810 MW en moyenne) et août le moins (38 124 MW). Dans les quatre saisons, la demande résiduelle fait un saut d'environ 2 000 MW entre 22 h et 23 h. L'éolien et le solaire ne peuvent pas l'expliquer, donc il vient de la consommation, peut-être du déclenchement des chauffe-eau en heures creuses, mais je ne l'ai pas vérifié.

Cette moyenne sur treize ans mélange des régimes différents, puisque l'éolien et le solaire ont beaucoup augmenté. La section 5.6 du notebook compare la forme de la journée sur trois périodes (2012 à 2014, 2017 à 2019 et 2022 à 2024) et la montée du soir avec et sans éolien et solaire, mais les moyennes ne disent rien des journées extrêmes.

## Limites

La météo utilisée est la météo observée. Une vraie prévision à J+1 utiliserait une prévision météo, donc mes résultats sont plutôt une borne haute de ce qu'on peut obtenir en pratique. Le test avec bruit n'est qu'un test de sensibilité fondé sur une hypothèse.

Les choix principaux ont été faits sur la validation de 2023, mais j'ai consulté les résultats de 2024 plusieurs fois pendant le travail. Le test le plus propre serait de refaire l'évaluation sur 2025 et 2026, que j'ai mises de côté, avec la météo de ces années.

La correction du biais glissant a été choisie en regardant l'erreur de 2023. Elle réduit nettement l'erreur de RTE et du modèle, mais son effet dépend de la dérive de l'année. Je ne sais pas non plus à quelle heure RTE publie sa prévision J-1. Si elle utilise des mesures plus récentes que mes retards de 48 heures, la comparaison est un peu défavorable à mon modèle.

Les écarts mesurés pendant les périodes de rupture ne sont pas des effets causaux. Les intervalles de confiance du notebook ne couvrent que l'incertitude statistique de la moyenne, pas l'erreur du modèle de référence. Enfin, un seul jeu de test d'un an est court pour départager des modèles dont les erreurs ne diffèrent que de quelques pourcents.

## Reproduire les résultats

1. Télécharger l'export éCO2mix national consolidé et définitif sur le portail Open Data Réseaux Énergies, et le nommer `eco2mix-national-cons-def.csv`.
2. Créer un dossier `projet_electricite` sur Google Drive (ou en local) et y placer ce fichier.
3. Ouvrir `projet_electricite.ipynb` dans Google Colab et installer les paquets manquants avec `!pip install holidays shap`.
4. Exécuter les cellules dans l'ordre. Le premier téléchargement de la météo prend quelques minutes, ensuite elle est conservée en cache.

Avec `RAPIDE = True` dans la première cellule, le notebook tourne avec des modèles plus petits pour vérifier que tout fonctionne. Pour les vrais résultats, il faut garder `RAPIDE = False` et prévoir plusieurs minutes de calcul.

## Pistes pour aller plus loin

Refaire le test sur 2025 et 2026, utiliser des prévisions météo archivées pour avoir une vraie prévision à J+1, étendre la prévision à J+7, ajouter les vacances scolaires dans les variables, et prévoir directement la production éolienne et solaire à partir du vent et du rayonnement.

## Sources

Les données de consommation et de production viennent de RTE (éCO2mix, portail Open Data Réseaux Énergies). La météo vient de l'API d'archive d'Open-Meteo.
