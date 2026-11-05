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

In Pureplay, open Settings and turn on Pureplay Accessory. It shows the port, default **9770**.

If your app runs on the same PC as the game, use `localhost`. If it runs on a phone, tablet or a
second PC, use the game PC's address on your network. UDP multicast is a possible expansion in a
later version.

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
    "connectionId": "c-8f21",
    "serverInstanceId": "s-3b90",
    "capabilities": {
      "events": ["match.*", "hole.*", "turn.changed", "phase.changed",
                 "player.*", "shot.*", "lie.changed", "ruling.*",
                 "score.updated", "context.invalidated"],
      "commands": ["aim.nudge", "aim.set", "aim.reset",
                   "club.set", "club.next", "club.prev", "club.reset"],
      "snapshotSections": ["session", "course", "hole", "turn",
                           "players", "shot", "lie", "ruling", "readiness"]
    }
} }
```

Check `version` is one you support. **Ignore any field you do not recognise**, in this and every
other message - we add fields over time and that must not break your app.

`capabilities` lists exactly what this build supports, so you can degrade gracefully against older
or newer versions instead of hard-coding assumptions.

`serverInstanceId` changes when Pureplay restarts. `connectionId` changes on every socket. Compare
them to tell a dropped socket apart from a game restart.

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

On success, whether by prompt or by a valid saved token:

```json
{ "type": "paired", "data": {
    "token": "PPL-4K7Q-92XF-1RM8",
    "clientId": "b1e7b0c2-0000-0000-0000-000000000000",
    "scopes": ["read", "control"]
} }
```

**Save the token.** Send it on every future connection and the player is never prompted again.

`scopes` tells you what you were granted. `read` means events and state. `control` means you may
also send commands. The player chooses at the prompt, so **do not assume you have `control`** -
check for it and hide your control UI if it is absent.

The player can see every paired app in Settings and revoke any of them at any time. Revoking closes
the connection immediately.

### 3. Subscribe

```json
{ "type": "subscribe", "data": {
    "events": ["shot.*", "turn.changed", "hole.*"],
    "snapshot": true
} }
```

- `events` accepts exact names or a **trailing family wildcard** such as `shot.*`. Patterns like
  `*.changed` are not supported. Omit `events` entirely to receive everything.
- `snapshot: true` sends the full current state immediately. Recommended.

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
| `interrupted` | The match is paused, or a menu has focus |
| `not_your_turn` | Another player is up, or that player is not on this machine |
| `shot_in_progress` | The ball is moving or the shot is still being resolved |
| `awaiting_decision` | The game is waiting on the player - a ruling, a drop, a choice |
| `controls_disabled` | Controls are turned off for this mode |

More than one can be present. An empty list with `ready: true` means go.

These say **why you cannot act**, not what the game is doing. If you need the detail - which menu,
which ruling, why play stopped - read `phase`, `ruling` and the match events. We deliberately keep
this list short and stable so your control logic does not have to track our internal states.

The event fires on **both** transitions - entering and leaving readiness. The same object appears in
the snapshot as `readiness`, so a client that connects late is never out of step.

Rate limiting is deliberately **not** a blocker. It is a property of your connection, not of the
golf shot, and it surfaces in the command response instead.

---

## Event reference

### Match

**`match.started`**
```json
{ "matchId": "match-123", "courseId": "StAndrews", "courseName": "St Andrews",
  "holeCount": 18, "totalPar": 72, "scoringModel": "Stroke",
  "isTeamGame": false, "playerIds": [1, 2] }
```

**`match.ended`**
```json
{ "matchId": "match-123", "reason": "completed",
  "standings": [ { "playerId": 1, "rank": 1, "totalStrokes": 71, "toPar": -1 },
                 { "playerId": 2, "rank": 2, "totalStrokes": 76, "toPar": 4 } ] }
```
`reason` is one of `completed`, `abandoned`.

**`match.paused`** / **`match.resumed`**
```json
{ "matchId": "match-123" }
```

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
`reason` is one of `new_hole`, `next_player`.

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

**`player.changed`** - any player attribute changed; carries the complete current object plus a
reason.
```json
{ "playerId": 1, "current": { }, "reason": "handedness_changed" }
```

**`player.club_changed`**
```json
{ "playerId": 1, "shotId": "shot-42",
  "previous": "Iron7", "current": "Iron8", "reason": "user_selected" }
