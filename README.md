# ddev-wp-deploy

Déploiement du **thème** d'un projet WordPress vers un environnement distant,
par `rsync` sur SSH, depuis le poste du dev.

```bash
ddev deploy                    # liste les environnements configurés
ddev deploy preprod            # construit, contrôle, transfère, vérifie
ddev deploy prod --dry-run     # montre ce qui partirait, n'envoie rien
```

Outil interne Propulse. *(Si ce dépôt devient public un jour, il lui faudra une
licence — la question n'est pas tranchée.)*

## Le principe

**Le script est le produit, pas la CI.** Il tourne sur la machine du dev, avec
sa clé SSH. Conséquences :

- **aucun secret dans le dépôt**, et rien à poser dans une interface GitHub que
  personne ne maintient ;
- la mise en route est **une fois par dev** (sa clé publique sur le serveur),
  pas une fois par projet ;
- la commande est **versionnée avec le projet** : qui clone a `ddev deploy`.

Il reste appelable depuis une CI plus tard — c'est le script qui porte la
logique, une CI ne serait qu'un déclencheur.

## Ce qu'il ne touche pas

Le transfert est circonscrit à `wp-content/themes/<thème>/`. **La base de
données, `wp-content/uploads` et les plugins ne sont jamais dans le périmètre** :
ils restent le domaine de *WP Migrate*, qui fait ça mieux que n'importe quel
script maison (search-replace dans les données sérialisées, médias, et dans les
deux sens).

> ### ⚠️ Le thème d'abord, les données ensuite
>
> Les définitions de champs ACF vivent dans `acf-json/` **du thème** et ne sont
> pas en base. Une base poussée avant le thème référence donc des clés de
> champs que le serveur ne connaît pas encore : ACF rend alors les valeurs
> brutes, les pages s'affichent vides, les écrans d'édition perdent leurs
> champs, et un champ image lu comme un tableau fait tomber le site en 500.

## Installation

```bash
ddev add-on get Team-Propulse/ddev-wp-deploy
git add .ddev && git commit -m "chore: ddev deploy"
```

`ddev add-on get <org>/<dépôt>` installe **la dernière release** du dépôt. Pour
épingler une version, ou travailler sur la branche par défaut :

```bash
ddev add-on get Team-Propulse/ddev-wp-deploy --version v1.0.0
ddev add-on get Team-Propulse/ddev-wp-deploy --default-branch
```

Puis, la première fois :

```bash
ddev deploy preprod
```

La commande **crée `.ddev/deploy/targets.conf` en te posant les questions**. Il
n'y a pas de fichier d'exemple à recopier — c'est délibéré : une configuration
qu'il faut aller chercher est une configuration qu'on oublie. Hors terminal
interactif, elle refuse et te dit de la lancer à la main une première fois.

Pour propager les corrections à tout le parc :

```bash
ddev add-on update
```

Pour retirer l'add-on d'un projet :

```bash
ddev add-on remove wp-deploy
```

`targets.conf` et `exclude.local.txt` **survivent** à une désinstallation :
l'add-on ne possède que `commands/host/deploy`, `deploy/exclude.txt` et
`deploy/.gitignore`.

## La configuration

`.ddev/deploy/targets.conf`, **non versionné** — l'add-on installe le
`.gitignore` qui l'écarte :

```ini
theme = mon-theme

[preprod]
host   = xxx.infomaniak.com
user   = xxxx_ben
port   = 22
path   = /home/clients/xxxx/sites/preprod.exemple.fr
url    = https://preprod.exemple.fr
branch = main

[prod]
host   = xxx.infomaniak.com
user   = xxxx_ben
path   = /home/clients/xxxx/sites/exemple.fr
url    = https://exemple.fr
branch = main
```

| Clé | Portée | Rôle |
|---|---|---|
| `theme` | globale | Nom du dossier dans `wp-content/themes/` |
| `host` | par env | Hôte SSH — **obligatoire** |
| `user` | par env | Utilisateur SSH — **obligatoire** |
| `path` | par env | **Racine du site**, celle qui contient `wp-content/` — **obligatoire** |
| `port` | par env | Port SSH, `22` par défaut |
| `url` | par env | URL publique, pour le contrôle final. Sans elle, pas de vérification HTTP |
| `branch` | par env | Branche git attendue. Une autre branche déclenche un avertissement, pas un refus |

Le nom de la section (`[preprod]`, `[prod]`, `[recette]`…) est libre : c'est
l'argument de `ddev deploy`. Seul `prod` a un traitement particulier — il
demande de taper `prod` pour confirmer.

> Si tu es tenté d'écrire un mot de passe dans ce fichier, c'est qu'il manque
> une clé publique sur le serveur. Le script n'accepte pas de mot de passe :
> `BatchMode=yes` interdit tout repli interactif, exprès.

### Pourquoi il n'est pas versionné

Le fichier ne contient aucun secret — la clé SSH du dev est le seul facteur
d'authentification — mais il nomme des **hôtes, des utilisateurs SSH et des
chemins absolus de serveurs**. Ce sont des informations d'infrastructure, elles
n'ont pas à voyager avec le code.

**Conséquence assumée :** chaque dev crée le fichier à son premier
`ddev deploy <env>`, guidé par l'assistant. Trente secondes, une fois par
machine.

`exclude.local.txt`, lui, **reste versionné** : c'est un réglage du projet et
non de la machine, et toute l'équipe doit déployer avec les mêmes exclusions.

## Ce qui ne part pas

Deux listes d'exclusions, et la séparation est volontaire :

| Fichier | Appartient à | Versionné | Mis à jour par `ddev add-on update` |
|---|---|---|---|
| `.ddev/deploy/exclude.txt` | l'add-on | oui | **oui** |
| `.ddev/deploy/exclude.local.txt` | le projet | oui | **jamais** |
| `.ddev/deploy/targets.conf` | la machine | **non** | jamais |

La liste commune écarte les **entrées du build** (`src/`, `node_modules/`,
`package.json`, la config Vite/Tailwind/PostCSS), le contrôle de version et les
débris. Ce qui aide à diagnostiquer en ligne — `docs/`, `tools/` — part : ça ne
pèse rien et ça vaut cher le jour où il faut comprendre quelque chose sur le
serveur.

Pour une exclusion propre à un projet, créer `exclude.local.txt` : la commande
passe **les deux** listes à rsync. C'est ce qui permet aux corrections de la
liste commune de se propager sans écraser les réglages d'un projet.

> ⚠️ rsync ne **supprime** pas les fichiers exclus déjà présents sur le
> serveur : un `src/` hérité d'un ancien téléversement FTP y survivra sans être
> mis à jour. Le nettoyer est une suppression manuelle, une fois.

## Ce que fait la commande

| | |
|---|---|
| **1 · Dépôt** | Arbre propre sur le thème, branche attendue, commit affiché |
| **2 · Build** | `ddev npm run prod` (et `npm ci` si `node_modules` manque) — jamais un `dist/` périmé |
| **3 · Contrôle** | Manifeste des assets, `acf-json/`, `dist/images` — voir plus bas |
| **4 · SSH** | Connexion testée, et présence de `wp-content/` à la racine indiquée |
| **5 · Confirmation** | En prod, il faut taper `prod` |
| **6 · Transfert** | `rsync -az --delete`, normalisation des droits sur le serveur, puis **vidage du cache de pages** |
| **7 · Vérification** | HTTP sur l'URL publique **et sur chaque asset du thème**, puis `tools/diag-acf.php` à distance si wp-cli est là — sous le nom `wp` **ou** `wp-cli` (celui d'Infomaniak) |

