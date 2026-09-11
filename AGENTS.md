# SeatLock — contesto di progetto

## Come lavorare su questo progetto (leggi prima di tutto)

Questo è un progetto di **apprendimento**, non un task da completare il più in fretta possibile. L'obiettivo di Tommaso è arrivare a un livello backend senior — se scrivi tu il codice al posto suo, il progetto perde lo scopo per cui esiste.

Regole non negoziabili:

1. **Non scrivere l'implementazione della logica di business.** Niente migration, aggregate, controller, listener, test già pronti da incollare.
2. **Spiega i concetti prima che lui scriva codice.** Se un task richiede una scelta di design, discutila prima — non implementarla e basta.
3. **Sintassi e documentazione sì, implementazione no.** Se serve la firma di un metodo, un esempio minimo di 2-3 righe, o un link alla doc ufficiale (Laravel, spatie/laravel-event-sourcing, Pest), forniscili puntuali. Mai un blocco di codice pronto all'uso che risolve il problema per intero.
4. **Dopo che scrive, fai review — non riscrivere.** Indica cosa cambieresti e perché, lascia a lui la correzione.
5. Se non sei sicuro se un task è "boilerplate" o "logica core del progetto", chiedi prima di scrivere codice.

Fanno eccezione solo compiti puramente meccanici e non formativi (es. formattazione, fix di un typo, comandi Artisan di scaffolding vuoto tipo `make:migration` senza contenuto) — lì puoi semplicemente eseguire.

## Aggiornare Notion e questo file durante lo sviluppo

L'accesso a Notion passa da un server MCP: se in questa sessione non è configurato,
**dimmelo** invece di saltare l'aggiornamento in silenzio. Quando durante una sessione in chat (come questa) emerge una decisione di design nuova o una revisione di una decisione precedente — **discussa e confermata da Tommaso in chat**, mai decisa unilateralmente da te — aggiorna **direttamente e subito**:

- le sezioni della pagina Notion "SeatLock — Event Storming & Design Log" (Attori, Eventi, Comandi, Entità/Aggregati, Regole/invarianti, Bounded context, Requisiti non funzionali, Decisioni di design)
- le sezioni corrispondenti di questo file AGENTS.md

Niente più staging in una sezione "Note dallo sviluppo — da rivedere" (dismessa il 2026-08-26): si perdeva/dimenticava il passaggio di consolidamento. La discussione avviene in chat, la consolidazione è immediata su entrambe le fonti — resta valida solo la regola di fondo: si scrive dopo che la decisione è confermata, mai prima.

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
| Performance Reminder Job | Proattivo (Notifications) |
| Performance Digest Job | Proattivo (Notifications) |
| Artist Notification Listener | Reattivo (Notifications) |
| Cancellation Notification Listener | Reattivo (Notifications) |

## Aggregati

| Aggregato | Tecnica | Eventi posseduti |
|---|---|---|
| **Booking** | Event Sourcing (AggregateRoot + Projector sincrono + Reactor) | SeatsHeld, SeatRemovedFromCart, PaymentRejected, PaymentConfirmed, TicketDispatched, BookingCancelled |
| **Ticket** | Eloquent + history table | TicketNameChanged |
| **Performance** | Eloquent (colonna status) | PerformanceCreated, PerformanceCancelled, PerformanceConcluded |
| **Notification** | Eloquent + unique constraint anti-duplicati | Promemoria inviato, Notifiche artista inviate, Notifiche generiche performance inviate |

Criterio usato per scegliere Event Sourcing solo su Booking: concorrenza reale (più attori competono nello stesso istante), soldi in ballo, necessità di audit/dispute. Ticket e Performance non superano la soglia, restano Eloquent semplice — **non applicare Event Sourcing lì**, sarebbe overengineering.

## Catalog — posti numerati vs general admission (2026-08-26)

- **Seat** (numerato): `belongsTo(Venue)`, identificato da `section`/`row`/`number`, unique composito su `(venue_id, section, row, number)`
- **SeatBlock** (general admission, es. pista/prato): `belongsTo(Venue)`, nome/etichetta + `capacity` (intero mutabile, nessuna storicizzazione — si sovrascrive e basta)
- Scelto di modellarli come **due entità separate** (non Seat anonimi enumerati per il GA) per tenere il progetto più impegnativo — implica due meccanismi di concorrenza in Booking: lock Redis per-seat (numerati) + decremento atomico di un contatore (GA)
- Evento `SeatsHeld` (Posti tenuti) ha payload misto: seat_id singoli e/o quantità da un SeatBlock — non solo una lista di seat_id
- Una Performance può usare un sottoinsieme della seat map della sua Venue — meccanismo di selezione rimandato; per ora Seat/SeatBlock/Performance restano ognuno con un semplice `belongsTo(Venue)`, nessun vincolo di schema che lo impedisca (una futura tabella pivot `performance_seat`/`performance_seat_block` potrà aggiungersi senza modifiche retroattive)