```
`reason` is one of `user_selected`, `api_command`, `auto`.

**`player.aim_changed`**
```json
{ "playerId": 1, "shotId": "shot-42",
  "previous": [418.0, 90.0, 4.1], "current": [420.0, 91.0, 4.1],
  "reason": "api_command" }
```

### Shot

Every shot gets a **stable `shotId`** when `shot.initialized` fires. It appears on every event
belonging to that shot and is never reused within a match. A mulligan or re-hit creates a **new**
`shotId` that points back at the original.

**`shot.initialized`**
```json
{ "shotId": "shot-42", "matchId": "match-123", "playerId": 1,
  "holeNumber": 7, "strokeNumber": 2,
  "ballLocation": [141.0, 92.7, 3.9], "club": "Iron7",
  "distanceToHoleM": 152.0, "replacesShotId": null }
```

**`shot.launched`**
```json
{ "shotId": "shot-42", "playerId": 1,
  "ballSpeedMs": 61.2, "launchAngleDeg": 18.4, "directionDeg": -1.5,
  "backSpinRpm": 5200, "sideSpinRpm": -240 }
```

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

**`shot.replayed`**
```json
{ "shotId": "shot-51", "replacesShotId": "shot-42",
  "playerId": 1, "reason": "mulligan" }
```
`reason` is one of `mulligan`, `rehit`.

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
See the lie conventions section below - the signs are defined precisely and you should read them
before acting on these values.

### Ruling and score

**`ruling.required`**
```json
{ "rulingId": "ruling-9", "shotId": "shot-42", "playerId": 1,
  "outcome": "OB", "reason": "Shot finished out of bounds",
  "allowedActions": ["RehitWithPenalty", "TakeDrop", "ConcedeHole"],
  "blocksPlay": true }
```

**`ruling.resolved`**
```json
{ "rulingId": "ruling-9", "shotId": "shot-42", "playerId": 1,
  "outcome": "OB", "selectedAction": "TakeDrop",
  "penaltyStrokes": 1, "dropLocation": [130.2, 90.4, 3.8], "resolved": true }
```
Actions: `AcceptShot`, `RehitWithPenalty`, `TakeDrop`, `ConcedeHole`, `ApplyGimme`, `Discard`,
`Mulligan`.

**`score.updated`**
```json
{ "matchId": "match-123", "shotId": "shot-42",
  "holeNumber": 7, "playerId": 1, "teamIndex": null,
  "holeStrokes": 3, "totalStrokes": 32,
  "grossToPar": -1, "netToPar": -2,
  "holeResult": "Birdie", "holedOut": true,
  "final": true, "reason": "shot_completed" }
```
`reason` is one of `shot_completed`, `ruling_resolved`, `hole_completed`, `corrected`.
`final` is `true` once the value can no longer change for that hole.

Fields that do not apply to the active scoring model are `null`. In team games `teamIndex` is
populated and one event is sent per affected player or team.

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
                 "lie": "Green",
                 "distanceToHoleM": 7.4 } ],
  "shot":    { "shotId": "shot-42", "playerId": 1, "holeNumber": 7,
               "strokeNumber": 2, "state": "AwaitingLaunch",
               "replacesShotId": null },
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
| Ruling pending | `ruling` populated, `readiness.blockers` contains `awaiting_decision` |
| Match ended, results showing | `session.matchActive` false, `course` and `players` retained, `shot` and `lie` `null` |

### Units and conventions

- Distances are **metres**, speeds **metres per second**, angles **degrees**, spin **RPM**.
- Positions are `[x, y, z]` arrays in metres, world space.
- Enumerated values are always **strings** (`"AwaitShot"`, `"Iron7"`, `"Fairway"`), never numbers.
- `playerId` is a number, stable for the whole round. Match players between messages on it.
- `profileId` is optional, may be `null`, and is stable across rounds. It carries no account
  information.
- `handedness` is `"RH"`, `"LH"` or `"Unknown"`.
- Times are ISO-8601 UTC.

### Lie coordinate and slope conventions

Read this before driving anything physical from these numbers.

- **`normalWorld`** is the surface normal as a normalised vector in **world space**.
- **`slopeUpDownDeg`** and **`slopeLeftRightDeg`** are expressed relative to the active player's
  **target line**, which is what `slopeFrame: "target_line"` means. `slopeFrame` is present on
  every lie payload so a future frame can be added without ambiguity.
- **`slopeUpDownDeg` greater than zero means uphill toward the target.**
- **`slopeLeftRightDeg` greater than zero means the ball is above the player's feet.**
- **Handedness does not invert these.** The published physical meaning is the same for a
  right-handed and a left-handed player. Apply handedness yourself if your device needs it.

A caution on precision: these values come from sampling the ground around the ball, and are
intended for display, feedback and coarse platform positioning. They are not a survey-grade
measurement of the surface. If you are driving a moving floor, treat them as an input to your own
smoothing and limits rather than as an absolute target.

---

## Sending commands

Only if you were granted the `control` scope **and** `readiness.controllable` is true.

```json
{ "type": "command", "id": "c-17", "command": "club.set",
  "data": { "playerId": 1, "club": "Iron7" } }
