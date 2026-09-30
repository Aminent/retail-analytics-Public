# Captures d'écran

<!--
  À FAIRE avant publication : remplacer cette liste par les images.
  Syntaxe une fois le fichier déposé dans ce dossier :
      ![Analyse magasin](01-analyse-magasin.png)
-->

Captures à produire, dans cet ordre (la première est celle que le recruteur verra) :

| Fichier | Écran | Ce qu'il faut y voir |
|---|---|---|
| `01-analyse-magasin.png` | Analyse magasin | Les cartes d'indicateurs et la courbe horaire |
| `02-analyse-rayon.png` | Analyse par rayon | Les zones sur l'image caméra + les classements |
| `03-densite.png` | Carte de densité | Les points d'arrêt des clients |
| `04-conversion.png` | Conversion | Visiteurs, taux de conversion, panier moyen |
| `05-taux-achat-rayon.png` | Taux d'achat par rayon | L'indicateur qui croise ventes et visites |
| `06-parc.png` | Parc | L'état des boîtiers, avec les raisons lisibles |
| `07-zones-editeur.png` | Édition des zones | Le tracé d'un polygone au sol |
| `08-vision.png` | Sortie de la vision | Silhouettes détectées, ligne de comptage |

## À masquer impérativement avant de publier

**Les visages et les silhouettes identifiables.** Les vues caméra montrent des clients
réels dans un magasin réel. Sur toute image issue d'une caméra : flouter les personnes,
ou utiliser une image où il n'y a personne, ou remplacer la vue par le plan
schématique des zones (l'interface l'affiche déjà quand aucune image n'est disponible).

**Le nom du client et de l'enseigne** : nom du magasin, nom de la société, logo,
vitrine ou enseigne visible dans le champ de la caméra, ville si elle identifie le
magasin. À remplacer par un nom neutre (« Magasin Centre-Ville »).

**Les données techniques** : adresses IP des caméras, clés d'agent, adresses e-mail des
utilisateurs, jetons visibles dans une URL.

**Les chiffres d'affaires réels** du client, s'il ne t'a pas autorisé à les montrer.

Le plus simple et le plus sûr : faire les captures sur le **compte de démonstration**,
avec des données factices et un nom de magasin neutre.

## Conseils pratiques

- Thème **sombre** pour les captures principales, il rend mieux ; une ou deux en clair
  suffisent à montrer que les deux thèmes existent.
- Fenêtre large (1600 px au moins) pour que les grilles ne se replient pas.
- Format PNG. Compresser avant de commiter — un dépôt vitrine n'a pas besoin
  d'images de 4 Mo.
- Une capture d'un écran **mobile** montre que le dashboard est responsive.
