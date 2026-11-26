# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Base de capitalisation — contexte du projet

## État actuel du dépôt (à mettre à jour)

Au 2026-11-26, les étapes 1 à 4 sont faites :
- arborescence de `content/` créée. Chaque dossier de domaine contient un `.gitkeep` pour que git le conserve. Quartz ignore les fichiers commençant par un point (`globby` sans l'option `dot`, vérifié au build), donc un domaine n'apparaît sur le site qu'à sa première fiche. Le `.gitkeep` peut être supprimé dès que le dossier contient une fiche ;
- déploiement : `.github/workflows/deploy.yml` est copié **à l'identique** de `docs/hosting.md`, section GitHub Pages (la même doc que quartz.jzhao.xyz/hosting). Il se déclenche sur un push vers `v5`. Pages est activé avec la source GitHub Actions. Site : https://psand72.github.io/krimpa/ ;
- plugin `@quartz-community/cname` désactivé dans `quartz.config.yaml`. Selon son README, il sert aux domaines personnalisés, et nous n'en avons pas ;
- le dossier est un clone de `jackyzha0/quartz` (Quartz **v5.2.2**). `npx quartz create --template obsidian --strategy new --links shortest --baseUrl psand72.github.io/krimpa` a été exécuté ;
- `quartz.config.yaml` : `pageTitle: "Krimpa"`, `locale: fr-FR`, `analytics: null` (le modèle activait Plausible) ;
- remotes : `origin` = dépôt GitHub public `psand72/krimpa`, `upstream` = `jackyzha0/quartz` (utilisé par `npx quartz upgrade`) ;
- `.github/dependabot.yml`, hérité de Quartz, a été supprimé pour éviter des pull requests automatiques. Les autres workflows de `.github/` ne tournent que sur `jackyzha0/quartz`. Ils sont inertes ici et ont été laissés pour limiter les conflits lors de `npx quartz upgrade` ;
- identité git **locale** au dépôt : `psand72` / adresse noreply GitHub, pour ne pas exposer d'adresse réelle ;
- `content/index.md` est la page d'accueil, avec la liste des domaines en liens vers les pages de dossier (`[[cds/|CDS]]`). Un lien reste en 404 tant que son dossier n'a pas de fiche. Les « fiches récentes » viennent du plugin `@quartz-community/recent-notes`, affiché uniquement sur l'accueil (`condition: is-index`, `limit: 5`) ;

Branche et `sync`, vérifiés dans `quartz/cli/handlers.js` :
- la branche principale s'appelle **`v5`** et ne doit pas être renommée. `npx quartz sync` tire toujours `origin v5` (`QUARTZ_SOURCE_BRANCH` dans `quartz/cli/constants.js`) ;
- `sync` fait `git add .`, un commit (message par défaut `Quartz sync: <date>`, modifiable avec `-m`), puis un **push forcé** (`git push -uf origin <branche>`). Tout fichier non ignoré par `.gitignore` est donc publié.

Le dossier est dans Dropbox, par choix. `node_modules/` et `.quartz/` (plugins) sont exclus de la synchronisation Dropbox par l'attribut étendu `user.com.dropbox.ignored=1`. C'est l'équivalent de `attr -s com.dropbox.ignored -V 1`, la commande indiquée par l'aide Dropbox, mais `attr` n'est pas installé. Il faut le reposer si l'un de ces dossiers est supprimé puis recréé, par exemple après un `rm -rf node_modules`. `.git/` est synchronisé.

## Commandes

Il n'y a ni tests ni lint. Une fois Quartz installé :
- `npx quartz build --serve` : construit le site et le sert en local pour vérifier ;
- `npx quartz sync` : commit + push, ce qui déclenche la reconstruction par GitHub Actions. Cette commande n'a pas encore été vérifiée en v5 (voir le point 2 de « Points à vérifier »).

## Objectif

Base **personnelle** de fiches de référence, consultable à tout moment et depuis n'importe quel appareil (navigateur, mobile).
Ce n'est **pas** un outil de gestion documentaire : pas de workflow de validation, pas de statuts, pas de relecture.

- Mainteneur unique : moi.
- Rédaction : Obsidian.
- Publication : Quartz → GitHub Actions → GitHub Pages.
- Mise à jour : `npx quartz sync` (commit + push), puis reconstruction automatique du site.

## Décisions prises

| Sujet | Décision |
|---|---|
| Visibilité | **Site public** (option A) : fiches sans données client ni information confidentielle |
| Générateur de site | Quartz (v5 au moment du choix, Node ≥ 22) |
| Hébergement | GitHub Pages, déployé par GitHub Actions |
| Flux | Push direct sur la branche principale, sans branches ni pull requests |
| Contrôles | Aucun contrôle bloquant au départ ; à ajouter seulement si un besoin apparaît |
| Fiches générées par script | Optionnel, plus tard (squelette `generateur_fiches` disponible à part) |

## Périmètre du contenu

Les fiches ne contiennent que du savoir générique : aucune donnée client ni information confidentielle. Tout le contenu de `content/` (hors modèles) est donc publiable.

## Organisation du contenu

```
content/
├── index.md          # page d'accueil : domaines + fiches récentes
├── bw/
├── datasphere/
├── cds/
├── s4-embedded/
├── s4-fonctionnel/
├── sac/
├── abap/
├── python/
├── methode/
├── html/             # pages HTML autonomes en .htm (voir « Pages HTML »)
├── private/          # brouillons à relire : ni publiés, ni commités (voir « Brouillons »)
└── templates/        # modèles Obsidian, jamais publiés
```

Conventions :
- un fichier = une fiche ; nom en minuscules, mots séparés par des tirets, sans accents ;
- en-tête YAML minimal : `title`, `description`, `tags`, `date`, `draft` — valeurs texte **entre guillemets** (un YAML invalide bloque tout le build) ;
- `draft: true` pour masquer une fiche en cours ;
- liens entre fiches avec des wikilinks `[[nom-de-fiche]]` ;
- modèles : `content/templates/fiche.md` (fiche standard) et `content/templates/fiche-html.md` (fiche d'accompagnement d'une page HTML).

## Brouillons

`content/private/` est le dossier tampon des brouillons à relire et à mettre en forme. Il est exclu deux fois, vérifié en v5.2.2 :
- du site, par `private` dans `ignorePatterns` (`quartz.config.yaml`) ;
- de git, par `private/` dans `.gitignore`, donc `npx quartz sync` ne l'envoie pas sur le dépôt public.

Ne pas retirer ces deux exclusions. `draft: true` ne suffit pas pour un contenu non relu : la fiche est masquée du site, mais elle est quand même poussée sur le dépôt GitHub public. **Règles de contrôle d'un brouillon avant publication** (décidées le 2026-11-26) :
- lire le brouillon en entier et lister à l'utilisateur les références client ou projet trouvées, avec leurs numéros de ligne ;
- **conserver tels quels** les termes génériques (« projet », « client », « besoin », « métier », « cadrage », « POC »…) et les noms d'éditeurs, de solutions ou de logiciels (SAP, BO, Fivetran, Snowflake, Microsoft…) ;
- **reformuler sans changer le sens** toute phrase contenant un nom qui permet d'identifier une entreprise cliente ou une personne. Par exemple, un responsable nommé est remplacé par son rôle ;
- en cas de doute sur un nom ou un détail, demander à l'utilisateur plutôt que trancher.

## Pages HTML

Quartz copie les fichiers non-Markdown du dossier `content/` : ils sont en ligne, mais absents de la recherche, des tags, du graphe et de la navigation. Toute page HTML est donc **accompagnée d'une fiche Markdown** qui la rend trouvable.

**Extension `.htm` obligatoire** (vérifié dans le code de v5.2.2). À la copie, `slugifyFilePath` (`@quartz-community/utils`) retire les extensions `.md` **et `.html`**. Un `content/html/<nom>.html` sortirait donc en `public/html/<nom>`, sans extension, et GitHub Pages ne le servirait pas comme une page. L'extension `.htm` est conservée, et la page sort en `public/html/<nom>.htm`.

Convention :
- la page va dans `content/html/<nom>.htm`. Si elle a des fichiers annexes (CSS, JS, images), elle va dans un sous-dossier `content/html/<nom>/index.htm` et le lien vise alors `/html/<nom>/index.htm`. Cette variante n'a pas été testée : c'est une déduction tirée du même code, et un `index.html` y perdrait aussi son extension ;
- la fiche d'accompagnement va dans le dossier de son domaine, **avec le même `<nom>`** : `content/cds/<nom>.md` ;
- la fiche est créée à partir du modèle `fiche-html.md` (tag `page-html`). Le seul marqueur à remplacer est `NOM` ;
- **chemins** (vérifié au build) : écrire `/html/<nom>.htm` dans le lien et dans l'`<iframe>`. Quartz traite `/` comme la racine de `content/` et réécrit le chemin en relatif (`../html/<nom>.htm`), ce qui fonctionne sur le site de projet `/krimpa/`. Ne **pas** écrire `/krimpa/html/…` : Quartz le transformerait en `../krimpa/html/…`, un lien cassé ;
- le lien « plein écran » est une balise HTML `<a … data-router-ignore>`. Sans cet attribut, le routeur SPA de Quartz (`enableSPA: true`) intercepterait le clic et chargerait la page dans son propre gabarit, où ses scripts ne s'exécuteraient pas. Cet attribut est documenté dans `docs/features/SPA Routing.md` ;
- la `description` et le résumé de la fiche sont **le seul texte indexé** par la recherche : y mettre les mots-clés utiles ;
- la page HTML est autonome : CSS et JS intégrés ou en chemins relatifs, bibliothèques externes uniquement depuis un CDN ;
- laisser `publishAssets` à sa valeur par défaut (`"all"`).

## Points à vérifier sur la version de Quartz installée

Ces points viennent de la documentation v4 ou de connaissances non retestées en v5 :
1. ~~`templates` est-il dans `ignorePatterns` par défaut ?~~ **Vérifié (v5.2.2)** : oui, dans `quartz.config.default.yaml` et dans le modèle `obsidian`. Le build de test n'a lu que `content/index.md`.
2. `npx quartz sync` existe-t-il toujours et fait-il commit + push ?
3. ~~Le workflow de déploiement GitHub Pages recommandé par la documentation officielle.~~ **Fait** : repris tel quel depuis `docs/hosting.md`.
4. ~~Le wikilink échappé `[[fiche\|alias]]` dans un tableau s'affiche-t-il correctement ?~~ **Vérifié au build** : oui, le lien est correct et l'alias s'affiche.
5. ~~Une balise `<iframe>` écrite dans une fiche Markdown est-elle conservée ?~~ **Vérifié au build** : oui. Son `src` est réécrit comme un lien interne (voir « Pages HTML »).
6. Chemin de la page HTML : **réglé au build**, avec `/html/<nom>.htm` et l'extension `.htm` (voir « Pages HTML »). Reste à confirmer **sur le site publié** que la page s'ouvre en plein écran et dans l'iframe.
7. L'affichage de l'iframe sur mobile (hauteur, défilement).
8. ~~Le build génère un fichier `public/CNAME`.~~ **Réglé** : ce fichier vient du plugin `@quartz-community/cname`, qui est désactivé. Il n'est donc plus généré.

## Prochaines étapes

1. Créer le dépôt à partir du dépôt officiel Quartz, installer les dépendances.
2. Configurer le site : titre, langue `fr-FR`, `baseUrl` = `<utilisateur>.github.io/<dépôt>`.
3. Ajouter le workflow GitHub Actions de déploiement et activer Pages (source : GitHub Actions).
4. Créer l'arborescence `content/` ci-dessus et les modèles `fiche.md` et `fiche-html.md`.
5. Rédiger `index.md` et 3 à 5 premières fiches, puis vérifier en local (`npx quartz build --serve`), sur le site publié et sur mobile.
6. Déclarer `content/` comme coffre Obsidian et `content/templates/` comme dossier de modèles.

## Règles de travail pour l'assistant

- Distinguer ce qui est vérifié (source consultée, citée), ce qui relève de connaissances non vérifiées, et ce qui est une déduction.
- Ne jamais inventer de noms d'options, de commandes, de champs de configuration ou d'URL : vérifier dans la documentation officielle de Quartz (quartz.jzhao.xyz) ou dans le code du dépôt.
- Préférer « je ne sais pas » à une réponse approximative ; poser une question si une information manque.
- Rester simple : ne rien ajouter (CI, statuts, workflow) qui n'a pas été demandé.