### Les trois contrôles de l'étape 3

Ils ne sont pas décoratifs. **Chacun correspond à une panne réelle en
préproduction, et chacune était silencieuse** — aucune erreur, aucun log, un
site qui se vide.

- **Le manifeste des assets** (`dist/.vite/manifest.json`) — sans lui, ni CSS
  ni JavaScript ne sont chargés.
- **`acf-json/`** — ni un asset (donc pas dans `dist/`), ni en base (donc pas
  transporté par WP Migrate). C'est le dossier qui tombe dans tous les angles
  morts, et le seul dont l'absence casse *tout*. Le contrôle est piloté par
  **git** et non par le disque : un dossier suivi mais absent de la copie de
  travail est le cas le plus grave — `--delete` effacerait les définitions du
  serveur — et c'est celui qu'un test « si le dossier existe » ne voit pas.
- **`dist/images`** — un helper qui insère un SVG en ligne lit le fichier sur
  le disque. Sans le dossier construit, il rend une chaîne vide, sans rien dire.

Un contrôle en échec demande une confirmation explicite ; hors terminal, ou
avec `--yes`, il arrête le déploiement plutôt que de deviner.

### Les droits sont normalisés sur le serveur

Après le transfert, une passe explicite sur le dossier du thème : **755 pour
les dossiers, 644 pour les fichiers**.

Ce n'est pas de la coquetterie. Sur macOS, DDEV synchronise par mutagen, qui
écrit sur l'hôte en `600`/`700` tout ce que le conteneur a produit — donc les
assets construits par Vite. Un transfert qui préserve les droits les livre
illisibles pour le serveur web : **403 sur le CSS et le JS, page en 200,
entièrement dépouillée.**

> ⚠️ **Ne pas compter sur `--chmod`.** L'`openrsync` livré avec macOS accepte
> l'option, l'annonce dans son usage, et **ne l'applique pas** — sans
> avertissement. Un déploiement s'annonce alors réussi et le 403 reste. Elle est
> conservée dans la commande parce qu'elle fonctionne avec le rsync GNU, mais
> c'est la passe côté serveur qui garantit le résultat.

Cette passe **répare** aussi un serveur déjà dans cet état, ce qu'un simple
transfert ne fait pas. Elle est sautée en `--dry-run`.

### Le cache de pages est vidé

Depuis la v1.2.0. **Un cache de pages survit à un déploiement, et c'est une
panne.** Les assets construits portent un nom haché qui change à chaque build
(`main-C_b-2cCh.js`), et `--delete` supprime les anciens. Une page mise en
cache avant le déploiement garde l'ancienne URL : **son JavaScript répond 404**,
et son CSS — si l'extension l'a recopié — est servi périmé.

