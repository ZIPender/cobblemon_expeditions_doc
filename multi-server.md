# Multi-server storage

By default Cobblemon Expeditions keeps every player's expedition state in the world
save. That is correct for singleplayer and for a single server, and it is what you
get if you change nothing.

On a network of several servers it is also the reason the Expedition Board reads
empty on the second server: each world save has its own copy, so rank, active
expeditions, reward inbox and Pokémon locks all stop at the server boundary. Since
Cobblemon PC boxes are typically synced network-wide, that also means a Pokémon sent
on an expedition on one server is fully usable on another - it can battle, be traded,
and be released while it is supposed to be away.

Since 1.6.0 the storage layer is pluggable, so all of that can follow the player.

## Quickstart: turning sync on

Five steps, assuming your servers can share a directory. Back up every world first.

**1. Put all servers on 1.6.0** with identical expedition, loot table and mode JSON.
Mismatched content is the most common cause of "it works on one server only".

**2. Give every server one directory they all reach.** On a single machine that is a
bind mount or a plain shared path; on Pterodactyl it is Admin → Mounts, attached to
both the node *and* each server. Make sure the server user can write to it.

**3. On each server**, edit `config/cobblemon_expeditions.json`:

```json
"storage": {
  "backend": "shared_file",
  "sharedDirectory": "shared/cobblemon_expeditions",
  "autoSaveSeconds": 60
}
```

`sharedDirectory` is relative to the server's working directory. On Pterodactyl that
is `/home/container`, so a mount at `/home/container/shared` matches the value above
with no edits.

**4. Restart every server** and check each console for the same line:

```
Shared expedition storage at /home/container/shared/cobblemon_expeditions
Expedition storage backend 'shared_file' active; player records load on join
```

The paths must be identical across servers. If one still says `'world' active`, it
didn't pick up the config or can't see the mount, and it will silently keep using
its own world save.

**5. Verify with one account.** Start an expedition on server A, move to server B,
and confirm the board shows it running with the same rank. Then check
`shared/cobblemon_expeditions/players/` - there should be exactly **one** `.dat` per
player. Two files for one person means their UUID differs between servers, usually
an `online-mode` mismatch.

To turn it back off, set `"backend": "world"` on every server and restart. Each will
go back to its own world save, which still holds whatever it had before you switched.

## Choosing a backend

`config/cobblemon_expeditions.json`:

```json
"storage": {
  "backend": "world",
  "sharedDirectory": "shared/cobblemon_expeditions",
  "autoSaveSeconds": 60
}
```

| `backend` | What it does | Use it when |
| --- | --- | --- |
| `world` | Per-world save, exactly as before 1.6.0. | Singleplayer, or one server. **Default.** |
| `shared_file` | One directory every server can see. | A network whose servers share a filesystem - the same node, a bind mount, an NFS export. |
| *(addon-provided)* | Whatever an addon registers. | A network that already runs Redis, Mongo or SQL. |

An unrecognised `backend` logs an error and falls back to `world` rather than
refusing to boot - a server that starts without expedition sync beats a server pool
that won't start at all.

`autoSaveSeconds` only applies to shared backends; the world save is written by
vanilla's own autosave. Records are always persisted on player quit and on shutdown,
so this only bounds what a crash can cost.

## `shared_file`

```json
"storage": {
  "backend": "shared_file",
  "sharedDirectory": "shared/cobblemon_expeditions"
}
```

`sharedDirectory` is resolved relative to the server's working directory. Every
server must end up at the same physical directory and be able to write to it.

The layout it creates:

```
shared/cobblemon_expeditions/
├── players/
│   └── <player-uuid>.dat          one gzipped-NBT record per player
└── claims/
    ├── resolution/<instance-uuid> marker: this expedition has been resolved
    └── reward/<reward-uuid>       marker: this reward has been handed out
```

Records are written to a temp file and moved into place atomically, so a reader
never sees a partial record. The claim markers are the important part: they are
created with exclusive-create (`O_CREAT | O_EXCL`), which succeeds for exactly one
caller and fails for every other. That is what stops two servers resolving the same
expedition or granting the same reward twice.

This is atomic on a local filesystem and on any ordinary bind mount. On NFS it
depends on the server honouring exclusive create - true for NFSv3 and later, but if
you are running at a scale where that worries you, write an addon against the
database you already trust.

**One deliberate failure direction:** a reward's claim marker is written *before* the
items are handed out. If a server dies in between, that reward is lost rather than
granted twice. Duplicating command rewards would mint economy currency, so the trade
goes this way round.

## Writing your own backend

Networks already running Redis, Mongo or SQL can store expedition data there
instead. See [storage-api.md](storage-api.md) for the full interface, a working
Mongo implementation and a testing checklist.

## How resolution works across servers

Expeditions run on wall-clock time (`elapsedMillis` against `durationMillis`), which
is why this works at all: wall-clock is wall-clock wherever it is read.

- While a player is on a server, that server ticks their expeditions as usual.
- While they are elsewhere, nobody ticks them here - the timestamps just sit.
- On join, the whole window since the record was last touched is credited at once,
  and anything that finished is resolved then.

So an expedition started on the survival server and finished while the player was on
the creative server resolves the moment they come back, with the correct completion
time. The claim guard means it resolves exactly once even if both servers briefly
hold the record during a transfer.

## What does *not* sync

**The Pokémon themselves.** This mod syncs expedition state - rank, XP, running
expeditions, reward inbox, fatigue, cooldowns, augments, locks. It does not sync
party or PC boxes: Cobblemon keeps those in `<world>/pokemon`, per world, and moving
them between servers is your network's job. Most networks already do it (that is why
this feature is wanted in the first place).

If you switch this mod to a shared backend but have no box sync, you get the
confusing half-state where your rank follows you but your party is empty on the
second server. That is working as intended - the missing piece is Cobblemon sync,
not this.



**Expedition, loot table and mode JSON.** These are per-server files, and the in-game
admin GUI edits the local copy. Deploy identical content to every server and make
admin edits in one place, or they will drift. Putting definitions in shared storage
is deliberately out of scope - they are content, not player state.

**Board blocks.** A board is a block in a world; each server has its own. Only the
player state behind it is shared.

## Before you switch a live network

- **Back up every world.** The `.dat` in each world save is left untouched when you
  move to a shared backend, but it is no longer read, so it becomes a stale snapshot
  rather than a live fallback.
- **There is no automatic import.** Switching to a shared backend starts from empty
  records. On a network where progress already diverged per server there is no single
  correct answer for which copy wins, so the mod does not guess.
- **Test the interaction with your PC sync first.** The lock enforcer moves locked
  Pokémon from party back to PC once a second. If your network syncs PC boxes with
  its own snapshot-on-quit mechanism, those two writers need to be checked against
  each other on a test pair before this goes near production. This is the part most
  likely to bite, and it depends on software the mod can't see.
