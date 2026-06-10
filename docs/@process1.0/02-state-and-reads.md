# State And Reads

HyperBEAM changes the default AO read pattern. Instead of sending dry-run messages just to inspect state, expose read keys through the `patch@1.0` device and fetch them over HTTP.

## Patch Device

The [`patch@1.0` device]() pushes selected process data to HyperBEAM endpoints so it can be discovered and fetched by clients. Each key sent to the patch device becomes readable through the process endpoint.

Patch selected state:

```lua
Send({
  device = "patch@1.0",
  counter = "42",
  status = "active"
})
```

Read selected state:

```text
GET /<process-id>~process@1.0/compute/counter
GET /<process-id>~process@1.0/compute/status
```

With a node:

```sh
curl https://<hyperbeam-node>/<process-id>~process@1.0/compute/counter
```

## Initial Sync

Use an initial sync block so clients can read useful state immediately:

```lua
Balances = Balances or {}
TotalSupply = TotalSupply or "0"
InitialSync = InitialSync or "INCOMPLETE"

if InitialSync == "INCOMPLETE" then
  Send({
    device = "patch@1.0",
    balances = Balances,
    totalsupply = TotalSupply
  })

  InitialSync = "COMPLETE"
end
```

## Patch After Mutations

Patch state in every handler that changes read-facing data:

```lua
Handlers.add(
  "Transfer",
  Handlers.utils.hasMatchingTag("Action", "Transfer"),
  function(msg)
    local amount = tonumber(msg.Tags.Amount)
    local recipient = msg.Tags.Recipient

    if not amount or amount <= 0 or not recipient then
      return msg.reply({ Error = "Invalid transfer" })
    end

    Balances[msg.From] = tostring((tonumber(Balances[msg.From]) or 0) - amount)
    Balances[recipient] = tostring((tonumber(Balances[recipient]) or 0) + amount)

    Send({
      device = "patch@1.0",
      balances = Balances
    })

    return msg.reply({ Status = "OK" })
  end
)
```

## `compute` vs `now`

- `compute/<key>` reads an exposed value quickly.
- `now/<key>` can ask HyperBEAM to evaluate the freshest process state path.

Use `compute` for normal app reads. Use `now` when a workflow explicitly requires the most current state or when building dynamic read pipelines.

## Dynamic Reads

Dynamic reads run a Lua function over process state without changing the process.

Create a Lua module:

```lua
function sum(base, req)
  local total = 0
  local holders = 0

  if base.balances then
    for address, balance in pairs(base.balances) do
      total = total + (tonumber(balance) or 0)
      holders = holders + 1
    end
  end

  return {
    totalsupply = tostring(math.floor(total)),
    holders = tostring(holders)
  }
end
```

Upload it to Arweave:

```sh
npm i -g @permaweb/arx
arx upload sum.lua -w wallet.json -t arweave --content-type application/lua --tags Data-Protocol ao
```

Call it through the process:

```text
GET /<process-id>~process@1.0/now/~lua@5.3a&module=<lua-script-tx-id>/sum/serialize~json@1.0
```

JavaScript read:

```js
const node = "https://<hyperbeam-node>";
const url = `${node}/${processId}~process@1.0/now/~lua@5.3a&module=${moduleId}/sum/serialize~json@1.0`;
const data = await fetch(url).then((res) => res.json());
```

## User-Owned Processes

If users own their own processes, they may need to opt into patching. Add an enable handler:

```lua
Handlers.add(
  "EnableStatePatch",
  Handlers.utils.hasMatchingTag("Action", "EnableStatePatch"),
  function(msg)
    if msg.From ~= ao.id then
      return msg.reply({ Error = "Only owner process can enable patching" })
    end

    StatePatchEnabled = true

    Send({
      device = "patch@1.0",
      balances = Balances or {}
    })

    return msg.reply({ Status = "Patch enabled" })
  end
)
```

Then guard automatic patching:

```lua
if StatePatchEnabled then
  Send({ device = "patch@1.0", balances = Balances })
end
```

## Read Hygiene

- Patch only public read data.
- Never expose secrets or private wallet material.
- Prefer strings for numeric values when interoperating with clients.
- Keep patched payloads small.
- Use dynamic reads for aggregate views.
- Avoid keys that collide with device paths: `now`, `compute`, `state`, `info`, `test`.