Aucune extension ne le voit toute seule : WP Fastest Cache, par exemple, ne se
vide que sur des événements WordPress (publication, changement de thème, mise à
jour par le gestionnaire), **jamais sur un rsync**. Rencontré sur un site dont
le popup de vérification d'âge ne se ferme que par le JS : un nouveau visiteur
y serait resté bloqué.

Après le transfert, la commande **supprime donc les dossiers de cache connus**
sur le serveur — exactement ce que fait l'extension quand on clique « Vider le
cache ». Ils sont recréés à la visite suivante.

| Extension | Dossiers vidés, sous `wp-content/cache/` |
|---|---|
| WP Fastest Cache | `all` (pages), `wpfc-minified` (CSS/JS recopiés), `wpfc-mobile-cache` (si activé) |

- **La liste est fermée**, et limitée à ce qui a été vérifié. Pour une autre
  extension, lire d'abord son code de purge, puis ajouter ses dossiers à
  `CACHES_PAGES` dans `commands/host/deploy`.
- En `--dry-run`, la commande **dit** ce qu'elle viderait, sans y toucher.
- `--no-purge` saute l'étape.
- Sans cache sur le serveur, l'étape ne fait rien et le dit.
- Si la suppression échoue, le déploiement continue, mais **l'alerte demande de
  vider le cache à la main** : c'est la seule panne de ce déploiement qui ne se
  voit pas sur la page de vérification — celle-ci est justement régénérée.

### La vérification porte sur les assets, pas seulement sur la page

L'étape 7 relève les URL du thème réellement référencées dans le HTML rendu — ni
liste à maintenir, ni supposition sur le nom des fichiers hachés — et vérifie
chacune. Une page en 200 dont le CSS répond 403 est indiscernable d'un
déploiement réussi si on s'arrête au code de la page.

## Options

| | |
|---|---|
| `--dry-run`, `-n` | Montre ce qui partirait, ne transfère rien et ne touche pas aux droits |
| `--no-build` | Déploie `dist/` tel qu'il est sur le disque |
| `--no-purge` | Ne vide pas le cache de pages du serveur (voir plus haut) |
| `--yes`, `-y` | Pas de question. Un contrôle en échec **arrête** le déploiement au lieu de demander |

## Prérequis

- **DDEV ≥ 1.24**, et un projet `type: wordpress`
- **SSH** sur l'hébergement, avec la clé publique du dev autorisée
- `rsync` — celui livré avec macOS suffit. Les options utilisées sont
  volontairement comprises par l'`openrsync` d'Apple (protocole 29) autant que
  par le `rsync` GNU : pas de `--filter`, pas de `--info`, `-n` plutôt que
  `--dry-run`. **Aucun `brew install` à demander.**

## Quand ça coince

| Symptôme | Cause la plus probable |
|---|---|
| `unknown command "deploy"` | Pas lancé depuis un projet DDEV, ou add-on installé ailleurs. `ddev add-on list --installed` |
| Connexion SSH refusée | Clé publique absente du serveur, ou hôte pas encore dans `~/.ssh/known_hosts`. Le script affiche la commande `ssh` à lancer une fois à la main |
| `wp-content est introuvable` | `path` désigne le dossier du thème au lieu de la **racine du site** |
| **403 sur un asset** | Droits sur le serveur. Un déploiement les normalise ; s'il en reste, c'est un dossier **parent** non traversable |
| Un asset en 404 | `dist/` périmé sur le serveur, ou déployé avec `--no-build` |
| `not using a post-quantum key exchange` | Avertissement d'OpenSSH sur l'**ancienneté du SSH du serveur**. Rien à voir avec le script, et volontairement non masqué |

### La sonde qui localise un 403

Un fichier **inexistant** distingue les deux causes en une requête :

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://…/dist/assets/inexistant-xyz.txt
```

| Réponse | Diagnostic |
|---|---|
| **404** | le dossier est lisible et traversable — le problème est le fichier |
| **403** | le dossier lui-même n'a pas le bit d'exécution, et rien de ce qu'il contient n'est accessible |

## Ce qu'il ne fait pas, et pourquoi

- **Pas de `releases/` + symlink.** Le modèle a du sens pour une application
  qui migre une base et compile des dépendances sur le serveur. Un thème, non :
  le transfert dure deux secondes, il n'y a rien à migrer, et faire du dossier
  du thème un lien symbolique complique la résolution de chemins côté
  WordPress pour un gain nul.
- **Pas de rollback.** `git checkout <commit> && ddev deploy preprod` en tient
  lieu, et reste vrai.
- **Pas de déploiement automatique au merge.** Le *moment* du déploiement est
  un geste humain.
- **Pas de transport FTP.** Sans clé, il faudrait stocker un mot de passe ; et
  sans SSH, ni normalisation des droits ni diagnostic à distance.
- **Pas de déploiement des plugins.** Le périmètre est le thème. Un `--delete`
  sur `wp-content/plugins` supprimerait tout plugin installé depuis l'admin et
  non versionné — un piège sérieux en WordPress.
