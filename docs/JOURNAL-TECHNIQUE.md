# Journal technique : la fausse piste, puis la cause réelle

Le projet tient un journal des pannes rencontrées **en conditions réelles**. Chaque
entrée note la première hypothèse — presque toujours fausse — et la cause réellement
trouvée. C'est ce document qu'on relit avant de re-diagnostiquer un symptôme déjà vu.

La règle de travail qui en découle : **diagnostiquer avec des données avant de
corriger** — logs, base, mesures. Les incidents ci-dessous montrent pourquoi : dans
presque tous les cas, la correction évidente était inutile.

Les incidents liés au comptage et aux caméras sont décrits dans
[VISION.md](VISION.md). Voici les autres.

---

## Disque saturé sur le boîtier (0,9 Go libres)

**Cause immédiate** : téléchargement de l'image `:latest` de l'agent, qui pointait sur
la variante **GPU — 15,5 Go**, sur un boîtier CPU.

**Cause racine** : le workflow de publication ajoutait le tag `latest` aux **deux**
variantes ; la GPU, publiée en second, écrasait la CPU.

**Deux correctifs, pas un** : désactiver le tag flottant dans la CI, **et** épingler
une version explicite sur chaque boîtier. Corriger seulement la CI aurait laissé la
règle « ne jamais suivre `latest` en production » non écrite.

**À savoir (Windows)** : supprimer des images ne rend pas l'espace au disque. Le
disque virtuel de Docker ne rétrécit pas tout seul ; il faut quitter Docker, arrêter
WSL, puis compacter le fichier `.vhdx` explicitement.

## La version déclarée par l'agent était fausse

`AGENT_VERSION` était **écrit en dur** dans le code (`0.4.0`) et n'avait jamais été
mis à jour. Conséquence : impossible de repérer un boîtier en retard dans le parc —
tous annonçaient la même version. La vraie version est maintenant **injectée au build
par la CI**, donc elle ne peut plus mentir.

Le genre de bug qui ne casse rien et fait perdre une heure le jour où on cherche
pourquoi un magasin se comporte différemment des autres.

## Coupure de courant : PostgreSQL ne redémarre pas

Le service Windows restait arrêté après le retour du courant : l'API renvoyait 500 et
le dashboard affichait « Serveur injoignable ».

**Ce qui n'a pas été perdu** : aucun passage. L'agent compte en local et garde les
passages dans sa file — la panne côté cloud n'a coûté que de l'affichage, ce qui est
exactement le comportement recherché par la conception (voir
[ARCHITECTURE.md](ARCHITECTURE.md)).

**Décision** : prévoir un onduleur en magasin, et surveiller le redémarrage des
services plutôt que de supposer qu'ils reviennent seuls.

## Réimport des ventes : ignorer ce qui est déjà là

Un gérant exporte juillet depuis sa caisse, puis le mois suivant exporte
juillet + août — le même fichier contient donc une période déjà importée. Sans
protection, la conversion double sur juillet.

**Traitement retenu** : déduplication par **jour local du magasin** et par identifiant
de ticket, et l'écran annonce les périodes retenues et ignorées **avant** d'écrire.
Le fuseau compte : un ticket du 31 juillet à 23 h ne doit pas basculer en août.

## Encodage des CSV de caisse

Les exports CSV arrivaient avec les accents corrompus, ce qui faisait échouer le
rattachement des articles au catalogue — 54 504 DH de ventes « hors catalogue » alors
que les articles existaient bien.

**Cause** : la librairie de lecture interprétait un CSV UTF-8 sans BOM comme du
Latin-1. Le décodage est désormais fait explicitement (UTF-8 strict, repli
windows-1252), et les lignes déjà abîmées en base ont été réparées après un essai à
blanc.

**Leçon** : un import qui « fonctionne » peut produire des données silencieusement
fausses. Ici, le signal a été un montant qui ne collait pas — d'où l'intérêt d'afficher
des totaux de contrôle après un import.

## Un indicateur techniquement juste mais commercialement faux

La première version de l'indicateur croisant ventes et rayons mesurait le **chiffre
d'affaires par visite**. Calcul correct, conclusion trompeuse : un rayon informatique
écrase un rayon d'accessoires simplement parce que ses articles coûtent plus cher, ce
qui ne dit rien de la performance du rayon.

**Remplacé** par le **taux d'achat** : tickets contenant au moins un article du rayon
÷ visites du rayon. Un taux sur des tickets ne dépend pas du prix. Et le calcul ne
retient que les **jours où les caméras du rayon ont réellement mesuré**, sinon les
ventes des jours non mesurés gonflent mécaniquement le taux.

**Leçon** : la validation d'un indicateur n'est pas dans le test unitaire, elle est
dans la question « qu'est-ce que le commerçant va décider en le lisant ».
