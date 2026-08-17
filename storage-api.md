# Cobblemon Expeditions - Custom Storage Backend Guide

Cobblemon Expeditions keeps per-player expedition state behind a small interface, so
a network can store it wherever it already stores everything else. The mod ships two
backends of its own (`world` and `shared_file`) and no database driver, so if you run
Redis, Mongo or SQL you write a small addon mod that plugs into the same interface.

This guide is everything you need. You do not need the mod's source.

## What you are building

A separate mod jar that:

1. depends on Cobblemon Expeditions,
2. implements `ExpeditionDataStore`,
3. registers it under a name during mod init.

Server owners then set `"backend": "<your name>"` in
`config/cobblemon_expeditions.json`.

## Gradle setup

Add the Expeditions jar as a compile-only dependency. It is present at runtime
because it is a required dependency of your addon.

```kotlin
dependencies {
    // Fabric
    modCompileOnly(files("libs/cobblemon-expeditions-fabric-1.6.0.jar"))
    // NeoForge
    // compileOnly(files("libs/cobblemon-expeditions-neoforge-1.6.0.jar"))
}
```

Declare the dependency in `fabric.mod.json`:

```json
"depends": { "cobblemon_expeditions": ">=1.6.0" }
```

or in `neoforge.mods.toml`:

```toml
[[dependencies.your_addon]]
modId = "cobblemon_expeditions"
type = "required"
versionRange = "[1.6.0,)"
```

The interface is Kotlin, so Kotlin is the path of least resistance, but it is a plain
JVM interface and Java works fine.

## The interface

```kotlin
package com.cobblemonexpeditions.store

interface ExpeditionDataStore {
    /** Identifier this backend was registered under. Used in logs and config. */
    val id: String

    /** False for anything shared. See "The four rules" below. */
    val keepsOfflinePlayersResident: Boolean

    /** Once on server start, before any load. Open connections here, not in the factory. */
    fun start(server: MinecraftServer)

    /** Once on server stop, after the final flushAll(). */
    fun stop()

    /** The player's record, read from the backend, created empty if absent. Never null. */
    fun loadOrCreate(playerUuid: UUID): PlayerExpeditionData

    /** The record if it is already resident, without touching the backend. */
    fun peek(playerUuid: UUID): PlayerExpeditionData?

    /** Persist, then drop from memory. */
    fun unload(playerUuid: UUID)

    /** Everything held in memory. Swept every tick, so it must not hit the backend. */
    fun resident(): Collection<PlayerExpeditionData>

    /** Take exclusive network-wide ownership of resolving this expedition. */
    fun claimResolution(data: PlayerExpeditionData, instanceId: UUID): Boolean

    /** Atomically remove a reward from the inbox and return it, or null if already taken. */
    fun claimReward(data: PlayerExpeditionData, rewardId: UUID): RewardBundle?

    /** Persist one record if it has pending changes. */
    fun flush(data: PlayerExpeditionData)

    /** Persist every resident record with pending changes. */
    fun flushAll()
}
```

## Serializing a record

You never need to know what is inside `PlayerExpeditionData`. It round-trips through
NBT, and the registries come off the server handed to `start`:

```kotlin
val tag: CompoundTag = data.toNbt(registries)          // write
val data = PlayerExpeditionData.fromNbt(tag, registries)  // read
```

To get bytes for a blob column or a Redis value:

```kotlin
// tag -> bytes
val out = ByteArrayOutputStream()
NbtIo.writeCompressed(tag, out)
val bytes = out.toByteArray()

// bytes -> tag
val tag = ByteArrayInputStream(bytes).use {
    NbtIo.readCompressed(it, NbtAccounter.unlimitedHeap())
}
```

Useful members:

| Member | Purpose |
| --- | --- |
| `data.playerUuid` | The key. |
| `data.isDirty()` | Whether it needs writing. |
| `data.clearDirty()` | Call after a successful write. |
| `data.removeReward(rewardId)` | Takes a reward out of the inbox and returns it. |
| `data.getActiveExpedition(instanceId)` | Looks up one running expedition, or null. |

Records are typically a few KB compressed.

## The four rules

Get these right and everything else is ordinary CRUD.

**1. `loadOrCreate` must read through on a join.** Do not serve a cached copy from a
previous session. The scheduler credits offline progress from what you return, so a
stale record replays expeditions another server already resolved.

**2. `claimResolution` and `claimReward` must be atomic across the whole network.**
A conditional write in the database, never a read followed by a write:

| Store | Primitive |
| --- | --- |
| Mongo | `insertOne` on a unique `_id`, catch duplicate key |
| Redis | `SET key 1 NX` returns null when it already exists |
| SQL | `INSERT ... ON CONFLICT DO NOTHING`, check rows affected |

These are the only things standing between two servers and a duplicated reward. Since
1.6.0 loot pools can run arbitrary commands, so a double claim can mint economy
currency.

**3. `claimReward` must be durable before it returns.** The caller grants the items
and runs the command rewards immediately after. If you return non-null and then lose
the write, a reward that has been paid out is still claimable.

**4. `keepsOfflinePlayersResident = false`.** The mod then loads a record on join,
evicts it on quit, and only sweeps players online on this server. With `true` it would
expect every player in the world to be resident, which is only correct for a world
save.

## Threading

