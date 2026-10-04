---
name: testing
description: "Write, place and run tests for a server in this repo. Use when choosing between driving the server through an MCP client and calling its functions directly; when writing a protocol-level test for a TypeScript or Python server; when choosing the command that runs one suite or a coverage report; or when a test is skipped or passes against a stale build."
disable-model-invocation: false
---

# Testing

The **rules** are in [`AGENTS.md`](../../../AGENTS.md) under **Always test new
or modified code**: new behavior comes with tests, test at the protocol level
where you can, and keep expected error output off the console. This skill
explains which harness each server has **today**, how to write a test with it,
and which command runs what.

Driving a running server by hand, with the Inspector or an LLM client, is
`/client-smoke`. It is evidence for a PR, not a substitute for a test.

⚠️ **The harnesses are uneven, and this page describes them as they are.** The
three Python servers are driven in-process through a `ClientSession` and held
to a per-file coverage gate (#4855). Of the TypeScript servers only
`everything` can be driven in-process today, and none has a coverage
threshold yet; both are the work of #4854, and the TypeScript half of this
skill changes when it lands. Do not write a TypeScript test that assumes them.

## Where a test goes

| Server | Directory | File name | Runner |
| --- | --- | --- | --- |
| `everything`, `filesystem`, `memory`, `sequentialthinking` | `src/<server>/__tests__/` | `<subject>.test.ts` | vitest, `globals: true` |
| `fetch`, `git`, `time` | `src/<server>/tests/` | `test_<subject>.py` | pytest, with `pytest-asyncio` (`asyncio_mode = "auto"`) |
| Root tooling | `scripts/`, beside the script | `<name>.test.mjs` | `node --test` |

Each vitest config includes only `**/__tests__/**/*.test.ts`, so a test file
anywhere else in a workspace is never run and reports nothing. Shared test
helpers go in `__tests__/` without the `.test` suffix, as
`src/everything/__tests__/helpers.ts` does.

## What each server's tests do today

| Server | How its tests reach the code |
| --- | --- |
| `everything` | Call a `register…` function with a mocked `McpServer`, capture the handler it registered, and invoke the handler directly. `__tests__/helpers.ts` types the captured handlers |
| `filesystem` | Unit tests of `lib.ts` and the path helpers, plus an SDK `Client` over `StdioClientTransport` that spawns the built `dist/index.js` |
| `sequentialthinking` | Unit tests of `lib.ts`, plus an SDK `Client` over stdio against `dist/index.js` |
| `memory` | Direct calls on the exported `KnowledgeGraphManager` and `register…` functions, with a mocked server |
| `fetch`, `git`, `time` | A `ClientSession` linked in-process to the real `serve()` over in-memory streams (below), plus direct calls on the pure functions in `server.py` |

No TypeScript test uses an in-memory transport yet.

## Choosing a harness for a new test

Ask what the test must prove.

- **Logic** (a path check, a graph operation, a timezone conversion): call the
  function directly. That is what `lib.ts` and the functions in `server.py` are
  for.
- **What a client sees** (a tool's result shape, its advertised schema, an
  error result, a capability, a notification): drive the server through a
  client. A handler called directly skips the SDK's input validation, output
  schema validation and error mapping, which is where the wire-level bugs in
  this repo have been. The regression tests in `filesystem` and
  `sequentialthinking` that go through a `Client` exist for exactly that
  reason.

Which client harness is available depends on the server.

### `everything`: in-process, over an in-memory transport

`src/everything/server/index.ts` exports `createServer()`, so a test can link a
real `Client` to a real server with no process and no build:

```ts
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { InMemoryTransport } from "@modelcontextprotocol/sdk/inMemory.js";
import { createServer } from "../server/index.js";

describe("echo, over the protocol", () => {
  let client: Client;
  let cleanup: () => void;

  beforeEach(async () => {
    const created = createServer();
    cleanup = created.cleanup;
    const [clientTransport, serverTransport] =
      InMemoryTransport.createLinkedPair();
    client = new Client({ name: "test-client", version: "0.0.0" });
    await Promise.all([
      created.server.connect(serverTransport),
      client.connect(clientTransport),
    ]);
  });

  afterEach(async () => {
    await client.close();
    cleanup();
  });

  it("returns the message", async () => {
    const result = await client.callTool({
      name: "echo",
      arguments: { message: "hi" },
    });
    expect(result.content).toEqual([{ type: "text", text: "Echo: hi" }]);
  });

  it("reports bad input as a tool error", async () => {
    const result = await client.callTool({ name: "echo", arguments: {} });
    expect(result.isError).toBe(true);
  });
});
```

Call `cleanup()` in `afterEach`: `createServer()` starts timers (the roots sync
after `initialize`, the task store) that otherwise outlive the test. A tool
registered by `registerConditionalTools` appears only when the `Client`
declares the matching capability (`roots`, `sampling`, `elicitation`) in its
constructor options.

### `filesystem`, `memory`, `sequentialthinking`: a client over stdio, against the build

These three build their server at module scope and connect stdio when
`index.ts` is imported, so there is no factory to call and the in-memory
recipe above does not apply to them. A protocol-level test spawns the built
server instead:

```ts
const distIndexPath = path.join(packageRoot, "dist", "index.js");
const transport = new StdioClientTransport({
  command: process.execPath,
  args: [distIndexPath],
  stderr: "pipe",
});
const client = new Client({ name: "test-client", version: "0.0.0" });
try {
  await client.connect(transport);
  // … assert …
} finally {
  await client.close(); // this is what stops the child process
}
```

Closing in `finally` (or in `afterEach`, as the `filesystem` tests do) matters:
a failed assertion that skips `close()` leaves the spawned server running.

`src/sequentialthinking/__tests__/input-schema.test.ts` is the reference.
`stderr: "pipe"` keeps the server's startup banner off the console.

Give the server what it needs to start. `filesystem` takes its allowed
directories as arguments (`args: [distIndexPath, testDir]`, with `testDir` a
`realpath`-resolved temporary directory, as
`src/filesystem/__tests__/structured-content.test.ts` does); with none, and a
client that offers no Roots, every file operation is refused. Environment
variables such as `MEMORY_FILE_PATH` go in the transport's `env` option: the
child inherits only the SDK's default safelist (`PATH`, `HOME`, `USER` and the
like), not the other variables of the test process.

