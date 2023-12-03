# Broker
Business entity that coordinates the betting process.
- allows users to make bets
- communicates with users on bet results
- keeps tracks of all transactions
- communicates with either `bookie` or `bank` regarding money transfers

Primary focus is UX:
- accept bets as fast as possible
- never lose users' money

For simplicity, assume that bets are made
per `game_id` instead of `{ game_id, round_id }` this does not change the process.

## Core entities

### game
Schema
- id: string
- bet_status: 'open' | 'closed'
- status: 'scheduled' | 'started' | 'finished'
- outcome?: number
- created_at: Date
- started_at?: Date
- finished_at?: Date
- overturned_at?: Date
- postponed_at?: Date
- rescheduled_at?: Date

Bets can be made between `scheduled` and `started` events.

Games status transitions:
```ts
scheduled: [ 'started', 'finished' ]
started: [ 'finished' ]
finished: [] // final status
```

### bet
Consistency between broker and bookie is achieved via idempotent bookie endpoints and corresponding intermediate state on broker side,
which can be:
- processing-related fields embedded in `bet` db entity
- separate `update_bet_task` in db
- message in broker (RabbitMQ or Kafka)

Two-phase commits are undesirable due to extra complexity required on the side of bookie/partner.

Schema
- id: string
- game_id: string
- outcome: number // user prediction
- status
  * `pending` initial status, bet exists only in broker
  * `placed` bet successfully created on bookie side
  * `settling` result is known, awaiting transaction on bookie side
  * `overturning` new result came, awaiting transaction on bookie side
  * `settled` win/lose money transferred on bookie side
- result?: 'win' | 'lose'

Some extra fields might be needed to track request timings / retries to bookie.

Bet status transitions:
```ts
pending: [ 'placed' ]
placed: [ 'settling' ]
settling: [ 'overturning', 'settled' ]
overturning: [ 'settled' ]
settled: [ 'overturning' ]
```

Note that statuses with continuous names (-ing) indicate that communication
with bookie regarding this bet is either scheduled or ongoing.

If task db entity or message broker is used, those statuses are not strictly necessary, but useful for visibility anyway.

## Produced Events
Assuming both broker and users connected to PlayCast,
broker doesn't have to send updates to users about game status.

### bet_update
Happens when user's bet changes status.
Payload is `{ ...bet }` with some system fields omitted.

## Exposed methods
All methods below require authorization with role `user`
  and have access to `user_id` from session or jwt.

Communication protocol might be
- HTTP/2 using json or protobuf
- gRPC over HTTP/2

### connect
- perform auth handshake
- establish duplex communication channel

### create_bet
- accept `{ bet_id, game_id, outcome, amount }`
  where amount > 0
- validate request - all fields are required, amount > 0
- execute transaction:
  * if account with specified `user_id` does not exist
    abort transaction and return (422) `{ code: 'account_not_found' }`
  * if corresponding `game` is not found
    abort transaction and return (422) `{ code: 'invalid_game_id' }`
  * if `game.bet_status` is not `open`
    abort transaction and return (422) `{ code: 'bets_off' }`
  * create `bet` entity with provided params and status `pending`
  * return (200) `{ ...bet }`

Created bet schedules [`update_bet_task`](#update_bet_task).

### cancel_bet
Similar approach is in [`create_bet`](#create_bet).

### update_bet
Similar approach is in [`create_bet`](#create_bet).

## Internal methods

### update_game
Executed as a reaction on PlayCast event.
Updates game, schedules processing for bet updates.

Game updates with invalid payload are logged and discarded.
Valid updates are recorded into db (`game_update` entity).

- accept `{ game_id, status, outcome, overturned_at }`
- validate payload
  * if `game_id` is missing
    throw `{ code: 'missing_game_id' }`.
  * if `status` is `finished` and `outcome` is missing
    throw `{ code: 'missing_outcome' }`.
- execute transaction:
  * update the `game` entity with the provided parameters
  * create `game_update` record
  * if `status` transition is not allowed
    throw `{ code: 'invalid_game_status' }`, abort transaction
  * if `status` is `scheduled`
    set `game.bet_status` to `open`
  * if `status` is `started` or `finished`
    set `game.bet_status` to `closed`
  * if `status` is `finished`
    set `game.outcome` and `game.finished_at`
  * if handling an overturn (`overturned_at` is provided)
    set `game.outcome` to the new `outcome`
    update `game.overturned_at` with the provided timestamp.

Note that game status can change from `scheduled` to `finished`,
it means that update with `started` status was missed.

In order to mitigate the risk of losing money in that case,
most obvious solution would be to schedule `bet_status.closed` update
when `bet_status` is set to `open` based on known `scheduled_start_time`
or any other information (e.g. configured `max_betting_interval`).

If `game_update.status` is not `scheduled` it triggers [`on_game_updated_task`](#on_game_updated_task).

## Async processing

Everything that requires updating large amount of db entities or communication with external service (bookie) should be offloaded
from the default application execution flow and possibly
from primary broker application itself.

In order to offload work reliably we need to save state somewhere.

Using primary db allows to schedule task transactionally with
update that caused it, but increases risks of outage for primary db.

Keeping task state is also necessary in order to handle
retries, crashes, redeploys, etc.

The options are:
1. use message broker for tasks (RabbitMQ, Kafka, etc.)
2. use separate storage for tasks (Redis, MongoDb, etc.)
3. ad-hoc task processing code, keep tasks in primary db
4. ad-hoc task processing code, embed required system fields
  into corresponding entities like `game_update` or `bet`
  (e.g. `status`, `locked_by`, `locked_until`, `started_at`, etc.)
5. external task scheduler (e.g. `pgboss`), keep tasks in primary db

When selecting an option it's important to think about:
- additional delay in processing (we need to react as fast as possible)
- how reliable is worker subscription
  periodic polling might not be an option since we need realtime
- subscriptions and worker scalability

#### on_game_updated_task
`game_update` triggers N updates in bets.

Depending on async processing option,
for each bet with corresponding `game_id`
  schedule or execute task [`update_bet_task`](#update_bet_task) with unique key `{game_id}_${bet_id}`


### update_bet_task

Invoke bookie `create_bet` / `complete_bet` / `overturn_bet`.

Since request to bookie is idempotent, it's safe to use retries in case of failure.

If partner interface (bookie) does not operate with bet entity, calculate transaction to be made on broker side and pass with `transaction_id`.
