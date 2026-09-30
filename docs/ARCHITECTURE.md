# Architecture

## Vue d'ensemble

Monorepo npm workspaces, quatre briques :

| Brique | Rôle |
|---|---|
| **API** | Express + TypeScript + Prisma + PostgreSQL. Multi-tenant : tout est cloisonné par entreprise. REST + Socket.IO + worker d'agrégation. |
| **Dashboard** | React 18 + Vite. CSS maison, thèmes clair/sombre, aucune librairie de composants. |
| **Agent** | Le boîtier posé en magasin : Node/TypeScript pour la file et la synchronisation, Python pour la vision. En Docker. |
| **Shared** | Schémas Zod et types partagés entre les trois — un seul contrat de données. |

## Le chemin d'un visiteur

1. Une caméra IP diffuse un flux **RTSP** sur le réseau local du magasin.
2. Un **worker Python par caméra** lit le flux, détecte les silhouettes (YOLO11),
   les suit d'une image à l'autre (ByteTrack) et décide : franchissement d'une ligne
   (entrée/sortie) ou présence dans une zone (rayon).
3. Le passage est écrit dans une **file SQLite** locale, avec son horodatage.
4. Le processus Node envoie les passages par lots en **HTTPS sortant**, avec reprise
   et dédoublonnage par identifiant.
5. L'API valide (Zod), écrit, et un **worker** agrège en statistiques horaires et
   journalières — c'est ce que lit le dashboard, jamais les événements bruts.
6. Le dashboard reçoit les compteurs temps réel par **Socket.IO**.

## Pourquoi le calcul en magasin

Envoyer les flux vidéo au cloud aurait signifié : la bande passante montante de
chaque magasin (rarement suffisante), le coût GPU côté serveur multiplié par le
nombre de caméras, et surtout des **images de clients qui quittent le magasin**.
En calculant sur place, ce qui sort tient dans quelques centaines d'octets par heure
et ne contient aucune donnée personnelle.

Conséquence assumée : il faut administrer un parc de boîtiers à distance — d'où la
page « Parc » et les alertes qualité décrites plus bas.

## Aucun port entrant

L'agent ne reçoit jamais de connexion : il **appelle** l'API. Cela évite de demander
au commerçant une IP fixe ou une redirection de port sur sa box, et supprime une
surface d'attaque. Les commandes à distance (recharger la configuration, redémarrer)
passent par une file que l'agent vient consulter, pas par un port ouvert.

L'agent s'authentifie par une **clé de magasin** révocable, distincte des comptes
utilisateurs.

## Durabilité du comptage

Le comptage et la synchronisation sont **séparés**. Le comptage écrit dans SQLite ;
la synchronisation lit la file et la vide à mesure que l'API confirme. Une coupure
réseau, un redémarrage de l'API ou un arrêt du boîtier ne perd donc aucun passage,
et l'envoi par lots avec identifiant stable garantit **0 doublon** à la reprise.

L'agent garde aussi une **copie locale de sa configuration** : privé de réseau au
démarrage, il compte quand même. Il expose un petit dashboard local, utile en magasin
pour vérifier une installation sans accès au cloud.

## Modèle de données

24 tables. Les principales :

- **Company → Store → Camera → Zone** : la hiérarchie, cloisonnée par entreprise.
- **VisitorEvent** : les passages bruts (entrée/sortie), source de vérité.
- **ZoneVisit** : une présence dans un rayon (début, fin, durée).
- **HourlyStat / DailyStat / ZoneHourlyStat** : agrégats pré-calculés. Le dashboard
  ne parcourt jamais les événements bruts — c'est ce qui garde les écrans rapides
  sur un an de données.
- **Transaction** : les ventes importées du logiciel de caisse.
- **StoreArticle** : le catalogue d'articles du magasin, chaque article rattaché à un
  rayon — c'est ce qui permet de croiser ventes et fréquentation.

Toute modification du schéma passe par une **migration versionnée** ; la CI les
applique sur une base vierge à chaque exécution, ce qui interdit les dérives entre
le code et la base.

## Croiser les ventes et la fréquentation

Le logiciel de caisse du client exporte un CSV. Deux difficultés réelles :

**Les réimports.** Un gérant exporte juillet, puis le mois suivant exporte
juillet + août. Le second import doit **ignorer juillet** sans créer de doublons.
La déduplication se fait par **jour local du magasin** et par identifiant de ticket,
et l'écran annonce les périodes retenues avant d'écrire.

**Rattacher un article à un rayon.** Demander une colonne « rayon » au client, c'est
du travail pour lui et il ne l'aura pas toujours. Les mots-clés par zone ont été
écartés (trop fragiles). Retenu : le magasin fournit la **liste de ses articles**, et
l'équipe support affecte chaque article à une zone depuis le dashboard. Un réimport
du catalogue **ajoute** les nouveaux articles sans écraser les affectations déjà
faites.

D'où l'indicateur final : **taux d'achat d'un rayon** = tickets contenant au moins un
article de ce rayon ÷ visites du rayon. Un taux sur des *tickets* ne dépend pas du
prix — un rayon d'informatique ne « gagne » pas contre un rayon d'accessoires parce
que ses articles coûtent plus cher, ce qui était le défaut de la première version
basée sur le chiffre d'affaires. Le calcul ne retient que les **jours où les caméras
du rayon ont réellement mesuré**, sinon les ventes de jours non mesurés gonflent le
taux.

## Exploiter un parc de boîtiers

Un système qui répond « en ligne » peut très bien compter faux. Deux niveaux de
surveillance :

- **État technique** — remonté à chaque battement de cœur : CPU, mémoire, disque,
  cadence réelle comparée à la consigne, redémarrages, version de l'agent. L'API
  classe chaque magasin *critique / à surveiller / OK*, **avec des raisons lisibles**.
- **Qualité du comptage** — fréquentation anormale comparée **au même créneau des
  semaines précédentes** (aucun horaire d'ouverture à saisir), déséquilibre
  entrées/sorties, proportion de passages « enfant » trop élevée, dérive de
  calibration.

## Déploiement

Images Docker publiées par la CI sur GHCR, en deux variantes **CPU** et **GPU**. Les
boîtiers **épinglent une version** au lieu de suivre `:latest` — après un incident où
le tag `latest` pointait sur l'image GPU de 15,5 Go et a saturé un disque.

Isolation multi-tenant : le filtrage applicatif par entreprise est la première
barrière. Une fondation de **Row-Level Security PostgreSQL** existe en défense en
profondeur, pour qu'un filtre oublié dans le code ne puisse pas provoquer de fuite
entre enseignes — utile le jour où un client exige un audit de sécurité.
