# HyperBEAM Quickstart

Use this page to create a process, send it a message, and read exposed state through the `process@1.0` device.

## Prerequisites

- Node.js 20 or newer.
- A terminal.
- An Arweave wallet keyfile, or let AOS create one.
- A HyperBEAM node URL selected from `https://lunar.arweave.net` or another current marketplace/list.

## Install AOS

```sh
npm i -g https://get_ao.arweave.net
```

## Create a Process

```sh
aos process_name --url https://<hyperbeam-node>
```

Example:

```sh
aos counter --url https://push.forward.computer
```

Replace `process_name` with a friendly local name for the process, such as `counter`. Reuse the same `aos process_name --url https://node.url` command to re-enter the process console after exiting. If you omit `--url https://node.url`, the current AOS mainnet release defaults to a HyperBEAM node like `push.forward.computer`. Passing `--url https://node.url` keeps the selected node explicit as the executor of the process.

To use a specific wallet:

```sh
aos process_name --wallet ./wallet.json --url https://<hyperbeam-node>
```

## Create A Counter

To paste multiline Lua into the AOS prompt, open the inline editor first:

```lua
.editor
```

Paste the counter code below into the editor, then type this on its own line to exit the editor and execute the code:

```lua
.done
```

Alternatively, save the code below to an `example.lua` file and load it with:

```lua
.load path/to/example.lua
```

```lua
Counter = Counter or 0

Handlers.add(
  "Increment",
  Handlers.utils.hasMatchingTag("Action", "Increment"),
  function(msg)
    Counter = Counter + 1

    Send({
      device = "patch@1.0",
      counter = tostring(Counter),
      lastupdate = tostring(os.time()),
      updatedby = msg.From
    })

    return msg.reply({
      Status = "OK",
      Counter = tostring(Counter)
    })
  end
)
```

Send a message to your own process:

```lua
Send({ Target = ao.id, Tags = { Action = "Increment" } })
```

## Read State Through `process@1.0`

Use your selected node and process ID:

```sh
curl https://<hyperbeam-node>/<process-id>~process@1.0/compute/counter
```

Example:

```sh
curl https://push.forward.computer/<process-id>~process@1.0/compute/counter
```

The response is the patched `counter` value. This is the core HyperBEAM read pattern: mutate process state through messages, then expose selected read state over HTTP.

## Initial State Sync

Patch important read keys once when the process starts:

```lua
Counter = Counter or 0
InitialSync = InitialSync or "INCOMPLETE"

if InitialSync == "INCOMPLETE" then
  Send({
    device = "patch@1.0",
    counter = tostring(Counter)
  })

  InitialSync = "COMPLETE"
end
```

## Naming Rules

- Prefer lowercase patch keys: `counter`, `balances`, `totalsupply`.
- Avoid reserved path words as keys: `now`, `compute`, `state`, `info`, `test`.
- Prefer dash-separated tag names: `Example-Tag`, `Transfer-Id`.
- Do not rely on mixed-case tag normalization. HyperBEAM can normalize tag casing in ways that break older handlers.

## Local HyperBEAM Node

For local development, start HyperBEAM with the profile your environment requires, then point AOS at it:

```sh
aos --url http://localhost:8734 myLocalProcess
```

The upstream cookbook commonly references a `genesis_wasm` profile for local AOS execution. Treat the exact local boot command as environment-specific and keep your public docs focused on the selected node URL.
