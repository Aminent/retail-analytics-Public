# Storalytics — comptage et analyse de fréquentation en magasin

Plateforme d'analyse vidéo pour le commerce de détail : des caméras déjà installées
en magasin comptent les visiteurs, mesurent le temps passé par rayon et rapprochent
la fréquentation des ventes. Conçue, développée et déployée en conditions réelles
au Maroc.

> **Le code source est privé** (projet commercial). Ce dépôt documente
> l'architecture, les choix techniques et les problèmes résolus. Les mesures citées
> proviennent du terrain, pas d'un banc d'essai.

---

## Le problème

Un commerçant connaît son chiffre d'affaires, mais pas combien de personnes sont
entrées. Il ne sait donc pas si une mauvaise journée vient d'un manque de clients
ou d'une mauvaise conversion, ni quels rayons attirent sans vendre. Les solutions du
marché imposent des capteurs dédiés ; ici, on part des **caméras de surveillance
déjà en place**.

## Ce que fait le système

| Fonction | Détail |
|---|---|
| **Comptage entrées / sorties** | Franchissement de ligne, par caméra, temps réel |
| **Présence par rayon** | Zones au sol (polygones) : visites, temps passé, occupation |
| **Carte de densité** | Où les clients s'arrêtent réellement dans le champ de la caméra |
| **Conversion** | Import des ventes (CSV du logiciel de caisse) ÷ visiteurs |
| **Taux d'achat par rayon** | Tickets contenant un article du rayon ÷ visites du rayon |
| **Surveillance du parc** | État de chaque boîtier magasin, sans lire les logs un par un |
| **Alertes qualité** | Détecte un magasin qui compte *faux* alors qu'il est « en ligne » |
| **Rapports** | PDF programmés, envoyés par e-mail |

## Architecture

```
    MAGASIN (réseau local)                        CLOUD
 ┌──────────────────────────────┐        ┌───────────────────────────┐
 │  Caméras IP (RTSP)           │        │  API Express + Prisma     │
 │        │                     │        │  PostgreSQL               │
 │        ▼                     │        │  Worker d'agrégation      │
 │  Boîtier « agent » (Docker)  │ HTTPS  │  Socket.IO (temps réel)   │
 │   • 1 worker Python / caméra │ ─────► │                           │
 │     YOLO11 + ByteTrack       │ sortant│  Dashboard React          │
 │   • file SQLite durable      │        │                           │
 └──────────────────────────────┘        └───────────────────────────┘
```

Trois décisions structurent tout le reste :

**1. Le calcul se fait en magasin.** Les images ne quittent jamais le local : seuls
des **chiffres agrégés** sont envoyés. Moins de bande passante, et un argument
décisif face aux clients sensibles à la confidentialité.

**2. Rien n'est ouvert en entrant.** L'agent établit une connexion **sortante**.
Aucun port à ouvrir, aucune IP fixe à demander au commerçant, aucune faille exposée.

**3. Une coupure réseau ne perd aucun passage.** Les passages sont écrits dans une
file **SQLite** locale, puis rejoués avec reprise. Coupure d'électricité comprise :
le comptage local continue, la synchronisation reprend au retour.

→ Détail : [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)

## Pile technique

**Back-end** TypeScript · Express · Prisma · PostgreSQL · Socket.IO · Zod
**Front-end** React 18 · Vite · CSS maison (thèmes clair/sombre, aucune librairie UI)
**Vision** Python · YOLO11 · ByteTrack · OpenCV · FFmpeg
**Infra** Docker · GitHub Actions (CI + publication d'images GHCR) · monorepo npm workspaces

Multi-tenant : tout est cloisonné par entreprise, avec cinq rôles
(super-admin, support, administrateur d'enseigne, gérant de magasin, démo).

## Le plus intéressant : la vision

Le comptage ne se joue pas dans le choix du modèle de détection — il se joue dans
les cas que le modèle ne résout pas. Quelques exemples, chacun tranché par la mesure :

- **Compter le personnel fausse tout.** Un employé traverse le champ vingt fois par
  jour. Le badge sur cordon a été testé et **abandonné** : mesuré de dos, il ne
  reste qu'un cordon de 2-3 px. Le **brassard fluo** a été retenu — sur 722 images :
  tache de 197-258 px, détectée sur ~71 % des images d'une personne, **0 fausse
  détection** sur 203 images sans brassard.
- **Un seuil de taille identique pour toutes les caméras fait disparaître des
  adultes** : 7 passages sur 22 perdus. Un adulte occupe 0,55-0,66 de l'image sur une
  entrée, 0,45-0,49 sur l'autre. D'où une **calibration par caméra**, puis
  **automatique** à partir des passages réellement mesurés.
- **L'appartenance à une zone se décide par la position des pieds**, pas par le
  corps entier : en perspective, le sol devant un rayon est un trapèze, pas un
  rectangle.
- **Pas de reconnaissance d'âge ni de genre.** Choix assumé : le traitement
  biométrique relève de la loi 09-08 et de la CNDP au Maroc. Le système ne compte que
  des silhouettes — contrainte devenue argument commercial.

→ Détail et mesures : [docs/VISION.md](docs/VISION.md)

## Diagnostiquer avant de corriger

Le projet tient un journal des pannes réelles, avec la **fausse piste** suivie
d'abord et la **cause réelle** ensuite. Trois exemples :

| Symptôme | Fausse piste | Cause réelle |
|---|---|---|
| 1900 % de CPU, cadence sous la consigne | variables d'environnement de threads | la librairie de détection **réécrit** le nombre de threads au premier appel |
| Une caméra refuse le mot de passe en RTSP | mot de passe corrompu | elle annonçait `MD5/SHA256` ; le client ne gère que MD5 |
| Disque saturé sur le boîtier | fuite de logs | le tag `:latest` pointait sur l'image **GPU** de 15,5 Go |

Résultat mesuré sur le premier : CPU ramené de ~1900 % à **~300 %**, cadences
tenues. Et la lecture RTSP à la place des instantanés JPEG a fait passer le CPU des
caméras de ~45 % à **10 %**.

→ Le journal complet : [docs/JOURNAL-TECHNIQUE.md](docs/JOURNAL-TECHNIQUE.md)

## Captures d'écran

→ [captures/](captures/)

## Ordres de grandeur

- ~24 000 lignes (TypeScript, Python, SQL, CSS), 172 commits
- 24 tables, migrations versionnées, CI obligatoire avant chaque fusion
- 29 fichiers de tests, ~150 tests automatisés
- Déployé et exploité en magasin réel : 3 caméras IP, boîtier Docker, un an de données

## Contact

Amine Ntela — [ntelaamine4@gmail.com](mailto:ntelaamine4@gmail.com)

Le code source peut être présenté et commenté en entretien.
