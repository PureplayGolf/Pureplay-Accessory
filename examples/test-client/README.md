# Test client

A single-file browser client that exercises the whole Version 1 API: pairing, subscriptions and
event filters, snapshots, every event, all seven commands, the readiness gate, rate limiting and
the rejection codes. Use it to see what Pureplay actually sends before you write your own client,
or to check your understanding of the [spec](../../docs/protocol.md).

No server, no build step, no dependencies.

## Running it

1. Open `index.html` in Chrome, Edge or Firefox. Some embedded browsers and app webviews refuse
   connections to `localhost`; if nothing connects, try a normal browser window first.
2. Leave **Host** as `127.0.0.1` when Pureplay runs on the same PC, or enter the game PC's address on
   your network. `0.0.0.0` is not an address you can connect to.
3. Press **Connect**. A `hello` message should arrive straight away.
4. Press **Pair as new client** and approve the request in Pureplay.

Pureplay must have **Allow apps from** set to *This PC* or *Your network* in Settings > Accessories,
or nothing listens on the port. See [Implementation status](../../docs/protocol.md#implementation-status)
for what the current build supports.

## A suggested test pass

Work down the numbered panels.

| Step | What to do | What it shows you |
|---|---|---|
| Connect | Press **Connect** | The `hello` message and what the build supports |
| 1 Pair | **Pair as new client**, then **Deny** in the game | `pairing.pending` (no `seq`), then `unauthorised` and close `4403`. Pair again at once: `rate_limited` with `retryAfterMs`, then close `4429` - the pairing cooldown |
| | After the cooldown, pair and allow, then reload the page and press **Pair** | A token is issued, and the saved token reconnects without asking the player again |
| 2 Subscribe | **Subscribe + snapshot** | The acknowledgement, then a snapshot before any live event |
| | Set filters to `shot.*` and subscribe again | Trailing-wildcard family filters. `*.changed` matches nothing, by design |
| 3 Readiness | Play a shot and watch the panel | `shot.ready_changed` firing on each transition, with its blockers |
| 4 Commands | `club.set`, `club.next`, `aim.nudge`, then the two resets | Each reply reports the resulting state, so clamping or snapping is visible |
| 5 Error paths | Each button | One specific rejection code apiece, including a 65-character `id` |

Try a command while a shot is in progress, or on another player's turn. It should come back
`not_ready` with the blocker list attached.

## What it checks for you

- **Events seen** - a counter for every event in the spec, so you can see which ones you have not
  received yet. Any event the client does not know about gets its own counter.
- **seq** - the per-connection event counter. It should never skip; a gap means messages were lost,
  and the client logs it.
- **Unparseable messages** - every message should be one JSON object. Anything else is logged with
  its first 80 characters.
- **The token** is kept in `localStorage` under `ppaccessory_token`. **Forget token** clears it.