Every method is called from the server thread, and `PlayerExpeditionData` is not
thread-safe, so do not hand records to a background pool. Blocking I/O is expected;
just keep it quick, because a slow call is a tick stall. Joins and quits are one small
read or write each. The periodic autosave is already spread over several ticks by the
mod, so you do not need to batch it yourself.

## Worked example: Mongo

```kotlin
class MongoExpeditionDataStore(db: MongoDatabase) : ExpeditionDataStore {

    override val id = "mongo"
    override val keepsOfflinePlayersResident = false

    private val records = db.getCollection("expedition_players")
    private val claims = db.getCollection("expedition_claims")
    private val resident = ConcurrentHashMap<UUID, PlayerExpeditionData>()
    private lateinit var registries: HolderLookup.Provider

    override fun start(server: MinecraftServer) {
        registries = server.registryAccess()
    }

    override fun stop() {
        flushAll()
        resident.clear()
    }

    override fun loadOrCreate(playerUuid: UUID): PlayerExpeditionData =
        resident.getOrPut(playerUuid) { read(playerUuid) ?: PlayerExpeditionData(playerUuid) }

    override fun peek(playerUuid: UUID): PlayerExpeditionData? = resident[playerUuid]

    override fun unload(playerUuid: UUID) {
        resident.remove(playerUuid)?.let(::write)
    }

    override fun resident(): Collection<PlayerExpeditionData> = resident.values

    override fun claimResolution(data: PlayerExpeditionData, instanceId: UUID): Boolean {
        val instance = data.getActiveExpedition(instanceId) ?: return false
        if (instance.resolved) return false
        return claim("resolution:$instanceId")
    }

    override fun claimReward(data: PlayerExpeditionData, rewardId: UUID): RewardBundle? {
        if (!claim("reward:$rewardId")) return null
        val reward = data.removeReward(rewardId) ?: return null
        write(data)   // durable before we return
        return reward
    }

    override fun flush(data: PlayerExpeditionData) {
        if (data.isDirty()) write(data)
    }

    override fun flushAll() {
        resident.values.forEach(::flush)
    }

    /** Exactly one caller network-wide gets true: the unique _id makes every later insert fail. */
    private fun claim(key: String): Boolean =
        try {
            claims.insertOne(Document("_id", key).append("at", System.currentTimeMillis()))
            true
        } catch (e: MongoWriteException) {
            if (e.error.category == ErrorCategory.DUPLICATE_KEY) false else throw e
        }

    private fun read(playerUuid: UUID): PlayerExpeditionData? {
        val doc = records.find(Filters.eq("_id", playerUuid.toString())).first() ?: return null
        val bytes = doc.get("nbt", Binary::class.java).data
        val tag = ByteArrayInputStream(bytes).use {
            NbtIo.readCompressed(it, NbtAccounter.unlimitedHeap())
        }
        return PlayerExpeditionData.fromNbt(tag, registries)
    }

    private fun write(data: PlayerExpeditionData) {
        val out = ByteArrayOutputStream()
        NbtIo.writeCompressed(data.toNbt(registries), out)
        records.replaceOne(
            Filters.eq("_id", data.playerUuid.toString()),
            Document("_id", data.playerUuid.toString()).append("nbt", Binary(out.toByteArray())),
            ReplaceOptions().upsert(true)
        )
        data.clearDirty()
    }
}
```

Claim documents are never read again once the thing they guard is finished. Give the
collection a TTL index of a couple of weeks so it does not grow forever:

```javascript
db.expedition_claims.createIndex({ at: 1 }, { expireAfterSeconds: 1209600 })
```

## Notes for Redis

The claim maps cleanly onto `SET NX`:

```kotlin
private fun claim(key: String): Boolean =
    jedis.set(key, "1", SetParams.setParams().nx().ex(14 * 24 * 3600)) != null
```

One thing to watch: **Redis is not durable by default.** With RDB snapshots only, a
crash can lose the last few seconds of writes, which breaks rule 3 - a reward could be
granted and then become claimable again. Either enable `appendonly yes` with
`appendfsync everysec` or better, or keep records in Mongo/SQL and use Redis only for
the claim markers, where losing one costs a single unclaimed reward rather than a
duplicated one.

## Registering it

Anywhere in your mod's init, before the server starts. Loader order does not matter:

```kotlin
ExpeditionDataStores.register("mongo") { MongoExpeditionDataStore(myDatabase) }
```

The factory takes no arguments and is called once per server start. Open connections
in `start`, not in the factory.

Server owners then set:

```json
"storage": {
  "backend": "mongo",
  "autoSaveSeconds": 60
}
```

An unregistered name logs an error and falls back to the world save rather than
failing the boot, so a missing addon degrades instead of taking the network down.

## Testing checklist

Two servers against one database, one account:

- [ ] Start an expedition on A, join B. Rank, timer and Pokémon lock all present.
- [ ] The locked Pokémon cannot battle, trade or be released on B.
- [ ] Let it finish while on B. It resolves once, and the reward lands in the inbox.
- [ ] Claim on B, go back to A. The reward is gone and cannot be claimed again.
- [ ] Exactly one record row per player. Two means the UUIDs differ between servers,
      usually an `online-mode` mismatch rather than a storage problem.
- [ ] Start an expedition, stop both servers, wait past its duration, start one and
      join. It resolves once with the correct completion time.
- [ ] Kill a server mid-session. On restart the player's state is at most
      `autoSaveSeconds` old, and no reward is duplicated.

The last two are the ones that catch a wrong claim implementation.
