# tldraw — known issue: canvas "flicker" / disappears

**Symptom:** the drawing area appears, then vanishes; refreshing brings it back
briefly, then it drops again (a flicker/refresh loop).

**Root cause — a bug in the `node-tldraw` *server*, not this deployment.**
The homelab side is healthy: HTTPS page loads (200), the WebSocket reaches the
server through the Tailscale proxy, and SQLite persistence works.

The server's WebSocket upgrade handler
([`server/sync-server.mjs`](https://github.com/bradmartin333/node-tldraw/blob/main/server/sync-server.mjs),
the `server.on('upgrade', …)` block) rejects the sync connection with a raw
`HTTP/1.1 404` whenever `boardExistsForRoom(roomId)` is false. tldraw's
`useSync` treats a rejected socket as a **fatal store error and unmounts the
canvas**, then retries. Any time the client opens a sync socket for a room
whose board row isn't present at that instant — a default/empty-state board,
or a race where `POST /api/boards` hasn't committed before the `wss` connect —
the server 404s it and the canvas disappears.

**Fix belongs in the `bradmartin333/node-tldraw` repo** (baked into the image;
not patchable from these manifests). Options, most robust first:

1. Client: `await` the `POST /api/boards` response before mounting `useSync`,
   so the board row always exists first. Fixes the race, keeps the server's
   "no orphaned rooms" guard. *(Recommended.)*
2. Server: on upgrade for an unknown room, close the socket cleanly with a
   WebSocket close code the client can handle, instead of a raw HTTP 404.
3. Server: auto-create the board row on first sync (simplest; drops the
   anti-orphan guard).

`latest` == `2.0.0` on Docker Hub (same digest), so there is no newer image to
pull — the server must be rebuilt after fixing. Once a new tag is pushed,
update the image in `tldraw.yaml` (or let phase 9 image automation do it).
