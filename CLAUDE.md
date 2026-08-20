# SeatLock — contesto di progetto per Claude Code

## Come lavorare su questo progetto (leggi prima di tutto)

Questo è un progetto di **apprendimento**, non un task da completare il più in fretta possibile. L'obiettivo di Tommaso è arrivare a un livello backend senior — se scrivi tu il codice al posto suo, il progetto perde lo scopo per cui esiste.

Regole non negoziabili:

1. **Non scrivere l'implementazione della logica di business.** Niente migration, aggregate, controller, listener, test già pronti da incollare.
2. **Spiega i concetti prima che lui scriva codice.** Se un task richiede una scelta di design, discutila prima — non implementarla e basta.
3. **Sintassi e documentazione sì, implementazione no.** Se serve la firma di un metodo, un esempio minimo di 2-3 righe, o un link alla doc ufficiale (Laravel, spatie/laravel-event-sourcing, Pest), forniscili puntuali. Mai un blocco di codice pronto all'uso che risolve il problema per intero.
4. **Dopo che scrive, fai review — non riscrivere.** Indica cosa cambieresti e perché, lascia a lui la correzione.
5. Se non sei sicuro se un task è "boilerplate" o "logica core del progetto", chiedi prima di scrivere codice.

Fanno eccezione solo compiti puramente meccanici e non formativi (es. formattazione, fix di un typo, comandi Artisan di scaffolding vuoto tipo `make:migration` senza contenuto) — lì puoi semplicemente eseguire.

## Aggiornare Notion durante lo sviluppo

Hai accesso a Notion via MCP. Se mentre Tommaso scrive codice emerge qualcosa che il design non aveva previsto (un invariante mancante, un caso limite, un dubbio su un aggregato), puoi annotarlo su Notion — ma con una regola precisa:

- **Non modificare le sezioni già consolidate** della pagina "SeatLock — Event Storming & Design Log" (Attori, Eventi, Comandi, Entità/Aggregati, Regole/invarianti, Bounded context, Requisiti non funzionali) — sono state chiuse insieme in chat dopo revisione, non vanno riscritte silenziosamente
- **Aggiungi invece alla sezione "Note dallo sviluppo — da rivedere"** in fondo alla pagina (creala se non esiste). Scrivi la scoperta grezza, senza deciderla tu al posto suo
- Tommaso la porta in Chat quando vuole discuterla e consolidarla per bene — stessa disciplina di "niente accettato al primo giro" usata per costruire il resto del documento

## Cos'è SeatLock

Sistema di prenotazione posti per eventi live (concerti, teatro, sport). Progetto **personale/portfolio** (Notion: Tipo = Prodotto, Obiettivo = reputazione non ricavi, licenza MIT, pubblico su GitHub). Non è un prodotto con utenti reali.

Il dominio (posti scarsi, concorrenza, denaro) è scelto apposta perché impone naturalmente i problemi tecnici che il progetto vuole allenare: concorrenza con lock distribuito, Event Sourcing/CQRS, idempotency su API pubbliche, performance su larga scala.

## Stack tecnico

