# ddev-wp-deploy

Déploiement du **thème** d'un projet WordPress vers un environnement distant,
par `rsync` sur SSH, depuis le poste du dev.

```bash
ddev deploy preprod
ddev deploy prod --dry-run
```

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

Puis, la première fois :

```bash
ddev deploy preprod
```

La commande **crée `.ddev/deploy/targets.conf` en te posant les questions**. Il
n'y a pas de fichier d'exemple à recopier — c'est délibéré : une configuration
qu'il faut aller chercher est une configuration qu'on oublie.

Pour propager les corrections à tout le parc :

```bash
ddev add-on update
```

## La configuration

`.ddev/deploy/targets.conf`, **versionné**, sans aucun secret :

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

`path` désigne la **racine du site**, celle qui contient `wp-content/` — pas le
dossier du thème. Le script en déduit la cible et sait où lancer `wp`.

> Si tu es tenté d'écrire un mot de passe dans ce fichier, c'est qu'il manque
> une clé publique sur le serveur. Le script n'accepte pas de mot de passe :
> `BatchMode=yes` interdit tout repli interactif, exprès.

## Ce que fait la commande

| | |
|---|---|
| **1 · Dépôt** | Arbre propre sur le thème, branche attendue, commit affiché |
| **2 · Build** | `ddev npm run prod` (et `npm ci` si `node_modules` manque) — jamais un `dist/` périmé |
| **3 · Contrôle** | Manifeste des assets, `acf-json/`, `dist/images` — voir plus bas |
| **4 · SSH** | Connexion testée avant le transfert, avec les causes probables en cas d'échec |
| **5 · Confirmation** | En prod, il faut taper `prod` |
| **6 · Transfert** | `rsync -az --delete`, exclusions de `deploy/exclude.txt` |
| **7 · Vérification** | HTTP sur l'URL publique, et `tools/diag-acf.php` à distance si `wp` est là |

### Les trois contrôles de l'étape 3

Ils ne sont pas décoratifs. **Chacun correspond à une panne réelle en
préproduction, et chacune était silencieuse** — aucune erreur, aucun log, un
site qui se vide.

- **Le manifeste des assets** (`dist/.vite/manifest.json`) — sans lui, ni CSS
  ni JavaScript ne sont chargés.
- **`acf-json/`** — ni un asset (donc pas dans `dist/`), ni en base (donc pas
  transporté par WP Migrate). C'est le dossier qui tombe dans tous les angles
  morts, et le seul dont l'absence casse *tout*.
- **`dist/images`** — un helper qui insère un SVG en ligne lit le fichier sur
  le disque. Sans le dossier construit, il rend une chaîne vide, sans rien dire.

Un contrôle en échec demande une confirmation explicite ; hors terminal, il
arrête le déploiement.

## Options

| | |
|---|---|
| `--dry-run`, `-n` | Montre ce qui partirait, ne transfère rien |
| `--no-build` | Déploie `dist/` tel qu'il est sur le disque |
| `--yes`, `-y` | Passe les confirmations (pour un usage scripté) |

## Prérequis

- **DDEV ≥ 1.24**, et un projet `type: wordpress`
- **SSH** sur l'hébergement, avec la clé publique du dev autorisée
- `rsync` — celui livré avec macOS suffit. Les options utilisées sont
  volontairement comprises par l'`openrsync` d'Apple (protocole 29) autant que
  par le `rsync` GNU : pas de `--filter`, pas de `--info`, `-n` plutôt que
  `--dry-run`. **Aucun `brew install` à demander.**

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
