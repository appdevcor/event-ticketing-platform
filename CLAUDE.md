# CLAUDE.md : Plan de tâches (géré par @planner)

Source de vérité de l'architecture : `architect.md` (validé). Ce fichier ne détaille que la tranche **T0 Fondations**. Les tranches suivantes seront planifiées une par une, après la revue de la précédente.

## Rappel du projet

Plateforme de billetterie/réservation d'événements, construite pour **apprendre l'ingénierie backend avancée** (pas un produit commercial).
Stack : Laravel (dernière version stable), PHP 8.4+, PostgreSQL 17+, Redis, Docker Compose écrit à la main (pas de Sail), Caddy + PHP-FPM, Pest, GitHub Actions.
Principe structurant : **PostgreSQL décide, Redis accélère.**

Profil de l'utilisateur : développeur PHP, SQL intermédiaire, **débutant** en Docker, Linux, tests et Git. L'utilisateur écrit le code lui-même ; l'assistant guide, challenge et relit.

## Conventions

- Code, noms de fichiers, identifiants, commits, PR et documentation du projet : **en anglais**. Échanges avec l'assistant : en français.
- **Aucune mention de génération par IA** dans le code, les commentaires, les commits, les PR ou la documentation (pas de `Co-Authored-By` d'IA, pas de signature).
- **Commits conventionnels** : `type(scope): description` à l'impératif, en anglais (`feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `ci`, `build`). Exemple : `build(docker): add php-fpm image`.
- **Git** : trunk-based. `main` toujours vert. Une branche courte par tâche (`feat/...`, `chore/...`, `ci/...`), une PR vers `main` même en solo, merge par squash après CI verte.
- **Une fonctionnalité n'est terminée qu'avec ses tests.** Tout bug corrigé reçoit un test de régression.
- Aucun secret dans Git : `.env` ignoré, `.env.example` versionné sans valeur sensible.
- Simplicité d'abord : pas de sur-ingénierie, pas d'optimisation prématurée.

## Cycle de travail par tâche

Chaque tâche suit les 7 étapes, dans l'ordre :

1. **Besoin** : l'assistant explique le problème à résoudre.
2. **Concept** : explication courte des notions à apprendre.
3. **Proposition** : l'utilisateur propose sa solution (avant tout code).
4. **Discussion** : compromis, alternatives, validation.
5. **Implémentation** : par l'utilisateur, sur une branche dédiée.
6. **Revue** : relecture (`@reviewer`), corrections.
7. **Note d'apprentissage** : l'utilisateur rédige 5 à 10 lignes dans `docs/learning/T0.x-<slug>.md` (en anglais) une fois la tâche mergée.

Cas de blocage : en cas d'erreur inattendue, passer par `@debugger` plutôt que de deviner.

> Note : `docs/learning/` est un dossier de notes personnelles (convention du plan, non prévue dans l'arborescence d'`architect.md`). Il est créé par l'utilisateur lors de T0.1.

## Vue d'ensemble de T0 (semaines 1-2, ~30 h)

| Tâche | Sujet | Heures | Dépend de |
|---|---|---|---|
| T0.1 | Terminal Linux, dépôt Git, premier commit | 2.5 | aucune |
| T0.2 | Workflow Git : branches, PR, commits conventionnels | 2 | T0.1 |
| T0.3 | Image Docker PHP-FPM | 3 | T0.2 |
| T0.4 | Compose : Caddy + PHP-FPM | 3.5 | T0.3 |
| T0.5 | Compose : PostgreSQL, Redis, Mailpit | 3 | T0.4 |
| T0.6 | Laravel installé et connecté aux services | 3.5 | T0.5 |
| T0.7 | Pest et premiers tests sur vrai PostgreSQL | 3 | T0.6 |
| T0.8 | Endpoint `/up` et healthchecks | 2 | T0.7 |
| T0.9 | Logs JSON avec request-id | 3.5 | T0.7, T0.8 |
| T0.10 | CI GitHub Actions (Pint + Pest) | 3 | T0.7 (T0.9 recommandée) |
| T0.11 | README, commandes, clôture de tranche | 1 | T0.1 à T0.10 |

Total : 30 h.

Ordre conseillé : T0.1 à T0.9 dans l'ordre. T0.10 peut être faite avant T0.9 si la CI est plus motivante tôt ; dans ce cas, T0.9 doit garder la CI verte.

---

## T0.1 : Terminal Linux, dépôt Git, premier commit

**Objectif :** disposer d'un dépôt Git propre dans `/home/abdessamad/backend-engineering`, relié à un dépôt GitHub, avec les fichiers de cadrage versionnés.

**Concepts à apprendre :** navigation et commandes de base du shell (`cd`, `ls`, `cat`, `grep`, `less`, `chmod`, variables d'environnement, `$PATH`) ; working tree / index / commit ; `.gitignore` ; différence entre `git` local et GitHub ; clé SSH pour GitHub.

**Étapes concrètes :**
1. Vérifier les outils installés : `git --version`, `docker --version`, `docker compose version`. Installer ce qui manque (Docker Engine + plugin Compose ; ajouter l'utilisateur au groupe `docker`).
2. `git init` avec la branche par défaut `main`. Configurer `user.name`, `user.email`.
3. Créer une clé SSH, l'ajouter à GitHub, tester `ssh -T git@github.com`.
4. Créer le dépôt GitHub (privé ou public, au choix), ajouter le remote.
5. Écrire un `.gitignore` minimal (`.env`, `vendor/`, `node_modules/`, `storage/*.key`, fichiers d'éditeur) et un `.gitattributes` optionnel.
6. Créer `docs/learning/` et le dossier `ideas/`, `reviews/`, `bugs/` (avec `.gitkeep`).
7. Premier commit (`chore: initial project documents`) contenant `architect.md`, `CLAUDE.md`, `.gitignore`. Pousser sur `main`.

**Critères d'acceptation :**
- [ ] `git status` est propre et `git log --oneline` montre au moins un commit conventionnel.
- [ ] `git remote -v` affiche le dépôt GitHub et `git push` fonctionne sans mot de passe (SSH).
- [ ] `docker run --rm hello-world` s'exécute sans `sudo`.
- [ ] `.env` est dans `.gitignore` (vérifiable avec `git check-ignore -v .env`).
- [ ] L'utilisateur sait expliquer à voix haute la différence entre working tree, index et commit.

**Estimation :** 2.5 h
**Dépend de :** aucune
**Note d'apprentissage attendue :** `docs/learning/T0.1-git-and-shell.md` : les commandes shell utilisées et leur rôle, le chemin d'un fichier de « modifié » à « poussé », pourquoi `.env` n'est jamais commité.

---

## T0.2 : Workflow Git : branches courtes, PR, commits conventionnels

**Objectif :** pratiquer le workflow qui sera utilisé pendant tout le projet (branche, commits, PR, merge) sur un petit changement réel.

**Concepts à apprendre :** branche, `merge` vs `squash` vs `rebase` (survol) ; pull request et revue ; commits conventionnels ; protection de branche ; `git diff`, `git restore`, `git stash`, `git reflog` (filet de sécurité).

**Étapes concrètes :**
1. Créer la branche `docs/pr-template`.
2. Ajouter `.github/pull_request_template.md` (sections : What, Why, How tested, Checklist avec « tests added », « no secrets »).
3. Ajouter une courte section « Git workflow » dans un futur `CONTRIBUTING.md` ou dans le README minimal (3 à 5 lignes : nommage des branches, format des commits).
4. Faire 2 commits conventionnels, pousser, ouvrir une PR vers `main`, relire son propre diff dans GitHub, merger en squash.
5. Dans GitHub, activer : squash merge uniquement, suppression automatique de la branche après merge. Activer la protection de `main` (PR requise) ; la vérification CI obligatoire sera ajoutée en T0.10.
6. Revenir sur `main` en local (`git switch main && git pull`), supprimer la branche locale.
7. Exercice de sécurité : créer une branche jetable, faire une erreur volontaire, la récupérer avec `git restore`, `git reset` ou `git reflog`.

**Critères d'acceptation :**
- [ ] Une PR mergée en squash est visible sur GitHub ; son titre est un commit conventionnel.
- [ ] `main` refuse un push direct (protection active).
- [ ] La branche de la PR est supprimée sur le remote et en local.
- [ ] L'utilisateur a récupéré avec succès un commit « perdu » via `git reflog`.
- [ ] Aucun commit ni PR ne contient de mention de génération par IA.

**Estimation :** 2 h
**Dépend de :** T0.1
**Note d'apprentissage attendue :** `docs/learning/T0.2-git-workflow.md` : le workflow en 6 lignes, pourquoi des branches courtes, la différence squash/merge, comment annuler une erreur.

---

## T0.3 : Image Docker PHP-FPM

**Objectif :** construire à la main une image PHP 8.4 FPM avec les extensions nécessaires à Laravel et PostgreSQL, et la lancer.

**Concepts à apprendre :** image vs conteneur ; `Dockerfile` (`FROM`, `RUN`, `COPY`, `WORKDIR`, `USER`) ; couches et cache de build ; `docker build`, `run`, `exec`, `logs`, `ps`, `images` ; pourquoi ne pas tourner en root ; PHP-FPM (pool, port 9000) ; extensions PHP (`pdo_pgsql`, `redis`, `pcntl`, `intl`, `zip`, `bcmath`...) ; Composer copié depuis l'image officielle.

**Étapes concrètes :**
1. Vérifier la dernière version stable de PHP (8.4+) et de l'image `php:*-fpm` correspondante ; la noter dans la PR.
2. Créer `docker/php/Dockerfile` : base `php:8.4-fpm` (ou version retenue), dépendances système, extensions PHP, Composer, utilisateur non-root dont l'UID/GID correspond à celui de l'hôte (argument de build) pour éviter les problèmes de permissions sur le volume monté.
3. Ajouter un `.dockerignore`.
4. Construire l'image, la lancer, entrer dedans (`docker exec -it`), vérifier `php -v`, `php -m`, `composer --version`, `whoami`.
5. Comprendre le cache : modifier une ligne tardive du Dockerfile et observer quelles couches sont reconstruites.
6. Commit sur une branche `build/php-image`, PR, merge.

**Critères d'acceptation :**
- [ ] `docker build -f docker/php/Dockerfile -t ticketing-app:dev .` réussit.
- [ ] Dans le conteneur : `php -v` indique 8.4 ou plus, `php -m` contient `pdo_pgsql`, `redis`, `pcntl`, `intl`, `bcmath`, `zip`.
- [ ] `whoami` dans le conteneur ne renvoie pas `root`.
- [ ] Un fichier créé dans un volume monté appartient à l'utilisateur de l'hôte (pas à root).
- [ ] L'utilisateur explique pourquoi l'ordre des instructions du Dockerfile affecte la vitesse de build.

**Estimation :** 3 h
**Dépend de :** T0.2
**Note d'apprentissage attendue :** `docs/learning/T0.3-php-image.md` : image vs conteneur, rôle de chaque bloc du Dockerfile, le cache de couches, le piège des permissions sur volumes.

---

## T0.4 : Compose : Caddy + PHP-FPM

**Objectif :** un `compose.yaml` minimal où Caddy sert une page PHP via FastCGI vers PHP-FPM.

**Concepts à apprendre :** Docker Compose (services, volumes, réseaux, `depends_on`, `ports`) ; réseau interne et résolution par nom de service ; bind mount vs volume nommé ; Caddyfile (`root`, `php_fastcgi`, `file_server`, `encode`) ; FastCGI ; `docker compose up/down/logs/exec/ps`.

**Étapes concrètes :**
1. Créer un dossier `public/` temporaire avec un `index.php` affichant `phpinfo()` ou un simple « hello » (sera remplacé par Laravel en T0.6).
2. Écrire `docker/caddy/Caddyfile` : écoute sur `:80`, `root` vers `/var/www/html/public`, `php_fastcgi app:9000`, `encode gzip`, logs sur stdout. Ajouter un `request_id` généré par Caddy transmis à PHP via l'en-tête `X-Request-Id` (`header_up`), sans le détailler : il sera exploité en T0.9.
3. Écrire `compose.yaml` avec deux services : `app` (build du Dockerfile, code monté en volume) et `web` (image `caddy`, Caddyfile monté, port `8080:80`).
4. Lancer, ouvrir `http://localhost:8080`, lire les logs des deux services.
5. Expérimenter volontairement : arrêter `app` et observer l'erreur 502 de Caddy ; la corriger.
6. Branche `build/compose-web`, PR, merge.

**Critères d'acceptation :**
- [ ] `docker compose up -d` démarre `web` et `app` ; `docker compose ps` les montre « running ».
- [ ] `curl -i http://localhost:8080` renvoie `200` et la page PHP.
- [ ] Arrêter `app` produit un `502` ; le redémarrer rétablit le `200` (démontré par l'utilisateur).
- [ ] Modifier `index.php` sur l'hôte est reflété sans reconstruire l'image.
- [ ] `docker compose down` puis `up` fonctionne sans erreur.
- [ ] Un seul port est exposé sur l'hôte (celui de Caddy).

**Estimation :** 3.5 h
**Dépend de :** T0.3
**Note d'apprentissage attendue :** `docs/learning/T0.4-compose-web.md` : schéma navigateur → Caddy → FastCGI → PHP-FPM, rôle du réseau Compose, bind mount vs volume nommé, ce que signifie un 502.

---

## T0.5 : Compose : PostgreSQL, Redis, Mailpit

**Objectif :** ajouter les services de données et de mail local, avec persistance, healthchecks et variables d'environnement propres.

**Concepts à apprendre :** volumes nommés et persistance ; `healthcheck` et `depends_on: condition: service_healthy` ; variables d'environnement et fichier `.env` (Compose vs Laravel) ; `psql`, `redis-cli` ; ne pas exposer inutilement les ports ; AOF Redis (persistance) ; Mailpit (SMTP 1025, UI 8025).

**Étapes concrètes :**
1. Ajouter `postgres` (image 17 ou plus récente stable, volume nommé `postgres_data`, `POSTGRES_DB/USER/PASSWORD` depuis `.env`, healthcheck `pg_isready`).
2. Ajouter `redis` (volume nommé, `--appendonly yes`, healthcheck `redis-cli ping`).
3. Ajouter `mailpit` (ports SMTP interne, UI exposée sur `8025`).
4. Créer `.env.example` (variables Compose : noms de base, utilisateur, mots de passe de développement factices) ; copier vers `.env` (ignoré). Définir une base `ticketing` pour le dev et une base `ticketing_test` pour les tests (script d'initialisation dans `docker/postgres/` monté dans `/docker-entrypoint-initdb.d`).
5. Faire dépendre `app` de `postgres` et `redis` en mode `service_healthy`.
6. Se connecter : `docker compose exec postgres psql -U ... -d ticketing -c '\l'`, `docker compose exec redis redis-cli ping`.
7. Prouver la persistance : créer une table de test, `docker compose down`, `up`, vérifier que la table existe ; puis `down -v` et constater la perte.
8. Branche `build/compose-data`, PR, merge.

**Critères d'acceptation :**
- [ ] `docker compose ps` affiche `postgres` et `redis` en `healthy`.
- [ ] Les bases `ticketing` et `ticketing_test` existent (`\l`).
- [ ] `redis-cli ping` renvoie `PONG` ; `redis-cli config get appendonly` renvoie `yes`.
- [ ] L'interface Mailpit est accessible sur `http://localhost:8025`.
- [ ] Les données PostgreSQL survivent à `down`/`up` et disparaissent avec `down -v` (démontré).
- [ ] Les ports de PostgreSQL et Redis ne sont pas exposés sur l'hôte, ou le sont uniquement sur `127.0.0.1` (choix justifié dans la PR).
- [ ] `.env` n'est pas versionné ; `.env.example` l'est et ne contient aucun secret réel.

**Estimation :** 3 h
**Dépend de :** T0.4
**Note d'apprentissage attendue :** `docs/learning/T0.5-compose-data.md` : volumes et persistance, healthcheck vs simple démarrage, pourquoi Redis n'est jamais la source de vérité, ce que fait `down -v`.

---

## T0.6 : Laravel installé et connecté aux services

**Objectif :** installer la dernière version stable de Laravel dans le projet, via le conteneur, connecté à PostgreSQL, Redis et Mailpit.

**Concepts à apprendre :** structure d'un projet Laravel (`app/`, `config/`, `routes/`, `database/`, `bootstrap/`, `storage/`) ; `.env` Laravel et `APP_KEY` ; drivers `pgsql`, cache/session/queue sur Redis ; migrations ; `artisan` ; permissions de `storage/` et `bootstrap/cache` ; exécuter les commandes dans le conteneur et non sur l'hôte.

**Étapes concrètes :**
1. Vérifier la version stable actuelle de Laravel et ses prérequis PHP ; la noter dans la PR.
2. Installer dans le conteneur `app` (`composer create-project laravel/laravel` dans un dossier temporaire puis déplacement à la racine, ou équivalent), sans écraser `architect.md`, `CLAUDE.md`, `compose.yaml`, `docker/`. Ne pas installer de kit de démarrage (Breeze/Jetstream).
3. Configurer `.env.example` Laravel : `DB_CONNECTION=pgsql`, `DB_HOST=postgres`, `CACHE_STORE=redis`, `QUEUE_CONNECTION=redis`, `SESSION_DRIVER=redis`, `REDIS_HOST=redis`, `MAIL_HOST=mailpit`, `MAIL_PORT=1025`. Générer `APP_KEY` dans le `.env` local.
4. Remplacer `public/index.php` temporaire par celui de Laravel (déjà fourni) ; vérifier la page d'accueil via Caddy.
5. Lancer `php artisan migrate` ; inspecter les tables créées avec `psql`.
6. Vérifier Redis : `php artisan tinker` puis `Cache::put('k','v',60)` et `Cache::get('k')` ; relire la clé avec `redis-cli`.
7. Vérifier le mail : envoyer un mail de test depuis `tinker` et le voir dans Mailpit.
8. Activer le log des requêtes SQL en dev, uniquement quand c'est utile, pour garder la visibilité sur le SQL (principe d'`architect.md`).
9. Branche `feat/laravel-install`, PR, merge.

**Critères d'acceptation :**
- [ ] `http://localhost:8080` affiche la page d'accueil Laravel.
- [ ] `php artisan about` (dans le conteneur) indique PHP 8.4+, driver `pgsql`, cache/queue/session sur `redis`.
- [ ] `php artisan migrate:status` montre les migrations exécutées ; les tables sont visibles dans `psql`.
- [ ] Une valeur écrite via `Cache::put` est lisible avec `redis-cli`.
- [ ] Un e-mail de test apparaît dans Mailpit.
- [ ] `.env` et `vendor/` ne sont pas versionnés ; `composer.lock` l'est.
- [ ] Aucune commande PHP/Composer n'a besoin d'être installée sur l'hôte.

**Estimation :** 3.5 h
**Dépend de :** T0.5
**Note d'apprentissage attendue :** `docs/learning/T0.6-laravel-install.md` : où vit chaque réglage (`.env`, `config/`), ce que contient chaque dossier, comment Laravel atteint PostgreSQL/Redis/Mailpit par nom de service, pourquoi `composer.lock` est versionné.

---

## T0.7 : Pest et premiers tests sur vrai PostgreSQL

**Objectif :** installer Pest, configurer l'environnement de test sur le vrai PostgreSQL (base `ticketing_test`) et écrire les premiers tests.

**Concepts à apprendre :** pourquoi tester sur le vrai PostgreSQL (pas SQLite, ADR-017) ; Pest (`test()`, `it()`, `expect()`) ; Unit vs Feature ; `phpunit.xml` et `.env.testing` ; `RefreshDatabase` et son principe (transaction englobante) ; niveau d'isolation entre test et base de dev ; cycle rouge, vert, refactor.

**Étapes concrètes :**
1. Installer Pest et le plugin Laravel via Composer (dans le conteneur) ; initialiser (`pest --init` ou équivalent), supprimer les exemples générés.
2. Configurer `phpunit.xml` (ou `.env.testing`) : `APP_ENV=testing`, `DB_DATABASE=ticketing_test`, Redis sur une base logique distincte (ex. `REDIS_DB=1`), mail en `array`, cache en `array` ou Redis dédié.
3. Activer `RefreshDatabase` pour les tests Feature dans `tests/Pest.php`.
4. Écrire un premier test **qui échoue** : par exemple, un test unitaire sur une petite fonction utilitaire de votre choix, ou un test Feature vérifiant que `GET /` répond 200. Le voir échouer, puis le faire passer.
5. Écrire un test qui prouve que les tests tournent sur PostgreSQL : vérifier `DB::connection()->getDriverName() === 'pgsql'` et `DB::getDatabaseName() === 'ticketing_test'`.
6. Vérifier que la base de dev `ticketing` n'est jamais touchée par les tests (compter des lignes avant/après).
7. Créer les dossiers `tests/Unit` et `tests/Feature` ; les dossiers `Architecture`, `Concurrency`, etc. viendront plus tard.
8. Commande standard documentée : `docker compose exec app php artisan test` (ou `./vendor/bin/pest`).
9. Branche `test/pest-setup`, PR, merge.

**Critères d'acceptation :**
- [ ] `docker compose exec app ./vendor/bin/pest` passe, avec au moins 1 test Unit et 2 tests Feature.
- [ ] Un test démontre que la connexion est `pgsql` sur la base `ticketing_test`.
- [ ] Les données de la base `ticketing` ne changent pas après une exécution des tests.
- [ ] L'utilisateur a observé au moins un test rouge avant de le rendre vert (capture ou note dans la PR).
- [ ] L'utilisateur explique ce que `RefreshDatabase` fait, et pourquoi il ne conviendra pas aux tests de concurrence de T2.

**Estimation :** 3 h
**Dépend de :** T0.6
**Note d'apprentissage attendue :** `docs/learning/T0.7-pest-setup.md` : pourquoi pas SQLite, structure d'un test Pest, ce que fait `RefreshDatabase`, cycle rouge/vert.

---

## T0.8 : Endpoint `/up` et healthchecks

**Objectif :** exposer `/up` (liveness), le tester, et l'utiliser comme healthcheck Docker de l'application.

**Concepts à apprendre :** liveness vs readiness (`/up` ici, `/health/ready` plus tard en T10) ; `health:` de Laravel dans `bootstrap/app.php` ; codes HTTP ; healthcheck Compose (`test`, `interval`, `retries`, `start_period`) ; pourquoi `/up` ne doit **pas** interroger PostgreSQL ni Redis.

**Étapes concrètes :**
1. Constater que Laravel déclare déjà `/up` ; lire où et comment (`bootstrap/app.php`), et ce que la route renvoie.
2. Écrire un test Feature : `GET /up` renvoie `200` sans authentification et sans toucher à la base (aucune requête SQL exécutée, vérifiable avec `DB::enableQueryLog()`).
3. Ajouter un healthcheck sur `web` (Caddy, via `wget`/`curl` vers `/up` ou un contrôle équivalent disponible dans l'image) et/ou sur `app` (au minimum : le processus PHP-FPM répond).
4. Écrire dans `README` (section provisoire) la différence liveness/readiness en 3 lignes.
5. Exercice : arrêter PostgreSQL, constater que `/up` reste `200` (liveness) ; relancer.
6. Branche `feat/up-endpoint`, PR, merge.

**Critères d'acceptation :**
- [ ] `curl -i http://localhost:8080/up` renvoie `200`.
- [ ] Un test Pest couvre `/up` (statut 200, aucune requête SQL).
- [ ] `docker compose ps` affiche le service healthcheck concerné en `healthy`.
- [ ] Avec PostgreSQL arrêté, `/up` répond toujours `200` (démontré).
- [ ] L'utilisateur explique la différence entre liveness et readiness et pourquoi les confondre est risqué.

**Estimation :** 2 h
**Dépend de :** T0.7
**Note d'apprentissage attendue :** `docs/learning/T0.8-up-endpoint.md` : liveness vs readiness, à quoi sert un healthcheck Docker, risques d'un healthcheck trop strict (redémarrages en cascade).

---

## T0.9 : Logs JSON avec request-id

**Objectif :** produire des logs JSON sur stdout, chaque ligne portant un `request_id` repris de `X-Request-Id` (ou généré), renvoyé dans la réponse.

**Concepts à apprendre :** logs structurés vs texte ; stdout/stderr et `docker compose logs` (12-factor) ; canaux de log Laravel (`config/logging.php`, `stderr`/`stack`, formateur Monolog JSON) ; middleware HTTP ; `Context` de Laravel (propagation aux logs et plus tard aux jobs) ; corrélation d'une requête de bout en bout.

**Étapes concrètes :**
1. Configurer un canal de log écrivant en JSON sur stdout/stderr (Monolog `JsonFormatter`), sélectionné par `LOG_CHANNEL`/`LOG_STACK` dans `.env`. Garder un format lisible possible en local si souhaité (au choix, documenté).
2. Écrire un middleware (dans `app/Shared/Observability/Http/`, ou `app/Http/Middleware/` si la structure Shared n'existe pas encore ; à trancher en discussion) qui : lit `X-Request-Id` ; le valide (longueur et caractères limités, sinon génère un UUID) ; l'ajoute à `Context` ; l'ajoute à l'en-tête de réponse `X-Request-Id`.
3. Enregistrer le middleware globalement dans `bootstrap/app.php`.
4. S'assurer que la ligne de log contient `request_id` (via `Context`, qui l'ajoute aux logs) et au moins : horodatage, niveau, message, `request_id`.
5. Écrire les tests Feature : (a) une requête sans en-tête reçoit un `X-Request-Id` généré dans la réponse ; (b) un en-tête valide fourni est repris à l'identique ; (c) un en-tête invalide (trop long ou caractères dangereux, ex. retour à la ligne) est remplacé ; (d) un log émis pendant la requête contient le même `request_id`.
6. Vérifier manuellement : `curl -H 'X-Request-Id: demo-123' ...` puis `docker compose logs app | grep demo-123`.
7. Ne jamais logger de données sensibles (mots de passe, jetons) ; l'écrire dans la note.
8. Branche `feat/json-logs`, PR, merge.

**Critères d'acceptation :**
- [ ] Chaque ligne de log applicative est un JSON valide (`docker compose logs app` pipé dans `jq` sans erreur sur les lignes applicatives).
- [ ] Une requête avec `X-Request-Id: demo-123` produit une réponse portant `X-Request-Id: demo-123` et des logs contenant `"request_id":"demo-123"`.
- [ ] Sans en-tête, un identifiant est généré et renvoyé.
- [ ] Un en-tête malformé est rejeté (remplacé), jamais recopié tel quel dans les logs.
- [ ] Au moins 4 tests Pest couvrent les cas (a) à (d).
- [ ] Les tests précédents (`/up`, etc.) restent verts.

**Estimation :** 3.5 h
**Dépend de :** T0.7, T0.8
**Note d'apprentissage attendue :** `docs/learning/T0.9-json-logs.md` : pourquoi du JSON et pourquoi stdout, comment un request-id permet de suivre une requête, risque de l'injection de logs via en-tête, rôle de `Context`.

---

## T0.10 : CI GitHub Actions (Pint + Pest)

**Objectif :** un workflow qui, à chaque push et PR, exécute Pint et Pest avec de vrais services PostgreSQL et Redis, rendu obligatoire pour merger.

**Concepts à apprendre :** intégration continue ; workflow, job, step, runner ; `on: push/pull_request` ; service containers (et healthchecks) ; cache Composer ; différence entre l'environnement CI et Compose local ; Laravel Pint (style de code, mode `--test`) ; secrets vs variables (aucun secret requis ici) ; checks requis dans la protection de branche.

**Étapes concrètes :**
1. Vérifier que Pint est présent (inclus dans Laravel) et lancer `docker compose exec app ./vendor/bin/pint --test` localement ; corriger le style si nécessaire (commit `style:` ou `chore:` séparé).
2. Créer `.github/workflows/ci.yml` : déclenché sur `pull_request` et `push` sur `main` ; setup PHP 8.4+ (`shivammathur/setup-php`) avec les mêmes extensions que l'image ; cache Composer ; services `postgres:17+` et `redis` avec healthchecks ; `composer install --no-interaction --prefer-dist` ; copie de `.env.example` en `.env`, `php artisan key:generate` ; étape Pint (`pint --test`) ; étape Pest.
3. Les variables d'environnement de CI pointent vers `127.0.0.1` (service containers) et non vers les noms Compose ; documenter cette différence en commentaire de session ou dans la note (pas de commentaires superflus dans le YAML).
4. Définir les permissions minimales du workflow (`permissions: contents: read`) et épingler les actions par version majeure.
5. Ouvrir une PR `ci/github-actions`. Observer la CI échouer au moins une fois si possible (ex. `pint --test` sur un fichier mal formaté, puis corriger) pour comprendre les logs.
6. Une fois verte, activer dans GitHub : check `ci` requis avant merge sur `main`.
7. Vérifier le blocage : ouvrir une PR de test avec un test volontairement rouge ; constater que le merge est bloqué ; la fermer.
8. Ajouter un badge de statut CI au README (en T0.11).

**Critères d'acceptation :**
- [ ] Le workflow est vert sur la PR et sur `main` après merge.
- [ ] Le job exécute réellement Pint (`--test`) et Pest contre PostgreSQL et Redis (visible dans les logs de CI : driver `pgsql`).
- [ ] Une PR avec un test rouge ou un style invalide est **bloquée** au merge (démontré).
- [ ] Le workflow a des `permissions` minimales et ne requiert aucun secret.
- [ ] Durée d'exécution notée dans la PR (cible : moins de 5 minutes avec le cache).
- [ ] L'utilisateur sait relire un log de CI et reproduire localement une erreur de CI.

**Estimation :** 3 h
**Dépend de :** T0.7 (T0.8 et T0.9 recommandées pour avoir plus de tests)
**Note d'apprentissage attendue :** `docs/learning/T0.10-ci.md` : ce que garantit une CI verte et ce qu'elle ne garantit pas, service containers, différences CI/local, comment déboguer une CI rouge.

---

## T0.11 : README, commandes et clôture de tranche

**Objectif :** qu'un nouveau venu (ou vous dans 6 mois) puisse cloner et lancer le projet en moins de 10 minutes ; clôturer T0 proprement.

**Concepts à apprendre :** documentation de démarrage ; reproductibilité (cloner, `up`, `migrate`, `test`) ; tags Git et jalons ; rétrospective courte.

**Étapes concrètes :**
1. Rédiger `README.md` (anglais) : description en 3 lignes, prérequis, démarrage rapide (`cp .env.example .env`, `docker compose up -d`, installation Composer, `key:generate`, `migrate`), commandes courantes (tests, Pint, logs, shell dans le conteneur, reset complet), URLs locales (app, Mailpit), badge CI, lien vers `architect.md`.
2. Option simple : un `Makefile` ou quelques alias documentés, **seulement** si cela évite la répétition ; pas obligatoire.
3. **Test de reproductibilité** : cloner le dépôt dans un dossier temporaire, suivre uniquement le README, corriger tout ce qui manque.
4. Passer en revue la checklist de fin de tranche (ci-dessous) avec l'assistant.
5. Poser le tag `t0-foundations` sur `main` et le pousser.
6. Rédiger une courte rétrospective (`docs/learning/T0-retro.md`) : ce qui a été difficile, temps réel passé vs estimé, ajustements pour T1.

**Critères d'acceptation :**
- [ ] Un clone neuf, suivant uniquement le README, arrive à `/up` en `200` et à des tests verts.
- [ ] Le README contient le badge CI, qui est vert.
- [ ] Le tag `t0-foundations` existe sur GitHub.
- [ ] Les notes d'apprentissage T0.1 à T0.10 et la rétrospective existent dans `docs/learning/`.
- [ ] Toute la checklist de fin de tranche est cochée.

**Estimation :** 1 h
**Dépend de :** T0.1 à T0.10
**Note d'apprentissage attendue :** `docs/learning/T0-retro.md` : écarts d'estimation, 3 points durs, 3 points à améliorer pour T1.

---

## Checklist de fin de tranche T0 : ce que l'utilisateur doit savoir expliquer

**Git**
- [ ] La différence entre working tree, index et commit, et comment annuler une erreur (`restore`, `reset`, `reflog`).
- [ ] Pourquoi des branches courtes et des PR même en solo ; ce qu'apporte le squash merge.
- [ ] Le format d'un commit conventionnel et son intérêt.

**Docker et Compose**
- [ ] La différence entre image et conteneur, et le rôle du cache de couches.
- [ ] Le chemin d'une requête : navigateur → Caddy → FastCGI → PHP-FPM → PostgreSQL/Redis.
- [ ] Volume nommé vs bind mount ; ce que fait `down` vs `down -v`.
- [ ] Pourquoi un healthcheck est différent d'un simple démarrage.
- [ ] Pourquoi le conteneur ne tourne pas en root et ce que cela implique pour les permissions.

**Laravel et tests**
- [ ] Où se règlent la base, le cache, la file et le mail (`.env`, `config/`).
- [ ] Pourquoi les tests tournent sur un vrai PostgreSQL (ADR-017) et ce que fait `RefreshDatabase`.
- [ ] Pourquoi `RefreshDatabase` ne conviendra pas aux tests de concurrence.
- [ ] Liveness (`/up`) vs readiness (`/health/ready`, plus tard).

**Observabilité et CI**
- [ ] Pourquoi des logs JSON sur stdout et comment le `request_id` permet de suivre une requête.
- [ ] Ce que garantit une CI verte (et ce qu'elle ne garantit pas) ; comment reproduire localement une CI rouge.
- [ ] Pourquoi `main` est protégé et que le check CI est requis.

**Vérifications finales**
- [ ] Clone neuf + README : application fonctionnelle en moins de 10 minutes.
- [ ] CI verte sur `main` ; tag `t0-foundations` posé.
- [ ] Aucune mention d'IA dans le dépôt ; aucun secret versionné.

---

## Aperçu des tranches suivantes (non détaillées)

- **T1 Catalogue (sem. 3-4)** : organisations, événements, types de billets, catalogue public, UUIDv7, `Money`, problem+json, OpenAPI (Scramble), tests d'architecture.
- **T2 Réservation et concurrence (sem. 5-7)** : `inventory_pools`, réservations, labo de concurrence (5 stratégies), tests parallèles, `Idempotency-Key`.
- **T3 Expiration et files (sem. 8-9)** : job retardé `ExpireReservation`, Horizon, sweeper `SKIP LOCKED`, scheduler, `Clock`.
- **T4 Premier déploiement (sem. 10)** : VPS durci, image de prod multi-stage, Caddy TLS, CD par SSH, backup nocturne (VPS à choisir avant).
- **T5 Paiement Stripe (sem. 11-13)** : Checkout Session, webhooks entrants, machines à états Order/Payment, paiement tardif, remboursement total.
- **T6 Outbox, billets, emails (sem. 14-15)** : démo de la perte avec `afterCommit`, outbox + relay, billets PDF/QR, emails, retries et DLQ.
- **T7 Multi-organisateurs (sem. 16)** : membres, rôles, policies, tests d'isolation, check-in atomique.
- **T8 API publique et webhooks sortants (sem. 17-18)** : clients d'API, rate limiting par paliers, webhooks HMAC, rotation de secret, protection SSRF.
- **T9 Montée en charge (sem. 19)** : cache catalogue, ETag, anti-stampede, limiteurs, 3 scénarios k6, rapport.
- **T10 Observabilité et exploitation (sem. 20)** : `/metrics`, Grafana, alertes, `/health/ready`, test de restauration, runbooks.