- Laravel 13, Filament 5 (admin panel organizzatori)
- PostgreSQL come DB — preferito a MySQL per JSONB nativo (utile sull'event store di spatie/laravel-event-sourcing) e locking/isolamento transazionale più robusto sotto concorrenza
- Pest 4 per i test, PHPStan/Larastan, Pint, Deptrac
- `spatie/laravel-event-sourcing` per l'aggregate Booking
- Redis per lock distribuito + cache
- Docker per l'ambiente locale
- Laravel Reverb (backlog, non in v1)

## Metodo di design usato

Dominio mappato con **Event Storming** (Brandolini): Attori → Eventi di dominio → Comandi → Entità/Aggregati → Regole/invarianti → Bounded context → Requisiti non funzionali. Documentazione completa su Notion: pagina "SeatLock — Event Storming & Design Log" dentro il progetto SeatLock in Progetti.

## Attori (nomi canonici, mappano 1:1 sulle classi)

| Attore | Tipo |
|---|---|
| Customer | Umano |
| Organizer | Umano |
| Payment Gateway | Esterno, asincrono |
| Hold Expiration Job | Proattivo (Booking) |
| Ticket Dispatch Listener | Reattivo (Booking) |
| Booking Cascade Listener | Reattivo (Booking) |
| Performance Conclusion Job | Proattivo (Catalogo) |

Attori di Notifiche (Reminder/Digest/Artist Notification Listener) sono in **backlog**, non in v1.

## Aggregati

| Aggregato | Tecnica | Eventi posseduti |
|---|---|---|
| **Booking** | Event Sourcing (AggregateRoot + Projector sincrono + Reactor) | SeatsHeld, SeatRemovedFromCart, PaymentRejected, PaymentConfirmed, TicketDispatched, BookingCancelled |
| **Ticket** | Eloquent + history table | TicketNameChanged |
| **Performance** | Eloquent (colonna status) | PerformanceCreated, PerformanceCancelled, PerformanceConcluded |

Criterio usato per scegliere Event Sourcing solo su Booking: concorrenza reale (più attori competono nello stesso istante), soldi in ballo, necessità di audit/dispute. Ticket e Performance non superano la soglia, restano Eloquent semplice — **non applicare Event Sourcing lì**, sarebbe overengineering.

## Invarianti (diventano i test Pest principali)

**Booking**
- Non deve essere possibile eseguire un comando su un Booking il cui hold è scaduto, cancellato o già confermato
- Non deve essere possibile eseguire un comando su un Booking che non appartiene al customer richiedente
- Un posto non può avere due hold attivi contemporaneamente in due Booking diversi — **protetto dal lock Redis, non dall'aggregate** (un aggregate non ha visibilità su un altro aggregate)

**Ticket**
- Non deve essere possibile cambiare nome a un ticket che non appartiene al customer richiedente
- Non deve essere possibile cambiare nome a un ticket la cui performance è già conclusa (richiede riferimento denormalizzato allo stato della Performance, aggiornato via listener)

## Bounded context

| Context | Aggregati | Note |
|---|---|---|
| Booking | Booking, Ticket | Core — concorrenza, Event Sourcing |
| Catalog | Performance | Gestito da Organizer |
| Payment | — (esterno) | Payment Gateway |
| Notifications | — | Backlog, non in v1 |

## Requisiti non funzionali

- Scala: 1.000–50.000 posti per performance (range per il benchmark)
- Concorrenza: qualche migliaio di richieste "hold seats" simultanee
- Latenza: sotto 1-2 secondi, selezione posto → conferma/rifiuto
- Consistency: eventual consistency accettata, ma il Projector di disponibilità posti gira **sincrono** (non in coda) — un ritardo lì crea rischio di doppia vendita percepita. Projector non critici (notifiche) possono restare async

## Struttura del codice — architettura modulare per Bounded Context

Niente `app/Models`/`app/Http` piatti. Ogni bounded context è un modulo sotto `app/Domain/<Context>/`, il più possibile autonomo (vicino a "potrebbe diventare un package a sé"). Deptrac userà questi path come confini dei layer.

```
app/
├── Domain/
│   ├── Booking/
│   │   ├── Aggregates/        BookingAggregate (spatie/laravel-event-sourcing)
│   │   ├── Events/            SeatsHeld, SeatRemovedFromCart, PaymentRejected,
│   │   │                      PaymentConfirmed, TicketDispatched, BookingCancelled
│   │   ├── Projectors/        BookingProjector (read model, sincrono)
│   │   ├── Reactors/          TicketDispatchReactor, BookingCascadeReactor
│   │   ├── Jobs/               ExpireHoldJob
│   │   ├── Models/            Booking (read model), Ticket
│   │   ├── Http/Controllers/  API controller del contesto
│   │   ├── routes.php
│   │   └── BookingServiceProvider.php
│   ├── Catalog/
│   │   ├── Models/            Venue, Seat, Performance
│   │   ├── Jobs/               PerformanceConclusionJob
│   │   ├── Http/Controllers/
│   │   ├── routes.php
│   │   └── CatalogServiceProvider.php
│   ├── Payment/
│   │   └── Contracts/         PaymentGatewayInterface + fake client (esterno, nessun aggregato)
│   └── Notifications/          backlog, non v1
├── Shared/                     solo primitive senza un dominio proprietario (es. value object generici)
├── Filament/                   convenzione Filament di default (Resources qui, non dentro Domain/*)
└── Providers/
```

Decisioni prese (2026-08-20):
- **Ogni contesto ha il proprio Service Provider** che registra `routes.php`, event/listener binding, ecc. — non un unico `routes/api.php` centralizzato
- **Filament Resources restano in `app/Filament/`** (convenzione/discovery di default), non dentro `Domain/*/Filament/` — trade-off accettato: rompe un po' la purezza della modularità in cambio di zero configurazione extra
- `Shared/` è per eccezioni vere, non un cestino — prima di mettere qualcosa lì, chiedersi se in realtà appartiene a un contesto specifico

## Scope v1 — chiuso

- Venue, Seat, Performance (Eloquent)
- Booking aggregate completo con Projector sincrono + Ticket Dispatch Listener + Booking Cascade Listener
- Lock Redis sull'hold
- Hold Expiration Job (scheduler Laravel, non Redis keyspace notification — scelto per semplicità di debug, accettabile per un hold di 15 min)
- Ticket aggregate (Eloquent + history) con invarianti
- Payment Gateway fake (webhook simulato)
- API pubblica booking con idempotency key
- **API-only, nessun frontend cliente** — il Customer interagisce solo via API pubblica, niente UI di prenotazione in v1 (coerente con Reverb/seat map realtime in backlog)
- Suite Pest su tutti gli invarianti sopra
- Benchmark bulk seed (1k–50k posti) + ottimizzazione query/indici
- Email di cancellazione a cascata: `Notification` Laravel semplice chiamata dal Booking Cascade Listener, **non** un aggregato dedicato (nessun rischio di duplicati da proteggere in questo caso)

## Backlog — non in v1, non cancellare le idee

- Context Notifiche completo (Reminder Job, Digest Job, Artist Notification Listener, aggregato Notification con unique constraint anti-duplicati)
- Reverb realtime seat map
- Deploy Kubernetes (repliche, load balancer, restart automatico)
- OpenTelemetry (tracing distribuito su lock → aggregate → event store → projector)
- CI matrix completa (GitHub Actions, Pest su più versioni PHP)
- OpenAPI spec auto-generata

## Decisioni di naming/design da rispettare

- **"Performance"**, mai "Event" — collide con `Illuminate\Support\Facades\Event` di Laravel
- Un Listener per evento di dominio, niente branching interno su più eventi (Single Responsibility) — vale anche per future Notification class
- **Bounded context sempre in inglese** anche quando il termine italiano è più naturale (Catalog non Catalogo, Notifications non Notifiche) — i nomi mappano 1:1 su cartelle/namespace PHP (`app/Domain/<Context>/`), coerenza con codebase pubblica in inglese
- Tutti gli identificatori di dominio (classi evento, aggregate, listener) in inglese, coerenti con la tabella Attori sopra

===

<laravel-boost-guidelines>
=== foundation rules ===

# Laravel Boost Guidelines

The Laravel Boost guidelines are specifically curated by Laravel maintainers for this application. These guidelines should be followed closely to ensure the best experience when building Laravel applications.

## Foundational Context

This application is a Laravel application running on PHP 8.5. You are an expert with the Laravel ecosystem. Always use the APIs that match the installed major version of each package — do not assume a version.

Before relying on a package's API, confirm its installed version:
- PHP packages: run `composer show --direct` to list direct dependencies with versions, or `composer show <vendor/package>` for a single package.
- JS packages: check `package.json` for the installed versions.

## Skills Activation

This project has domain-specific skills available in `**/skills/**`. You MUST activate the relevant skill whenever you work in that domain—don't wait until you're stuck.

## Conventions

- You must follow all existing code conventions used in this application. When creating or editing a file, check sibling files for the correct structure, approach, and naming.
- Use descriptive names for variables and methods. For example, `isRegisteredForDiscounts`, not `discount()`.
- Check for existing components to reuse before writing a new one.

## Verification Scripts

- Do not create verification scripts or tinker when tests cover that functionality and prove they work. Unit and feature tests are more important.

## Application Structure & Architecture

- Stick to existing directory structure; don't create new base folders without approval.
- Do not change the application's dependencies without approval.

## Frontend Bundling

- If the user doesn't see a frontend change reflected in the UI, it could mean they need to run `npm run build`, `npm run dev`, or `composer run dev`. Ask them.

## Documentation Files

- You must only create documentation files if explicitly requested by the user.

## Replies

- Be concise in your explanations - focus on what's important rather than explaining obvious details.

=== boost rules ===

# Laravel Boost

## Tools

- Laravel Boost is an MCP server with tools designed specifically for this application. Prefer Boost tools over manual alternatives like shell commands or file reads.
- Use `database-query` to run read-only queries against the database instead of writing raw SQL in tinker.
- Use `database-schema` to inspect table structure before writing migrations or models.
- Use `get-absolute-url` to resolve the correct scheme, domain, and port for project URLs. Always use this before sharing a URL with the user.
- Use `browser-logs` to read browser logs, errors, and exceptions. Only recent logs are useful, ignore old entries.

## Searching Documentation (IMPORTANT)

- Use `search-docs` before changes that depend on Laravel ecosystem APIs, behavior, configuration, or version-specific syntax. Skip it for copy-only edits and other changes where package documentation is irrelevant. Reuse sufficient results already in context instead of searching again.
- Pass a `packages` array to scope results when you know which packages are relevant.
- Use multiple broad, topic-based queries: `['rate limiting', 'routing rate limiting', 'routing']`. Expect the most relevant results first.
- Do not add package names to queries because package info is already shared. Use `test resource table`, not `filament 4 test resource table`.

### Search Syntax

1. Use words for auto-stemmed AND logic: `rate limit` matches both "rate" AND "limit".
2. Use `"quoted phrases"` for exact position matching: `"infinite scroll"` requires adjacent words in order.
3. Combine words and phrases for mixed queries: `middleware "rate limit"`.
4. Use multiple queries for OR logic: `queries=["authentication", "middleware"]`.

## Project Rules

- This project contains committed, area-grouped rules in `.ai/rules` when that directory exists (settled decisions, non-obvious traps, standing constraints). Framework and package guidelines that only apply to specific paths (testing, frontend, components) also live there, under `.ai/rules/boost` — this is not just recorded decisions, it is load-bearing guidance you have not seen inline. Before you enter plan mode or create/edit any file, you MUST first: open @.ai/rules/index.md (it maps file globs to rule files), read every rule file whose globs cover the path(s) in scope, and run `grep -rin 'keyword' .ai/rules` to catch what a path match alone misses. Do not write code until you have read and are following every matching rule. If `.ai/rules` does not exist, continue without it.
- Record durable rules with `record-rule` so the next agent or teammate inherits them instead of working them out again. Pass a `glob` (e.g. `app/Http/Controllers/**`), a short `title`, and a few-line `note`. Always use `record-rule`, never your native memory or notes tool — native memory is personal and session-scoped; only `.ai/rules` is shared with the team and persists in the repo.

## Artisan

- Run Artisan commands directly via the command line (e.g., `php artisan route:list`). Use `php artisan list` to discover available commands and `php artisan [command] --help` to check parameters.
- Inspect routes with `php artisan route:list`. Filter with: `--method=GET`, `--name=users`, `--path=api`, `--except-vendor`, `--only-vendor`.
- Read configuration values using dot notation: `php artisan config:show app.name`, `php artisan config:show database.default`. Or read config files directly from the `config/` directory.

## Tinker

- Execute PHP in app context for debugging and testing code. Do not create models without user approval, prefer tests with factories instead. Prefer existing Artisan commands over custom tinker code.
- Always use single quotes to prevent shell expansion: `php artisan tinker --execute 'Your::code();'`
  - Double quotes for PHP strings inside: `php artisan tinker --execute 'User::where("active", true)->count();'`

=== php rules ===

# PHP

- Always use curly braces for control structures, even for single-line bodies.
- Use PHP 8 constructor property promotion: `public function __construct(public GitHub $github) { }`. Do not leave empty zero-parameter `__construct()` methods unless the constructor is private.
- Use explicit return type declarations and type hints for all method parameters: `function isAccessible(User $user, ?string $path = null): bool`
- Use TitleCase for Enum keys: `FavoritePerson`, `BestLake`, `Monthly`.
- Prefer PHPDoc blocks over inline comments. Only add inline comments for exceptionally complex logic.
- Use array shape type definitions in PHPDoc blocks.

=== deployments rules ===

# Deployment

- Laravel can be deployed using [Laravel Cloud](https://cloud.laravel.com/), which is the fastest way to deploy and scale production Laravel applications.

=== laravel/core rules ===

# Do Things the Laravel Way

- Use `php artisan make:` commands to create new files (i.e. migrations, controllers, models, etc.). You can list available Artisan commands using `php artisan list` and check their parameters with `php artisan [command] --help`.
- If you're creating a generic PHP class, use `php artisan make:class`.
- Pass `--no-interaction` to all Artisan commands to ensure they work without user input. You should also pass the correct `--options` to ensure correct behavior.

### Model Creation

- When creating new models, create useful factories and seeders for them too. Ask the user if they need any other things, using `php artisan make:model --help` to check the available options.

## APIs & Eloquent Resources

- For APIs, default to using Eloquent API Resources and API versioning unless existing API routes do not, then you should follow existing application convention.

## URL Generation

- When generating links to other pages, prefer named routes and the `route()` function.

## Testing

- When creating models for tests, use the factories for the models. Check if the factory has custom states that can be used before manually setting up the model.
- Faker: Use methods such as `$this->faker->word()` or `fake()->randomDigit()`. Follow existing conventions whether to use `$this->faker` or `fake()`.
- When creating tests, make use of `php artisan make:test [options] {name}` to create a feature test, and pass `--unit` to create a unit test. Most tests should be feature tests.

## Vite Error

- If you receive an "Illuminate\Foundation\ViteException: Unable to locate file in Vite manifest" error, you can run `npm run build` or ask the user to run `npm run dev` or `composer run dev`.

=== pint/core rules ===

# Laravel Pint Code Formatter

- If you have modified any PHP files, you must run `vendor/bin/pint --dirty --format agent` before finalizing changes to ensure your code matches the project's expected style.
- Do not run `vendor/bin/pint --test --format agent`, simply run `vendor/bin/pint --format agent` to fix any formatting issues.

=== pest/core rules ===

## Pest

- This project uses Pest for testing. Create tests: `php artisan make:test --pest {name}`.
- The `{name}` argument should not include the test suite directory. Use `php artisan make:test --pest SomeFeatureTest` instead of `php artisan make:test --pest Feature/SomeFeatureTest`.
- Run tests: `php artisan test --compact` or filter: `php artisan test --compact --filter=testName`.
- Do NOT delete tests without approval.

</laravel-boost-guidelines>