⚠️ **These tests run the last build, not your edit.** `npm test` does not build.
A test that spawns `dist/index.js` passes or fails on whatever `tsc` last
wrote, so after changing a server's source, build before trusting the result,
or run `npm run validate -w src/<server>`, which builds before it tests.

⚠️ **A missing `dist/` is handled two ways.** The `sequentialthinking` tests
wrap themselves in `skipIf(!existsSync(distIndexPath))`, so with no build they
are reported as **skipped**, and the run stays green having tested nothing over
the wire. The `filesystem` stdio tests and `memory`'s dist-layout test have no
such guard and **fail**. A skipped test in this repo usually means "not
built".

### Python: `ClientSession` over in-memory streams, in-process

Each Python server builds its `Server` inside `serve()` and serves it over the
`stdio_server()` it imports into `server.py`. A test replaces that one name
with a context manager that yields the server side of a pair of in-memory
streams, runs the real `serve()` in a task group, and opens a `ClientSession`
on the client side. No process, no build, and the SDK's input validation and
error mapping run exactly as they do for a real client:

```python
from contextlib import asynccontextmanager
from typing import Any, AsyncIterator

import anyio
import pytest
from mcp import ClientSession
from mcp.shared.memory import create_client_server_memory_streams

import mcp_server_time.server as server_module


@asynccontextmanager
async def connect(**serve_kwargs: Any) -> AsyncIterator[ClientSession]:
    async with create_client_server_memory_streams() as (client, server):

        @asynccontextmanager
        async def fake_stdio_server() -> AsyncIterator[Any]:
            yield server

        with pytest.MonkeyPatch.context() as mp:
            mp.setattr(server_module, "stdio_server", fake_stdio_server)
            async with anyio.create_task_group() as tg:
                tg.start_soon(lambda: server_module.serve(**serve_kwargs))
                async with ClientSession(*client) as session:
                    await session.initialize()
                    yield session
                tg.cancel_scope.cancel()


async def test_get_current_time() -> None:
    async with connect() as session:
        result = await session.call_tool("get_current_time", {"timezone": "UTC"})
        assert result.isError is False
```

