# worldgen-fork

Outil en ligne de commande qui rend **n'importe quel datapack de génération de
monde** (Terralith, etc.) utilisable sur **un seul monde**, sans modifier la
génération des autres mondes du serveur (en particulier la survie).

Né du projet PlanetY (voir `planety-archive`), puis généralisé pour pouvoir
refaire l'opération sans aide.

## Le problème

Sur Paper, la génération (biomes, bruit, structures) vient des datapacks, et
les datapacks sont **globaux** : un seul dossier `world/datapacks/` pour tous
les mondes, y compris ceux créés par Multiverse (de simples dimensions de plus
dans `world/dimensions/`). Un datapack comme Terralith redéfinit des entrées
`minecraft:*` : installé tel quel, il change la génération de **tous** les
mondes.

## La solution

`worldgen-fork` copie le datapack en **renommant tout son contenu vers un
espace de noms privé** (par exemple `ply:` au lieu de `minecraft:`), réécrit
toutes les références internes, puis crée une nouvelle dimension
`minecraft:<monde>` qui est la seule à pointer vers cet espace de noms. Les
autres mondes ne voient jamais `ply:*` et restent en `minecraft:*` pur.

## Utilisation

```bash
worldgen-fork <datapack source : dossier, .zip ou .jar> \
  --ns <espace_de_noms> --world <nom_du_monde> --out <dossier_sortie> \
  [--dimension-type ...] [--install <world/datapacks/>] [--force]
```

- Détecte tout seul le `pack_format` / `min_format` du `pack.mcmeta` source.
- Extrait automatiquement les `.zip` et `.jar`.
- Python 3, bibliothèque standard uniquement (pip n'est pas disponible côté
  serveur).

La procédure complète (téléchargement, test dans un conteneur Docker jetable,
déploiement, création du monde Multiverse, dépannage) est dans
[GUIDE.md](GUIDE.md).

## Bug historique trouvé grâce à l'outil

En comparant la sortie de l'outil avec l'ancien fork déployé, on a découvert
que le pack PlanetY contenait **en plus** une copie non renommée de
`minecraft:overworld/temperature` et `minecraft:overworld/vegetation`. Ces
fichiers écrasaient les valeurs vanilla et modifiaient discrètement la
génération des nouveaux chunks de la survie pendant 2 mois. L'outil ne produit
plus jamais de fichier `minecraft:` en dehors de la nouvelle dimension.

## Coût

Outil ponctuel : aucun processus résident. L'exécution prend quelques secondes
et quelques dizaines de Mo de RAM. Le coût à l'exécution du jeu est celui d'un
monde supplémentaire (sa génération et ses chunks chargés).
