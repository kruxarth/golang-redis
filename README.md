# Redis Clone in Go

This project is a small Redis-inspired server written in Go. It speaks the Redis RESP protocol over TCP, stores string keys in memory, and supports a focused set of Redis-style commands for reads, writes, expiration, persistence, transactions, monitoring, and basic server introspection.

The server listens on `:6379`, keeps all data in an in-memory map, and optionally persists state through:

- `AOF` replay and rewrite
- `RDB` snapshot save and load
- configurable memory limits with eviction policies

## What This Project Supports

Implemented command groups:

- basic key-value: `GET`, `SET`, `DEL`, `EXISTS`, `KEYS`, `DBSIZE`, `FLUSHDB`
- expiration: `EXPIRE`, `TTL`
- persistence: `SAVE`, `BGSAVE`, `BGREWRITEAOF`
- auth: `AUTH`
- transactions: `MULTI`, `EXEC`, `DISCARD`
- observability: `MONITOR`, `INFO`
- compatibility/no-op style command: `COMMAND`

Values are stored as strings. Complex Redis data types such as lists, sets, hashes, and streams are not implemented in this codebase.

## Running It

The entrypoint is [main.go].

```bash
go run .
```

At startup the server:

1. reads `redis.conf`
2. creates the application state
3. replays the AOF file if enabled
4. loads the RDB snapshot if configured
5. starts listening on TCP port `6379`

You can connect with `redis-cli`:

```bash
redis-cli -p 6379
```

## File Structure

This repository uses a flat layout instead of nested packages.

- [main.go](./main.go): bootstraps config, persistence recovery, and the TCP server loop
- [handlers.go](./handlers.go): command registry, request dispatcher, and all command implementations
- [db.go](./db.go): in-memory database, locking, memory tracking, eviction, and lazy expiration
- [item.go](./item.go): stored value model with expiry and access metadata
- [appstate.go](./appstate.go): global runtime state such as config, AOF, stats, monitors, and transaction state
- [value.go](./value.go): RESP request parsing into internal `Value` objects
- [writer.go](./writer.go): RESP response serialization
- [client.go](./client.go): per-connection client state and `MONITOR` output
- [transaction.go](./transaction.go): queued command representation for `MULTI/EXEC`
- [conf.go](./conf.go): `redis.conf` parsing and config model
- [aof.go](./aof.go): append-only file setup, replay, and rewrite
- [rdb.go](./rdb.go): snapshot save/load and periodic save trackers
- [info.go](./info.go): `INFO` output generation
- [mem.go](./mem.go): key sampling helper used by eviction logic
- [utils.go](./utils.go): small shared helpers
- [redis.conf](./redis.conf): example runtime configuration

## Architecture

The project follows a simple single-process server model.

### 1. Network layer

`main.go` accepts TCP connections and spawns one goroutine per client. Each connection is wrapped in a `Client`, and incoming bytes are read with a buffered reader.

### 2. Protocol layer

`value.go` parses RESP arrays and bulk strings into the internal `Value` type. The server expects commands in array form, which matches how `redis-cli` sends requests.

`writer.go` performs the reverse conversion by serializing `Value` instances into RESP replies:

- simple strings
- bulk strings
- integers
- arrays
- errors
- nulls

### 3. Command dispatch

`handlers.go` contains a global `Handlers` map from command name to handler function. The `handle` function is the central dispatcher:

1. read command name from the parsed array
2. look up a handler
3. reject unknown commands
4. enforce authentication when `requirepass` is configured
5. queue commands if a transaction is active
6. execute the handler
7. write the reply
8. update command stats
9. relay the command to monitor clients

### 4. Storage layer

The in-memory database is the global `DB` in [db.go](./db.go). It uses:

- `map[string]*Item` for key storage
- `sync.RWMutex` for concurrency control
- an internal memory counter for eviction decisions

Each `Item` stores:

- the string value
- optional expiration time
- last access time
- access count

That metadata is used by `TTL`, lazy expiration, and LFU/LRU-style eviction.

### 5. Runtime state

`AppState` carries process-wide state that is not part of the raw key-value map:

- parsed config
- AOF writer/file state
- background save/rewrite flags
- monitor subscribers
- transaction queue
- server start time
- connected client count
- memory/high-water stats
- persistence and command statistics

### 6. Persistence layer

The project implements two persistence mechanisms.

#### AOF

`aof.go` appends write commands to an append-only file. On startup, the server replays AOF commands by parsing them back into `Value` objects and calling `SET` logic again.

`BGREWRITEAOF` rewrites the file from a database snapshot so the log becomes a compact sequence of `SET` commands for the current dataset.

#### RDB

`rdb.go` writes the in-memory map to disk using Go `gob` encoding. Snapshot schedules from `redis.conf` create tickers that trigger `SaveRDB` when enough keys changed during the configured interval.

`BGSAVE` copies the database map and saves it asynchronously.

## Request Flow

A typical `SET key value` request moves through the system like this:

1. the client sends a RESP array
2. `value.go` parses it into `Value{typ: ARRAY, ...}`
3. `handlers.go` routes `SET` to the `set` handler
4. `db.go` updates the in-memory map and memory accounting
5. AOF is appended if enabled
6. RDB change trackers are incremented if snapshotting is configured
7. `writer.go` returns `+OK`

`GET` follows the same path but reads from memory, checks for expiration, updates access metadata, and returns either a bulk string or null.

## Persistence and Recovery

The bundled [redis.conf](./redis.conf) enables both AOF and RDB by default:

- data directory: `./data`
- AOF file: `backup.aof`
- fsync mode: `everysec`
- RDB file: `backup.rdb`
- snapshot rule: save after `5` seconds if at least `3` keys changed

