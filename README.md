# Pureplay Accessory

Pureplay Accessory is a WebSocket API for apps that follow a round of golf in Pureplay as it
happens - and, if the player allows it, help them line up by changing aim and club. It never plays
the shot for them.

## What you can build

- stream overlays, scoreboards and shot tracers
- physical button boxes, tournament consoles and bay hardware
- accessibility controllers
- shot loggers, coaching dashboards and practice tools

## In this repository

| Path | What it is |
|---|---|
| [`docs/protocol.md`](docs/protocol.md) | The Version 1 specification: connecting, pairing, events, the snapshot, commands |
| [`examples/browser`](examples/browser) | A minimal overlay client - one HTML file, no dependencies |
| [`examples/test-client`](examples/test-client) | A full test client that exercises every message, command and error |

## Two things every client must get right

Most first attempts fail on one of these:

1. **Ask for the `binary` subprotocol.** It is the only one Pureplay offers.
2. **Frames are binary, not text.** Decode each message as UTF-8 before parsing the JSON.

```js
const ws = new WebSocket("ws://localhost:9770/", "binary");
ws.binaryType = "arraybuffer";

ws.onmessage = (e) => {
  const msg = JSON.parse(new TextDecoder("utf-8").decode(e.data));
  // ...
};
```

A client that treats messages as text connects without error and then fails on the first one.

