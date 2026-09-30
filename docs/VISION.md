# Vision : compter juste

Détecter des personnes est résolu par une librairie. **Compter juste** ne l'est pas :
tout se joue dans les cas particuliers. Chaque décision ci-dessous a été tranchée par
la mesure sur des images réelles du magasin, pas par intuition.

## Comment ça compte

Un **worker Python par caméra**, selon le rôle de la caméra :

- **Entrée / caisse** → franchissement d'une **ligne** tracée sur l'image. La
  personne doit être vue des deux côtés pour qu'un passage soit validé.
- **Rayons** → présence dans des **zones**, polygones tracés au sol.

Détection YOLO11, suivi ByteTrack pour garder la même personne d'une image à l'autre.
Le suivi est indispensable : sans lui, on compterait la même personne à chaque image.

## L'appartenance à une zone se décide par les pieds

Une zone est un **polygone au sol**, pas un rectangle sur l'image. En perspective,
le sol devant un rayon est un trapèze. Et c'est la **position des pieds** (bas de la
silhouette) qui décide de l'appartenance, pas le corps entier : une personne debout
dans l'allée a la tête « dans » le rayon du fond sans y être.

## Exclure le personnel

Un employé traverse le champ de la caméra vingt fois par jour, et se tient dans les
rayons. Le compter fausse tout.

**Écarté : distinguer une tenue.** Certains magasins habillent leur personnel en
blanc — une couleur très courante chez les clients. Détecter « haut blanc + pantalon
foncé » retirerait des clients du comptage, ce qui est **pire** qu'un employé compté.

**Écarté : une entrée réservée au personnel.** Les employés restent visibles dans les
rayons, où ils faussent les visites et la carte de densité.

**Testé puis abandonné : le badge sur cordon.** Mesures sur 70 s de vidéo et 208
silhouettes à une entrée : la carte n'est visible que **de face** ; de dos il ne reste
que le cordon, large de **2 à 3 px**. La carte est blanche (inutilisable), et le bleu
du cordon est la couleur des jeans.

**Retenu : le brassard fluo.** Mesures sur 722 images (entrée et rayon) :

| Critère | Résultat |
|---|---|
| Taille de la tache | 197 à 258 px (médiane) |
| Visibilité | de face, de dos, de profil, penché |
| Détection | ~71 % des images d'une personne portant le brassard |
| Fausses détections | **0** sur 203 images d'une personne sans brassard |

Trois points d'implémentation qui ont demandé du travail :

1. La tache est cherchée **dans la bande des bras**, sur toute la largeur de la
   silhouette. La règle héritée du gilet (« 18 % du torse ») ne verrait jamais un
   brassard, qui représente 2 à 4 % de la silhouette.
2. **Le décor de la même couleur est appris puis ignoré** : un objet orange derrière
   une personne entrait dans son cadre de détection. L'apprentissage se fait
   **hors des personnes** — sinon un employé immobile verrait son propre brassard
   appris comme décor.
3. La décision porte sur **le passage entier**, pas sur une image : au moins 2 images
   et au moins 20 % des images du passage.

**Limite connue, documentée, non corrigée** : la décision est prise à la fin de chaque
visite de zone. Une visite très courte **au tout début** du parcours peut être comptée
avant que le brassard ait été vu deux fois (observé : une visite de 2 s). Correctif
identifié : différer la décision jusqu'à la fin du suivi de la personne.

**Consignes magasin qui en découlent** : un brassard à **chaque bras** (le corps en
cache un), porté haut sous l'épaule, couleur fluo — et caméras **en mode jour**, car
en noir et blanc toute détection de couleur cesse de fonctionner.

## Exclure les enfants, sans se tromper d'adulte

**Symptôme observé** : 7 passages sur 22 disparaissaient du comptage à une entrée.