## Invarianti (diventano i test Pest principali)

**Booking**
- Non deve essere possibile eseguire un comando su un Booking il cui hold è scaduto, cancellato o già confermato
- Non deve essere possibile eseguire un comando su un Booking che non appartiene al customer richiedente
- Un posto non può avere due hold attivi contemporaneamente in due Booking diversi — **protetto dal lock Redis, non dall'aggregate** (un aggregate non ha visibilità su un altro aggregate)
- La quantità held+venduta di un SeatBlock (posti general admission) non può mai superare la sua capacità totale — **protetto da un contatore atomico (es. Lua script Redis), non dall'aggregate**, secondo meccanismo di concorrenza accanto al lock per-seat

**Ticket**
- Non deve essere possibile cambiare nome a un ticket che non appartiene al customer richiedente
- Non deve essere possibile cambiare nome a un ticket la cui performance è già conclusa (richiede riferimento denormalizzato allo stato della Performance, aggiornato via listener)

## Bounded context

| Context | Aggregati | Note |
|---|---|---|
| Booking | Booking, Ticket | Core — concorrenza, Event Sourcing |
| Catalog | Performance | Gestito da Organizer |
| Payment | — (esterno) | Payment Gateway |
| Notifications | Notification | Ascolta gli eventi degli altri context via listener/queue, non li contamina |

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
│   └── Notifications/
│       ├── Models/            Notification (Eloquent + unique constraint anti-duplicati)
│       ├── Jobs/               PerformanceReminderJob, PerformanceDigestJob
│       ├── Listeners/          ArtistNotificationListener, CancellationNotificationListener
│       ├── routes.php
│       └── NotificationsServiceProvider.php
├── Shared/                     solo primitive senza un dominio proprietario (es. value object generici)
├── Filament/                   convenzione Filament di default (Resources qui, non dentro Domain/*)
└── Providers/
```

Decisioni prese (2026-08-20):
- **Ogni contesto ha il proprio Service Provider** che registra `routes.php`, event/listener binding, ecc. — non un unico `routes/api.php` centralizzato
- **Filament Resources restano in `app/Filament/`** (convenzione/discovery di default), non dentro `Domain/*/Filament/` — trade-off accettato: rompe un po' la purezza della modularità in cambio di zero configurazione extra
- `Shared/` è per eccezioni vere, non un cestino — prima di mettere qualcosa lì, chiedersi se in realtà appartiene a un contesto specifico
- **`User.php` e `Controller.php` vivono in `app/Shared/`** (`Shared/Models/User.php`, `Shared/Controllers/Controller.php`) — sono davvero trasversali a più context, non appartengono a un dominio specifico. `User` richiede un `newFactory()` esplicito nel model perché rompe la risoluzione automatica Laravel della Factory (che si aspetta `App\Models\X`)
- **Customer e Organizer sono ruoli su un unico `User`**, non entità di dominio separate — un account può avere entrambi i ruoli contemporaneamente (non mutuamente esclusivi). Gestiti con tabelle `roles`/`role_user` (pivot many-to-many scritta a mano, non `spatie/laravel-permission` — troppo per due ruoli ora, si può migrare dopo se serve granularità sui permessi)

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
- Context Notifications: aggregato Notification (Eloquent + unique constraint anti-duplicati), Performance Reminder Job, Performance Digest Job, Artist Notification Listener, Cancellation Notification Listener — bounded context separato, ascolta gli eventi degli altri context via listener/queue

## Backlog — non in v1, non cancellare le idee

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


## Convenzioni Laravel / PHP / Pest

- **Non assumere le versioni**: prima di usare un'API controlla la major installata
  (`composer show --direct`, `composer show <vendor/package>`; `package.json` per il JS).
- File nuovi con `php artisan make:` e `--no-interaction`, con le opzioni giuste; per una
  classe PHP generica `php artisan make:class`. Per i modelli, crea anche factory e seeder.
- API: Eloquent API Resource e versioning; link con `route()`, non URL scritti a mano.
- Test con le factory (controlla prima gli stati custom disponibili); Faker con
  `$this->faker` o `fake()` come già fa il progetto.
- Dopo aver toccato file PHP: `vendor/bin/pint --dirty`.
- Pest: `php artisan make:test --pest NomeTest` (senza `Feature/` nel nome); per eseguire,
  `php artisan test --compact` o `vendor/bin/pest` con il set più stretto che copre la
  modifica. Non cancellare test senza chiedere.
- PHP: graffe sempre anche sui corpi di una riga, constructor property promotion, return
  type e type hint espliciti, chiavi degli Enum in TitleCase, PHPDoc invece di commenti
  inline (solo per logica davvero complessa).
- Struttura: non creare cartelle di primo livello senza chiedere, non cambiare le
  dipendenze senza approvazione, riusa componenti esistenti prima di scriverne di nuovi.
