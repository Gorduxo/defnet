# DefNet

[![Tests](https://github.com/ernisto/defnet/actions/workflows/check.yml/badge.svg)](https://github.com/ernisto/defnet/actions/workflows/check.yml)

DefNet is a small Roblox networking library for declaring remotes once in a shared module. The server creates the
`RemoteEvent`, `RemoteFunction`, and state folders; the client waits for the same tree and receives typed wrappers.

Current features:

- Shared remote declarations with deterministic creation order.
- Typed `RemoteEvent` and `RemoteFunction` wrappers.
- Optional schema validation for incoming payloads.
- Optional per-player rate limits for server handlers.
- Safe callback execution with warning logs instead of broken remote callbacks.
- Optional binary codec for compact payloads and replicated state diffs.
- Server-authoritative state replication with per-player targets.
- Single multiplexed state remote, using numeric remote ids internally.

## Installation

### Wally

Add DefNet under `[dependencies]`:

```toml
defnet = "ernisto/defnet@0.1.0"
```

### Pesde

```sh
pesde add wally#ernisto/defnet
```

### Release Model

You can also download `defnet.rbxm` from the latest GitHub release.

## Quick Start

Create one shared module that both server and client require:

```lua
-- ReplicatedStorage/Shared/net.luau
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local defnet = require(ReplicatedStorage.Packages.defnet)
local types = require(ReplicatedStorage.Packages.defnet.utils.types)

return defnet.remotes({
	chat = {
		send = defnet.event({
			schema = { types.string },
			rate_limit = 8,
			rate_window = 1,
		}),

		history = defnet.func({
			schema = {},
			timeout = 5,
		}) :: defnet.func_type<(), { string }>,
	},

	world = {
		players = defnet.state({} :: { [string]: Vector3 }),
	},
})
```

Server:

```lua
local net = require(ReplicatedStorage.Shared.net)

net.chat.send.on_server(function(player, message)
	print(player.Name, message)
	net.chat.send.fire_all(`{player.Name}: {message}`)
end)

net.chat.history.on_server_invoke = function(player)
	return { "welcome" }
end

net.world.players.server[player] = {}
net.world.players.server[player][player.Name] = player.Character:GetPivot().Position
```

Client:

```lua
local net = require(ReplicatedStorage.Shared.net)

net.chat.send.on_client(function(message)
	print(message)
end)

net.chat.send.fire_server("hello")

local history = net.chat.history.invoke_server()
print(history)

net.world.players.client.changed(function(patch, current)
	print("state changed", patch, current)
end)
```

## Defining Remotes

```lua
local remotes = defnet.remotes(definition, root?)
```

`definition` is a nested table. Each leaf must be one of:

- `defnet.event(options?)`
- `defnet.func(options?)`
- `defnet.state(default?)`

On the server, DefNet creates the Roblox instances. On the client, DefNet calls `WaitForChild` for those instances.
The definition table is replaced in-place with wrapper objects and returned frozen.

If `root` is omitted, DefNet creates or waits for a `rbx` folder under the package module script. Passing `root` lets
you place remotes somewhere else.

## Event API

```lua
local event = defnet.event(options?)
```

Options:

```lua
type event_options = {
	codec: codec_impl?,
	schema: { checker }?,
	rate_limit: number?,
	rate_window: number?,
}
```

Wrapper methods:

| Method | Side | Description |
| --- | --- | --- |
| `fire_server(...)` | Client | Sends payload to the server. |
| `fire_client(player, ...)` | Server | Sends payload to one player. |
| `fire_all(...)` | Server | Sends payload to every player. |
| `fire_list(players, ...)` | Server | Sends payload to a player list. |
| `fire_except(player, ...)` | Server | Sends payload to everyone except one player. |
| `on_client(callback)` | Client | Connects to `OnClientEvent`. |
| `on_server(callback)` | Server | Connects to `OnServerEvent`. |

Server handlers are protected by:

- Rate limiting when `rate_limit` is set.
- Tuple schema validation when `schema` is set.
- NaN validation through `nan_scanner`.
- `pcall` callback isolation through `guard.safe_call`.

With a codec, DefNet wraps encode/decode with the same schema, so encoded and decoded values are checked too.

## Function API

```lua
local remote_function = defnet.func(options?)
```

Options:

```lua
type func_options = {
	schema: { checker }?,
	rate_limit: number?,
	rate_window: number?,
	timeout: number?,
}
```

Wrapper methods and callbacks:

| Member | Side | Description |
| --- | --- | --- |
| `invoke_server(...)` | Client | Calls server and returns server result. |
| `invoke_client(player, ...)` | Server | Calls client and returns client result. |
| `on_server_invoke = function(player, ...)` | Server | Handles client invokes. |
| `on_client_invoke = function(...)` | Client | Handles server invokes. |

`invoke_client` runs through a timeout loop. The default timeout is `8` seconds; set `timeout` to override it.
Invoke handlers also use rate limits, schema validation, NaN validation, and safe callback execution.

## State API

```lua
local state = defnet.state(default?)
```

State is server-authoritative. Each player can receive a different target table.

```lua
net.inventory.server[player] = {
	coins = 100,
	items = {},
}

net.inventory.server[player].coins += 25
```

Client state:

```lua
local snapshot = net.inventory.client.get()

net.inventory.client.changed(function(patch, current)
	print(patch, current)
end)
```

State wrapper:

| Member | Side | Description |
| --- | --- | --- |
| `client.value` | Client | Current replicated snapshot table. |
| `client.get()` | Client | Returns the current snapshot. |
| `client.changed(callback)` | Client | Fires when a patch is applied. Callback receives `patch, current`. |
| `changed(callback)` | Client/Server | Connects to the internal changed event. |
| `step(player?)` | Server | Flushes one state syncer, optionally for one player. |
| `server[player]` | Server | Target state table for a player. |
| `codec` | Internal | Diff codec used by state sync. |

DefNet flushes state on `RunService.Heartbeat`. You can flush manually:

```lua
defnet.step_sync()
defnet.step_sync(player)
```

Internally, state replication uses one shared remote and `remote_index` ids. Each player has a codec pool so repeated
values can be encoded more compactly across updates.

## Schema Helpers

The `utils/types` module provides checker functions for schemas:

```lua
local types = require(ReplicatedStorage.Packages.defnet.utils.types)

local chat_schema = {
	types.string,
	types.optional(types.number),
}
```

Available checkers:

| Checker | Passes when |
| --- | --- |
| `types.any` | Always true. |
| `types.string` | `typeof(value) == "string"`. |
| `types.number` | Number and not NaN. |
| `types.boolean` | Boolean. |
| `types.table` | Table. |
| `types.vector3` | `Vector3`. |
| `types.instance` | Any Roblox `Instance`. |
| `types.player` | A `Player` instance. |
| `types.optional(check)` | `nil` or `check(value)`. |
| `types.array(check)` | Table where every item passes `check`. |
| `types.shape(schema)` | Table where each schema key passes its checker. |

Schemas are tuples. `schema = { types.string, types.number }` checks argument 1 as string and argument 2 as number.

## Codec

The built-in codec is available at:

```lua
local codec = defnet.codec
```

Public codec API:

| Member | Description |
| --- | --- |
| `codec.impl.encode(value, pool?)` | Encodes a value into a `buffer`, returning `buffer, unknowns?`. |
| `codec.impl.decode(buffer, pool?, unknowns?)` | Decodes a value from a buffer. |
| `codec.create_pool()` | Creates an identity pool used to compact repeated values. |

Supported inline types:

- `nil`
- `boolean`
- `number`
- ASCII `string`
- `Vector3`
- `table`
- `buffer`

Values that cannot be serialized inline use an `unknowns` side channel. Pass the second return from `encode` into
`decode` when unknown values are present.

## Utility Modules

These modules are used internally and can be required directly when needed:

| Module | Purpose |
| --- | --- |
| `utils/guard` | Rate limits, tuple validation, and safe callback execution. |
| `utils/codec_guard` | Wraps codecs so schema validation runs around encode/decode. |
| `utils/nan_scanner` | Rejects NaN values in payloads and return tuples. |
| `utils/diff` | Creates and applies table diffs, including array shifts. |
| `utils/diff_codec` | Encodes and decodes diff patches. |
| `utils/sync` | Tracks per-player targets, snapshots, and sends patches. |
| `remote_index` | Maps state remote ids to callbacks and manages the shared state remote. |

## Notes

- Define the same remote tree on server and client by requiring the same shared module.
- Keep server handlers authoritative. Schema validation rejects bad shapes; it does not make client data trusted.
- `rate_limit` is per player and remote name. `rate_window` defaults to `1` second.
- Avoid NaN in networked data. DefNet warns or rejects NaN payloads because NaN breaks stable equality and diffs.
- `RemoteFunction:InvokeClient` can hang in Roblox; DefNet wraps client invokes with a timeout.

## Contributors

- [Gorduxo](https://github.com/Gorduxo)

## License

MIT