```

You always get a reply with the same `id`. Successful replies **return the resulting state**,
because Pureplay may clamp, snap or normalise what you asked for:

```json
{ "type": "response", "id": "c-17", "ok": true,
  "data": { "playerId": 1, "club": "Iron7" } }
```
```json
{ "type": "response", "id": "c-17", "ok": false,
  "error": { "code": "not_ready",
             "message": "Not accepting commands.",
             "blockers": ["shot_in_progress"] } }
```

| Command | Data | What it does |
|---|---|---|
| `aim.nudge` | `{ playerId, deltaDeg }` | Turn the aim left (negative) or right (positive) |
| `aim.set` | `{ playerId, target: [x,y,z] }` | Aim at a world position |
| `aim.reset` | `{ playerId }` | Undo your changes and restore the aim Pureplay had set |
| `club.set` | `{ playerId, club }` | Select a club by name |
| `club.next` / `club.prev` | `{ playerId }` | Step through the player's bag |
| `club.reset` | `{ playerId }` | Undo your changes and restore the club Pureplay had selected |

Club names: `Driver`, `Wood2` to `Wood9`, `Hybrid2` to `Hybrid9`, `Iron1` to `Iron9`,
`PitchingWedge`, `GapWedge`, `SandWedge`, `LobWedge`, `Putter`.

**What the reset commands restore.** The first time an API command changes the aim or the club for a
shot, Pureplay remembers what it was beforehand. `aim.reset` and `club.reset` put that value back,
so the player ends up on whatever they would have had if your app had never touched it.

- Aim and club are remembered **independently** - resetting one leaves the other alone.
- The remembered value is captured on your first aim or club command **for that shot**, and cleared
  when the shot ends. Reset always refers to the current shot, never an earlier one.
- If you have not changed anything, reset succeeds and does nothing.
- Reset undoes **your app's** changes, not the player's. If the player moved the aim themselves
  after you did, resetting still returns to the pre-API value.

Give your users a way to reach these. Any app that changes aim or club should be able to hand
control back - it matters most for accessibility controllers and button boxes, where a player can
otherwise be left lined up somewhere they did not intend.

That is the entire control surface. There is deliberately **no command to play a shot**. Your app
can help the player line up. It cannot swing for them.

### Validation and limits

- `id` must be unique among your **pending** commands. Reusing an id after its response has
  arrived is fine.
- `id` maximum 64 characters. `client` name maximum 64 characters.
- `deltaDeg` is clamped to plus or minus 45 degrees per call.
- `aim.set` targets outside the playable area are rejected with `bad_request`.
- Rate limits are **per connection**: 20 commands per second, burst 40. Rejected commands do not
  consume the allowance.

A rate-limited response tells you when to try again:

```json
{ "type": "response", "id": "c-18", "ok": false,
  "error": { "code": "rate_limited", "message": "Slow down.", "retryAfterMs": 250 } }
