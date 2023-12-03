# PlayCast
Service that provides realtime updates about games.

Game is offline event (game/races/fights) which is broadcasted to general audience
or can be attended physically.

For simplicity purposes let's assume `outcome` is an integer number between 1 and `N` where `N` is known constant.

## Produced Events
1. Start of the game `{ game_id, started_at }`
2. End of the game `{ game_id, finished_at, outcome }`
3. Overturn game result `{ game_id, overturned_at, outcome }`