- **Patch the name `server.py` looks up** (`mcp_server_<name>.server.stdio_server`),
  not `mcp.server.stdio.stdio_server`: `server.py` imported the function, so
  the module attribute is what `serve()` calls.
- **Cancel the task group after the session closes.** `serve()` runs until its
  streams end, so without the cancel the test never returns.
- **Assert on the wire shape where you can**, for example
  `result.model_dump(by_alias=True, mode="json")`, rather than on Python
  attribute names: the SDK v2 port (#4851) renames them, and a wire-level
  assertion survives it.
- **No network in a test.** `fetch` replaces `httpx.AsyncClient` with one on an
  `httpx.MockTransport`; `git` works against a temporary repository its
  fixture creates and closes before removing.

`pytest-asyncio` with `asyncio_mode = "auto"` is a dev dependency of all three,
so an `async def test_…` runs as it stands.

A subprocess test over stdio (`StdioServerParameters(command=sys.executable,
args=["-m", "mcp_server_<name>"])` with `mcp.client.stdio.stdio_client`) is
kept to a thin smoke of the entry point. coverage.py does not measure a child
process, so code reached only that way reads as uncovered.

## Running them

| Scope | Command |
| --- | --- |
| One TypeScript server | `npm test -w src/<server>` |
| One file, or one test | `npm test -w src/<server> -- __tests__/<file>.test.ts -t "<name>"` |
| Every TypeScript server | `npm test` at the root |
| One TypeScript server's whole chain | `npm run validate -w src/<server>` (format check, lint, typecheck, build, test) |
| One Python server | `uv run pytest` in `src/<server>` |
| One Python test | `uv run pytest <path to the test file>::<name>` in `src/<server>` (for example `tests/test_server.py::test_git_checkout_existing_branch`) |
| One Python server's whole chain | `npm run validate:py -- <server>` |
| One Python server's coverage gate | `npm run coverage:py -- <server>` at the root |
| Every Python server's coverage gate | `npm run coverage:py` at the root |
| Root tooling | `npm run test:scripts` |

`npm run typecheck -w src/<server>` is the only step that typechecks the test
files in `filesystem`, `memory` and `sequentialthinking`: their build
`tsconfig.json` excludes tests, and vitest does not typecheck.

## `test` versus `coverage`

`test` is `vitest run`. `coverage` is `vitest run --coverage`: the same suite,
plus a v8 coverage report over the workspace's `**/*.ts` (tests and `dist/`
excluded), written to the terminal and to the ignored `coverage/` directory.

```sh
npm run coverage -w src/<server>    # one server
npm run coverage                    # all four
```

For the TypeScript servers the report is **informational** today. No vitest
config sets a threshold, so `coverage` fails only when a test fails, and
neither `validate` nor CI runs it. Read the report to find what a change left
untested; do not describe it as a gate.

**The Python servers' coverage is a gate.** `npm run coverage:py -- <server>`
runs, in `src/<server>`:

```sh
uv run --frozen pytest --cov --cov-report=term-missing --cov-report=json
```

which prints coverage.py's report (the missing lines and partial branches)
and writes `coverage.json` (ignored by git). `scripts/lib/py-coverage.mjs` then
checks **every file** in it: at least 90% of its lines and 90% of its branches,
each file on its own. What is measured is the server's `[tool.coverage.run]`
(`branch = true`, `source` set to its package). `local:gate` and CI's
**Coverage \<server\>** job both run it. It is kept out of the plain
`uv run pytest` loop, which stays fast and uninstrumented.

A line or branch that genuinely cannot run is marked
`# pragma: no cover  # <reason>`, with the reason on the same line; never
lower the gate. Diagnosing a red run is `/pre-push-gate`.

⚠️ **A spawned server is invisible to the report.** v8 coverage measures the
vitest process, not a child it starts, so code reached only through a stdio
test reads as uncovered: `sequentialthinking`'s `index.ts` reports 0% while its
stdio tests exercise it. A low number on an entry file is not, by itself,
missing tests.

## Error output

A test that expects an error keeps it off the console. Spy on the logger for
the duration of the test (`vi.spyOn(console, "error").mockImplementation(() =>
{})`, restored afterwards), and pipe a spawned server's stderr rather than
inheriting it.
