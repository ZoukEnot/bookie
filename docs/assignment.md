# Overview

The project is built around short bets on game events. The project is designed to integrate with existing bookmaker sites - we don't deal with payments and balances, only game events. There are three main systems in the system:

- Client - a client application installed on the partner's site. In it, the player follows the course of the game and, among other things, presses the "Make / cancel bet" buttons, as well as keeps track of his wins and losses.
- The backend of our platform. It accepts requests from customers, stores the transaction history, and routes requests for debit for bets to the appropriate partners.
- Partner's backend. An external system that stores information about the player's payment balance, but knows nothing about the game itself.

We'll look at a key part of our system - processing player bets, and see what the requirements and complexities are.

# Requirements

## Business rules
The general principle of the game:

1. Our platform receives external events from some board game. For simplicity, let's say it's the throwing of a regular six-sided die. We have three types of events:
- Start of round (nothing happened, players can bet)
- End of betting (no more bets can be placed, the draw begins)
- End of round (some result has occurred, we need to calculate the results of bets).
2. We transfer this information to the client installed on the sites of different bookmakers (there can be many of them), which shows it to the players - and the players can react by pressing buttons in the interface.
4. When a player makes a bet, we poll the relevant bookmaker in realtime that the player can afford to make such a bet (has the money), and block the money in the account until the end of the round.
5. When the round ends, we need to record the wins / losses - ie, to inform all bookmakers, who to charge money, and who to write off. And we need to do it quickly, and there can be quite a lot of messages (e.g. 10k).

Accordingly, from the parameters we have:
- `round_id` - round ID
- `partner_id` - ID of the bookmaker, from which the player came.
- `partner_player_id` - ID of the player inside the bookmaker (since the players come not from us, each player is bound to some bookmaker).

Mechanics of bets and logic of the game (what is the game, what are the results, what bet means what, what are the multipliers to win) lives completely inside our service, only figures about money (plus/minus) go outside.

Approximate numbers:
- up to 200k online players
- up to 10k bets per round
- round length - 15-20 seconds (floating).

## Problems

We have two fundamental issues that complicate the system:

1. Sometimes requests to bookmakers end with an error - for various reasons. Server reboot, network failure, etc. We have to do something about them. The system should calmly survive possible failures and not lose money.

3. sometimes we may receive _incorrect_ results of a round - let's say we got 4, and the operator accidentally entered 5. When the error is detected, we will receive a corresponding event with new results of the round - and we should recalculate all bets (if someone was credited with a loss by mistake - return the money and credit the victory) and bring this change to the bookmakers. While the event can come both almost immediately and after a significant period of time (up to a week).

# Questions

1. We are asking bookmakers to make some HTTP JSON API for integration according to our specification.

How to design the interface of communication with bookmakers so that we can do all the necessary operations, and at the same time minimize the risk of losing money in case of errors?

Omit authorization and security issues. The answer will be a description of types and order of requests with examples of request-response format for each case.

2. Which technologies to use and how to organize such a system of bet processing from the architectural point of view?

3. What risks of the system are seen in production, what means are needed to eliminate them?
