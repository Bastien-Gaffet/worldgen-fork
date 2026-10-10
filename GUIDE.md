# Outil `worldgen-fork` — scoper un datapack de génération à un seul monde

Généralise le script ad hoc écrit pour le fork Terralith->PlanetY (voir
[Serveur Minecraft](minecraft-serveur.md#monde)) en un outil réutilisable :
prendre **n'importe quel datapack de génération de monde** et le rendre
utilisable sur **un seul nouveau monde Multiverse**, sans toucher à la
génération des autres mondes (en particulier la survie).

## Le problème que ça résout

Sur Paper, toute la génération de terrain (biomes, bruit, structures) est
pilotée par des **datapacks**, et les datapacks sont **globaux au niveau**
(`world/datapacks/`), partagés par tous les mondes/dimensions qui vivent dans
ce même conteneur `world/` — y compris les mondes créés par Multiverse (sur
Paper 26.2, un monde Multiverse est juste une dimension de plus dans
`world/dimensions/<namespace>/<nom>`).

Un datapack de génération comme Terralith redéfinit des entrées `minecraft:*`
(ex. `minecraft:overworld` en tant que biome_source). Si on l'installe tel
quel, il modifie la génération de **tous** les mondes qui utilisent ce
registre — y compris la survie. `worldgen-fork` évite ça en **renommant tout
le contenu du datapack source vers un namespace privé** (ex. `ply:` au lieu
de `minecraft:`), puis en créant une nouvelle dimension `minecraft:<monde>`
qui est la seule à référencer ce namespace. La survie ne voit jamais passer
`ply:*` et continue de tourner en pur `minecraft:*`.

## Emplacement

- Sur le VPS : `/usr/local/bin/worldgen-fork` (exécutable, `python3` pur,
  aucune dépendance externe — important car pip n'est pas dispo côté serveur).
- Miroir local (source de vérité pour l'historique/modifs) :
  [`vps/bin/worldgen-fork`](../vps/bin/worldgen-fork). Après une modif locale,
  redéployer avec :
  ```
  scp vps/bin/worldgen-fork root@VOTRE_VPS:/usr/local/bin/worldgen-fork
  ssh root@VOTRE_VPS 'chmod +x /usr/local/bin/worldgen-fork'
  ```

## Usage

```
worldgen-fork <source> --ns <namespace> --world <nom_du_monde> --out <dossier_de_sortie> [options]
```

| Argument | Obligatoire | Rôle |
|---|---|---|
| `<source>` | oui | Dossier extrait, `.zip` ou `.jar` du datapack source (Terralith, etc.) |
| `--ns` | oui | Namespace cible pour le worldgen forké (ex. `ply`). Ne peut pas être `minecraft`. Choisir un nom **unique**, pas déjà utilisé par un autre monde forké. |
| `--world` | oui | Nom du nouveau monde/dimension (ex. `planety`). Devient `minecraft:<world>`. |
| `--out` | oui | Dossier où écrire le datapack généré (créé, échoue s'il existe déjà sauf `--force`) |
| `--dimension-type` | non (défaut `minecraft:overworld`) | Type vanilla à cloner : `minecraft:overworld`, `minecraft:the_nether`, ou `minecraft:the_end` |
| `--source-dimension` | non | Chemin (relatif à `data/`) du fichier dimension à cloner, si le datapack source ne suit pas la convention `data/minecraft/dimension/<basename>.json` |
| `--pack-format` | non | Forcé si le `pack.mcmeta` source n'est pas lisible (rare) |
| `--description` | non | Texte du `pack.mcmeta` généré |
| `--drop-registry` | non, répétable | Registre worldgen additionnel à exclure du fork (en plus des 3 par défaut, voir plus bas) |
| `--install DIR` | non | Copie le pack généré directement dans `DIR` (le `world/datapacks/` visé) et aligne le propriétaire (uid/gid) sur celui de `DIR` — pratique pour éviter un `chown amp:amp` manuel |
| `--force` | non | Écrase `--out` s'il existe |

### Exemple complet (reproduction du fork PlanetY)

```
worldgen-fork /root/tlsrc/terralith_extract \
  --ns ply --world planety \
  --out /root/tlbuild/ply_pack \
  --install /home/amp/.ampdata/instances/PCSI201/Minecraft/world/datapacks
```

Sortie attendue (exemple réel) :
```
ply worldgen: 128 fichiers | autres namespaces: 1686 fichiers | erreurs JSON: 0
dimension minecraft:planety generee (type=minecraft:overworld, biome_source avait un preset: False -> retire, 1710 biomes inline)
pack ecrit dans /root/tlbuild/ply_pack
installe dans /home/.../world/datapacks/ply_pack (proprietaire aligne sur .../world/datapacks)
```

**`erreurs JSON: 0` est le signal à surveiller** : si ce nombre n'est pas 0,
le pack est quand même généré (les fichiers en erreur sont copiés tels quels,
non réécrits), mais ça veut dire que certains fichiers referencent le
namespace `minecraft:` d'une façon que le script n'a pas su résoudre — à
inspecter avant de déployer (voir Dépannage).

## Ce que le script fait exactement

1. Extrait `<source>` si c'est un `.zip`/`.jar` (recherche automatique du
   dossier contenant `data/`, y compris dans un sous-dossier wrapper).
2. Répertorie tout le contenu de `data/minecraft/worldgen/` (registre par
   registre : `biome`, `noise`, `density_function`, `placed_feature`,
   `configured_feature`, `configured_carver`, `noise_settings`, etc.), en
   excluant 3 registres par défaut :
   - `world_preset` — définit des préréglages de génération complets, sans
     rapport avec un monde scopé, et source d'erreurs de démarrage
     (`Missing overworld dimension`) si on les garde.
   - `flat_level_generator_preset` — idem, hors-sujet.
   - `multi_noise_biome_source_parameter_list` — format "lithostitché" qui
     dépend de fichiers assemblés à l'échelle du serveur ; inutile ici car la
     nouvelle dimension définit ses biomes en ligne (`biomes: [...]`).
3. Recopie **tout le reste du datapack** (les namespaces non-`minecraft`,
   ex. `terralith:`, `c:`, `tectonic:`...) à l'identique côté chemin, mais en
   réécrivant leur contenu interne.
4. Pour chaque fichier JSON traité (qu'il soit dans `data/<ns>/worldgen/` ou
   ailleurs), réécrit toute chaîne `"minecraft:xxx"` qui correspond à un id
   qui a été forké, en `"<ns>:xxx"` — **sauf** les clés `"type"` (qui
   désignent une classe Java interne, jamais un id de contenu data-driven).
   La clé JSON dans laquelle l'id apparaît sert à deviner dans **quel
   registre vanilla** le résoudre (une chaîne `"biome_source"`/`"noise"` ne
   veut pas dire la même chose selon qu'elle apparaît dans un fichier biome
   ou un fichier noise_settings) — voir la fonction `target_reg` dans le
   script, commentée, si un nouveau cas de collision apparaît.
5. Clone `data/minecraft/dimension/<basename du dimension-type>.json` du
   datapack source (ex. `overworld.json`), réécrit ses références internes,
   retire les clés de compat mod-loader (`fabric:load_conditions`,
   `neoforge:conditions`), force `type` sur le `--dimension-type` demandé,
   et retire un éventuel `preset` du `biome_source` (les presets sont
   toujours exclus, donc plus résolvables). Écrit le résultat dans
   `data/minecraft/dimension/<world>.json`.
6. Écrit un `pack.mcmeta` (détecte `pack_format` OU `min_format`/`max_format`
   selon le schéma du pack source).

Le fichier `data/minecraft/dimension/<world>.json` reste volontairement sous
le namespace `minecraft:` (c'est ce qui permet à Multiverse/Paper de
reconnaître `minecraft:<world>` comme une dimension standard) ; **c'est le
seul fichier du pack généré qui touche à `data/minecraft/`**. Tout le reste
du worldgen vit sous `data/<ns>/`.

## Limites connues

- **Ne fonctionne que sur des datapacks qui redéfinissent une dimension
  entière** (biome_source complet). Un datapack qui se contente d'*ajouter*
  des biomes/structures à la génération vanilla existante (sans fournir son
  propre `data/minecraft/dimension/overworld.json`) n'a pas de fichier à
  cloner à l'étape 5 — le script s'arrête avec un message clair plutôt que de
  produire un pack cassé. Fournir `--source-dimension` si le datapack utilise
  une convention de nommage différente.
- **Pas de changement de génération a posteriori** : une fois qu'un monde a
  généré des chunks avec un profil donné, changer le datapack ne modifie que
  les *futurs* chunks (comportement vanilla normal, pas une limite de l'outil).
- **`--ns` doit être unique par monde forké.** Deux mondes qui utilisent le
  même namespace partageraient leurs registres — reviendrait au problème
  qu'on essaie d'éviter.
- Testé et validé sur des datapacks de génération "classiques" (style
  Terralith : structure `data/minecraft/worldgen/*` + namespaces de compat
  annexes). Un datapack à l'architecture très différente peut nécessiter un
  ajustement du script (il est court et commenté, exactement pour ça).

## Procédure complète, du datapack téléchargé au monde jouable

1. **Récupérer le datapack** (ex. depuis Modrinth) sur le VPS, dans un dossier
   de travail, ex. `/root/forkbuild/<projet>/src.zip`.
2. **Générer le pack scopé** :
   ```
   worldgen-fork /root/forkbuild/<projet>/src.zip \
     --ns <namespace> --world <nom_monde> \
     --out /root/forkbuild/<projet>/pack
   ```
   Vérifier `erreurs JSON: 0` dans la sortie.
3. **Tester dans un conteneur Docker jetable AVANT de toucher la prod**
   (règle absolue : un datapack de worldgen mal formé fait planter Paper au
   boot — vécu une fois sur ce serveur avec Terralith non forké correctement).
   ```
   mkdir -p /root/tltest
   cp /home/amp/.ampdata/instances/PCSI201/Minecraft/paperclip.jar /root/tltest/
   rm -rf /root/tltest/world
   mkdir -p /root/tltest/world/datapacks
   cp -r /root/forkbuild/<projet>/pack /root/tltest/world/datapacks/<nom_pack>

   docker rm -f tltest 2>/dev/null
   docker run -d --name tltest --user root --memory=1500m --memory-swap=1900m \
     -v /root/tltest:/srv -w /srv \
     --entrypoint /usr/lib/jvm/temurin-25-jdk-amd64/bin/java \
     cubecoders/ampbase:java -Xms256M -Xmx850M -jar paperclip.jar --nogui

   sleep 30
   docker logs tltest | grep -iE "Done \(|Unbound|Failed to parse|Unknown|Missing tag|Missing.*dimension"
   docker rm -f tltest
   ```
   - `Done (` sans erreur au-dessus = boot propre, le pack est valide.
   - Toute ligne `Unbound`/`Unknown`/`Failed to parse`/`Missing tag` = un id
     n'a pas été résolu correctement (voir Dépannage).
4. **Déployer en prod** une fois le test propre :
   ```
   cp -r /root/forkbuild/<projet>/pack \
     /home/amp/.ampdata/instances/PCSI201/Minecraft/world/datapacks/<nom_pack>
   chown -R amp:amp /home/amp/.ampdata/instances/PCSI201/Minecraft/world/datapacks/<nom_pack>
   ```
   (ou directement avec `--install .../world/datapacks` à l'étape 2, qui fait
   la copie + le `chown` en un coup — mais alors on saute le test Docker, donc
   à réserver aux cas déjà validés une fois).
5. **Redémarrer le serveur** (`rcon "restart"` ou panel AMP) pour que Paper
   charge la nouvelle dimension `minecraft:<nom_monde>`.
6. **Créer le monde côté Multiverse** :
   ```
   rcon "mv import <nom_monde> normal"
   rcon "mv modify <nom_monde> set gamemode creative"    # ou survival, etc.
   rcon "mv modify <nom_monde> set difficulty peaceful"  # au choix
   ```
   Vérifier `rcon "mv list"`.
7. **Vérifier la génération** : se téléporter dedans (`/mv tp <joueur>
   <nom_monde>` ou `/mvtp`) et constater le terrain, ou en console :
   `rcon "execute in minecraft:<nom_monde> run forceload add 0 0"` puis
   observer les logs / `/locate biome`.

## Dépannage

Erreurs rencontrées pendant la mise au point du fork original (Terralith),
classées par cause — utile si `worldgen-fork` sort des `erreurs JSON` ou si
le test Docker échoue au boot :

| Symptôme | Cause | Remède |
|---|---|---|
| `Missing overworld dimension` au boot | Un `world_preset`/preset de génération complet a été inclus et résolu de travers | Vérifier qu'il est bien dans `--drop-registry` (inclus par défaut) |
| `Unknown registry key` / id genre `ply:temperature` invalide dans un `material_condition` ou une `tag` | Réécriture trop agressive : un id a été renommé alors qu'il devait rester `minecraft:` (ex. une clé qui ressemble à un id mais désigne un type Java, ou une référence vers `data/minecraft/tags/`) | `data/minecraft/tags/` et `data/minecraft/dimension/` (sauf le fichier cloné) ne sont jamais touchés par construction — si l'erreur persiste, le datapack source référence un id ambigu présent dans plusieurs registres forkés ; ajouter un cas explicite dans `target_reg()` |
| `Failed to parse` / `Unbound` sur un density_function (`overworld/temperature`, `overworld/vegetation`, etc.) | Id de bruit non résolu de façon unique (présent dans 2 registres forkés à la fois) | Ajouter le cas dans `target_reg()` plutôt que corriger le fichier généré à la main (voir encadré ci-dessous) |
| Erreur JSON listée en sortie du script (`errors`) | Fichier du datapack source malformé, ou schéma inattendu | Le fichier est copié tel quel dans le pack de sortie (non réécrit) — l'ouvrir et regarder s'il référence vraiment du contenu forké |

> **Ne jamais corriger un fichier généré à la main.** En creusant l'ancien
> pack `PlanetY_Terralith` pour écrire cet outil, on a trouvé un vrai bug
> resté 2 mois en prod sans être vu : un correctif ponctuel avait laissé une
> **copie orpheline** de `overworld/temperature.json` et
> `overworld/vegetation.json` sous `data/minecraft/worldgen/...` (en plus de
> la bonne copie sous `data/ply/worldgen/...`), ce qui redéfinissait
> silencieusement ces deux density functions **pour tous les mondes,
> survie comprise**. Le nouvel outil ne reproduit pas ce bug (il n'écrit
> jamais rien sous `data/minecraft/` à part le fichier dimension), et le
> pack fautif a été retiré du serveur le 2026-09-14 (fermeture du projet
> PlanetY). La leçon : si un id ne se
> résout pas bien, corriger la règle dans le script et régénérer tout le
> pack — jamais patcher une sortie à la main, l'erreur redevient invisible
> au prochain régénérage et personne ne se souvient de l'avoir faite.

## Voir aussi

- [Serveur Minecraft](minecraft-serveur.md) — contexte général, `world/datapacks/`
- [Architecture générale](architecture.md#principes) — principe de réversibilité/vérification