Startup recovery order in `main.go` is:

1. replay AOF if enabled
2. load RDB if configured

That order is important to note because it differs from the usual expectation that the most recent persistence source should win. The README documents the current code behavior, not ideal Redis behavior.

## Memory Management and Eviction

The database tracks approximate memory usage per key using [item.go](./item.go). When `maxmemory` is set and a `SET` would exceed it, the database calls `evictKeys`.

Implemented eviction policies in the code:

- `noeviction`
- `allkeys-random`
- `allkeys-lru`
- `allkeys-lfu`

The volatile policies are present in the config enum, but eviction logic currently implements only the all-keys variants above.

Eviction candidates are taken from a fixed-size sample of keys using [mem.go](./mem.go), then selected according to the configured policy.

## Expiration Model

Expiration is lazy. Keys are checked for expiry when they are read through `GET` or `TTL` via `tryExpire`.

What this means:

- `EXPIRE key seconds` sets an absolute expiration timestamp on the item
- expired keys are removed when accessed later
- there is no active background expiry loop in this codebase

## Transactions

Transactions are implemented in [transaction.go](./transaction.go) and coordinated from `handlers.go`.

Behavior:

- `MULTI` starts queuing commands
- commands are stored as `TxCommand` values containing the original request and its handler
- non-`EXEC` and non-`DISCARD` commands return `QUEUED` while the transaction is active
- `EXEC` runs queued commands in order and returns an array of replies
- `DISCARD` drops the queue

This is a lightweight transaction queue. It does not implement Redis features like `WATCH`.

## Monitoring and Introspection

### MONITOR

`MONITOR` registers the client in `state.monitors`. After each command executes, the server relays a formatted log line to other monitor clients.

### INFO

`INFO` returns grouped server information from [info.go](./info.go):

- server details
- client count
- memory usage and peak memory
- persistence state
- general counters such as total commands, evictions, and expirations

## Supported Commands

### Core key-value commands

- `SET key value`
  - stores or overwrites a string value
  - updates memory accounting
  - appends to AOF when enabled
  - contributes to RDB change counters

- `GET key`
  - returns the stored string value
  - returns null if the key does not exist
  - lazily expires the key if its TTL has passed
  - updates access count and last access time

- `DEL key [key ...]`
  - deletes one or more keys
  - returns the number of keys that existed

- `EXISTS key [key ...]`
  - counts how many of the provided keys currently exist

- `KEYS pattern`
  - matches keys using Go's `filepath.Match`
  - useful for glob-style inspection such as `*`, `user:*`, or `cache_*`

- `DBSIZE`
  - returns the number of keys currently in the map

- `FLUSHDB`
  - clears the entire in-memory map

### Expiration commands

- `EXPIRE key seconds`
  - assigns a TTL in seconds
  - returns `1` if the key exists, `0` otherwise

- `TTL key`
  - returns remaining seconds if the key has an expiry
  - returns `-1` if the key exists without an expiry
  - returns `-2` if the key does not exist or has already expired

### Persistence commands

- `SAVE`
  - writes an RDB snapshot synchronously

- `BGSAVE`
  - copies the current map and writes the snapshot in a goroutine
  - rejects the call if a background save is already running

- `BGREWRITEAOF`
  - rebuilds the AOF file from the current dataset in the background
  - increments rewrite stats when finished

### Security command

- `AUTH password`
  - required when `requirepass` is enabled in config
  - only `AUTH` and `COMMAND` are allowed before authentication

### Transaction commands

- `MULTI`
  - starts command queuing

- `EXEC`
  - executes all queued commands and returns their replies as an array

- `DISCARD`
  - clears the queued transaction without executing it

### Introspection commands

- `MONITOR`
  - subscribes the client to server command logs

- `INFO`
  - returns server, memory, persistence, and stats information

- `COMMAND`
  - currently returns `OK`
  - acts more like a compatibility placeholder than a full Redis `COMMAND` implementation

## Configuration

Configuration is read from [redis.conf](./redis.conf) by [conf.go](./conf.go).

Supported directives in this project:

- `dir`
- `appendonly`
- `appendfilename`
- `appendfsync`
- `save`
- `dbfilename`
- `requirepass`
- `maxmemory`
- `maxmemory-policy`
- `maxmemory-samples`

Example:

```conf
dir ./data
appendonly yes
appendfilename backup.aof
appendfsync everysec
save 5 3
dbfilename backup.rdb
maxmemory 256
maxmemory-policy allkeys-lfu
maxmemory-samples 50
```

## Design Notes

A few implementation choices are worth calling out:

- persistence is intentionally simple and uses Go `gob` for snapshots instead of the real Redis RDB binary format
- AOF replay reuses the `SET` handler instead of a separate recovery path
- the project uses a single global database instance instead of Redis-style multiple logical DBs
- expiration is lazy rather than actively scanned
- command handling is straightforward and easy to extend through the `Handlers` map

## How To Extend It

If you want to add new commands, the main places to change are:

1. add a handler to the `Handlers` map in [handlers.go](./handlers.go)
2. implement the command logic in that file or a dedicated file
3. parse any extra arguments from `Value.array`
4. return a proper RESP reply using the `Value` types
5. update persistence behavior if the command mutates state
6. consider how it should behave with auth, transactions, `MONITOR`, and `INFO`

## Summary

This project is a focused Redis-style clone that demonstrates:

- TCP server design in Go
- RESP parsing and serialization
- concurrent request handling
- in-memory key-value storage
- lazy TTL expiration
- configurable eviction
- AOF and snapshot persistence
- simple Redis-like transactions and monitoring

It is a strong learning project because the main Redis building blocks are all visible in a small codebase and each subsystem is implemented directly in plain Go.
