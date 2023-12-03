# Bookie
Business entity that might holds deposits from certain users.
- allows users to authorize brokers
- exposes methods for brokers to make or overturn bets

User account is represented by two numbers:
- liquid amount (can be withdrawn or spent by user if greater than 0)
- frozen amount (can not be withdrawn or spent by user)

## Exposed methods
Interaction between user an bookie is out of scope of this POC.

Permission must be explicitly granted from user to broker to make bets on his behalf.

All methods below require authorization with role `broker`
  and have access to `broker_id` from session or jwt.

For each method below:
- if user haven't granted or revoked permission to certain `broker`
  return 403 `{ code: 'no_user_permission' }`.

### get_user_balance
- accept `{ user_id }`
- if account exists
  return (200) `{ user_id, liquid_amount, frozen_amount }`
- if user account for `user_id` does not exist
  return (422) `{ code: 'account_not_found' }`

### get_bet
- accept `{ bet_id }`
- if bet does not exist return (422) `{ code: 'bet_not_found' }`
- return (200) `{ ...bet }`

### create_bet
- accept `{ user_id, bet_id, amount }`
  where amount > 0
- validate request:
  * if `amount` is missing or not positive number
    return (400) `{ code: 'invalid_amount' }`
- execute transaction:
  * if account with specified `user_id` does not exist
    abort transaction and return (422) `{ code: 'account_not_found' }`
  * if `account.liquid_amount` < `amount`
    abort transaction and return (422) `{ code: 'insufficient_funds' }`
  * decrement `account.liquid_amount` by `value`
  * increment `account.frozen_amount` by `value`
  * create `bet` entity with given `bet_id`, `user_id`, `amount`, `broker_id`
    and status `active`
  * return (200) `{ liquid_amount, frozen_amount }`

### complete_bet
- accept `{ user_id, bet_id, result, win_amount? }`
  where `result` is either `win` or `lose`, `win_amount` > 0 or `undefined`
- validate request:
  * if `result === 'win'` and `win_amount` missing
    return (400) `{ code: 'missing_win_amount' }`
  * if `result === 'win'` and `win_amount` not positive number
    return (400) `{ code: 'invalid_win_amount' }`
  * if `result === 'lose'` and `win_amount` is present
    return (400) `{ code: 'unexpected_win_amount' }`
- execute transaction:
  * if `bet.broker_id` != `broker_id`
    return (403) `{ code: 'different_broker' }`, abort transaction
  * if `bet` with specified `bet_id` and `user_id` is not found
    return (422) `{ code: 'bet_not_found' }`, abort transaction
  * if `bet.status` is not `active`
    return (422) `{ code: 'bet_not_active' }`, abort transaction
  * if `account.frozen_amount < bet.amount`
    return (422) `{ code: 'insufficient_frozen_amount' }`
  * if `broker_account.liquid_amount < win_amount`
    return (422) `{ code: 'insufficient_broker_funds' }`
  * if `result === 'win'`
    increment `account.liquid_amount` by `bet.amount + win_amount`
    decrement `account.frozen_amount` by `bet.amount`
    decrement `broker_account.liquid_amount` by `win_amount`
    set `bet.win_amount` to `win_amount`
  * if `result === 'lose'`
    decrement `account.frozen_amount` by `bet.amount`
    increment `broker_account.liquid_amount` by `bet.amount`
  * set `bet.result` to `result`
  * set `bet.status` to `completed`
  * return (200) `{ liquid_amount, frozen_amount }`

### overturn_bet
Happens when user's bet changes `result` from `win` to `lose` or vice versa.
This method might cause `account.liquid_amount` to become negative.
- accept `{ user_id, bet_id, result, win_amount? }`
  where `result` is either `win` or `lose`
- validation same as in `complete_bet` method
- execute transaction:
  * if `bet.broker_id` != `broker_id`
    return (403) `{ code: 'different_broker' }`, abort transaction
  * if `bet` with specified `bet_id` and `user_id` is not found
    return (422) `{ code: 'bet_not_found' }`, abort transaction
  * if `bet.result` === `result`
    return (208) `{ code: 'already_reported' }`
  * if `bet.status` is not `completed`
    return (422) `{ code: 'bet_not_completed' }`, abort transaction
  * if `result` == `win` and `broker_account.liquid_amount < 2 * bet.amount + win_amount`
    return (422) `{ code: 'insufficient_broker_funds' }`
  * if `result === 'win'`
    increment `account.liquid_amount` by `2 * bet.amount + win_amount`
    decrement `broker_account.liquid_amount` by `2 * bet.amount + win_amount`
    set `bet.win_amount` to `win_amount`
  * if `result === 'lose'`
    decrement `account.liquid_amount` by `2 * bet.amount + bet.win_amount`
    unset `bet.win_amount`
  * set `bet.result` to `result`
  * set `bet.status` to `completed`
  * return (200) `{ liquid_amount, frozen_amount }`

## Internal methods

### Validate db bet
This method might be invoked internally before responding to exposed methods:
- get_bet
- create_bet
- overturn_bet

If bet is invalid, terminates processing, aborts transaction if any
and returns (500) `{ code: 'corrupted_bet' }`

Assuming `bet_id` is primary key, it cannot be missing.

When bet can be considered invalid:
- `bet.user_id` is missing
- `bet.status` is `active` and `bet.result` is not null
- `bet.status` is `completed` and `bet.result` is not `win` or `lose`
- `bet.result` is `win` and `bet.win_amount` is null or less than 0

### Money withdrawal
Happens when transfers money from bookie account to external bank or withdraws cash. Can be executed when `liquid_amount` is positive.

Because bets can be overturned it might be useful to enforce limits
on minimal `liquid_amount` if user has recently won some bets.

### Money transfer
There are 2 types of money transfers besides withdrawal, which is a separate case.
1. Transfer within same account between `liquid_amount` and `frozen_amount`.
2. Transfer from user account and broker account.

Money cannot appear from nowhere:
- every win is a transfer from broker account to user's `liquid_amount`.
- every loss is a transfer from user's `frozen_amount` to broker account.

This leads to potential bottleneck in obtaining the write lock on broker account.
Solution might be:
- split broker account into multiple sub-accounts to reduce concurrent access
- allow broker account go negative - use insert instead of update
- throw money on the problem and buy super expensive server
