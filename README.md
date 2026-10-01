# Neural Speech

Projet de calcul embarqué réalisé en 3ème année d'école d'ingénieur : reconnaissance de mots à partir de leurs caractéristiques vocales, avec une chaîne de traitement sur Arduino Due et un réseau de neurones entraîné en Python.

## Objectif

Le projet met en œuvre une chaîne de reconnaissance vocale embarquée :

1. Acquisition du microphone par l'ADC de l'Arduino Due, déclenchée par un timer à 32 kHz.
2. Enregistrement d'une seconde et réduction de la fréquence à 8 kHz.
3. Découpage du signal en trames de 256 échantillons, avec un décalage de 128 échantillons entre trames.
4. Calcul de 13 coefficients MFCC par trame, soit 61 trames et 793 valeurs par enregistrement.
5. Classification sur l'Arduino Due avec les poids entraînés en Python.
6. Indication du résultat à l'aide de deux LED.

L'entraînement est réalisé sur ordinateur ; l'inférence prévue par le sketch de classification est exécutée sur la carte. Le sujet de projet interdit de déporter la reconnaissance elle-même sur l'ordinateur.

## Matériel et logiciels

- Arduino Due
- Microphone MAX9814 connecté à l'entrée analogique utilisée par le sketch (A0 / canal ADC 7)
- Bouton connecté à la broche 22
- Deux LED connectées aux broches 23 et 24
- Arduino IDE avec le support de carte Arduino SAM/Due
- Bibliothèques Arduino `arduinoMFCC` et `NeuralNetwork`
- Python 3 avec `numpy`, `scikit-learn` et `matplotlib` pour les scripts d'entraînement et de visualisation

Installer les dépendances Python avec :

```bash
python -m pip install numpy scikit-learn matplotlib
```

Les bibliothèques Arduino ne sont pas vendored dans ce dépôt : elles doivent être installées séparément dans l'environnement Arduino IDE.

## Organisation du dépôt

| Fichier ou dossier | Rôle |
| --- | --- |
| `et9/et9.ino` | Sketch de classification embarquée avec MFCC, réseau neuronal et sorties LED. |
| `et6/et6.ino`, `et6.ino` | Acquisition audio, filtrage, découpage en trames et calcul des MFCC. |
| `ia.py` | Chargement des jeux de données, entraînement du classifieur et export des poids vers `parametre.h`. |
| `dataset_training.txt` | Données utilisées pour l'entraînement. |
| `dataset_test.txt` | Données utilisées pour mesurer les résultats sur le jeu de test. |
| `parametre.h` | Poids et biais inclus par le sketch de classification. |
| `plot_mfcc-2.py`, `et6_preemphasis.py` | Scripts de visualisation et de validation du traitement du signal. |
| `validation_et6_mel/` | Sketch de validation du banc de filtres MEL. |
| `verificationet1/`, `et2/` à `et9/` | Sketches et étapes intermédiaires du projet. |
| `projet/` | Copie de travail et fichiers associés au projet. |

## Entraîner le réseau

Depuis la racine du dépôt, lancer :

```bash
python ia.py
```

Le script lit `dataset_training.txt` et `dataset_test.txt`, attend 100 exemples d'entraînement et 10 exemples de test, chacun représenté par 61 trames de 13 coefficients, puis entraîne un `MLPClassifier` de structure 793 → 64 → 32 → 2. Il affiche les prédictions et génère `parametre.h`, utilisé par le sketch Arduino.

Le script suppose que les 50 premiers exemples de chaque fichier correspondent à la classe 0 et les 50 suivants à la classe 1 pour l'entraînement (5 puis 5 pour le test). Respecter cet ordre lors du remplacement des données. Le fichier exporté est créé à la racine ; pour mettre à jour le sketch `et9`, reporter ce fichier dans `et9/parametre.h` avant le téléversement.

## Téléverser et tester

1. Installer les bibliothèques Arduino requises et sélectionner la carte **Arduino Due** ainsi que le port série correspondant.
2. Ouvrir `et9/et9.ino` dans l'IDE Arduino. Vérifier que `parametre.h` est accessible depuis le sketch et correspond au réseau entraîné.
3. Téléverser le programme et ouvrir le moniteur série à **115200 bauds**.
4. Faire un appui bref puis relâcher le bouton pour lancer l'enregistrement d'une seconde. Le sketch affiche les deux estimations et allume la LED correspondant à la classe choisie ; si les estimations sont proches, les deux LED restent éteintes.

Les broches et paramètres de traitement sont définis au début du sketch. Adapter le câblage si nécessaire, sans oublier la masse commune.

## Validation du traitement

Le dépôt contient également des éléments pour examiner différentes étapes du projet :

- `et6_preemphasis.py` compare les signaux avant et après préaccentuation à partir de `pre_before.txt` et `pre_after.txt`.
- `plot_mfcc-2.py` visualise des données MFCC.
- `verification_et5.py` illustre le recouvrement des trames.
- Les fichiers `testaudio.txt`, `signal1kHz.txt` et leurs variantes binaires sont des données de test ou de transfert.

Ces scripts de visualisation attendent leurs fichiers de données dans le répertoire courant. Les validations expérimentales (oscilloscope, spectre, temps de calcul et écoute dans Audacity) doivent être interprétées avec les mesures réellement effectuées sur la carte.

## État et limites

Le code fourni implémente une classification à **deux classes** (sorties annoncées dans le moniteur série comme « Jaune » et « Vert »), et non la reconnaissance de dix mots décrite comme objectif final dans le sujet. L'architecture entraînée dans `ia.py` est un perceptron multicouche (MLP), pas un CNN.

Quelques points à vérifier avant de réutiliser ou de présenter les résultats :

- Dans `ia.py`, le meilleur modèle parmi 50 essais est sélectionné selon sa précision sur le jeu de test. Ce jeu intervient donc dans le choix du modèle et ne constitue pas une mesure finale totalement indépendante.
- Dans les sketches, un échantillon sur quatre est conservé avant l'application du filtre RIF. Pour satisfaire rigoureusement l'exigence anti-repliement, vérifier la réponse du filtre et son placement avant le sous-échantillonnage.
- La normalisation des MFCC est faite séparément sur chaque enregistrement dans le sketch. Elle doit rester cohérente avec le prétraitement des données d'entraînement.
- La précision dépend du dataset et de l'environnement d'enregistrement. Le seuil vérifié par le script ne remplace pas une campagne de validation sur des voix et conditions variées.

Le sujet demande notamment une atténuation d'au moins 30 dB au-dessus de 4 kHz, un temps de filtrage inférieur à 31 µs, une MSE de test inférieure à 0,05 et moins de 5 % d'erreur sur au moins dix mots. Ces seuils sont des critères du sujet ; ce README ne les présente pas comme validés par les seuls fichiers du dépôt.