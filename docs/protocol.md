# Pureplay Accessory API

Version 1 - preview for feedback.

> **Status: preview.** Version 1 is published so the people who will build against it can tell us
> what is wrong with it before it ships. Nothing here is in a public Pureplay release yet, and parts
> of it are still being built. See [Implementation status](#implementation-status) for what works
> today.

## What it is

Pureplay Accessory lets your app see what is happening in a round of golf as it happens, and optionally
expose typical game controls but not inject launch data.

It is a WebSocket you connect to and JSON messages you exchange. If you can use a WebSocket, you
can build for Pureplay.

Things we expect people to build:

- stream overlays, scoreboards and shot tracers
- physical button boxes, tournament consoles and bay hardware
- accessibility controllers
- shot loggers, coaching dashboards and practice tools

## Why a WebSocket

We looked at plain HTTP polling, server-sent events, a raw TCP socket and a few other options.
WebSocket won because it is the only one that is browser-native, push-based and bidirectional at
the same time. Overlays need to run in a browser with no shim, hardware needs low latency, and both
events and commands need to share one connection.

Version 1 is WebSocket only. If you need something other than a network socket, talk to us.

## Before you start

In Pureplay, open **Settings > Accessories**. Three switches matter:

- **Allow apps from** - *Nowhere* (off), *This PC* (loopback only) or *Your network* (every
  interface). The tab shows the address apps should connect to, on port **9770** by default.
- **Ask about new apps** - when off, new apps are turned away and only previously approved apps
  connect.
- **Allow aim and club** - the master switch for commands. When off, every app is read-only
  whatever it was granted.

Changes take effect when the player presses Apply or Save. Changing *Allow apps from* drops every
connection; approved apps reconnect with their saved token.

If your app runs on the same PC as the game, use `localhost` with *This PC*. If it runs on a phone,
tablet or a second PC, the player must choose *Your network* and you use the game PC's address. UDP
multicast is a possible expansion in a later version.

---

## Quick start

A complete working client. Paste into an HTML file and open it while the game is running.

```html
<pre id="log"></pre>
<script>
// Two things are required by the transport:
//   1. ask for the "binary" subprotocol - it is the only one the server offers;
//   2. read frames as binary - the server never sends text frames, so decode before parsing.
const ws = new WebSocket("ws://localhost:9770/", "binary");
ws.binaryType = "arraybuffer";

ws.onopen = () => {
  // Say who you are. Send back the token we were given last time, if we have one.
  ws.send(JSON.stringify({ type: "pair", data: {
    client: "My First Overlay",
    token: localStorage.getItem("ppaccessory_token")
  }}));
};

ws.onmessage = (e) => {
  const text = (typeof e.data === "string")
    ? e.data
    : new TextDecoder("utf-8").decode(e.data);
  const msg = JSON.parse(text);

  if (msg.type === "pairing.pending") {
    log("Check the game - approve the connection request.");
  }

  if (msg.type === "paired") {
    // Save the token so next time we connect without a prompt.
    localStorage.setItem("ppaccessory_token", msg.data.token);
    ws.send(JSON.stringify({ type: "subscribe", data: { snapshot: true } }));
    log("Connected.");
  }

  if (msg.type === "event") {
    log(msg.event + " " + JSON.stringify(msg.data));
  }
};

function log(s) { document.getElementById("log").textContent += s + "\n"; }
</script>
```

The first time you run it, Pureplay asks the player to approve your app. After that it connects
silently.

---

## Connecting

### 1. Open the socket

```
ws://<host>:9770/
```

The server immediately sends a `hello`:

```json
{ "type": "hello", "data": {
    "protocol": "ppaccessory",
    "version": 1,
    "game": "Pureplay",
    "gameVersion": "1.4.2",
    "paired": false,
    "connectionId": "c-8f21a3c0d4e5",
    "serverInstanceId": "s-3b90f1e2a7b8",
    "capabilities": {
      "events": ["state.snapshot", "context.invalidated",
                 "match.started", "match.ended", "match.paused", "match.resumed",
                 "hole.started", "hole.completed", "turn.changed", "phase.changed",
                 "player.joined", "player.left", "player.changed",
                 "player.club_changed", "player.aim_changed",
                 "player.bag_changed",
                 "shot.initialized", "shot.launched", "shot.completed",
                 "shot.cancelled", "shot.replayed", "shot.ready_changed",
                 "lie.changed", "ruling.required", "ruling.resolved", "score.updated"],
      "commands": ["aim.nudge", "aim.set", "aim.reset",
                   "club.set", "club.next", "club.prev", "club.reset"],
      "snapshotSections": ["session", "course", "hole", "turn",
                           "players", "shot", "readiness", "lie", "ruling"]
    }
} }
```

Check `version` is one you support. **Ignore any field you do not recognise**, in this and every
other message - we add fields over time and that must not break your app.

`capabilities` lists exactly what this build supports, so you can degrade gracefully against older
or newer versions instead of hard-coding assumptions. `events` lists **exact event names**, never
families: an event that is not listed is never sent. Match them with the same rule as a
subscription filter. `gameVersion` is an opaque string (development builds report
`"LocalDevBuild"`). `paired` is always `false` here, because `hello` arrives before you pair.

`serverInstanceId` changes when Pureplay restarts. `connectionId` changes on every socket. Compare
them to tell a dropped socket apart from a game restart. Both are opaque strings.

### 2. Pair

Send this as your first message, always:

```json
{ "type": "pair", "data": {
    "client": "My First Overlay",
    "version": "1.0",
    "token": null
} }
```

- `client` is the name shown to the player in the approval prompt. Use something recognisable.
- `token` is `null` the first time. After that, send the token you saved.

There is no password to copy and no config file to edit. If you have no token, the game shows an
approval prompt and you get:

```json
{ "type": "pairing.pending", "data": { "expiresInSec": 60 } }
```

Show the user something like "Approve the connection in Pureplay", then wait.

`pairing.pending` is sent when your request reaches the front of the queue, and that is when its
countdown starts; a request behind another one waits silently first. The rules around the prompt:

- One pending request per device address. A second `pair` from the same address while one is
  waiting gets `bad_request`, as does a second `pair` on the socket that is already waiting.
- After a request from your address is **denied, times out, or its app is removed** in Pureplay,
  further `pair` messages from that address are answered `rate_limited` with `retryAfterMs` (two
  minutes by default) and the socket then closes `4429`. No prompt is shown. Honour
  `retryAfterMs` before trying again.
- A valid saved token is never subject to the queue or the cooldown.
- If the player has turned off **Ask about new apps**, `pair` without a valid token is answered
  `forbidden_scope` and the socket stays open.
- Pureplay accepts a limited number of connections (eight by default). Beyond that a socket is
  closed `4429` without a `hello`.

Every refusal that closes the socket is a `response` first (`unauthorised` or `rate_limited`,
carrying your `pair` message's `id`), then the close frame with the code from the table below.

On success, whether by prompt or by a valid saved token:

```json
{ "type": "paired", "data": {
    "token": "PPA-6F1C2A9B3D4E4F50A1B2C3D4E5F60718",
    "clientId": "B1E7B0C2-9A3F-4C1D-8E2B-5F6A7B8C9D0E",
    "scopes": ["read", "control"]
} }
```

`token` and `clientId` are opaque strings of up to 64 characters. Store them, compare them, never
parse them.

**Save the token.** Send it on every future connection and the player is never prompted again.

`scopes` tells you what you were granted. `read` means events and state. `control` means you may
also send commands. The player chooses at the prompt, so **do not assume you have `control`** -
check for it and hide your control UI if it is absent.

The player can see every paired app in Settings > Accessories and remove any of them at any time.
Removing an app sends it an `unauthorised` response and closes its connection with `4401`.

### 3. Subscribe

```json
{ "type": "subscribe", "id": "sub-1", "data": {
    "events": ["shot.*", "turn.changed", "hole.*"],
    "snapshot": true
} }
```

- `events` accepts exact names or a **trailing family wildcard** such as `shot.*`. Patterns like
  `*.changed` are not supported. Omit `events` entirely to receive everything. Note that `shot.*`
  includes `shot.ready_changed`.
- `snapshot: true` sends the full current state immediately. Recommended.

**You receive no events until you subscribe.** Before that, a paired connection gets only
responses and the snapshots it asks for with `state.get`. `subscribe` is acknowledged with
`{ "type": "response", "id": "sub-1", "ok": true, "data": {} }` before the snapshot.

**Ordering guarantee.** When you subscribe with `snapshot: true`, Pureplay activates the
subscription and captures the snapshot in the same step, then sends the snapshot before any live
event. The snapshot carries a `seq`, and every event after it has a higher `seq`. You will never
receive an event that is already reflected in the snapshot you were given, and you will never miss
one that is not.

---

## Message envelope

Every event looks like this:

```json
{ "type": "event",
  "event": "shot.completed",
  "seq": 128,
  "time": "2026-07-27T14:03:11.123Z",
  "data": { } }
```

- `seq` counts **only `type: "event"` messages**, on this connection, starting at 1 and increasing
  by exactly one. Responses and other message types do not consume a sequence number.
- `time` is the time Pureplay raised the event, in UTC. Not the time you received it.
- `data` holds the payload documented below.
- Entity identifiers (`matchId`, `shotId`, `playerId`, `rulingId`) always live **inside `data`**,
  never in the envelope.

If `seq` jumps, you missed something. Request a fresh snapshot (see below).

### Payloads are complete objects, not deltas

Every event payload is a **complete, self-contained object** for the thing it describes. We do not
send partial patches. Change events additionally carry `previous` and `current` where a before and
after is meaningful.

- A field that is **absent** was not applicable to this event.
- A field that is **`null`** is known to be empty or unavailable right now.
- Property names are `lowerCamelCase` everywhere.

### Resynchronising

`state.get` is a **request, not a command** - it needs only the `read` scope.

```json
{ "type": "state.get", "id": "s-17" }
```

You get back a normal snapshot event tagged with your request id:

```json
{ "type": "event", "event": "state.snapshot", "requestId": "s-17",
  "seq": 129, "time": "...", "data": { } }
```

---

## Readiness: when is it safe to act

This is the signal to build on, for both UI state and physical hardware. Do not try to work
readiness out yourself from `phase` and `turn` - it depends on conditions we do not all expose
individually, so Pureplay computes it for you.

```json
{ "type": "event", "event": "shot.ready_changed", "seq": 130, "time": "...", "data": {
    "shotId": "shot-42",
    "playerId": 1,
    "ready": true,
    "controllable": true,
    "allowedCommands": ["aim.nudge", "aim.set", "aim.reset",
                        "club.set", "club.next", "club.prev", "club.reset"],
    "blockers": []
} }
```

When something is in the way:

```json
{ "shotId": "shot-42", "playerId": 1,
  "ready": false, "controllable": false,
  "allowedCommands": [],
  "blockers": ["awaiting_decision"] }
```

- **`ready`** - the game is genuinely waiting for this shot to be played.
- **`controllable`** - *your* connection may send commands for this player right now. This is
  distinct from your granted scope: scope says whether you are authorised at all, `controllable`
  says whether the game currently permits it.
- **`allowedCommands`** - which commands will be accepted. A ranked or restricted mode can shrink
  this list, so drive your UI from it rather than hard-coding our rules.
- **`blockers`** - why not, when `ready` or `controllable` is false.

Blocker values:

| Blocker | Meaning |
|---|---|
| `not_in_play` | No round in progress, or still loading |
| `interrupted` | The match is paused (the in-round menu is open), or the game is between phases |
| `not_your_turn` | No player is up, or the player who is up is seated on another machine |
| `shot_in_progress` | The ball is moving or the shot is still being resolved |
| `awaiting_decision` | The game is waiting on the player for a ruling that blocks play |
| `controls_disabled` | The player has turned off **Allow aim and club** in Settings > Accessories |

More than one can be present. An empty list with `ready: true` means go. `controls_disabled` is
the one blocker that does not affect `ready`: you can see `ready: true, controllable: false,
blockers: ["controls_disabled"]`, meaning the shot is waiting but apps may not steer it.

These say **why you cannot act**, not what the game is doing. If you need the detail - which menu,
which ruling, why play stopped - read `phase`, `ruling` and the match events. We deliberately keep
this list short and stable so your control logic does not have to track our internal states.

The event fires on **both** transitions - entering and leaving readiness, and only when something
in the object actually changed. The same object appears in the snapshot as `readiness`, so a
client that connects late is never out of step: read it from the snapshot, do not wait for the
event.

Rate limiting is deliberately **not** a blocker. It is a property of your connection, not of the
golf shot, and it surfaces in the command response instead.

---

## Event reference

Example ids below (`match-123`, `shot-42`, `ruling-9`) are illustrative. Real `matchId` and
`shotId` values are UUID strings; `rulingId` is `ruling-<n>`. Treat all of them as opaque.

### Lifecycle, in order

- Loading a course with no round in play: `context.invalidated { reason: "loading" }`.
- The first hole begins: `match.started`, then `hole.started`.
- Every later hole: `hole.started`.
- The round finishes: `match.ended { reason: "completed" }`.
- The player quits to the menu mid-round: `match.ended { reason: "abandoned" }`, then
  `context.invalidated { reason: "ended" }`.
- A practice restart mid-round: `match.ended { reason: "abandoned" }`, then
  `context.invalidated { reason: "reset" }`.

`match.*` events are only sent for a round whose start Pureplay published to you. A client that
connects mid-round sees no `match.started`; the snapshot has everything it needs.

### Multiplayer

The API is meant to run on the machine that is **hosting** the round. On a machine that joined
someone else's round, readiness is approximate (it may never report `shot_in_progress`), and the
shot family (`shot.*`), `lie.changed`, `hole.completed` and the answers in `ruling.resolved` are
not sent. `handedness`, `profileId` and the club in `shot.initialized` are known only for players
seated on the connected machine.

### Match

**`match.started`**
```json
{ "matchId": "match-123", "courseId": "StAndrews", "courseName": "St Andrews",
  "holeCount": 18, "totalPar": 72, "scoringModel": "Stroke",
  "isTeamGame": false, "playerIds": [1, 2] }
```
`scoringModel` is one of `Stroke`, `Stableford`, `MatchPlay`.

**`match.ended`**
```json
{ "matchId": "match-123", "reason": "completed",
  "standings": [ { "playerId": 1, "rank": 1, "totalStrokes": 71, "toPar": -1 },
                 { "playerId": 2, "rank": 2, "totalStrokes": 76, "toPar": 4 } ] }
```
`reason` is one of `completed`, `abandoned`. `rank` is `null` when the round was abandoned before it
was scored. `toPar` is `null` unless the scoring model is `Stroke`.

**`match.paused`** / **`match.resumed`**
```json
{ "matchId": "match-123" }
```
Sent when the player opens and closes the in-round menu, which pauses the game. While paused,
readiness carries the `interrupted` blocker.

**`context.invalidated`**
```json
{ "reason": "loading", "matchId": null }
```
Raised when previous shot, lie and player context can no longer be trusted - course loading, a
round reset, or a return to the main menu. **If you drive physical hardware, treat this as a
command to return to a safe neutral state.** `reason` is one of `loading`, `reset`, `ended`.

### Hole

**`hole.started`**
```json
{ "matchId": "match-123", "holeNumber": 7, "par": 4, "strokeIndex": 3,
  "yards": 410, "pin": [420.0, 91.0, 4.1], "playerIds": [1, 2] }
```
`pin` is the cup in play. `yards` is measured from the active player's tee and is `null` when the
course has no yardage for that tee.

**`hole.completed`**
```json
{ "matchId": "match-123", "holeNumber": 7,
  "results": [ { "playerId": 1, "holeStrokes": 3, "grossToPar": -1,
                 "netToPar": -2, "holeResult": "Birdie", "holedOut": true },
               { "playerId": 2, "holeStrokes": 5, "grossToPar": 1,
                 "netToPar": 1, "holeResult": "Bogey", "holedOut": true } ] }
```

### Turn and phase

**`turn.changed`**
```json
{ "matchId": "match-123", "previousPlayerId": 2, "currentPlayerId": 1,
  "holeNumber": 7, "reason": "next_player" }
```
`reason` is one of `new_hole`, `next_player`. `previousPlayerId` is `null` for the first turn of a
round. The event is not sent when the turn simply clears at the end of a hole (the snapshot shows
`turn.activeShooterId` as `null`), nor when the same player is selected again on the same hole.

**`phase.changed`**
```json
{ "matchId": "match-123", "previous": "ResolveShot", "current": "AwaitShot" }
```
Phases: `InitMatch`, `InitHole`, `TurnSelect`, `AwaitShot`, `ResolveShot`, `HoleScoring`,
`MatchEnd`. For deciding whether to act, prefer `shot.ready_changed` over this.

### Players

**`player.joined`** - the complete player object (see the snapshot for its shape).

**`player.left`**
```json
{ "playerId": 3 }
```

**`player.changed`** - a player's identity changed; carries the complete current object plus a
reason.
```json
{ "playerId": 1, "current": { }, "reason": "handedness_changed" }
```
`reason` is one of `name_changed`, `team_changed`, `handicap_changed`, `local_changed`,
`finished_changed`, `handedness_changed`, `profile_changed`. Scores, ball, aim and club are not
reported here; they have their own events.

**`player.club_changed`**
```json
{ "playerId": 1, "shotId": "shot-42",
  "previous": "Iron7", "current": "Iron8",
  "previousClubId": "9b1c0000-0000-0000-0000-000000000001",
  "currentClubId": "9b1c0000-0000-0000-0000-000000000002",
  "reason": "user_selected" }
```
`reason` is one of `user_selected`, `api_command`, `auto` (Pureplay's own suggestion at the start
of a turn). `previous` is `null` for the first club of a turn. Sent for players seated on the
connected machine. `previousClubId` / `currentClubId` are bag ids (see *The bag*); `null` when the held club is not a bag entry.

**`player.aim_changed`**
```json
{ "playerId": 1, "shotId": "shot-42",
  "previous": [418.0, 90.0, 4.1], "current": [420.0, 91.0, 4.1],
  "reason": "api_command" }
```
`previous` or `current` is `null` when the aim marker is hidden (on the green, or close enough to
aim straight at the cup).

**`player.bag_changed`** - the player's bag changed (a club added, removed or edited, or another bag
made active). Carries the complete bag; replace your copy.
```json
{ "playerId": 1, "bag": [ ... ] }
```
`bag` has the shape documented under *The bag* and is `null` for a player seated on another machine.
A `clubId` you were holding may no longer exist after this event.

### Shot

Every shot gets a **stable `shotId`** when `shot.initialized` fires. It appears on every event
belonging to that shot and is never reused within a match. A mulligan or re-hit creates a **new**
`shotId` that points back at the original.

**`shot.initialized`**
```json
{ "shotId": "shot-42", "matchId": "match-123", "playerId": 1,
  "holeNumber": 7, "strokeNumber": 2,
  "ballLocation": [141.0, 92.7, 3.9], "club": "Iron7",
  "clubId": "9b1c0000-0000-0000-0000-000000000001",
  "distanceToHoleM": 152.0, "replacesShotId": null }
```
`club` and `clubId` are `null` for a player seated on another machine.

**`shot.launched`**
```json
{ "shotId": "shot-42", "playerId": 1,
  "ballSpeedMs": 61.2, "launchAngleDeg": 18.4, "directionDeg": -1.5,
  "backSpinRpm": 5200, "sideSpinRpm": -240 }
```
These are the values the ball actually flew with, after any lie penalty or power adjustment, so a
tracer drawn from them matches the flight. `directionDeg` is positive to the right of the target
line. Spin comes from the launch monitor when one reported it, otherwise it is derived from the
total spin and axis.

**`shot.completed`**
```json
{ "shotId": "shot-42", "matchId": "match-123", "playerId": 1,
  "holeNumber": 7, "strokeNumber": 2,
  "outcome": "Green", "penaltyStrokes": 0, "isPutt": false,
  "carryM": 138.4, "totalM": 145.1, "apexM": 24.6,
  "startDistToHoleM": 152.0, "endDistToHoleM": 7.4,
  "start": [12.4, 88.1, 3.2], "end": [141.0, 92.7, 3.9],
  "lie": "Green", "holeStrokes": 2 }
```
`outcome` is one of `InPlay`, `Green`, `Holed`, `OB`, `Water`, `Lost`, `Unplayable`,
`PenaltyOnly`, `Gimme`, `AutoPutt`.

**`shot.cancelled`**
```json
{ "shotId": "shot-42", "playerId": 1, "reason": "discarded" }
```
`reason` is one of `discarded` (a played shot the game dropped, for example a scramble candidate the
team did not pick) or `superseded` (a shot that was set up but never played: a skipped turn, a
withdrawal, a reset).

**`shot.replayed`**
```json
{ "shotId": "shot-51", "replacesShotId": "shot-42",
  "playerId": 1, "reason": "mulligan" }
```
`reason` is one of `mulligan`, `rehit`. Sent straight after the replacement's `shot.initialized`,
which is the next time that player is up. An out-of-bounds re-hit is automatic and needs no ruling.

**`lie.changed`**
```json
{ "shotId": "shot-42", "playerId": 1,
  "surface": "Fairway",
  "normalWorld": [0.02, -0.05, 0.998],
  "slopeUpDownDeg": 1.2,
  "slopeLeftRightDeg": -0.4,
  "slopeFrame": "target_line",
  "reason": "ball_at_rest" }
```
`reason` is one of `teed` (the ball is on the tee), `ball_at_rest` (it came to rest somewhere new,
a drop included) or `aim_changed` (the ball did not move but the target line did, so the slopes
changed). Only the active player's lie is tracked. While the ball is in the air or not yet placed,
`normalWorld` and the two slopes are `null` and `surface` is the last known value or `null`. Values
are rounded before comparison, so a resting ball does not produce a stream of events.

See the lie conventions section below - the signs are defined precisely and you should read them
before acting on these values.

### Ruling and score

A ruling is a decision Pureplay is waiting on. Some block play (`blocksPlay: true`) and add the
`awaiting_decision` blocker; others are offers the player may ignore by simply playing on. The
rulings Pureplay raises in Version 1:

| Ruling | `outcome` | `allowedActions` | `blocksPlay` |
|---|---|---|---|
| Hazard relief | `Water` | `AcceptShot`, `TakeDrop`, and `Mulligan` when allowed | `false` |
| Unplayable ball, drop needed | `Unplayable` | `TakeDrop`, `AcceptShot` | `false` |
| Choose how many putts to hole out | `Green` | `AcceptShot` | `true` |
| Team ball selection (scramble) | `null`, `playerId` `null` | `AcceptShot` | `true` |

**`ruling.required`**
```json
{ "rulingId": "ruling-9", "shotId": "shot-42", "playerId": 1,
  "outcome": "Water", "reason": "Shot finished in a hazard; relief has been taken",
  "allowedActions": ["AcceptShot", "TakeDrop"],
  "blocksPlay": false }
```

**`ruling.resolved`**
```json
{ "rulingId": "ruling-9", "shotId": "shot-42", "playerId": 1,
  "outcome": "Water", "selectedAction": "TakeDrop",
  "penaltyStrokes": 1, "dropLocation": [130.2, 90.4, 3.8], "resolved": true }
```
`selectedAction` is one of `AcceptShot`, `RehitWithPenalty`, `TakeDrop`, `ConcedeHole`,
`ApplyGimme`, `Discard`, `Mulligan`. `dropLocation` is `null` when the answer involved no drop.
When a ruling is withdrawn rather than answered (the turn moved on, a reset, or this machine is not
hosting the round) `resolved` is `false`, `selectedAction` and `dropLocation` are `null` and
`penaltyStrokes` is `0`.

**`score.updated`**
```json
{ "matchId": "match-123", "shotId": "shot-42",
  "holeNumber": 7, "playerId": 1, "teamIndex": null,
  "holeStrokes": 3, "totalStrokes": 32,
  "grossToPar": -1, "netToPar": -2,
  "holeResult": "Birdie", "holedOut": true,
  "final": true, "reason": "shot_completed" }
```
`reason` is one of `shot_completed`, `ruling_resolved`, `hole_completed`, `corrected` (a change
with no single cause, such as a score recomputed). `final` is `true` only with reason
`hole_completed`: until then a mulligan can still undo a hole-out. `holeResult` is one of `Eagle`,
`Birdie`, `Par`, `Bogey`, `DoubleBogey`, clamped at both ends (an albatross reports `Eagle`, a
triple bogey reports `DoubleBogey`).

`grossToPar` and `holeResult` are `null` before the player's first stroke on the hole. `netToPar`
is `null` in match play. `holedOut` is `false` for a conceded hole. In team games `teamIndex` is
populated; one event is still sent per player, never per team.

Pureplay publishes the interpreted result so you never have to reimplement our scoring rules.

---

## The state snapshot

Everything at once. Most overlays need only this plus `turn.changed` and `shot.completed`.

```json
{ "session": { "apiVersion": 1, "matchId": "match-123", "phase": "AwaitShot",
               "isTeamGame": false, "scoringModel": "Stroke", "matchActive": true },
  "course":  { "courseId": "StAndrews", "displayName": "St Andrews",
               "holeCount": 18, "totalPar": 72 },
  "hole":    { "number": 7, "par": 4, "strokeIndex": 3, "yards": 410,
               "pin": [420.0, 91.0, 4.1] },
  "turn":    { "activeShooterId": 1, "remainingPlayerIds": [1, 2] },
  "players": [ { "playerId": 1,
                 "profileId": "p-77c1",
                 "name": "Joe",
                 "handedness": "RH",
                 "isLocal": true,
                 "teamIndex": 0,
                 "handicap": 8.0,
                 "totalStrokes": 31,
                 "toPar": 2,
                 "finished": false,
                 "hole": { "strokes": 2, "putts": 0, "holedOut": false,
                           "onGreen": true, "handicapStrokes": 1 },
                 "ball": [141.0, 92.7, 3.9],
                 "aim": [420.0, 91.0, 4.1],
                 "club": "Iron7",
                 "clubId": "9b1c0000-0000-0000-0000-000000000001",
                 "lie": "Green",
                 "distanceToHoleM": 7.4,
                 "bag": [
                   { "clubId": "3f2a0000-0000-0000-0000-000000000002", "type": "Driver", "name": "Driver" },
                   { "clubId": "9b1c0000-0000-0000-0000-000000000001", "type": "Iron7", "name": "7 Iron" } ] } ],
  "shot":    { "shotId": "shot-42", "playerId": 1, "holeNumber": 7,
               "strokeNumber": 2, "state": "AwaitingLaunch" },
  "readiness": { "shotId": "shot-42", "playerId": 1,
                 "ready": true, "controllable": true,
                 "allowedCommands": ["aim.nudge", "aim.set", "aim.reset",
                                     "club.set", "club.next", "club.prev", "club.reset"],
                 "blockers": [] },
  "lie":     { "surface": "Green",
               "normalWorld": [0.02, -0.05, 0.998],
               "slopeUpDownDeg": 1.2,
               "slopeLeftRightDeg": -0.4,
               "slopeFrame": "target_line" },
  "ruling":  null }
```

- `session.scoringModel` is one of `Stroke`, `Stableford`, `MatchPlay`.
- `shot.state` is `AwaitingLaunch` while the game waits for the shot, else `Resolving`.
- `players[].toPar` is **gross** strokes over par across the holes that player has completed.
- `players[].lie` and `lie.surface` are the course's own surface names, not a closed list. Common
  values: `Tee`, `Fairway`, `Fringe`, `Semi`, `Rough`, `DeepRough`, `Bunker`, `Green`, `Water`,
  `Hazard`. Expect others on some courses, and `null` when unknown.
- `ruling` is the active player's pending ruling, else the first one that blocks play. A ruling
  that is only an offer (hazard relief, an unplayable drop) is shown here without the
  `awaiting_decision` blocker.
- `hole.yards` is `null` when the course has no yardage for the player's tee.

### The bag

`players[].bag` is the player's active bag in the order Pureplay shows it, or `null` for a player on
another machine. Each entry:

| Field | Meaning |
|---|---|
| `clubId` | Opaque id, the value `club.set` takes. Stable while the club stays in the bag |
| `type` | One of `Driver`, `Wood2` to `Wood9`, `Hybrid2` to `Hybrid9`, `Iron1` to `Iron9`, `PitchingWedge`, `GapWedge`, `SandWedge`, `LobWedge`, `Putter` |
| `name` | The club's display name, as the player sees it in Pureplay |

That is the whole entry: no loft, make or model. A bag can hold two clubs of the same `type` (two sand
wedges, say), which is why `club.set` takes an id and not a type; tell them apart by `name`. `players[].clubId` is the held club's id. It is `null` when the
player is holding a club that is not in the bag, and for a player seated on another machine.

### Empty and transitional states

The snapshot is always valid, whatever the game is doing. Sections that do not currently apply are
**`null`** - never missing, never stale, never a placeholder you have to detect.

| Situation | What the snapshot looks like |
|---|---|
| Main menu | `session.matchActive` false; `course`, `hole`, `turn`, `shot`, `lie`, `ruling` all `null`; `players` empty |
| Match loading | `session.matchActive` false, `course` populated, rest `null`, `readiness.blockers` contains `not_in_play` |
| No active player | `turn.activeShooterId` `null`, `shot` `null` |
| Ball not yet placed | `shot` populated, `lie` `null` |
| Player holed out | player's `hole.holedOut` true, `shot` `null` until the next turn |
| Ruling pending | `ruling` populated; `readiness.blockers` contains `awaiting_decision` when it blocks play |
| Match ended, results showing | `session.matchActive` false, `course` and `players` retained, `shot` and `lie` `null` |

### Units and conventions

- Distances are **metres**, speeds **metres per second**, angles **degrees**, spin **RPM**.
- Positions are `[x, y, z]` arrays in metres, world space, `z` up.
- Enumerated values are always **strings** (`"AwaitShot"`, `"Iron7"`, `"Fairway"`), never numbers.
- `playerId` is a number, stable for the whole round. Match players between messages on it.
- `profileId` is optional, may be `null`, and is stable across rounds. It identifies a profile
  slot on the game PC (`"local-3"`), so it is set for players seated on the machine you are
  connected to and `null` for players on another machine. It carries no account information.
- `handedness` is `"RH"`, `"LH"` or `"Unknown"`. It is known for players seated on the connected
  machine only; others report `"Unknown"`.
- Ids (`matchId`, `shotId`, `rulingId`, `clientId`, `connectionId`, `serverInstanceId`, tokens)
  are opaque strings. Compare them; never parse them or depend on their format.
- Times are ISO-8601 UTC.
- `club` fields (`players[].club`, `shot.initialized.club`, `player.club_changed`, command replies) use the same vocabulary as a bag entry's `type`.

### Lie coordinate and slope conventions

Read this before driving anything physical from these numbers.

- **`normalWorld`** is the surface normal as a normalised vector in **world space**.
- **`slopeUpDownDeg`** and **`slopeLeftRightDeg`** are expressed relative to the active player's
  **target line** - the line from the ball to the aim marker, or to the cup when no marker is set.
  That is what `slopeFrame: "target_line"` means. `slopeFrame` is present on every lie payload so a
  future frame can be added without ambiguity.
- **`slopeUpDownDeg` greater than zero means uphill toward the target.**
- **`slopeLeftRightDeg` greater than zero means the ground rises to the right of the target line**,
  seen from behind the ball looking at the target. For a right-handed player that is "ball above
  the feet"; for a left-handed player it is "ball below the feet".
- **Handedness does not invert these.** The geometry above is fixed. Apply handedness yourself if
  your device needs it.

A caution on precision: these values come from sampling the ground around the ball, and are
intended for display, feedback and coarse platform positioning. They are not a survey-grade
measurement of the surface. If you are driving a moving floor, treat them as an input to your own
smoothing and limits rather than as an absolute target.

---

## Sending commands

Only if you were granted the `control` scope **and** `readiness.controllable` is true.

```json
{ "type": "command", "id": "c-17", "command": "club.set",
  "data": { "playerId": 1, "clubId": "9b1c0000-0000-0000-0000-000000000001" } }
```

You always get a reply with the same `id`. Successful replies **return the resulting state**,
because Pureplay may clamp, snap or normalise what you asked for. Every successful reply carries
`playerId`, the resulting `club` and `clubId` (`null` when the held club is not a bag entry); aim commands add the resulting `aim` (`null` when the marker
is hidden), and `aim.nudge` adds the `deltaDeg` actually applied after clamping:

```json
{ "type": "response", "id": "c-17", "ok": true,
  "data": { "playerId": 1, "club": "Iron7", "clubId": "9b1c0000-0000-0000-0000-000000000001" } }
```
```json
{ "type": "response", "id": "c-18", "ok": true,
  "data": { "playerId": 1, "club": "Iron7", "clubId": "9b1c0000-0000-0000-0000-000000000001", "aim": [420.0, 91.0, 4.1], "deltaDeg": 5 } }
```
```json
{ "type": "response", "id": "c-17", "ok": false,
  "error": { "code": "not_ready",
             "message": "Not accepting commands.",
             "blockers": ["shot_in_progress"] } }
```

| Command | Data | What it does |
|---|---|---|
| `aim.nudge` | `{ playerId?, deltaDeg }` | Turn the aim left (negative) or right (positive) |
| `aim.set` | `{ playerId?, target: [x,y,z] }` | Aim at a world position |
| `aim.reset` | `{ playerId? }` | Undo your changes and restore the aim Pureplay had set |
| `club.set` | `{ playerId?, clubId }` | Select a club from the player's bag |
| `club.next` / `club.prev` | `{ playerId? }` | Step through the player's bag |
| `club.reset` | `{ playerId? }` | Undo your changes and restore the club Pureplay had selected |

`playerId` is optional and defaults to the active player. If you send it, it must be the active
player, or the command is refused with `not_ready`.

**`aim.nudge`** rotates the current aim point about the ball by `deltaDeg`. If no aim is set yet it
starts from Pureplay's default aim for the shot. It is a one-shot delta, not the held-key nudge the
player has on screen, so the two do not move in lockstep.

**`club.set`** installs the bag club with that `clubId`, fitting included. An id that is not in the
active bag (including one that was removed after you read it) is refused with `bad_request`.

**What the reset commands restore.** The first time an API command changes the aim or the club for a
shot, Pureplay remembers what it was beforehand. `aim.reset` and `club.reset` put that value back,
so the player ends up on whatever they would have had if your app had never touched it.

- Aim and club are remembered **independently** - resetting one leaves the other alone.
- The remembered value is captured on your first aim or club command **for that shot**, and cleared
  when the shot ends. Reset always refers to the current shot, never an earlier one.
- If your app has not changed the club this shot, `club.reset` succeeds and does nothing.
- If your app has not changed the aim this shot, `aim.reset` restores **Pureplay's default aim**
  for the shot, which also discards any aim the player placed themselves. Only send `aim.reset`
  after one of your own aim commands.
- Otherwise reset undoes **your app's** changes, not the player's. If the player moved the aim
  themselves after you did, resetting still returns to the pre-API value.

Give your users a way to reach these. Any app that changes aim or club should be able to hand
control back - it matters most for accessibility controllers and button boxes, where a player can
otherwise be left lined up somewhere they did not intend.

That is the entire control surface. There is deliberately **no command to play a shot**. Your app
can help the player line up. It cannot swing for them.

### Validation and limits

- `id` must be unique among your **pending** commands. Reusing an id after its response has
  arrived is fine.
- `id` maximum 64 characters. `client` name maximum 64 characters. Longer values are refused with
  `bad_request`.
- `deltaDeg` is clamped to plus or minus 45 degrees per call. The reply reports the value applied.
- `aim.set` accepts any three finite numbers today. Rejecting targets outside the playable area is
  planned (see Implementation status).
- Rate limits are **per connection**: 20 commands per second, burst 40. The allowance starts empty
  when you connect and fills at 20 per second, so a command sent in the first few milliseconds of a
  connection can be rate limited. Commands refused for scope, readiness or the wrong player do not
  consume the allowance; a command refused for a bad argument does.

A rate-limited response tells you when to try again:

```json
{ "type": "response", "id": "c-18", "ok": false,
  "error": { "code": "rate_limited", "message": "Slow down.", "retryAfterMs": 50 } }
```

### Why a command gets rejected

Drive your UI from `readiness` and most rejections disappear. The codes:

| Code | Meaning |
|---|---|
| `unauthorised` | Not paired yet; or your token is dead, your request was denied, timed out, or the app was removed (the socket then closes with the matching code below) |
| `forbidden_scope` | You were granted read-only, the player has turned off **Allow aim and club**, or (on `pair`) the player has turned off **Ask about new apps** |
| `not_ready` | The game will not accept that right now - see `blockers` on the response |
| `rate_limited` | Slow down, see `retryAfterMs` - also sent to `pair` during the pairing cooldown |
| `unknown_command` | Check spelling, or the game is older than your app |
| `bad_request` | Malformed JSON, unknown message type, a value out of range, `id` or `client` too long, an unknown club or player, or a second pairing request while one is waiting |

A `not_ready` response caused by readiness repeats the current blockers so you do not have to go
looking. A `not_ready` for a `playerId` that is not the active player, or when no round is running,
carries a message only.

### Connection close codes

Each of these is preceded by a `response` (`unauthorised`, or `rate_limited` for the cooldown) so
your client can log the reason before the socket goes.

| Code | Meaning | What to do |
|---|---|---|
| `4401` | Token rejected, or the player removed your app | Delete your saved token and pair again |
| `4403` | The player denied your request, or has turned off new apps | Tell the user, do not retry automatically |
| `4408` | Approval timed out | Offer a retry button |
| `4429` | Too many pairing attempts (see `retryAfterMs`), or too many connections (no `hello` was sent) | Back off before retrying |

---

## Reconnecting

Sequence numbers are **per connection**. They cannot tell you what you missed while you were away.
Version 1 does not replay missed events. The contract:

- `seq` restarts at 1 on every new connection.
- On reconnect you must request a snapshot - subscribe with `snapshot: true`.
- The snapshot is authoritative current state.
- **Events that occurred while you were disconnected are not replayed.** A shot may have been
  played, a hole completed, a penalty applied. You will see the result, not the events.
- Because of that, be careful about firing one-shot automation (a celebration, a light cue, a
  sound) from differences you infer between an old cached state and a fresh snapshot.

Compare `serverInstanceId` from `hello` against the one you saw before. If it changed, Pureplay
restarted and you should discard all cached state rather than diffing against it.

Reconnect with a backoff: 1s, 2s, 4s, up to about 30s. Do not hammer it.

---

## Versioning

`version` in `hello` is the major version. We add fields and events without bumping it, so:

- ignore fields you do not recognise
- ignore event types you do not recognise
- check `capabilities` rather than assuming a feature exists

Breaking changes bump the major version, and the old one keeps working for a deprecation period.

---

## Security and network exposure

Be aware of what this is before you deploy it in a venue:

- With **Allow apps from** set to *Your network*, Pureplay listens on every interface, not just
  loopback, and any device that can reach the port can attempt to pair. *This PC* keeps it to
  loopback.
- Pairing requires a person to approve the request on screen, so nothing is served to an unapproved
  client - it receives only `hello` and may send only `pair`.
- The connection is **`ws://`, not encrypted.** On an untrusted network, pairing tokens and game
  data are visible to anyone able to observe the traffic. Treat LAN mode as a trusted-network
  feature.
- Tokens are bound to the paired client entry, not to an origin or IP. Store yours somewhere your
  users would consider reasonable.
- The player can remove any app at any time, which closes its connection immediately.
- Pureplay stores only a hash of your token, never the token itself.

---

## Tips

**Always save the token.** If you lose it the player gets prompted again, which gets annoying fast.

**Handle read-only.** Check `scopes` and hide any control UI you cannot use.

**Drive UI from `readiness`, not from `phase`.** It is the only authoritative answer, and it tells
you *why* when the answer is no.

**Do not poll.** Events are pushed. Take a snapshot when you connect, then keep up with events.

**Handle `context.invalidated`.** Especially if you control hardware - it is your signal to return
to a safe neutral state.

**Be honest in your client name.** It is what the player sees when deciding whether to trust you.

**Test the unhappy paths.** Reconnects, pauses, rulings, mulligans and the main menu are where
integrations break, not the ordinary shot.

---

## Feedback wanted

This is a preview. Specific things we would like opinions on:

1. **Is the event list right now?** What is still missing, and what would you never use?
2. **Is `readiness` sufficient?** Are there blockers you need that are not listed?
3. **Are the lie slope conventions usable** for the hardware you are building?
4. **Is the command set too small?** Tell us what you cannot build, and why that should change.
5. **Snapshot shape** - is anything awkward to consume, or missing for a scoreboard?
6. **WebSocket only** - does having no HTTP interface cause you a real problem?

Start a thread in
[Discussions](https://github.com/PureplayGolf/pureplay-accessory/discussions). If you are
connecting over something other than a network socket, or you want an event that does not exist
yet, tell us - the event list will grow based on what people actually ask for. Use
[Issues](https://github.com/PureplayGolf/pureplay-accessory/issues) for mistakes in this document
or the examples.

---

## Implementation status

Nothing in Version 1 is in a public Pureplay release yet. Everything below exists in Pureplay
development builds. **Verified** means it has been exercised end to end against a live game.
**Built** means the code is in place and covered by automated tests, with the in-game pass still
to run. **Planned** means designed, not built. This table is updated as parts land.

| Area | Status | Notes |
|---|---|---|
| WebSocket transport, `binary` subprotocol, binary frames | Verified | |
| `hello` with `gameVersion`, `serverInstanceId`, `connectionId` and `capabilities` | Verified | `capabilities.events` lists exact names |
| Pairing: approval prompt, `pairing.pending`, saved tokens | Verified | |
| Settings > Accessories: turning the API on, removing an app, the three switches | Verified | |
| Pairing cooldown, connection limit, close codes `4401`, `4403`, `4408`, `4429` | Verified | |
| `subscribe`, event filters, `snapshot: true`, `state.get`, envelope and `seq` | Verified | |
| All 24 events with the payloads in this document | Built | |
| Snapshot with `null` sections and every documented field | Built | |
| `aim.set`, `aim.nudge`, `aim.reset`, `club.next`, `club.reset` | Verified | Replies carry the resulting club and `clubId` |
| `club.set` by `clubId`, `club.prev`, aim replies carrying the resulting `aim`, `id` and `client` length limits | Built | |
| Rate limit of 20 per second, burst 40; `deltaDeg` clamp; rejection codes | Verified | |
| Rejecting `aim.set` targets outside the playable area | Planned | Only non-numeric targets are rejected today |
| `players[].bag`, `player.bag_changed` | Built | |
