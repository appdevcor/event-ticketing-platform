# architect.md — Event Booking & Ticketing Platform

> Statut : **validé**. Projet d'apprentissage de l'ingénierie backend avancée (pas un produit commercial).
> Budget : environ 15 h/semaine pendant 20 semaines (~300 h).
> Le code, les noms et la documentation technique du projet sont en **anglais**.

## 0. Décisions validées

| # | Point | Décision |
|---|---|---|
| 1 | Placement | Admission générale uniquement (places numérotées = extension hors périmètre) |
| 2 | TTL de réservation | 15 min, avec expiration explicite de la session Stripe |
| 3 | Identifiants | UUIDv7 partout (exception : `outbox_messages.id` en `bigint identity`) |
| 4 | OpenAPI | Générée depuis le code avec Scramble |
| 5 | Observabilité | Prometheus + Grafana |
| 6 | RLS PostgreSQL | Extension optionnelle |
| 7 | Serveur web | Caddy + PHP-FPM (pas d'Octane) |
| 8 | Versions | Dernière version stable de Laravel, PHP 8.4+, PostgreSQL 17+ |
| 9 | Achat | Compte obligatoire |
| 10 | Remboursements | Totaux uniquement ; annulation d'événement avec remboursement en masse = optionnelle |
| 11 | Tests | Pest |
| 12 | VPS | **À trancher avant la tranche T4 (semaine 10)** : fournisseur, budget, nom de domaine |
| 13 | Rythme | 20 semaines, premier déploiement en semaine 10 |

Stack : Laravel, PostgreSQL, Redis, Docker. Stripe Checkout en mode test uniquement. API seule, front limité aux pages `success`/`cancel` (Blade, sans style). Exclus : marché/TVA/conformité fiscale, interface soignée, microservices, Kubernetes, multi-devises.

## 1. Vue d'ensemble

Une API REST versionnée qui permet :
- aux **organisateurs** (multi-tenant) de créer des événements, types de billets et quotas, gérer leurs membres, consulter les commandes, contrôler les billets à l'entrée et recevoir des webhooks ;
- aux **acheteurs** de consulter le catalogue, bloquer temporairement des places, payer via Stripe Checkout (test) et recevoir leurs billets PDF avec QR code par email ;
- aux **partenaires** d'utiliser une API publique avec des clés d'API.

```
   Clients: curl/Bruno, k6, partner apps, minimal Blade pages
                         │ HTTPS
                         ▼
               ┌───────────────────┐
               │ Caddy              │  TLS (prod), compression, request-id
               └─────────┬─────────┘
                         │ FastCGI
               ┌─────────▼─────────────────────────────────────────┐
               │ Laravel app (PHP-FPM) — modular monolith           │
               │  Identity │ Organizations │ Catalog │ Booking      │
               │  Ordering │ Payments │ Ticketing │ Notifications   │
               │  Integrations (API keys, outgoing webhooks)        │
               │  Shared kernel: Money, Outbox, Idempotency, Health │
               └───┬───────────────────────────────┬───────────────┘
                   │ SQL (transactions)             │ cache / rate limit / queues
          ┌────────▼─────────┐              ┌──────▼──────┐
          │ PostgreSQL        │              │ Redis        │
          │ SOURCE OF TRUTH   │              │ never source │
          │ + outbox_messages │              │ of truth     │
          └────────▲─────────┘              └──────▲──────┘
                   │ poll (SKIP LOCKED)             │ jobs
          ┌────────┴─────────┐   dispatch    ┌─────┴──────────────────────┐
          │ outbox-relay      │──────────────▶│ workers (Horizon)           │
          │ (long-running)    │               │ queues: critical, default,  │
          └──────────────────┘               │ notifications, documents,   │
          ┌──────────────────┐               │ webhooks-out                │
          │ scheduler         │──────────────▶└──┬─────────┬──────────┬────┘
          │ (sweeper, prune)  │                  ▼         ▼          ▼
          └──────────────────┘             Stripe API   SMTP     Organizer webhook
                                                        (Mailpit  endpoints (HMAC)
   Stripe ──webhooks──▶ POST /webhooks/stripe            in dev)

   Observability: JSON logs → stdout ; /metrics → Prometheus → Grafana ; /up, /health/ready
```

**Principe structurant : PostgreSQL décide, Redis accélère.**

## 2. Style architectural : monolithe modulaire

Une base de code, une base PostgreSQL, une image Docker lancée avec plusieurs commandes (web, worker, scheduler, relay).

| Module | Responsabilité | Tables possédées |
|---|---|---|
| Identity | Comptes, authentification (Sanctum), tokens | users, personal_access_tokens |
| Organizations | Tenants, membres, rôles, policies | organizations, organization_members |
| Catalog | Événements, types de billets | events, ticket_types |
| Booking | Inventaire, blocages temporaires, expiration | inventory_pools, reservations, reservation_items |
| Ordering | Commandes, machine à états, historique | orders, order_status_transitions |
| Payments | Passerelle Stripe, webhooks entrants, remboursements | payments, refunds, inbound_webhook_events |
| Ticketing | Billets, QR signés, PDF, contrôle d'accès | tickets |
| Notifications | Emails transactionnels | — |
| Integrations | Clés d'API, webhooks sortants | api_clients, webhook_endpoints, webhook_deliveries |
| Shared | Money, Clock, Outbox, Idempotency, ProblemDetails, Health, Observability | outbox_messages, processed_messages, idempotency_keys |

Structure interne d'un module :

```
app/Modules/Booking/
  Domain/          Eloquent models, state enums, value objects, domain exceptions, events
  Application/     Actions (one use case = one class), Queries
  Infrastructure/  Implementations of external ports (only for real boundaries)
  Http/            Controllers, FormRequests, API Resources, routes.php
  Jobs/  Console/  Listeners/
  BookingServiceProvider.php
```

Règles de frontière :
1. Seul le module propriétaire écrit dans ses tables.
2. Les autres modules passent par ses Actions, Queries publiques ou événements.
3. Les clés étrangères entre modules sont autorisées (base unique).
4. Les règles sont vérifiées par des tests d'architecture Pest (`arch()`).

Choix : Eloquent dans le domaine, interfaces seulement pour les vraies frontières (`PaymentGateway`, `Clock`, `TicketRenderer`, `WebhookSender`).
Écartés : MVC plat, DDD tactique complet / CQRS / event sourcing, microservices, hexagonal pur.

## 3. Modèle de données

Conventions :
- **IDs** : UUIDv7 natif PostgreSQL. Commandes : `number` lisible unique (`ORD-2026-000123`).
- **Montants** : `bigint` en unités mineures + `currency char(3)` avec `CHECK (currency = 'EUR')`. Value object `Money`. Jamais de `float`. Prix figés (snapshot) sur `reservation_items` et `orders`.
- **Temps** : `timestamptz` en UTC ; `events.timezone` (IANA) pour l'affichage ; interface `Clock` pour les tests.
- **États** : colonne `text` + `CHECK (status IN (...))` + enum PHP. Pas d'enum PostgreSQL.
- **Multi-tenancy** : `organization_id` sur les tables « organisateur », dénormalisé sur les tables filles si utile.
- Pas de soft deletes : des statuts.

```
organizations        id, name, slug UNIQUE, created_at
users                id, email (UNIQUE on lower(email)), password, name
organization_members (organization_id, user_id) PK, role CHECK IN (owner, admin, manager, finance, scanner)
                     partial UNIQUE: one owner per organization

events               id, organization_id FK, title, description, venue_name, address, timezone,
                     starts_at, ends_at, sales_start_at, sales_end_at,
                     status CHECK IN (draft, published, cancelled, completed)
                     CHECK (ends_at > starts_at), CHECK (sales_end_at <= ends_at)
ticket_types         id, event_id FK, organization_id, name, price_amount bigint CHECK >= 0, currency,
                     max_per_order smallint CHECK > 0, status, UNIQUE (event_id, name)

inventory_pools      ticket_type_id PK/FK, capacity int CHECK >= 0,
                     reserved int CHECK >= 0, sold int CHECK >= 0,
                     CHECK (reserved + sold <= capacity)          -- anti-oversell safety net
reservations         id, user_id, event_id, organization_id, status, expires_at,
                     confirmed_at, released_at, created_at
                     partial INDEX (expires_at) WHERE status = 'active'
reservation_items    (reservation_id, ticket_type_id) PK, quantity CHECK > 0, unit_price_amount

orders               id, number UNIQUE, reservation_id UNIQUE FK, user_id, organization_id,
                     status, total_amount, currency, paid_at, created_at, updated_at
order_status_transitions  id, order_id, from_status, to_status, reason, actor_type, actor_id,
                          metadata jsonb, created_at

payments             id, order_id FK, provider, checkout_session_id UNIQUE,
                     payment_intent_id UNIQUE NULL, status, amount, currency, expires_at
                     partial UNIQUE (order_id) WHERE status = 'pending'
refunds              id, payment_id FK, provider_refund_id UNIQUE, amount, status, reason
inbound_webhook_events  id, provider, provider_event_id, type, payload jsonb, received_at,
                        processed_at, attempts, last_error, UNIQUE (provider, provider_event_id)

tickets              id, order_id, ticket_type_id, event_id, organization_id,
                     code UNIQUE (random, 128 bits), holder_name,
                     status CHECK IN (issued, checked_in, voided), checked_in_at, document_path

idempotency_keys     id, scope (user/api_client id), key, request_fingerprint, status
                     (in_progress, completed), response_status, response_body jsonb,
                     locked_until, expires_at, UNIQUE (scope, key)
outbox_messages      id bigint identity, message_id uuid UNIQUE, type, aggregate_type, aggregate_id,
                     organization_id, payload jsonb, occurred_at, published_at NULL, attempts
                     partial INDEX (id) WHERE published_at IS NULL
processed_messages   (consumer, message_id) PK, processed_at

api_clients          id, organization_id, name (Sanctum tokens attached, with abilities)
webhook_endpoints    id, organization_id, url, secret (encrypted), previous_secret NULL,
                     previous_secret_expires_at, events text[], status, consecutive_failures
webhook_deliveries   id, endpoint_id, message_id, event_type, payload jsonb, status
                     (pending, succeeded, retrying, dead), attempts, next_attempt_at,
                     last_response_code, last_error, UNIQUE (endpoint_id, message_id)
```

`inventory_pools` est séparé de `ticket_types` : la ligne de stock est très sollicitée en écriture, alors que `ticket_types` est surtout lue et mise en cache. Cela réduit la contention et évite d'invalider le cache du catalogue à chaque réservation.

## 4. Machines à états

Enum PHP avec `canTransitionTo()`. Transition appliquée par un **UPDATE gardé** :
`UPDATE orders SET status = :to WHERE id = :id AND status = :from`. Zéro ligne modifiée = transition concurrente ou doublon, traité comme idempotent. Chaque transition écrit une ligne d'historique et un message outbox dans la **même transaction**. Pas de package de machine à états ni de moteur de workflow.

### Reservation
```
             create (stock reserved)
                    │
                    ▼
               ┌─────────┐  user cancels             ┌───────────┐
               │ active  │──────────────────────────▶│ cancelled │ (stock released)
               └────┬────┘                           └───────────┘
       TTL reached  │   payment confirmed
   (job / sweeper / │──────────────────────────────▶ ┌───────────┐
    lazy check)     │                                │ confirmed │ (reserved → sold)
                    ▼                                └───────────┘
               ┌─────────┐   late payment AND stock re-acquired   ▲
               │ expired │────────────────────────────────────────┘
               └─────────┘   (otherwise: order goes to refund)
```

### Order
```
 checkout started
       │
       ▼
 ┌──────────────────┐ session expired / hold expired / user cancels ┌───────────┐
 │ awaiting_payment │───────────────────────────────────────────────▶│ cancelled │
 └───┬──────────┬───┘                                                └───────────┘
     │          │ async_payment_failed                               ┌───────────┐
     │          └───────────────────────────────────────────────────▶│ failed    │
     │ checkout.session.completed (paid) / async_payment_succeeded   └───────────┘
     ▼
 ┌──────┐  tickets issued   ┌───────────┐
 │ paid │──────────────────▶│ fulfilled │
 └──┬───┘                   └─────┬─────┘
    │ late payment without stock  │ refund requested / event cancelled
    ▼                             ▼
 ┌───────────┐  charge.refunded / refund.updated(succeeded)  ┌──────────┐
 │ refunding │──────────────────────────────────────────────▶│ refunded │
 └───────────┘                                               └──────────┘
```
- `Payment` : `pending → succeeded | failed | expired`. Une seule tentative `pending` par commande (index unique partiel).
- `Ticket` : `issued → checked_in | voided`. Check-in par UPDATE atomique `WHERE status = 'issued'`.
- Remboursements **totaux uniquement**.

## 5. Stratégies pour les problèmes difficiles

### 5.1 Concurrence sur la dernière place
Retenu : compteur atomique conditionnel + `CHECK` comme filet de sécurité, en READ COMMITTED.
```
UPDATE inventory_pools
SET reserved = reserved + :qty
WHERE ticket_type_id = :id AND reserved + sold + :qty <= capacity
-- 0 rows affected => sold out (409 Conflict)
```
- Plusieurs types de billets : verrouillage dans un ordre déterministe (tri par `ticket_type_id`) pour éviter les deadlocks ; retry borné sur `40P01` et `40001`.
- **Labo pédagogique (T2)** : implémenter derrière une même interface, puis comparer par tests de concurrence et k6 : (1) naïve (démontrer la survente), (2) `SELECT ... FOR UPDATE`, (3) optimiste avec `version`, (4) `SERIALIZABLE` + retry, (5) compteur atomique + CHECK.

| Alternative | Raison de l'écarter en production |
|---|---|
| Pessimiste `FOR UPDATE` | Verrou tenu plus longtemps, plus de contention |
| Optimiste avec version | Les retries explosent sous forte demande sur une ligne |
| SERIALIZABLE | Retry partout, coût en performance (exercice utile) |
| Décrément Redis comme autorité | Double écriture, incohérences : viole « Postgres décide » |
| Une ligne par place + `SKIP LOCKED` | Seulement pour le placement numéroté (hors périmètre) |

### 5.2 Réservation temporaire et expiration
Source de vérité : `reservations.expires_at` (TTL **15 min**). Trois mécanismes :
1. **Job retardé** `ExpireReservation` (chemin nominal).
2. **Sweeper planifié** chaque minute : `WHERE status = 'active' AND expires_at < now() ORDER BY expires_at LIMIT 100 FOR UPDATE SKIP LOCKED` (filet de sécurité).
3. **Vérification à la lecture et au checkout** : une réservation dépassée est considérée expirée.

Expiration idempotente : UPDATE gardé `WHERE status = 'active'` puis décrément de `reserved`, dans une transaction.
Interaction Stripe : une Checkout Session dure au minimum 30 min. À l'expiration, on **expire explicitement la session** via l'API Stripe. Si un paiement arrive malgré tout, on tente de reprendre le stock ; sinon la commande passe en `refunding` (remboursement automatique).
Écartés : keyspace notifications Redis, TTL Redis comme source de vérité.

### 5.3 Idempotence
- **Requêtes entrantes** (`POST /v1/reservations`, `POST /v1/orders/{id}/checkout`) : header `Idempotency-Key` obligatoire. `INSERT ... ON CONFLICT DO NOTHING` ; clé terminée = réponse rejouée ; `in_progress` = 409 ; empreinte différente = 422. Finalisation dans la même transaction que l'écriture métier.
- **Webhooks Stripe entrants** : vérifier la signature, `INSERT ... ON CONFLICT (provider, provider_event_id) DO NOTHING`, répondre 200, traiter dans un job `critical`. L'ordre d'arrivée n'est jamais supposé : en cas de doute, relire l'objet via l'API Stripe.
- **Appels sortants vers Stripe** : `Idempotency-Key` dérivé de `payment.id`.

### 5.4 Outbox
- Les changements d'état importants (`ReservationConfirmed`, `OrderPaid`, `OrderRefunded`, `EventCancelled`, `TicketCheckedIn`...) insèrent une ligne outbox dans la même transaction que la modification métier.
- **Relay** (commande artisan longue durée, conteneur dédié) : `SELECT ... WHERE published_at IS NULL ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED`, dispatch, puis `published_at`.
- Livraison **au moins une fois** ; consommateurs protégés par `processed_messages`.
- Piège documenté : ne jamais paginer par `id > dernier_id_vu` (les ids sont attribués avant le commit).
- Purge planifiée des messages publiés depuis plus de N jours.
- Progression : T5 avec événements Laravel `ShouldDispatchAfterCommit`, simulation d'un crash pour **constater la perte**, puis migration vers l'outbox en T6.

Écartés : dispatch direct dans la transaction, `afterCommit` seul, CDC (Debezium/Kafka), `LISTEN/NOTIFY` (optimisation optionnelle du relay seulement).

### 5.5 Queues, retries, dead letters
- Redis + **Horizon**. Files : `critical`, `default`, `notifications`, `documents`, `webhooks-out`.
- Chaque job : `$tries`, `backoff` exponentiel, `$timeout`, `failed()`, idempotent.
- DLQ : `failed_jobs` avec commande de rejeu documentée et alerte ; DLQ métier pour les webhooks sortants (`webhook_deliveries.status = 'dead'`).
- Redis avec persistance AOF ; l'outbox et le sweeper permettent de reconstituer le travail.

### 5.6 Cache
- Cache-aside Redis pour le catalogue public, clés versionnées (`catalog:v1:event:{id}:{version}`), invalidées par les messages outbox `EventUpdated`.
- Disponibilité affichée : cache court (2 à 5 s), explicitement approximative. La décision d'achat passe toujours par le compteur atomique.
- Anti-stampede : `Cache::flexible` ou `Cache::lock`. `ETag` et `Cache-Control` sur les endpoints publics.
- Écartés : cache de page complet, cache comme autorité sur le stock.

### 5.7 Rate limiting
- Limiteurs Laravel nommés sur Redis : `login` (email + IP), `reservations` (par utilisateur + `Redis::funnel` par événement), `public-api` (par client, paliers), `organizer-api` (par token).
- Réponses `429` avec `Retry-After` et `RateLimit-*`. Webhook Stripe exclu (protégé par signature).
- Optionnel : token bucket Lua, salle d'attente.

### 5.8 Multi-tenancy et autorisations
- Base et schéma partagés, colonne `organization_id`. Tenant résolu depuis la route (`/v1/organizations/{organization}/...`) ou le token du client d'API.
- Scope global Eloquent + Policies avec matrice rôle → permissions en code (enum). Pas de package de permissions.
- Tests d'isolation endpoint par endpoint.
- RLS PostgreSQL : extension optionnelle. Écartés : schéma ou base par tenant.
- Acheteurs : utilisateurs globaux, **compte obligatoire**.

### 5.9 API publique et webhooks sortants
- Versioning `/v1`, erreurs RFC 9457 (`application/problem+json`), pagination par curseur, OpenAPI via Scramble.
- Authentification Sanctum (abilities) ; Passport/OAuth écarté.
- Webhooks sortants : signature `HMAC-SHA256` sur `timestamp + "." + body` (header `Webhook-Signature: t=...,v1=...`), `message_id` stable, rotation de secret (deux secrets valides en transition), retries exponentiels sur ~24 h puis `dead`, désactivation après N échecs, protection **SSRF** (IP privées/loopback refusées, résolution DNS vérifiée).
- Billets : QR contenant `ticket_code` signé HMAC ; PDF généré sur la file `documents`, stocké sur volume Docker.

## 6. Stratégie de tests

**Pest** sur un vrai PostgreSQL et un vrai Redis (pas de SQLite).

| Niveau | Contenu |
|---|---|
| Unitaires | Money, transitions d'enums, HMAC, empreinte d'idempotence, matrice des rôles |
| Intégration | Endpoints, policies, isolation des tenants, machines à états via fixtures Stripe signées, outbox, sweeper avec `Clock` contrôlée ; `PaymentGateway` fake |
| Architecture | Frontières entre modules, pas de `float` pour les montants, contrôleurs minces |
| Contrat | Réponses validées contre OpenAPI |
| Concurrence | N processus parallèles (`Process::pool`) sur la dernière place, invariant `sold + reserved <= capacity` vérifié en SQL ; **sans transaction englobante** (pas de `RefreshDatabase`) |
| Bout en bout Stripe | Manuel avec la Stripe CLI avant chaque jalon de paiement |
| Charge (k6) | (1) pic de lecture du catalogue ; (2) ruée de 500 VUs pour 100 billets, zéro survente ; (3) tempête de webhooks dupliqués. Lancé hors du VPS testé |

Règle : tout invariant métier a au moins un test ; chaque bug corrigé reçoit un test de régression (lien avec `bugs/`).

## 7. Infrastructure

### Dev (`compose.yaml`)
Services : `web` (Caddy), `app` (PHP-FPM), `worker` (Horizon), `scheduler`, `outbox-relay`, `postgres`, `redis`, `mailpit`, `stripe-cli`. Code monté en volume, Xdebug optionnel. Pas de Laravel Sail : écrire son Compose fait partie de l'apprentissage.

### Prod (`compose.prod.yaml`)
- Une image multi-stage (Composer puis PHP-FPM non-root, OPcache), lancée avec les commandes `app`, `worker`, `scheduler`, `outbox-relay`.
- Caddy avec TLS automatique ; `postgres` et `redis` en volumes nommés, non exposés ; `restart: unless-stopped`, healthchecks, limites mémoire.
- Secrets dans un `.env` présent seulement sur le serveur (`chmod 600`). Stripe en clés de test.
- Prometheus et Grafana derrière Caddy, avec authentification.

### CI/CD (GitHub Actions)
1. CI à chaque push/PR : Pint, Larastan (niveau relevé progressivement), Pest avec service containers Postgres et Redis, validation OpenAPI, scan de secrets (gitleaks).
2. CD sur `main` : build de l'image, push GHCR (tag SHA), déploiement SSH (`docker compose pull`, `php artisan migrate --force`, `docker compose up -d`), smoke test `/health/ready`.
3. Migrations expand/contract (compatibles N-1). Courte interruption acceptée.

Git : trunk-based, branches courtes, PR vers `main` même en solo, commits conventionnels.

### VPS
~2 vCPU / 4 Go de RAM, Debian ou Ubuntu LTS. Utilisateur `deploy` non-root, SSH par clé, `ufw` (22, 80, 443), `fail2ban`, `unattended-upgrades`. Runbooks dans `docs/runbooks/` (redémarrage, rejeu DLQ, restauration).

### Observabilité
- **Logs** JSON sur stdout ; `request_id` (`X-Request-Id`) propagé aux jobs via `Context` ; champs `organization_id`, `order_id`, `message_id`.
- **Métriques** `/metrics` : latence par route, profondeur des files, jobs en échec ; métier : réservations créées/expirées, refus pour stock épuisé, âge du plus vieux message outbox non publié, délai de traitement des webhooks Stripe, livraisons `dead`. Quelques alertes.
- **Healthchecks** : `/up` et `/health/ready` (Postgres, Redis, retard outbox, workers).
- Option : Sentry (offre gratuite).

### Backups
`pg_dump -Fc` nocturne, chiffré, envoyé hors serveur via restic (7 jours / 4 semaines / 3 mois). **Test de restauration mensuel**, documenté et chronométré. PDF régénérables ; Redis non sauvegardé. Option : WAL-G / pgBackRest (PITR).

### Arborescence
```
backend-engineering/
├── CLAUDE.md                 task list (@planner)
├── architect.md
├── ideas/  reviews/  bugs/
├── docs/
│   ├── runbooks/             restore.md, replay-dlq.md, deploy.md
│   └── concurrency-lab.md
├── app/
│   ├── Modules/              Identity, Organizations, Catalog, Booking, Ordering,
│   │                         Payments, Ticketing, Notifications, Integrations
│   └── Shared/               Money, Clock, Outbox, Idempotency, Http/ProblemDetails, Health, Observability
├── bootstrap/  config/  public/  resources/views/ (checkout success/cancel only)
├── database/migrations/
├── routes/api.php
├── tests/                    Unit, Feature, Architecture, Contract, Concurrency, Fixtures/stripe
├── loadtests/k6/             catalog-spike.js, on-sale-rush.js, webhook-storm.js
├── docker/                   php/Dockerfile, caddy/, observability/
├── compose.yaml  compose.prod.yaml
├── scripts/                  backup.sh, restore.sh
└── .github/workflows/        ci.yml, deploy.yml
```

## 8. Décisions d'architecture (ADR)

| # | Décision | Statut |
|---|---|---|
| ADR-001 | Laravel, PostgreSQL, Redis, Docker ; Stripe Checkout test ; API seule avec OpenAPI | Accepté |
| ADR-002 | Monolithe modulaire, frontières vérifiées par tests d'architecture | Accepté |
| ADR-003 | PostgreSQL source de vérité ; Redis jamais autorité | Accepté |
| ADR-004 | UUIDv7 comme clé primaire (exception : outbox `bigint identity`) | Accepté |
| ADR-005 | Montants en `bigint` d'unités mineures, devise unique EUR, `Money` | Accepté |
| ADR-006 | Inventaire par compteur atomique conditionnel + `CHECK`, READ COMMITTED | Accepté |
| ADR-007 | Expiration : TTL 15 min, job retardé + sweeper `SKIP LOCKED` + vérification à la lecture, expiration explicite de la session Stripe | Accepté |
| ADR-008 | Idempotence des requêtes via `Idempotency-Key` dans Postgres | Accepté |
| ADR-009 | Webhooks Stripe : signature, stockage dédupliqué, traitement asynchrone | Accepté |
| ADR-010 | Machines à états : enums PHP, UPDATE gardés, historique des transitions | Accepté |
| ADR-011 | Outbox transactionnelle, relay par polling, consommateurs idempotents | Accepté |
| ADR-012 | Files Redis + Horizon ; `failed_jobs` comme DLQ + DLQ métier webhooks sortants | Accepté |
| ADR-013 | Multi-tenancy par `organization_id`, scopes et policies ; RLS optionnel | Accepté |
| ADR-014 | `/v1`, erreurs RFC 9457, pagination par curseur, Sanctum, OpenAPI via Scramble | Accepté |
| ADR-015 | Webhooks sortants HMAC-SHA256 horodatés, rotation de secret, protection SSRF | Accepté |
| ADR-016 | Caddy + PHP-FPM en dev et prod ; pas d'Octane | Accepté |
| ADR-017 | Pest sur vrai Postgres et Redis ; tests de concurrence sans transaction englobante | Accepté |
| ADR-018 | Logs JSON avec request-id ; métriques Prometheus + Grafana | Accepté |
| ADR-019 | Déploiement : image GHCR, SSH, Docker Compose sur VPS unique | Accepté (fournisseur à choisir avant T4) |
| ADR-020 | Backups `pg_dump` + restic hors serveur, test de restauration mensuel | Accepté |
| ADR-021 | Admission générale uniquement, compte obligatoire pour acheter, remboursements totaux uniquement | Accepté |

## 9. Risques et ordre de construction

| Risque | Mitigation |
|---|---|
| Périmètre trop large pour ~300 h | Tranches livrables seules ; liste d'éléments à couper |
| Friction Docker et Git au départ | T0 dédiée aux fondations ; Compose minimal ; PR systématiques |
| Tests verts malgré des bugs de concurrence | Suite `Concurrency/` sans transaction englobante ; démontrer la survente avant de la corriger |
| La « magie » Laravel cache le SQL | Logger les requêtes en dev ; SQL explicite pour stock, sweeper et relay |
| Paiement tardif après expiration | Chemin explicite (reprise du stock ou remboursement), testé par fixture |
| Webhooks Stripe en désordre ou dupliqués | Store-then-process, UPDATE gardés, relecture via l'API |
| VPS sous-dimensionné (Prometheus + Grafana) | Limites mémoire ; repli sur Pulse + Sentry |
| Premier déploiement tardif | Déploiement dès T4, puis à chaque merge |
| Fuite de secrets dans Git | `.env` ignoré, Stripe test, gitleaks en CI |
| Mesures k6 faussées | Ne jamais lancer k6 depuis le VPS testé ; noter la configuration |

### Tranches verticales (~20 semaines)

| Sem. | Tranche | Livrable | Compétences |
|---|---|---|---|
| 1–2 | **T0 Fondations** | Dépôt Git, Laravel dans Compose, Pest, CI verte, `/up`, logs JSON | Git, Docker, CI, premier test |
| 3–4 | **T1 Catalogue** | Organisation, événements, types de billets, catalogue public, UUIDv7, Money, problem+json, OpenAPI, tests d'architecture | Structure modulaire, migrations, contraintes |
| 5–7 | **T2 Réservation et concurrence** | `inventory_pools`, réservation, labo de concurrence, tests parallèles, `Idempotency-Key` | Transactions, isolation, verrous |
| 8–9 | **T3 Expiration et files** | Job retardé, Horizon, sweeper, scheduler, `Clock` | Queues, ordonnancement |
| 10 | **T4 Premier déploiement** | VPS durci, image de prod, Caddy TLS, CD SSH, backup nocturne | Linux, Docker prod, CD |
| 11–13 | **T5 Paiement Stripe** | Checkout Session, webhooks, machines à états Order/Payment, paiement tardif, remboursement | Intégration externe, idempotence |
| 14–15 | **T6 Outbox, billets, emails** | Démo de la perte `afterCommit`, outbox + relay, billets PDF/QR, emails, retries, DLQ | Fiabilité asynchrone |
| 16 | **T7 Multi-organisateurs** | Membres, rôles, policies, tests d'isolation, check-in atomique | Autorisations, tenancy |
| 17–18 | **T8 API publique et webhooks sortants** | Clients d'API, rate limiting par paliers, webhooks HMAC, rotation, SSRF | Conception d'API, sécurité |
| 19 | **T9 Montée en charge** | Cache, ETag, stampede, limiteurs, 3 scénarios k6, rapport | Performance |
| 20 | **T10 Observabilité et exploitation** | `/metrics`, Grafana, alertes, `/health/ready`, restauration, runbooks | Exploitation |

À couper en premier en cas de retard : RLS, salle d'attente, token bucket Lua, PITR, Grafana (remplacé par Pulse), annulation d'événement avec remboursement en masse.

## 10. Points ouverts

- **VPS** : fournisseur, budget mensuel et nom de domaine à choisir avant T4 (semaine 10). Le domaine est nécessaire pour le TLS et les webhooks Stripe de test en production.