```

### Why a command gets rejected

Drive your UI from `readiness` and most rejections disappear. The codes:

| Code | Meaning |
|---|---|
| `unauthorised` | Not paired yet |
| `forbidden_scope` | You were granted read-only |
| `not_ready` | The game will not accept that right now - see `blockers` on the response |
| `rate_limited` | Slow down, see `retryAfterMs` |
| `unknown_command` | Check spelling, or the game is older than your app |
| `bad_request` | Malformed message or a value out of range |

A `not_ready` response repeats the current blockers so you do not have to go looking. There is one
vocabulary for "you cannot act right now", and it is the blocker list.

### Connection close codes

| Code | Meaning | What to do |
|---|---|---|
| `4401` | Token rejected or revoked | Delete your saved token and pair again |
| `4403` | The player denied your request | Tell the user, do not retry automatically |
| `4408` | Approval timed out | Offer a retry button |
| `4429` | Too many attempts | Back off before retrying |

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

- Pureplay listens on your **local network**, not just loopback. Any device that can reach the
  port can attempt to pair.
- Pairing requires a person to approve the request on screen, so nothing is served to an unapproved
  client - it receives only `hello` and may send only `pair`.
- The connection is **`ws://`, not encrypted.** On an untrusted network, pairing tokens and game
  data are visible to anyone able to observe the traffic. Treat LAN mode as a trusted-network
  feature.
- Tokens are bound to the paired client entry, not to an origin or IP. Store yours somewhere your
  users would consider reasonable.
- The player can revoke any app at any time, which closes its connection immediately.

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

Nothing in Version 1 is in a public Pureplay release yet. This table tracks what works in Pureplay
development builds today, so you can tell the design apart from what has been built. It is updated
as parts land.

- **Available** - works as documented.
- **Partial** - works, but some documented fields are not populated yet.
- **Planned** - designed, not built yet.

### Connection

| Feature | Status | Notes |
|---|---|---|
| WebSocket transport, `binary` subprotocol, binary frames | Available | |
| `hello` | Partial | `gameVersion`, `serverInstanceId` and `capabilities` are not sent yet |
| Pairing with a saved token | Available | |
| Approval prompt and `pairing.pending` | Planned | **Until this ships, a new app cannot pair in a public build** |
| Turning the API on from Pureplay's Settings | Planned | |
| Revoking an app from Settings | Planned | |
| Connection close codes `4401` to `4429` | Planned | |
| `subscribe`, event filters, `snapshot: true` | Available | |
| `state.get` | Available | |
| Envelope, `seq` and `time` | Available | |

### Events

| Event | Status | Notes |
|---|---|---|
| `shot.ready_changed` | Partial | `shotId` is not populated yet |
| `phase.changed` | Partial | Sends `current` only |
| `turn.changed` | Partial | Sends `currentPlayerId` only |
| `hole.started` | Partial | Sends `holeNumber` only |
| `shot.completed` | Partial | Sends `playerId`, `holeNumber` and `holeStrokes` only |
| `player.changed` | Partial | Sent with an empty payload - request a snapshot for the detail |
| `match.started`, `match.ended`, `match.paused`, `match.resumed` | Planned | |
| `context.invalidated` | Planned | |
| `hole.completed` | Planned | |
| `player.joined`, `player.left`, `player.club_changed`, `player.aim_changed` | Planned | |
| `shot.initialized`, `shot.launched`, `shot.cancelled`, `shot.replayed` | Planned | |
| `lie.changed` | Planned | |
| `ruling.required`, `ruling.resolved` | Planned | |
| `score.updated` | Planned | |

### Snapshot

| Section | Status | Notes |
|---|---|---|
| `session`, `course`, `hole`, `turn`, `players`, `shot`, `readiness`, `lie` | Partial | Not populated yet: `session.matchId`, `hole.yards`, `shot.shotId`, `shot.replacesShotId`, `readiness.shotId`, `players[].profileId`, the slope fields of `lie`. `players[].handedness` is always `"Unknown"` |
| `ruling` | Planned | |
| `null` for sections that do not apply | Planned | Sections are currently always present, with default values |

### Commands

| Feature | Status | Notes |
|---|---|---|
| `aim.set`, `aim.nudge`, `aim.reset`, `club.set`, `club.next`, `club.reset` | Available | |
| `club.prev` | Planned | Currently answered with `unknown_command` |
| Replies return the resulting state | Partial | Club commands return the resulting club; aim commands do not return the resulting aim yet |
| Rate limit of 20 per second, burst 40 | Available | |
| `deltaDeg` clamp | Available | |
| `id` and `client` length limits | Planned | |
| Rejecting `aim.set` targets outside the playable area | Planned | |
| Rejection codes | Available | |