**Cause** : le seuil de taille partait d'une **valeur par défaut identique pour toutes
les caméras** (0,55 de la hauteur d'image). Or la fraction d'image occupée par un
adulte dépend de la hauteur de pose, de la distance et de l'angle. Mesuré :

| Caméra | Fraction d'image occupée par un adulte |
|---|---|
| Entrée 1 | 0,55 – 0,66 |
| Entrée 2 | 0,45 – 0,49 |

Le seuil par défaut classait donc comme « enfants » des adultes de la seconde entrée.

**Correctif** : calibration **par caméra** — on renseigne la fraction d'image occupée
par un adulte de taille connue *sur cette caméra*, et le seuil enfant s'en déduit.
Puis **calibration automatique** : l'API déduit cette valeur des passages réellement
mesurés (médiane de la moitié haute, 30 passages minimum) et le dashboard la
**suggère** au technicien, qui valide.

**Leçon générale** : une valeur par défaut « raisonnable » partagée entre des caméras
différentes est une source d'erreur silencieuse — elle ne fait pas planter, elle fait
compter faux.

## Où tracer la ligne

Autre cause de passages ratés, plus banale : une ligne tracée **tout en bas de
l'image**. Une personne qui entre apparaît déjà dessus ou au-delà, donc le compteur ne
la voit jamais « de l'autre côté » et ne valide rien. La ligne doit être placée assez
haut pour voir la personne **entière des deux côtés**.

Les fausses pistes explorées avant de trouver : cadence d'images trop basse, modèle
trop léger. Ni l'une ni l'autre.

## Cadence : au-delà de 6 images/s, on ne gagne plus rien

Une personne met environ **1 seconde** à franchir la ligne. Monter la cadence au-delà
de 5-6 images par seconde ne rattrape presque aucun passage supplémentaire — et si la
machine est saturée, les images demandées ne sont de toute façon pas réellement
analysées. Mesurer *ce qui est traité*, pas *ce qui est demandé*.

## Threads : la librairie ignore l'environnement

**Symptôme** : ~1900 % de CPU (19 cœurs sur 20 occupés) et 5,3 images/s pour 6
demandées.

**Fausse piste** : `OMP_NUM_THREADS`, pourtant correctement pris en compte au
démarrage.

**Cause réelle** : la librairie de détection **réécrit `torch.set_num_threads` au
premier appel d'inférence** (8 threads), quel que soit l'environnement. Avec un worker
par caméra : 24 threads de calcul pour 20 cœurs, donc de la contention pure.

**Correctif** : chaque worker **repose le plafond après un échauffement**, et OpenCV
comme le décodeur FFmpeg sont bornés aussi. Résultat : **~300 % de CPU**, consignes
tenues.

**Combien de threads par caméra** — mesuré, puis testé en charge réelle (3 caméras) :

| Threads par compteur | Coût par image | CPU total (charge réelle) | Cadence |
|---|---|---|---|
| 1 | 0,16 cœur·s | — | — |
| 2 | 0,22 cœur·s | **~330 %** | consignes tenues |
| 3 | 0,29 cœur·s | ~740 % | ~5 img/s (sous la consigne) |

**Décision** : dimensionnement automatique `(cœurs − 2) ÷ nombre de caméras`, borné à
[1, 2], un réglage explicite restant prioritaire.

**Leçon** : le test sur une scène vide masquait complètement l'écart. Il faut mesurer
avec des personnes dans le champ.

## Lire les caméras : RTSP plutôt que des instantanés

Première installation derrière un **NVR**, chaque caméra exposée sur un port. La
saturation des sessions du NVR provoquait des `Read timeout` et des caméras muettes.

Passage aux caméras en **IP directe** via un switch PoE… et le problème s'est déplacé :
les instantanés JPEG saturaient **les caméras elles-mêmes** (petits modèles), mémoire
à 96 % et timeouts. Une petite caméra ne tient pas plusieurs JPEG par seconde.

**Correctif** : lire le **flux RTSP** — une connexion continue, les images non
nécessaires étant sautées sans décodage.

| | CPU de la caméra |
|---|---|
| Instantanés JPEG | ~45 % |
| Flux RTSP | **10 %** |

Plus aucun timeout. Deux pièges rencontrés au passage :

- **Le NVR reprend les caméras.** Le switch avait été branché sur un **port PoE** du
  NVR, dont le réseau interne est en plug-and-play : le NVR a réadressé les trois
  caméras, qui ont disparu du réseau. Brancher sur le port **LAN**, ajouter les
  caméras à la main, désactiver le plug-and-play.
- **Une caméra refusait le RTSP en 401** malgré le bon mot de passe. Fausses pistes :
  mot de passe perdu, empreinte corrompue (le reposer n'a rien changé, créer un
  utilisateur neuf non plus). **Cause réelle** : ce firmware annonçait **plusieurs
  méthodes Digest à la fois** (`MD5/SHA256`) et le client ne gère que MD5. Le mode
  Basic fonctionnait — preuve que le mot de passe était bon. À savoir : trop d'échecs
  **bloquent l'IP 30 minutes**, et pendant le blocage le bon mot de passe est refusé
  exactement comme un mauvais. Ne pas multiplier les essais.

## Ce que le système ne fait pas, volontairement

**Aucune reconnaissance d'âge, de genre ou de visage.** Ce serait un traitement
biométrique, soumis à la loi 09-08 et à la CNDP au Maroc. Le système ne compte que des
silhouettes, et aucune image ne quitte le magasin. La contrainte est devenue un
argument commercial.

**Aucun suivi individuel entre caméras.** La durée moyenne de visite est calculée par
la **loi de Little** — temps total passé ÷ nombre d'entrées — ce qui donne une moyenne
fiable sans jamais rattacher un parcours à une personne. Limite documentée : la mesure
est sensible aux sorties ratées, qui laissent des « présents fantômes ».
