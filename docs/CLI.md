# `tanvrit` — the command-line interface

The `tanvrit` binary is the third first-party consumer of this repo's local
intelligence, after the desktop app and the VS Code extension. It exists to give
scripts, CI jobs and compute nodes the same command set a human gets, over a
contract a program can actually read.

## Status (2026-09-10, branch `main`)

Per CONSTITUTION §7.8 — shipped means installable today, "coming" is labeled —
this section is the honest state, and it is deliberately more prominent than the
reference below it.

| Piece | State |
|---|---|
| The command tree (`cli/CommandSpec.kt`) | **written** — 41 commands, and it is what the rest of this document describes |
| The `--json` envelope + exit codes (`cli/OutputEnvelope.kt`) | **landed** |
| The backend seam (`cli/CliBackend.kt`) | **landed** |
| `DesktopCliContext` (stderr + confirmation) | **landed** |
| `EmbeddedBackend` | **landed** — serves **37 of 41**; the other four are named stubs below |
| `RemoteBackend` | **landed** — serves **29 of 41**; the other twelve are refused with a written reason |
| `TanvritCliMain.kt` (argv parser + stdout renderer) | **landed** |
| `:composeApp:cliRuntimeImage` / `:composeApp:cliDistTar` / `:composeApp:runCli` | **landed** |
| A published `tanvrit` download | **shipped in v1.5.31** — as a standalone tarball, *not* inside the installers. See [Where the binary comes from](#where-the-binary-comes-from) |

### Coverage, both backends (verified 2026-09-10 by running the binary)

`COMMAND_TREE` holds **41** commands. Each backend partitions that tree exactly,
in code, and both partitions are written out rather than derived as "everything
not excused" — so a command added tomorrow lands in neither half and the parity
tests fail, which is the point:

| Backend | Serves | Refuses | The two sets that decide it |
|---|---|---|---|
| `EmbeddedBackend` | **37** | **4**, as `not_implemented` | `IMPLEMENTED` / `UNIMPLEMENTED_WITH_REASON` |
| `RemoteBackend` | **29** | **12**, as `remote_unsupported` | `SUPPORTED` / `UNSUPPORTED_WITH_REASON` |

Overlay the two and **39 of the 41 are reachable on at least one backend**: ten
are embedded-only, two (`agent stream`, `agent stop`) are remote-only, and
**two — `provider set` and `policy set` — are served by neither.** For those two
`--local` is *not* a remedy, and **both paths now say so.** With `--local`,
`EmbeddedBackend.notImplemented()` reads `RemoteBackend.SUPPORTED` and answers
"No backend implements it: …" followed by remote's own reason. With `--remote`,
`unsupportedError()` delegates to `otherBackendRemedy()`, which names `--local`
**only** when the id is in `EmbeddedBackend.IMPLEMENTED` and otherwise returns
the mirror-image "No backend implements it: …" carrying *embedded's* reason.
Each side computes its remedy from the other side's published set; neither has a
third hand-maintained list of "commands nobody serves" to go stale.

**With one gap, and it is the common case.** `otherBackendRemedy` is only
reached once a backend has been *selected*. `--remote` with no app listening
never gets that far: the reachability check in `dispatch` answers first, and its
remedy is unconditional. Measured 2026-09-11 on 1.5.32:

```console
$ tanvrit provider set ollama --remote --endpoint http://127.0.0.1:19311 --json --yes
{"schema":1,"kind":"error","command":"provider.set","backend":"remote","error":{"code":"app_unreachable","message":"--remote was requested but no Tanvrit AI app answered at http://127.0.0.1:19311.","remedy":"Start the desktop app, or drop --remote to run in-process."},"seq":0}
exit=5
```

"…or drop `--remote` to run in-process" is the closed road, offered for a
command the binary already knows nobody serves. So the accurate statement is:
*once a backend answers*, neither refusal points at the other one. The
`app_unreachable` path is not backend-aware and still does.

**And "exit 5 either way" needs `--yes` — on `--local` only.** The two
`supports()` implementations mean different things, which is why the same
invocation is refused at different points on the two paths:

| | `supports(id)` is | so a `neither` command without `--yes` answers |
|---|---|---|
| `EmbeddedBackend` | `COMMANDS_BY_ID.containsKey(id)` (`EmbeddedBackend.supports`) — **every id in the tree**, stubs included | `confirmation_required` / **64**. The stub's `not_implemented` is raised inside the handler, which runs *after* the gate. |
| `RemoteBackend` | `id in SUPPORTED` (`RemoteBackend.supports`) — the routed ids only | `remote_unsupported` / **5**, refused by `runOnBackend`'s capability check *before* the gate. |

Measured 2026-09-11: `tanvrit provider set ollama --local --json` → 64; the
same with a reachable `--remote` → 5. Pass `--yes` and both are 5. See
§"Known holes" below, items 2 and 3.

Both halves of that were wrong at different times on 2026-09-11, and the history
is worth keeping because each wrong version read as reassuring. This paragraph
first claimed the CLI "says so rather than sending the user down a road it
already knows is closed" while `unsupportedError` hard-coded "Re-run with
`--local`" for every remote refusal; it was then corrected to "**only one of the
two paths says so**", accurate for a few hours, until `otherBackendRemedy`
landed the same day and made the original promise true *by changing the code
rather than softening the sentence*. Check it with `tanvrit provider set <id>
--remote --json --yes`: the remedy must not name `--local`. What decides it is
`otherBackendRemedy` (`TanvritCliMain.kt`): it names `--local` if and only if
the id is in `EmbeddedBackend.IMPLEMENTED`, and otherwise prefixes "No backend
implements it:" to that command's own sentence in
`EmbeddedBackend.UNIMPLEMENTED_WITH_REASON`. `UnsupportedRemedyTest`
(`composeApp/src/desktopTest/kotlin/com/tanvrit/ai/cli/UnsupportedRemedyTest.kt`)
walks every command in `COMMAND_TREE` and asserts exactly that biconditional, so
the property is pinned by a test rather than by one capture. Rebuild before
reproducing it — a launcher built before `otherBackendRemedy` landed still
points every remote refusal at `--local`. See
[Not supported remotely](#not-supported-remotely--and-when---local-is-the-remedy).
The `Backend` column in the reference below carries this per command.

**Four commands answer `not_implemented`,** and each refuses for a reason
recorded in code rather than as a TODO —
`EmbeddedBackend.UNIMPLEMENTED_WITH_REASON` carries the sentence the error
envelope prints. **There is no fifth site.** This paragraph said there was one
until 2026-09-11: `mcp serve --stdio` refused the *option* with the same code,
so a grep for `not_implemented` found five places and four commands. `--stdio`
is served from later the same day (`EmbeddedMcpCommands.mcpServeStdio`), its
`not_implemented` branch is deleted, and the grep and the count now agree at
four — see the `mcp serve` row in the reference below.

The rows are the map's values **verbatim** (`EmbeddedBackend`'s `UNIMPLEMENTED` map, copied
2026-09-11). They were paraphrases until then — "No cross-process registry of
running agents" for "There is no cross-process registry of running agents to
stop", "the gateway" for "the AI gateway" — which a table introduced as *the
sentence the error envelope prints* may not be. The envelope prefixes each with
"`<command>` is not available on the embedded backend."

| Command | Why it refuses |
|---|---|
| `agent stream` | An embedded run is synchronous and process-local, so there is no live run in another process for this one to attach to. |
| `agent stop` | There is no cross-process registry of running agents to stop; an embedded run ends when its own invocation does. |
| `provider set` | The active provider is in-memory state seeded from registration order and persisted nowhere, so a CLI process would report a switch that dies with it. |
| `policy set` | Org policy is fetched from the AI gateway into a read-only flow with no local writer, and a governance rule the governed machine can rewrite is not a control. |

The first two ARE served by `RemoteBackend`, which has routes for them, so they
work against a running app; the last two are refused by *both* backends, for
different reasons each. A command being present in `COMMAND_TREE` means its
invocation, options and exit-code contract are fixed — not that either backend
does the work. `tanvrit <command> --json` remains the only authority.

`not_implemented` is *not* one of the seven codes pinned in `CliError.Companion`,
but it does map onto exit **5** alongside `remote_unsupported` and
`app_unreachable`, so a script that already branches on 5 for "this backend
cannot serve it" needs no change:

```console
$ tanvrit provider set ollama --local --yes
tanvrit: `provider set` is not available on the embedded backend. The active provider is in-memory state seeded from registration order and persisted nowhere, so a CLI process would report a switch that dies with it. (not_implemented)
  No backend implements it: Selecting the active provider steers every in-app conversation, not just this caller's; a CLI should not silently repoint the window the user is looking at.
exit=5
```

Note the remedy line: it is **computed, not guessed.** `notImplemented()` checks
`RemoteBackend.SUPPORTED` before writing it, so a command remote *can* serve is
told to try `--remote`, and one neither backend serves — this one — is told that
plainly and given remote's reason as well. An unconditional "try `--remote`"
here would be §7.8 in miniature.

## Where the binary comes from

**The CLI ships as its own artifact** — a standalone tarball published to the
GitHub Release beside the installers:

```
tanvrit-cli-1.5.32-Linux-arm64.tar.gz        # :composeApp:cliDistTar
```

**The OS token is capitalised.** `cliDistTar` builds the name from `cliHostOsTag`
(`composeApp/build.gradle.kts:1262-1269`; the name it assembles is at `:1285`),
which is `macOS` / `Windows` / `Linux` and never lowercase — the same spelling
the installers and the Download page use, for the reason
`scripts/check-download-names.sh` records: a second spelling of one platform is
a silent 404 for users rather than a red build. This line read
`tanvrit-cli-1.5.31-linux-arm64.tar.gz` until 2026-09-11, a filename
`cliDistTar` cannot produce; nothing scripts off this doc, so it cost a failed
copy-paste rather than a broken link. The version segment is `appVersionName`
(`composeApp/build.gradle.kts:155`), `1.5.32` today.

**Grep the symbol, not the line number.** Those two citations read
`:1187-1194` and `:133` until 2026-09-11, and both were wrong *by the time the
paragraph shipped*: the commit that wrote them added lines to the same
`build.gradle.kts`, so `:1187` landed inside the Windows `.bat` launcher
template and `:133` inside an unrelated comment. `build.gradle.kts:1282`'s own
"`appVersionName` (line ~106)" is stale from the same shift and is listed under
§"Known stale citations elsewhere" below, since it is not this file's to edit.
Line numbers here are a convenience for a reader who greps first.

All three installer workflows build the tarball in the SAME invocation as their
installer and fail the job if none is produced, so it cannot go missing
silently the way the sidecar did:

| workflow | invocation |
| --- | --- |
| `release-linux.yml` | `:composeApp:packageDeb :composeApp:cliDistTar` |
| `release-macos.yml` | `:composeApp:{notarize,package}Dmg :composeApp:cliDistTar` |
| `release-windows.yml` | `:composeApp:packageMsi :composeApp:cliDistTar` |

Each also asserts the built tarball carries that leg's `<Os>-<arch>` tag before
publishing it, and mirrors it to R2 — an artifact that reaches only the GitHub
Release 404s on `dl.tanvrit.com`, which is what every link here points at.
`scripts/check-cli-artifact-names.sh` pins those names, bidirectionally,
against `cliDistTar`'s own `archiveFileName` — all three workflows since
2026-09-11; before that it read `release-linux.yml` only, so a `macOS`→`macos`
respelling passed CI. It derives each leg's tag from that job's own
`case "$base" in *-<tag>.tar.gz)` assertion, so neutering the assertion fails
the gate rather than disabling it quietly.

The reverse direction has one exception, and it is the paragraph below: a
workflow may carry a `cli-tarball-not-yet-advertised:` marker listing names it
publishes that no page links yet. Those are reported as PENDING instead of
failing. The marker is itself checked — its list must equal what that workflow
publishes — and the forward direction still fires, so the first page edit that
adds a macOS or Windows row fails CI and names the marker line to delete.
"Do not add them to the download pages before a release has published them" is
therefore enforced, not just advised.

**v1.5.31 is the first release that contains a CLI at all, and it was Linux
only.** The macOS and Windows legs were added on 2026-09-11 and have never run:
until the next release actually cuts, `tanvrit-cli-<ver>-macOS-arm64.tar.gz`,
`-macOS-x64` and `-Windows-x64` are all 404 on `dl.tanvrit.com` (probed
2026-09-11). Do not add them to the download pages before a release has
published them.

**It is NOT inside the `.dmg` / `.msi` / `.deb`.** That is the second correction
this section has carried, and both are recorded rather than quietly overwritten,
because each was load-bearing and each was believed before it was checked.

**Correction 1.** This section originally said a second executable "cannot be put
inside the installers" because jpackage takes exactly one launcher and exposes no
`--add-launcher` passthrough through Compose's DSL. `installCliSidecar` was then
written on the theory that the premise was true but the conclusion was not: copy
the launchers into the jpackage app image *after* it is built and *before* it is
packaged, the same post-processing pattern `brandDmgIcon` already used.

**Correction 2 (2026-09-10).** The original conclusion was right after all, for a
reason nobody had checked: **`packageDeb` does not package the app image
`createDistributable` produces — it builds its own.** The two tasks' jpackage
argument files, written by the Compose plugin on every run, say so directly:

```console
$ grep -A1 -- --input composeApp/build/compose/tmp/createDistributable.args.txt
--input
".../composeApp/build/compose/tmp/createDistributable/libs"

$ grep -A1 -- --input composeApp/build/compose/tmp/packageDeb.args.txt
--input
".../composeApp/build/compose/tmp/packageDeb/libs"
```

Neither invocation passes `--app-image`. `createDistributable` writes its image
to `build/compose/binaries/main/app`, which is exactly what `installCliSidecar`
patches — and `packageDeb` never reads it. No amount of correct wiring in that
task can put `tanvrit` into an installer.

The published artifact agrees, and it was checked by unpacking rather than by
reading the build:

```console
$ dpkg-deb -x TanvritAI-1.5.30-Linux-arm64.deb .   # from dl.tanvrit.com
$ ls opt/tanvrit-ai/bin/
Tanvrit AI                    # that is the whole directory
```

v1.5.30 *also* had two independent wiring gaps inside `installCliSidecar`, both
silent and both since fixed: it did not depend on `cliRuntimeImage`, so
`build/cli/bin` was empty; and it used `mustRunAfter("createDistributable")`,
which only *orders* a task already in the graph and does not pull one in, so
there was no app image to copy into either. Both logged `Skipped.` at lifecycle
level on an otherwise green build. Both are now `dependsOn` and an empty CLI
directory throws instead of skipping. Those were real defects and fixing them was
right — but fixing them does not make the sidecar reach a `.deb`, which is why
the tarball, not the sidecar, is the delivery mechanism.

`installCliSidecar` still runs and still patches the app image, so a
`createDistributable` tree used directly does carry `tanvrit` and
`tanvrit-ai-agent`. Treat that as convenient for a local app-image run, **not**
as how the CLI is delivered to users.

So the binary is built:

```bash
# Build a self-contained runtime image containing the tanvrit launcher.
# Produces composeApp/build/cli/bin/{tanvrit,tanvrit-ai-agent}.
./gradlew :composeApp:cliRuntimeImage

# Pack that image into the artifact a release publishes.
# Produces composeApp/build/distributions/tanvrit-cli-<ver>-<Os>-<arch>.tar.gz,
# e.g. tanvrit-cli-1.5.32-Linux-arm64.tar.gz — the OS token is capitalised.
./gradlew :composeApp:cliDistTar

# Run it straight from the build, without producing an image.
./gradlew :composeApp:runCli -Pargs="status --json"
```

`-Pargs` is tokenized by Ant's `Commandline` parser, which understands single and
double quotes but **not** escaped quotes. An argument containing both quote
characters cannot be passed through it at all — use `--prompt-file` for prompts
with hostile quoting. This is the same trap `scripts/tanvrit-ai-agent.sh`
documents and re-quotes around; `runCli` inherits it because it is the same
Gradle mechanism, not because anyone chose it.

## The two backends

`CliBackend` (`cli/CliBackend.kt`) has exactly two implementations, and they are
not alternatives in the usual sense — they are the same command set reached two
ways.

| | `EmbeddedBackend` | `RemoteBackend` |
|---|---|---|
| How it runs | boots the Koin graph in-process, the way `HeadlessAgentMain` already does | drives the running desktop app's `LocalApiServer` on `127.0.0.1:19280` |
| Needs the app open | no | **yes** |
| Tool surface | `ToolSurfaceId.CLI` | inherits `LOCAL_API`'s posture |
| Command coverage | **37 of 41** (`IMPLEMENTED`) | **29 of 41** (`SUPPORTED`) — those with a real route |
| `backend` on every envelope | `"embedded"` | `"remote"` |

With neither `--local` nor `--remote`, the CLI probes `127.0.0.1:19280` and picks:
app reachable ⇒ remote, otherwise embedded. `--local` forces in-process and
ignores a running app. `--remote` requires the app and fails with
`app_unreachable` rather than silently falling back. `--endpoint <url>` overrides
the base URL.

Attaching to the running app is the default because it is the only way to see
state the app owns: a `RemoteBackend` `edits list` shows the edits the desktop UI
has staged, while an `EmbeddedBackend` in a second process would boot its own
graph and show none of them. Being a second consumer of the contract
`vscode-extension/src/api.ts` and `agentRunner.ts` already use is also the point
— the CLI is not a new surface with its own registry, which is the recurring
defect `ToolSurfaceId.kt` was written to name.

### Why `tanvrit status` always names the backend

Because the two are not equivalent, and a CLI that hid which one answered would
be lying about its own capabilities (§7.8). `EMBEDDED` runs under
`ToolSurfaceId.CLI` with a full capability ceiling; `REMOTE` is a client of a
loopback HTTP API and gets `LOCAL_API`'s posture and route coverage. "Command X
worked yesterday and errors today" is otherwise unattributable to the app having
been open yesterday.

One honest caveat about that posture, carried over from CLAUDE.md: on the
`LOCAL_API` surface `ToolGovernancePolicy.forSurface(LOCAL_API)` (= `LOCKED_DOWN`)
is *selected but inert* — `CapabilityCeilingToolRegistry` filters only when
`ExecutionServices.descriptors` is populated, and that surface has neither
descriptors nor an `ApprovalPort`. So "REMOTE has the tighter ceiling" describes
the wiring, not an enforced restriction. Read it as route coverage, which is
real, rather than as a security boundary, which is not there yet.

Both backends answer `supports()` for every `CommandId` in `COMMAND_TREE`, and
`CliBackendParityTest` walks the tree asserting each command is either supported
on both or carries a written reason. Being unsupported is allowed; being
*accidentally* unsupported is not — that is the same class of gap
`SurfaceCoverageTest` closes for tool surfaces.

## Output contract: stdout is data, stderr is everything else

This is the rule the CLI exists to fix. It applies to every command and both
output modes, with **exactly one named exception**, below:

- **stdout is data.** Under `--json`, every stdout line is exactly one
  `CliEnvelope` and nothing else. Without `--json`, stdout is the rendered result
  and nothing else.
- **stderr is everything else** — progress, warnings, diagnostics, confirmation
  prompts.

### The one exception: `mcp serve --stdio`

An MCP client that spawns the CLI reads JSON-RPC from **its** stdout, so for
that one parsed shape stdout belongs to the transport and not to the renderer.
`claimsExclusiveStdout` (`CliBackend.kt`) answers true for exactly
`mcp serve --stdio` without `--http`. The hand-over itself is **not** done from
argv: `runOnBackend` (`TanvritCliMain.kt`) calls `renderer.handOverStdout()` and
`ExclusiveStdout.offer(stdout)` as the *last* step before the handler runs, so
the renderer starts every run pointed at the real stdout and only moves if the
run is actually going to serve. From there the handler takes the stream
(`EmbeddedMcpCommands.kt`, `ExclusiveStdout.claim(STDIO_CLAIMANT)`), stdout
carries the client's JSON-RPC, the whole `--json` envelope stream — including
the terminal envelope the exit code comes from — is on stderr, and "every
stdout line is an envelope" stays literally true because stdout carries none.
`ExclusiveStdout.release()` runs in a `finally`.

Three consequences, and the ordering is the point of all three:

1. **Every refusal that can happen before the handler still lands on real
   stdout.** The capability check, the confirmation gate and the reachability
   probe all run above the hand-over, so `mcp serve --stdio` on a backend that
   cannot serve it prints its terminal `error` envelope where a `--json`
   consumer reads it. That ordering was the other way round earlier on
   2026-09-11: the hand-over was decided in `dispatch` from argv alone, and
   `tanvrit mcp serve --stdio --remote --endpoint <dead> --json` emitted **zero
   bytes on stdout**
   while the `remote_unsupported` refusal went to stderr. Fixed later the same
   day by moving the hand-over into `runOnBackend`, below the capability check;
   rebuild before reproducing either behaviour, because a launcher built before
   that move still writes the empty stdout.
2. **`--local` is still not optional in a client config.** `mcp.serve` is in
   `RemoteBackend.UNSUPPORTED_WITH_REASON`, so without it the backend probe
   hands the spawn to a running app that has no route for it and the client
   gets a refusal instead of a session — now a legible one on stdout, but still
   a refusal.
3. `--stdio --http` is deliberately *not* a claimant. That combination is
   `bad_usage`/64 before any transport starts, so the hatch must stay shut or
   the caller would lose the error too. It is `bad_usage`/64 on **both**
   backends and through the binary with the app running: `sharedUsageRefusal`
   (`CliCommandContract.kt`) is consulted by `EmbeddedBackend.execute`,
   `RemoteBackend.execute` and `TanvritCliMain.runOnBackend` above its
   capability check. Until 2026-09-11 the refusal lived only in
   `EmbeddedMcpCommands.mcpServe`, so the same two flags answered
   `remote_unsupported`/5 whenever the desktop app happened to be listening.

**A stdio session ends when the client hangs up, as of 2026-09-11.** Before
that the handler parked on a signal only: `tanvrit mcp serve --stdio --json
</dev/null` ran until `timeout` killed it, and a `jstack` on the parked
process showed the sole non-daemon thread was `main`, parked in `runBlocking`
— nothing had leaked, the handler simply never woke. A client that quit
without killing its child therefore left one JVM behind per session.
`EmbeddedMcpCommands.ClientPipe` now wraps `System.in` for the duration of
`runStdio` and `awaitStdioSessionEnd` races that EOF against the shutdown
drain. Measured on a launcher built from this tree: the same command exits in
**one second** with **exit 0**, `[mcp] stopped — the client closed our stdin`
on stderr, and one terminal `result` envelope carrying `state: stopped` —
also on stderr, because that run did serve and the client owned stdout.
Ctrl-C and SIGTERM still end a held-open session, unchanged and shared with
`--http`. `Main.kt`'s `--mcp-stdio` app path still has the old behaviour: it
hands the real `System.in` to `runStdio` with nothing watching it.

**This section said "it is absolute" and "there is no code path from a
`CliBackend` to stdout at all" until 2026-09-11.** Both were true when written
and both were falsified the same day, by the stdio transport landing — the
per-command `mcp serve` row in §"Command reference" was updated and this
section, six hundred lines earlier in the same file, was not. A reader who never reaches that
row is the one this note is for.

### Why this differs from the headless runner

`HeadlessAgentMain` prints its progress with `println`, i.e. to **stdout** —
`[index] 3/97`, `[approval] auto-approved …`, one line per transcript message,
`[grounding] Context Pack injected …`, all of them in
`composeApp/src/desktopMain/.../agent/HeadlessAgentMain.kt`. Its
machine-readable result is then the *last* stdout line. A consumer therefore has to buffer the entire run, split on newlines, take
the last one, and gamble that no progress line ever happens to end in `}`. That
is the contract compute nodes parse today.

The CLI enforces the replacement structurally rather than by convention: a
backend's only handle on the terminal is `CliContext`, and `DesktopCliContext`
holds a `PrintStream` that is stderr. A handler cannot *ask* for stdout — the
only way one ever touches it is to find that `dispatch` already left it in
`ExclusiveStdout` for the one shape above, which is decided before any handler
runs and dropped in a `finally`. Everywhere else stdout is written by the
renderer in `TanvritCliMain.kt` and by nothing else, so a future progress line
that "just does a `println`" fails review, not production.

`--quiet` silences routine progress; `--verbose` adds `[debug]` lines. Both write
to stderr, and passing both is not a contradiction — `--quiet` silences progress,
`--verbose` asks for diagnostics, and someone who passed both gets the
diagnostics they explicitly asked for.

## The `--json` envelope

One line per envelope, newline-delimited, so a reader can act on each line as it
arrives instead of buffering the run.

| Field | Type | Meaning |
|---|---|---|
| `schema` | int | `CliEnvelopeSchema.SCHEMA_VERSION`, currently **1**. Emitted on every envelope |
| `kind` | `"event"` \| `"result"` \| `"error"` | see below |
| `command` | string | the `CommandId`, e.g. `"agent.run"` |
| `backend` | `"embedded"` \| `"remote"` | which path answered |
| `payload` | object \| null | command-shaped; absent on `error` envelopes |
| `error` | object \| null | present on, and only on, `error` envelopes |
| `seq` | int | monotonic index within one invocation |

A **UNARY** command emits exactly one `result` or `error`. A **STREAMING**
command (`index build`, `agent run`, `agent stream`, `model pull`, `mcp serve`)
emits zero or more `event` envelopes and then exactly one `result` or `error`.
`result` and `error` are mutually exclusive and terminal; the constructor of
`CliEnvelope` rejects an `error` envelope with no error and a non-`error`
envelope that carries one, so a malformed envelope cannot be emitted.

`SCHEMA_VERSION` is a promise to scripts: it bumps only on a breaking shape
change. Adding an optional field is not breaking and does not bump it.

### Worked example

```console
$ tanvrit search --repo . --limit 2 --json "retry failed uploads"
{"schema":1,"kind":"result","command":"search","backend":"embedded","payload":{ … },"seq":0}
```

```console
$ tanvrit index build --repo . --json
{"schema":1,"kind":"event","command":"index.build","backend":"embedded","payload":{ … },"seq":0}
{"schema":1,"kind":"event","command":"index.build","backend":"embedded","payload":{ … },"seq":1}
{"schema":1,"kind":"result","command":"index.build","backend":"embedded","payload":{ … },"seq":2}
```

The `payload` objects are elided on purpose. `CliEnvelope.payload` is a
`JsonElement?` and **`status` is the only command whose payload shape is pinned
in code** (`STATUS_PAYLOAD_KEYS` + `requireStatusShape()`, and it is pinned
because the two backends had already drifted apart on it — see
[status / config](#status--config), where a real one is printed in full).
Giving every command a sealed payload class in commonMain would drag the whole
domain model into shared code for no gain, so each command documents its own
schema when its backend lands. Writing a plausible-looking payload here would be
inventing a contract this repo has not made. The envelope *around* it, above, is real — and
it is a real envelope from a real run, which is why `search` shows
`backend:"embedded"` and not `"remote"`.

That distinction is not cosmetic. `search` is one of the twelve in
`RemoteBackend.UNSUPPORTED_WITH_REASON` (there is no `/v1/search` route;
`/v1/knowledge/search` queries `kg_entities`, not the code index), so a remote
`search` can only ever emit a `remote_unsupported` **error** envelope and exit
`5` — the transcript is in
[Not supported remotely](#not-supported-remotely--and-when---local-is-the-remedy).
A `kind:"result"` carrying `backend:"remote"` for `search` is a state the
binary cannot reach. This example said exactly that until 2026-09-10 — a doc
showing output the product cannot produce is the failure CONSTITUTION §7.8
names, in the one file a reader would trust most.

### Error envelopes

Real, re-run 2026-09-11 on 1.5.32 against a loopback server answering only the
`/v1/status` probe:

```console
$ tanvrit policy set retention 30 --remote --json --yes ; echo "exit=$?"
{"schema":1,"kind":"error","command":"policy.set","backend":"remote","error":{"code":"remote_unsupported","message":"Org policy is written by the gateway that issues it; admitting a loopback writer would be a governance hole, which is why GET /v1/policy has no matching POST.","remedy":"No backend implements it: Org policy is fetched from the AI gateway into a read-only flow with no local writer, and a governance rule the governed machine can rewrite is not a control."},"seq":0}
exit=5
```

Two corrections are folded into that one line, both 2026-09-11.

The **message** printed ``"`policy set` is not available on the remote
backend."`` until that morning — a generic sentence the binary no longer emits
for any remote refusal. `unsupportedError` looks the per-command sentence up in
`RemoteBackend.UNSUPPORTED_WITH_REASON` and falls back to a generic one only for
a command missing from that map, which `CliBackendParityTest` makes unreachable.

The **remedy** read `"Re-run with --local to execute it in-process."` in this
transcript, which was a true capture until `otherBackendRemedy` landed later the
same day; `policy set` is in neither backend's implemented set, so that branch no
longer fires for it. The line above is **composed from the source**
(`otherBackendRemedy` + `EmbeddedBackend.UNIMPLEMENTED_WITH_REASON[policy.set]`)
rather than re-captured, and is labelled so here precisely so it is not read as
a transcript; rebuild before re-capturing it. Same defect class as the `search`
example above either way: a doc showing output the product cannot produce.

`code` is stable and scriptable; `message` is for humans and may be reworded, so
branch on the code — the wording above is what the binary prints today, not a
promise about tomorrow. `remedy` is present when there is a concrete next action.
`details` is an optional object. The codes pinned in `CliError.Companion`:

| Code | When |
|---|---|
| `remote_unsupported` | the command exists but this backend has no route for it |
| `unknown_command` | no `COMMAND_TREE` entry matches the argv |
| `bad_usage` | a missing required option, an `INT` option that isn't base-10, an unknown flag |
| `not_authenticated` | no stored credentials |
| `app_unreachable` | `--remote` was given and `127.0.0.1:19280` did not answer |
| `confirmation_required` | a mutating command, no TTY, no `--yes` |
| `internal` | anything else |

Four more codes are real and are **not** in that companion, because they are
produced on the desktop side of the seam only and widening a commonMain
contract that wasm and iOS also compile would buy nothing
(`TanvritCliMain.kt`'s desktop-side code constants, `McpStdioLauncher`). A consumer branching on
`code` sees them, so they are written down here:

| Code | When |
|---|---|
| `not_implemented` | the embedded backend has no handler — four commands, and since 2026-09-11 no option-level case (`mcp serve --stdio` is served) |
| `verification_failed` | an agent run finished and its post-run verification failed |
| `agent_error` | an agent run failed, timed out, or was refused by the provider mid-run |
| `app_launcher_missing` | a stdio MCP client was asked for and no Tanvrit AI app launcher was found beside this install |

## Exit codes

Defined in `ExitCode` (`cli/OutputEnvelope.kt`).

| Code | Name | Meaning |
|---|---|---|
| 0 | `OK` | success |
| 1 | `FAILURE` | the command ran and failed |
| 2 | `VERIFICATION_FAILED` | an agent run completed but verification failed; edits rolled back |
| 3 | `AGENT_ERROR` | agent error, timeout, or a provider that could not serve the model |
| 4 | `NOT_AUTHENTICATED` | no credentials |
| 5 | `UNSUPPORTED` | this backend cannot serve the command — see `--local` below |
| 64 | `BAD_USAGE` | bad argv |

**0, 2, 3 and 64 deliberately reuse `HeadlessAgentMain`'s values**, which are
declared as `EXIT_OK` / `EXIT_VERIFICATION_FAILED` / `EXIT_AGENT_ERROR` /
`EXIT_BAD_USAGE` in `HeadlessAgentMain.kt`, and are the only four it has. Compute
nodes and `scripts/tanvrit-ai-agent.sh` branch on them today and the CLI takes
that entry point over, so a script that already distinguishes "verification
failed" from "agent error" keeps working unchanged. 1, 4 and 5 are new and were
chosen from values the headless runner never returns, precisely so an existing
consumer cannot mistake one for a code it already knows.

Exit 3 covers more than a crash. A run the provider refuses **in band** — one
terminal chunk saying the model is not available, the key is stale, the request
was policy-blocked, the backend is unavailable or a subscription limit was hit —
did no work, so it is a failed run: the payload says `status: "failed"` and the
envelope carries `agent_error`. That is the largest real-world producer of exit
3, because `--model <id-not-installed>` on the default `tanvrit-local` provider
takes exactly this path; it exited **0** with `status: "completed"` until
2026-09-11.

### Which code exits with which number

`exitCodeForError` (`TanvritCliMain.kt`) is the entire mapping, and it
is the only place an `error.code` becomes a number:

| `error.code` | Exit |
|---|---|
| `bad_usage`, `unknown_command`, `confirmation_required` | **64** |
| `verification_failed` | 2 |
| `agent_error` | 3 |
| `not_authenticated` | 4 |
| `remote_unsupported`, `app_unreachable`, `not_implemented`, `app_launcher_missing` | 5 |
| `internal`, any code this table does not name | 1 |

Two of those rows are worth reading twice. **`confirmation_required` exits 64,
not 5** — the command was understood and the *invocation* is what needs fixing
(add `--yes`), which is what `EX_USAGE` means; this doc called it "exit 5's
sibling path" until 2026-09-11 and gave the number in no table.
**`app_launcher_missing` exits 5**, and did not for the few hours between the
code being added and the mapping landing: `exitCodeForError` had no case for
it, so it degraded to `FAILURE` (1) through the `else` branch, and this table
said so. 5 is the right answer because it is the same fact as
`remote_unsupported` — the caller should look somewhere else rather than retry
here — and putting a permanent, knowable property of the downloaded artifact in
the same bucket as an internal error would have made exit 1 unbranchable.

## Confirmation on mutating commands

`CommandSpec.mutating` marks the commands that change state a user would want
back (marked ✎ in the reference below). Before executing one, the CLI calls
`CliContext.confirm`, and `DesktopCliContext` applies three rules in order:

1. `--yes` wins everywhere, including non-interactively — it is an explicit,
   auditable answer, and it is what makes automation possible.
2. No TTY and no `--yes` is an immediate **refusal**, never a block:
   `confirmation_required`, **exit 64** (`exitCodeForError` groups it with
   `bad_usage`: the invocation is what needs fixing), and the process ends. This
   read "exit 5's sibling path" until 2026-09-11, which named the wrong number
   by implication and appeared in no table. A prompt
   written to a stderr nobody reads, blocking on a stdin nobody will type into,
   is a hung job that burns its whole wall-clock budget and reports a timeout —
   strictly worse than a named refusal the caller can fix with `--yes`.
3. With a real console, only `y` / `yes` counts. Anything else, including EOF,
   is "no": a mutating action needs a positive answer, not the absence of a
   negative one.

This implements §7.4 ("agents act, never rule") once, at the seam, rather than at
every call site.

## Global options

Accepted by every command and handled before dispatch (`GLOBAL_OPTIONS`).

| Option | Short | Meaning |
|---|---|---|
| `--json` | `-j` | emit the envelope stream on stdout |
| `--quiet` | `-q` | suppress progress output on stderr |
| `--verbose` | `-v` | verbose diagnostics on stderr |
| `--local` | | force in-process execution, ignoring a running app |
| `--remote` | | require the running app; fail if it is not reachable |
| `--endpoint <url>` | | Local API base URL |
| `--yes` | `-y` | assume yes for confirmation prompts |
| `--help` | `-h` | show help for this command |

Two shared options recur below with fixed spellings and defaults, declared once
so no subcommand can mean something slightly different by them.

**`--repo <dir>`** — default `.`. A **blank or whitespace-only** value is
`bad_usage`, exit `64`, rather than being promoted to `.`: `--repo "$UNSET"`
silently re-pointing at the current directory is the same wrong-scope-with-exit-0
failure that `search` refuses for an unindexed path. Omit the option to get `.`.

**`--limit <n>`** — default `20`, range **`0`–`100`**.

| Value | Behaviour |
|---|---|
| negative | `bad_usage`, exit `64`. It is not "no limit" — SQLite reads a negative `LIMIT` as unbounded, and `--limit -1` used to return **18,863 rows** in symbol mode for an option declared "Maximum results". |
| `0` | returns nothing, in every mode. |
| `1`–`100` | honoured. |
| `>100` | capped at 100, and **said out loud**: a `[limit]` warning on stderr, plus `requestedLimit` / `effectiveLimit` / `limitCapped: true` in the payload. The ceiling is `SemanticSearch`'s, which the CLI does not own; it was already silently capping hybrid queries before it was declared here. |

The rule is enforced once at dispatch, so it holds for every command declaring
`--limit` — `search`, `knowledge search`, `knowledge list`, `agent runs` — and
on **both** backends. The remote path forwarded the raw value as a query
parameter until 2026-09-10, which meant `--limit -1` was refused with the app
closed and accepted with it open.

## Command reference

Every command in `COMMAND_TREE`, in `tanvrit --help` display order.
✎ marks a mutating command (needs `--yes` without a TTY). ⇄ marks a streaming
command.

The **Backend** column says which of the two actually does the work, because
"the command exists" and "this backend can run it" are different facts and only
the second one decides whether your invocation succeeds:

| Value | Meaning |
|---|---|
| `both` | served embedded and remote |
| `local` | embedded only — remote answers `remote_unsupported`, exit 5, remedy `--local` |
| `remote` | remote only — embedded answers `not_implemented`, exit 5, remedy `--remote` |
| `neither` | a stub on both paths; with `--yes` and a reachable backend, exit 5 either way. Both of those paths emit a `remedy` and **neither names `--local`** — each says "No backend implements it: …" and quotes the *other* backend's reason (`otherBackendRemedy`, added 2026-09-11; until that day remote's remedy was the unconditional `--local`, a closed road for these two). Two earlier refusals are not: `--local` without `--yes` ⇒ `confirmation_required` / **64** (the embedded stub is refused inside the handler, after the gate), and `--remote` with no app ⇒ `app_unreachable` / 5 offering "drop `--remote`" |

Transcribed 2026-09-10 from `EmbeddedBackend.IMPLEMENTED` and
`RemoteBackend.SUPPORTED`. Those two sets are the authority; this column is a
convenience that a parity test does not police.

### auth

| Command | Backend | Options / args | Does |
|---|---|---|---|
| `auth login` ✎ | `local` | `--email <addr>` (optional; becomes an OAuth `login_hint`) | sign in through your browser |
| `auth logout` ✎ | `local` | | clear stored credentials — **both** the SDK session and the durable mirror |
| `auth whoami` | `local` | | show the signed-in account |

**`auth login` is a browser flow**, the same shape `wrangler`, `firebase`, `gcloud` and
`gh` use: the CLI binds a throwaway HTTP server on `127.0.0.1`, opens your browser at the
provider's authorize endpoint with that loopback as the `redirect_uri`, catches the
redirect and exchanges the code for tokens. OAuth 2.0 authorization code + **PKCE (S256)**;
the CLI is a public client and holds no secret. The URL is always printed as well, so the
flow still works over SSH or in a container where no browser can open.

Endpoints are **discovered**, never hardcoded — `GET {issuer}/.well-known/openid-configuration`,
where the issuer defaults to `Constants.TANVRIT_API_BASE`. `TANVRIT_OAUTH_ISSUER` points it
at a self-hosted deployment; `TANVRIT_OAUTH_CLIENT_ID` overrides the client id.

Tokens go to the **same** `AuthHandler` credential store the desktop app writes, so signing
in through either surface authenticates both. `--code` is gone: the one-time code is entered
in the browser, where the provider owns it.

> **Not usable yet — blocked on a server-side record (checked 2026-09-10).** The
> authorization server has no OAuth client for the CLI: `client_id=tanvrit-cli` (and
> `tanvrit-ai-cli`, `tanvrit-desktop`, `tanvrit-ai`, `cli`) all answer `400 invalid_client`,
> and the discovery document exposes no `registrationEndpoint`, so the CLI cannot
> self-register. Someone must create a **public** client server-side — token auth method
> `none`, PKCE required, redirect URIs covering `http://127.0.0.1:*/callback`. Until then
> `tanvrit auth login` fails fast with `oauth_client_not_registered` and a message naming
> exactly that, checked *before* a browser window is opened. No code change is needed once
> the client exists.

**`auth logout` signs out of both halves, as of 2026-09-10.** It used to call
only the SDK's `clearToken()`, which empties the in-memory secure store and
leaves the app-layer mirror — `access_token`, `refresh_token`, `user_id`,
`user_json` in `CredentialStorage` — sitting on disk. The next CLI boot
rehydrated it exactly as `Main.kt` does on the next app launch, so the command
reported `signedOut: true`, exited `0`, and the very next `tanvrit auth whoami`
printed the same signed-in user: a stale token stayed reusable after the user
had been told it was gone. It now purges the mirror too, mirroring `App.kt`'s
`performSignOut`.

Both halves are attempted independently, so one failure cannot skip the other,
and the result reports what actually happened rather than asserting success —
`signedOut`, `sdkSessionCleared`, `credentialsRemoved`. If any key survives, the
command fails with `internal` and names the keys, because "you are signed out"
is exactly the sentence that must not be printed on a guess.

### index

| Command | Backend | Options / args | Does |
|---|---|---|---|
| `index build` ✎ ⇄ | `both` | `--repo <dir>`, `--timeout <min>` (budget in minutes; `0` disables), `--force` (see below) | index a repository for search and grounding |
| `index status` | `both` | `--repo <dir>` | index freshness and counts |
| `index clear` ✎ | `both` | `--repo <dir>` | delete the index for a repository |

**`--force` is honoured on one backend only, and the doc used to promise both.**
Until 2026-09-10 this table said `--force` "re-indexes files already current",
full stop. That is true when a running app serves the command and false when it
does not:

| Backend | What `--force` does |
|---|---|
| `remote` | **honoured** — becomes `full: true` on `POST /v1/index/reindex`, and the app does a full pass |
| `embedded` | **inert** — `HeadlessRepoIndexer.indexRepo` takes a path and a budget and nothing else, so `FileIndexer` still decides per file from its stored hash |

The embedded path does not pretend otherwise. It warns on stderr and reports the
flag's real fate in the payload, which is the observable difference between a
flag that was ignored and one that was honoured:

```console
$ tanvrit index build --local --repo /tmp/demo-repo --force --yes
[index] --force is not wired on the embedded path; indexing incrementally.
[index] /tmp/demo-repo (budget 10m)
processed=0 total=1 currentFile=a.kt
[index] indexed 1 file(s) — 0 new, 0 updated, 0 removed, 1 unchanged (no embeddings)
repo            /tmp/demo-repo
state           lexical_only
summary         indexed 1 file(s) — 0 new, 0 updated, 0 removed, 1 unchange…
forceRequested  true
forceHonoured   false
files           1
semanticReady   false
```

`1 unchanged` with `forceHonoured: false` is the proof: the file was skipped, not
re-indexed. Branch on `forceHonoured`, never on the exit code — the command
succeeds either way, because indexing incrementally is not a failure.

Two further asymmetries on the remote path, both surfaced rather than hidden:
`POST /v1/index/reindex` reconciles directories the app has **already** indexed
and `404`s on an unknown one, so remote `index build` refreshes an index and
cannot create one; and the route answers once with no progress channel, so
`RemoteBackend` keeps the envelope *grammar* (one `event` per indexed directory,
then one `result`) and says plainly on stderr that mid-pass progress is not
available. Same parse on both backends, no invented progress.

### search / context

| Command | Backend | Options / args | Does |
|---|---|---|---|
| `search <query…>` | `local` | `--repo <dir>`, `--limit <n>`, `--mode` (`keyword` \| `semantic` \| `hybrid` \| `symbol`, default `hybrid`) | search the index |
| `context <task…>` | `local` | `--repo <dir>`, `--budget <n>` | build a Context Pack for a task — the same fused grounding call as the `tanvrit_context` MCP tool. See [AUTONOMY_LAYER.md](AUTONOMY_LAYER.md) |

Both take a variadic positional, so the query need not be quoted.

**One exception to `search`'s `local`, as of 2026-09-11.** `search --limit 0`
is answered above the backend seam — by `zeroLimitAnswer`
(`CliCommandContract.kt`), out of argv, with no rows fetched — so it exits `0`
with `{"mode":…,"results":[],"query":…,"emptyReason":"limit_zero"}`
(plus `semanticProvider: null` under `--mode semantic` / `hybrid`) whether or
not the app is running. It used to exit `0` in-process and `5`
`remote_unsupported` with the app up, off the same argv, because backend
selection is automatic. Explicit `--remote` with nothing listening is still
`app_unreachable`/`5`, because that probe lives in `dispatch`, above
`runOnBackend`. Every other `search` invocation is still `local`.

**`search --repo <unindexed>` now refuses, as of 2026-09-10.** It used to answer
— confidently, exit `0` — with hits from a completely different codebase. The
mechanism was quiet: `directoryFor` returns null for a path that is not an
indexed project, and every `repository.search*` call treats a null directory id
as "no filter", i.e. *search the entire global index*. Nothing in the payload
said the scope had been dropped, so a caller could not tell it had been served
results from a project it never asked about. A wrong answer that looks right is
worse than no answer:

```console
$ tanvrit search --local --repo /tmp/demo-repo "retry failed uploads" ; echo "exit=$?"
tanvrit: `/tmp/demo-repo` is not an indexed project, so there is nothing scoped to search. Indexed here: /home/you/Developer/tanvrit/ai, /tmp/tinyrepo. (bad_usage)
  Run `tanvrit index build --repo /tmp/demo-repo --yes` first.
exit=64
```

The refusal lists what *is* indexed rather than making the user guess why their
scope did not take. Note the code and exit status: this is `bad_usage` / `64`,
not an empty result — an unindexed `--repo` is a malformed request, not a
repository with nothing in it.

`index status` deliberately does **not** follow that rule: an unindexed repo
there is a `result` carrying `indexed: false`, because "is this indexed?" is the
question that command exists to answer, and refusing to answer it would be
absurd.

**An empty `results` array says why it is empty, as of 2026-09-11 — on the
embedded backend.** Every `search` payload carries `query` — the string the
process actually searched for — and, when `results` is empty, `emptyReason`:
one of `limit_zero`, `query_undecodable`, `query_charset_unverified`,
`semantic_unavailable` or `no_match`. Only the last is a claim about your
repository; the others say this run could not answer, and each of them outranks
`no_match` for that reason.

**`search` itself has only one backend**, so for that command the guarantee is
unqualified: `CommandId("search")` is in `RemoteBackend.UNSUPPORTED_WITH_REASON`
— there is no `/v1/search` route, and
`/v1/knowledge/search` queries `kg_entities`, not the code index — so every
`search` run is embedded and every one of them carries both fields.

**`knowledge search` carries them on both backends — since later on
2026-09-11.** `RemoteBackend.knowledgeSearch` runs the same three helpers the
embedded path does: `queryCharsetNotice` on stderr, `emptyResultReason` gating
the "no entries match" line so a mangled query is never told the knowledge base
is empty, and `attributedSearchPayload` to attach `query` and `emptyReason` to
the route's rows. This paragraph said "`knowledge search` carries the same two
fields" from earlier that day, when it was false: the attached path forwarded
the query and emitted the app's JSON verbatim, so a destroyed query got a bare
`{"results":[]}`, exit 0, and nothing on stderr — the default backend, saying
nothing, about the one command in this section that has two backends.

**`index build` is the half still missing** — see the note under its own
paragraph below, and §"Known holes", item 1.

The one that matters most is `query_charset_unverified`. argv is decoded by the
JVM before `main` is entered, using `sun.jnu.encoding`, which comes from the
inherited locale. Measured on the bundled Zulu 21.0.12.1 with a byte-identical
command line, argv bytes `72 C3 A9 73 75 6D C3 A9` (`résumé` in UTF-8):

| `LC_ALL` | `sun.jnu.encoding` | what the process searched for |
|---|---|---|
| `C.UTF-8` | `UTF-8` | `résumé` — the hit is returned |
| `en_US.ISO-8859-1` | `ISO-8859-1` | `rÃ©sumÃ©` — genuinely absent, so `results:[]` |
| `C` | `ANSI_X3.4-1968` | `r<U+FFFD><U+FFFD>sum<U+FFFD><U+FFFD>` — refused, `bad_usage`/`64` |

The middle row used to read as "your codebase does not contain that word". It
still exits `0` with a `result`, because a search that matched nothing *is* a
result — but the echoed `query` lets a caller compare what it sent against what
was searched, and stderr names the charset and the remedy.

`index build` had the same fault on the file-name side. `File.list()` decodes
directory entries with `sun.jnu.encoding` too, so under `LC_ALL=C` a file called
`café.py` comes back as `caf<U+FFFD><U+FFFD>.py`, whose re-encoding names no
file — `isFile()` answers false and the walk drops it before any counter sees
it. A four-file checkout indexed as `total: 3`, `files: 3`, `0 removed`, exit
`0`, nothing on stderr. When the path charset is not UTF-8 the result now
carries `pathCharset` and `countIsLowerBound: true`, and stderr says the count
is a lower bound. The count is **not** corrected: `HeadlessRepoIndexer` owns the
walk and its ignore rules, and a second walk from the CLI would report `.git`
objects as missing source files.

**Embedded only.** `pathCharset` and `countIsLowerBound` are emitted by
`EmbeddedIndexCommands`; `RemoteBackend.indexBuild` emits the
`/v1/index/reindex` rows and the response object untouched, so the attached
path — the default with the app running — carries neither key and prints no
lower-bound line. §"Known holes", item 1.

**The repair, also 2026-09-11, is in the generated POSIX launcher**
(`composeApp/build.gradle.kts`), which is the only place that can act —
`-Dsun.jnu.encoding=UTF-8` at runtime was measured not to work, because the
property is read before any `-D` applies. Before `exec`ing the JVM the launcher
probes `locale charmap` and, when it is not `UTF-8`, exports the first of
`<lang>_<REGION>.UTF-8`, `C.UTF-8`, `en_US.UTF-8` that `locale charmap` confirms
— as `LC_CTYPE` normally, or by rewriting `LC_ALL` in place when `LC_ALL` is
what is set, since it overrides `LC_CTYPE`. Measured after the change: all three
rows above come back exact, `café.py` is walked, and a `de_DE.UTF-8` caller
keeps `user.language=de` / `user.country=DE`. Two carve-outs, both deliberate:
the `.bat` launcher is untouched, because on Windows `sun.jnu.encoding` follows
the ANSI code page and nothing in a batch file was measured to move it; and
where `locale` is absent (musl) every probe is empty and the block is a no-op.
The `emptyReason` reporting above stays regardless — it is what makes the
failure legible when the launcher is bypassed (`java -cp … TanvritCliMainKt`,
a compute node, a container that calls the jar directly).

### agent

| Command | Backend | Options / args | Does |
|---|---|---|---|
| `agent list` | `both` | | list installed agents |
| `agent run` ✎ ⇄ | `both` | `--repo <dir>`, `--agent <id>`, `--prompt <text>`, `--prompt-file <path>`, `--model <id>`, `--provider <id>`, `--output-dir <dir>`, `--auto-approve`, `--max-tool-calls <n>` (default `50`), `--max-repair-attempts <n>` (default `2`), `--no-index` | run one agent task |
| `agent runs` | `both` | `--limit <n>` | list recent agent runs |
| `agent stream <runId>` ⇄ | `remote` | | attach to a running agent |
| `agent stop <runId>` ✎ | `remote` | | stop a running agent |

`agent stream` and `agent stop` are the two remote-only commands, and they share
one cause: both address a run *by id*, which presumes a registry of live runs
outliving the caller. The app has one (`LocalApiServer`'s `agentRuns` map, behind
`GET /v1/agent/stream/{id}`); an embedded run is a synchronous call inside a
single `tanvrit agent run` invocation with no IPC to reach it from a second
process. Replaying a finished run's transcript from `AgentRunStore` was
deliberately not substituted — `agent stream` promises "attach to a running
agent", and answering a different question under the same name is worse than
refusing.

**Six `agent run` options are silently unavailable remotely** and the CLI warns
on stderr rather than dropping them: `--provider`, `--output-dir`,
`--auto-approve`, `--max-tool-calls`, `--max-repair-attempts` and `--no-index`
have no field on `POST /v1/agent/run`. Pass `--local` if any of them matters to
the run.

**A provider that refuses is a failed run, as of 2026-09-11.** Some providers
refuse by throwing; the default `tanvrit-local` refuses *in band* — one terminal
chunk carrying an explanation and a `finish_reason` of `error`, `auth_error`,
`policy_blocked`, `backend_unavailable` or `subscription_limit`. Those five are
now classified: the payload says `status: "failed"`, the terminal envelope is an
ERROR with `agent_error` (exit **3**), the message names the provider id, the
`finish_reason` and the provider's own sentence, and `error.details` carries the
same run summary the success payload would have. Until then such a run reported
`status: "completed"` and exited **0** having done nothing — the most common way
to hit it is `--model <id-not-installed>` on the default provider. The five are
a denylist on purpose: providers forward upstream reasons untouched, so the set
of *success* reasons is open-ended and an allowlist would fail real runs on
`length`.

`agent run` is the command that supersedes the headless runner; its option names
were chosen to match `HeadlessAgentMain`'s so an existing invocation ports across
by changing the launcher and nothing else. The semantics of `--auto-approve`,
`--max-repair-attempts` and `--no-index` — plan-only by default, bounded
self-repair, pre-run indexing and the four-state index notice — are documented
once in [AUTONOMY_LAYER.md](AUTONOMY_LAYER.md) and are not restated here.

### edits

| Command | Backend | Options / args | Does |
|---|---|---|---|
| `edits list` | `both` | | list staged edits |
| `edits show <editId>` | `both` | | show one staged edit as a diff |
| `edits apply <editId>` ✎ | `both` | | apply one staged edit |
| `edits reject <editId>` ✎ | `both` | | discard one staged edit |
| `edits apply-all` ✎ | `both` | | apply every staged edit |

`both` here is the case where the backend choice changes the *answer*, not just
the route: staged edits live in whichever process staged them, so an embedded
`edits list` will not see what the desktop UI staged, and vice versa. This is
the command set the default probe-and-attach behaviour exists for.

### mcp

| Command | Backend | Options / args | Does |
|---|---|---|---|
| `mcp serve` ⇄ | `local` | `--stdio`, `--http`, `--client-id <id>` | serve the MCP tool catalogue. `--http` listens on 127.0.0.1:19281. **`--stdio` serves a client that spawned this process** — the transport *is* stdout, so for that one invocation `TanvritCliMain.runOnBackend` hands the real stdout to the handler (`CliBackend.claimsExclusiveStdout` names the shape) and moves the whole envelope stream to stderr; stdout then carries JSON-RPC and nothing else. A client config must pass `--local` as well: without it the backend probe hands the spawn to a running app that has no route for it, and the client gets a `remote_unsupported` refusal rather than a session. That refusal is rendered on the real stdout, because the hand-over is the last step before the handler — it was decided from argv earlier on 2026-09-11, which made the same run emit zero stdout bytes; see §"Output contract". This row said `--stdio` was refused (`not_implemented`, exit 5) until 2026-09-11; the transport landed later the same day. Exactly one of the two flags, or `bad_usage` |
| `mcp tools` | `local` | | list the advertised tool catalogue |
| `mcp credential` | `both` | | print this client's MCP credential |
| `mcp config` | `local` | `--client <name>` — one of `claude-desktop`, `claude-code`, `cursor`, `zed`, `continue-dev`, `windsurf`, `cline` | print MCP client configuration; omit `--client` for an index of all seven |

`--client-id` selects an `McpClientToolPolicy` exactly as the `X-Mcp-Client-Id`
header does on the SSE handshake — an unknown or absent id stays unrestricted.
`mcp credential` prints the per-client credential served by
`GET /v1/mcp/credential`, not the deprecated shared token.

**The seven aliases above are the values to type.** They are the set `--help`,
this row and the refusal's own remedy all name, and the set
`EnumeratedOptionValuesTest` re-derives from `McpClientId`. The *handler* is
looser than the list: `mcpConfig` trims and lowercases the argument and then
matches it against `clientAlias(profile)` **or** `profile.displayName.lowercase()`
(`EmbeddedMcpCommands.mcpConfig`), so `CURSOR`, `  cursor  `, `claude desktop`
and `continue.dev` are accepted too. Those synonyms are deliberately unlisted and
the test deliberately does not assert them, so nothing keeps them alive — type
one of the seven. This paragraph read "**The `--client` values are the seven
aliases above and nothing else**" for part of 2026-09-11: the absolute was false
against the very build the rest of this section was verified on, which is the
docs-truth defect it had just been written to fix.

This row and `CommandSpec.kt`'s option help both read "`claude` | `cursor` |
`zed`" until 2026-09-11: `claude` is not a value the command accepts —
`mcpConfig` matches
the argument against `McpClientProfiles.all`, so it answers `bad_usage`, exit
64, "Unknown MCP client 'claude'." — and the same string hid five values that do
work. The refusal's own remedy already listed all seven; help and docs a user
reads *before* typing are where the wrong list cost the round trip.
`EnumeratedOptionValuesTest` re-derives the set from `McpClientId` and fails if
the option help drifts from it again.

Five of the seven speak **stdio** (`claude-desktop`, `zed`, `continue-dev`,
`windsurf`, `cline`) and their rendered config names an executable, so on an
install with no Tanvrit AI app launcher beside it — the standalone CLI tarball,
which ships `bin/`, `lib/` and `runtime/` only — `mcp config` refuses with
`app_launcher_missing`. It used to emit `<install>/runtime/bin/java
--mcp-stdio`, which is the bundled JVM and exits 1 at its own argument parser.

That refusal used to be a dead end for the tarball, whose only remaining
suggestion was the two HTTP clients. Since 2026-09-11 the remedy names three
roads, cheapest first, and the first one works from the very install that
refused: point the client at your own `tanvrit` launcher with `"args": ["mcp",
"serve", "--stdio", "--local", "--client-id", "<alias>"]`. **`--local` is not
optional there** — without it the backend probe hands the spawn to a running
desktop app, which has no route for `mcp.serve`, so the config works with the
app closed and fails with it open. The path has to be supplied by the user
because a CLI started through `bin/tanvrit` cannot read its own launcher path
back out of the process table (`McpStdioLauncher.kt` records that measurement);
`mcp config` will not guess one. The other two roads are installing the desktop
app — which makes this command emit a block that needs no editing — and using
one of the two HTTP clients (`claude-code`, `cursor`) that need no launcher at
all.

### model / provider

| Command | Backend | Options / args | Does |
|---|---|---|---|
| `model list` | `both` | | list models |
| `model load <modelId>` ✎ | `both` | | load a local model |
| `model unload <modelId>` ✎ | `both` | | unload a local model |
| `model pull <modelId>` ✎ ⇄ | `local` | | download a model from the catalogue |
| `provider list` | `both` | | list AI providers and their status |
| `provider set <providerId>` ✎ | **`neither`** | | set the active provider |
| `provider test <providerId>` | `both` | | validate a provider's credentials |

### knowledge / packs / policy

| Command | Backend | Options / args | Does |
|---|---|---|---|
| `knowledge search <query…>` | `both` | `--limit <n>` | search team knowledge |
| `knowledge add <body…>` ✎ | `both` | `--title <text>` (**required**) | add a knowledge entry |
| `knowledge list` | `both` | `--limit <n>` | list knowledge entries |
| `pack list` | `both` | | list domain packs |
| `pack enable <packId>` ✎ | `both` | | enable a domain pack |
| `pack disable <packId>` ✎ | `both` | | disable a domain pack |
| `policy show` | `both` | | show the effective org policy |
| `policy set <key> <value>` ✎ | **`neither`** | | set an org policy value |

`knowledge add` **requires `--title`** because the store upserts on
`(kind, name)`: the title is the entry's identity, so re-running `knowledge add`
with the same title updates that entry instead of creating a second one, and the
envelope says `created: false` when it did. Deriving a title from the body would
make that key collide silently for two notes that open with the same sentence,
so a **missing** one is `bad_usage`, exit 64 — refused by `ArgParser`'s
required-option check before the confirmation gate, before Koin boots and before
a backend is chosen at all, which is why the refusal is identical on `--local`
and `--remote`. A **blank** one is a different story, because the grammar tests
*presence*: `--title ""` and `--title "   "` get past it, so it is the
*handler* that has to refuse them — and since 2026-09-11 **both handlers do,
with the same code, message and remedy**. `requireKnowledgeTitle`
(`EmbeddedKnowledgeCommands.kt`) is the single implementation;
`EmbeddedKnowledgeCommands.knowledgeAdd` calls it, and
`RemoteBackend.knowledgeAdd` calls the *same function*, re-raising its failure as an `ApiFailure` so it unwinds through the
remote flow. Measured on the 1.5.32 launcher, 2026-09-11, both exit **64**:

```console
$ tanvrit knowledge add "the quick brown fox" --title "" --local --json --yes
{"schema":1,"kind":"error","command":"knowledge.add","backend":"embedded","error":{"code":"bad_usage","message":"`knowledge add` needs a non-blank --title: it is the entry's identity, not a label.","remedy":"tanvrit knowledge add --title \"Why we pin the SDK\" <body…>"},"seq":0}
exit=64

# --remote needs something to reach: with nothing listening the reachability
# probe answers first (app_unreachable / 5), so this run is against a stub
# Local API on 127.0.0.1:19412 that logs every request it receives.
$ tanvrit knowledge add "the quick brown fox" --title "" --remote --endpoint http://127.0.0.1:19412 --json --yes
{"schema":1,"kind":"error","command":"knowledge.add","backend":"remote","error":{"code":"bad_usage","message":"`knowledge add` needs a non-blank --title: it is the entry's identity, not a label.","remedy":"tanvrit knowledge add --title \"Why we pin the SDK\" <body…>"},"seq":0}
exit=64
```

The remote refusal happens **before the request**. The stub's log for that run
and for a well-titled control run, in order:

```
GET  /v1/status                                  <- run A (--title ""): the probe, and nothing after it
GET  /v1/status                                  <- run B (--title T1): the probe
GET  /v1/mcp/credential?client_id=tanvrit-cli    <- run B
POST /v1/knowledge/entities BODY={"name":"T1","summary":"the quick brown fox"}
```

So nothing is upserted under a derived key. `LocalApiServer` also rejects a
blank `name` with 400 independently, so a hand-rolled POST cannot key an entry
on an empty title either.

Note the ordering this implies: the blank-title guard lives *inside*
`knowledgeAdd`, so on `--remote` the **reachability probe runs first**. With no
app up, `tanvrit knowledge add <body> --title "" --remote` answers
`app_unreachable` / exit 5, not `bad_usage` / 64. Missing `--title` is the
opposite — `ArgParser` refuses it before a backend is chosen at all, which is
why that envelope says `backend: "embedded"` even under `--remote`.

**One ordering wart survives, on both backends.** The confirmation gate runs
before the handler, so a blank title *without* `--yes` answers
`confirmation_required` / 64 first, and the user is told to pass a flag that
buys a second refusal:

```console
$ tanvrit knowledge add "hello" --title "" --local --json    # and --remote, identically
{"schema":1,"kind":"error","command":"knowledge.add","backend":"embedded","error":{"code":"confirmation_required","message":"`knowledge add` changes state and was not confirmed.","remedy":"Pass --yes to confirm non-interactively."},"seq":0}
exit=64
```

That is the same "told to pass `--yes` for a command that was never going to
run" complaint the *missing*-title fix removed, surviving for the blank value.
It is a gate-ordering question in `dispatch`, not a doc question; see
§"Known holes" below, item 3.

**Three corrections this paragraph has carried, all recorded rather than
overwritten.** (1) The row said `--title <text>` with no marking until
2026-09-11, and the embedded path refused a missing one *after* the mutating
gate. (2) The paragraph written that day to correct that claimed "a blank one is
refused by the handler with the same code" and "both backends' handlers already
refused a missing one" — the first was false for remote and the second was
false outright, since a missing `--title` never reaches a handler at all.
(3) A third paragraph, written later the same day to correct *those*, asserted
the opposite falsehood — that `RemoteBackend` "falls back to `firstLine(body)`"
and that "**that hole is open** as of 2026-09-11" — while the very commit that
shipped the sentence had added `requireKnowledgeTitle` to `RemoteBackend`
fifteen minutes earlier. It also cited `RemoteBackend.kt:667`, which at that
commit was a parameter list inside `configSet`, not `knowledgeAdd`. All of that
is replaced by the measured
transcripts above, 2026-09-11. Nothing in this section is struck through;
earlier wordings are quoted here and deleted from the body, which is what
"recorded rather than overwritten" means in this file.

`knowledge search` searches `kg_entities` — team knowledge, not the code index.
`search` is the code-index command, and the two are not interchangeable, which
is exactly why remote `search` is refused rather than quietly mapped onto
`/v1/knowledge/search`.

### status / config

| Command | Backend | Options / args | Does |
|---|---|---|---|
| `status` | `both` | | show app, server and backend status |
| `config get [key]` | `both` | | read a config value; the key is optional |
| `config set <key> <value>` ✎ | `both` | | write a config value |
| `config path` | `local` | | print config and data locations |

`status` is the one command whose payload shape *is* pinned:
`STATUS_PAYLOAD_KEYS` + `requireStatusShape()` are called inside both handlers,
because the two backends once disagreed — embedded emitted 16 keys and remote 4,
so `tanvrit status --json | jq .authenticated` answered with the app closed and
returned nothing with it open. **A field a backend cannot determine is `null`,
never absent:** null says "same schema, unknown here", absent says "different
schema", and only the first is safe to branch on.

This capture is from a **1.5.31** launcher, which is why `version` reads
`1.5.31` while `appVersionName` is `1.5.32` today; the key set, not the values,
is what it is here to show. Noted 2026-09-11 rather than silently edited — the
value in a transcript is evidence, and editing it would make the transcript a
mock-up.

```console
$ tanvrit status --local --json
{"schema":1,"kind":"result","command":"status","backend":"embedded","payload":{"backend":"embedded","toolSurface":"CLI","version":"1.5.31","sdkVersion":"3.0.1","endpoint":"http://127.0.0.1:19280","appRunning":false,"localApi":"unavailable","mcpServer":"unavailable","engine":"unavailable","authenticated":false,"account":null,"provider":"tanvrit-local","indexedProjects":5,"indexedFiles":1546,"indexedSymbols":21004,"stagedEdits":0},"seq":0}
```

`config path` is `local` on purpose rather than by omission: config and data
locations are *this process's* own paths, and asking the app would answer for
the app.

## Not supported remotely — and when `--local` is the remedy

`RemoteBackend.supports()` returns false for any command with no `LocalApiServer`
route behind it. **Twelve of the 41 are in that set; the other 29 are served.**

This section said something different until 2026-09-10. It named four
*categories* — "marketplace, org policy, provider config, knowledge writes" —
which was a reasonable summary of an earlier state and is now wrong in both
directions. It **over-listed**: marketplace (`pack *`), org policy reads
(`policy show`), provider config reads (`provider list`, `provider test`) and
knowledge writes (`knowledge add`) all have routes now and are served remotely.
And it **under-listed**: `auth *`, `search`, `context`, `mcp serve`,
`mcp tools`, `mcp config`, `model pull` and `config path` are refused and were
named by none of the four categories. Summarising a partition into categories
is what went wrong; the table below is transcribed from
`RemoteBackend.UNSUPPORTED_WITH_REASON` instead, which is exhaustive by
construction.

**Verbatim**, copied from `RemoteBackend.UNSUPPORTED_WITH_REASON` on 2026-09-11
(`RemoteBackend.UNSUPPORTED_WITH_REASON` — the map, not a line span,
is the reference), with no markdown styling added inside a sentence — because
since the per-command
sentences started reaching users (below), this table is a transcript of what
the binary prints and not a summary of it. Two rows were not verbatim before:
`search` dropped the word "route", and `model pull` was truncated at "no
progress channel." — cutting off the clause that carries the reasoning.

| Command | Why remote cannot serve it |
|---|---|
| `auth login` | Sign-in is interactive and stores credentials locally; the app exposes no /v1/auth route. |
| `auth logout` | Clearing stored credentials is a local-keychain operation with no /v1/auth route. |
| `auth whoami` | Account identity is a local credential fact, and putting the signed-in user's email behind a loopback bearer any local process can read would leak it for no offsetting need. |
| `search` | There is no /v1/search route; /v1/knowledge/search queries kg_entities, not the code index, and the app holds no SemanticSearch to answer --mode semantic honestly. |
| `context` | POST /v1/context stores editor context in a ring buffer; it neither builds nor returns a Context Pack. |
| `mcp serve` | Serving MCP is a long-lived process this CLI hosts itself; there is nothing on the app to proxy to. |
| `mcp tools` | The tool catalogue lives on the MCP server (19281) and the Local API exposes no catalogue route. |
| `mcp config` | Client configuration is rendered from local paths and the credential; no route returns it. |
| `model pull` | A multi-gigabyte download is the one command whose value IS the progress stream, and the Local API has no progress channel — the app can pull, but a route that answered once after twenty silent minutes would be worse than this refusal. |
| `provider set` | Selecting the active provider steers every in-app conversation, not just this caller's; a CLI should not silently repoint the window the user is looking at. |
| `policy set` | Org policy is written by the gateway that issues it; admitting a loopback writer would be a governance hole, which is why GET /v1/policy has no matching POST. |
| `config path` | Config and data locations are this process's own paths — asking the app would answer for the app. |

Ten of those twelve are served by `EmbeddedBackend`, so `--local` genuinely
fixes the invocation — and the remedy above says `--local` **because** of that,
not by default: `otherBackendRemedy` (`TanvritCliMain.kt`) tests the id against
`EmbeddedBackend.IMPLEMENTED` first. The exceptions are `provider set` and
`policy set`, which neither backend serves. With `--yes`, they answer `not_implemented` on
`--local` and `remote_unsupported` on a *reachable* `--remote`, exit 5 either
way, and since 2026-09-11 **neither of those two refusals offers the other
backend**: both say "No backend implements it: …" and quote the other side's
reason. The two refusals that come *earlier* are not backend-aware and are not
covered by that sentence — `confirmation_required` / 64 without `--yes`, and
`app_unreachable` / 5 with `--remote` and no app, whose remedy still says "drop
`--remote` to run in-process". The transcripts are at the top of this
file, in §"Status"; this sentence read "neither refusal offers the other
backend", unqualified, until 2026-09-11.

### What the refusal actually looks like

The previous version of this section showed an `error:` / `remedy:` transcript
that the binary has never printed. The real renderer is
`TanvritCliMain.renderHumanError`, and its shape is **`tanvrit: <message>
(<code>)`** on one line, followed by the remedy indented two spaces — both on
stderr. Re-run 2026-09-11 on 1.5.32:

```console
$ tanvrit search --remote "retry failed uploads" ; echo "exit=$?"
tanvrit: There is no /v1/search route; /v1/knowledge/search queries kg_entities, not the code index, and the app holds no SemanticSearch to answer --mode semantic honestly. (remote_unsupported)
  Re-run with --local to execute it in-process.
exit=5
```

A refusal costs one round trip and no more: the launcher probes `/v1/status` to
satisfy `--remote`, then answers from `supports()` **before** any request for
the command itself is built. So this output does not depend on what the app
would have said — which is also why the two transcripts here were captured
against a loopback server that answers only the probe.

```console
$ tanvrit search --remote --json "retry failed uploads" ; echo "exit=$?"
{"schema":1,"kind":"error","command":"search","backend":"remote","error":{"code":"remote_unsupported","message":"There is no /v1/search route; /v1/knowledge/search queries kg_entities, not the code index, and the app holds no SemanticSearch to answer --mode semantic honestly.","remedy":"Re-run with --local to execute it in-process."},"seq":0}
exit=5
```

The stable `code` is printed even in human mode on purpose: it is what a user
pastes into a search or a bug report, and a human-readable mode you cannot get
support for is not a kindness.

**The per-command sentences reach the user.** Both transcripts above were
re-captured on 2026-09-11 against 1.5.32, and both print the sentence from
`UNSUPPORTED_WITH_REASON` rather than a generic one.

That is a correction, not a boast. This paragraph said the opposite — "**The
per-command sentences in `UNSUPPORTED_WITH_REASON` do not reach the user
today**", because the launcher checked `supports()` before dispatch and wrote
its own generic message, leaving `RemoteBackend.execute`'s reason-carrying
branch reachable only from a test. That was true when it was written, and the
gap was closed by `unsupportedError` (`TanvritCliMain.kt`), which fetches
`RemoteBackend.UNSUPPORTED_WITH_REASON[spec.id]` at exactly that pre-dispatch
check; its KDoc records the old state in the past tense ("Twelve written
explanations, zero users"). The doc kept describing the gap after the fix
landed, and the three transcripts in this file kept showing the generic
message — output the binary can no longer produce, which is the failure
CONSTITUTION §7.8 names.

Unchanged, and still what a script should branch on: the `code` is
`remote_unsupported` and the exit is 5. `message` is for humans and may be
reworded (`CliError`'s KDoc in `OutputEnvelope.kt`), so a consumer that matched on the old
generic sentence was already outside the contract.

`--local` is a real remedy rather than a workaround because `EmbeddedBackend` is
not a degraded mode: it boots the same Koin graph the app does and executes
against the same repositories and services. The tradeoff is process-local state,
not reduced capability — an embedded `edits list` will not see edits the running
app staged in its own process.

### The routes behind the 29

Twelve commands moved from "unsupported" to "supported" when their routes
landed: `provider list`, `provider test`, `policy show`, `pack list`,
`pack enable`, `pack disable`, `config get`, `config set`, `knowledge list`,
`knowledge add`, `index status`, `index clear`. They had been excused for one
reason only — "there is no route" — which is a statement about this repo, not
about the command, and it meant a user's CLI was *worse* with the app open than
with it closed for no reason anyone had decided.

`/v1` routes on `LocalApiServer` today: `/v1/health`, `/v1/status`,
`/v1/config` (GET + POST), `/v1/models` (+ `{id}/load`, `{id}/unload`),
`/v1/embeddings`, `/v1/context`, `/v1/agent/run`, `/v1/agent/stream/{id}`,
`/v1/agent/runs`, `/v1/agents`, `/v1/edits` (+ `{id}/apply`, `{id}/reject`,
`apply-all`), `/v1/knowledge/search`, `/v1/knowledge/entities` (GET + POST),
`/v1/index/reindex`, `/v1/index/status`, `/v1/index/clear`, `/v1/packs`
(+ `{id}/enable`, `{id}/disable`), `/v1/policy`, `/v1/providers`
(+ `{id}/test`), `/v1/completions`, `/v1/chat/completions`, `/v1/mcp/credential`
(from `TenancyRoutes`) and the deprecated `/v1/mcp/token`.

**That list is context, not the contract.** The authority is `supports()`,
walked by `CliBackendParityTest`; do not derive a support matrix from this
paragraph, which will drift the moment a route is added. Derive it from the
`Backend` column above, or from `tanvrit <command> --json`.

## Relationship to the headless runner

`tanvrit agent run` supersedes `./gradlew :composeApp:runHeadlessAgent` and
`scripts/tanvrit-ai-agent.sh` as the way to run one non-interactive agent task.

**The compute-node command name `tanvrit-ai-agent` does not change.** It is
pinned in code as `COMPUTE_AGENT_COMMAND`
(`composeApp/src/commonMain/kotlin/com/friendlyai/agent/ComputeAgentSubmitter.kt:20`),
asserted by `ComputeAgentSubmitterTest`, and stamped on every `AI_TOOL` job and
cron schedule already submitted to the control plane. Renaming it would strand
every scheduled job on every provisioned node. What changes is what a node
*resolves that name to*; see [compute-agent-nodes.md](compute-agent-nodes.md).

## Known holes

Things the binary gets wrong that this document describes accurately. Listed
so a reader who hits one can tell "known" from "you are holding it wrong", and
so the next person in this code has the list in one place.

**Basis, because it matters here more than anywhere else in this file.** The
transcripts throughout this document were captured by running
`composeApp/build/cli/bin/tanvrit`, each against the build that existed when it
was captured. The CLI sources moved several times on 2026-09-11, so a transcript
records what a binary printed then, not what the one on your disk prints now:
for everything below the *source* is the authority and a transcript is a second
opinion, and where the two disagree it is said so explicitly. Rebuild before
re-running anything here. Nothing in this document was inferred from another
document.

| # | Hole | Where it lives |
|---|---|---|
| 1 | **`index build` says nothing about an untrustworthy file count on the attached backend.** `pathCharset` and `countIsLowerBound` are emitted by `EmbeddedIndexCommands`; `RemoteBackend` has zero occurrences of either, and `RemoteBackend.indexBuild` returns the route's rows and response object untouched. `index build` is `both`, so this is the default path whenever the app is running. The sibling command `knowledge search` had the same gap and no longer does — `RemoteBackend.knowledgeSearch` now calls `queryCharsetNotice`, `emptyResultReason` and `attributedSearchPayload`, which is the shape this one still needs. | `RemoteBackend.indexBuild` |
| 2 | **A `neither` command on `--local` asks for `--yes` before admitting nobody implements it.** `EmbeddedBackend.supports()` is `COMMANDS_BY_ID.containsKey(id)` — membership in the tree, stubs included — so the confirmation gate in `runOnBackend` fires above the stub's own `not_implemented`. `tanvrit provider set <id> --local` answers `confirmation_required` / 64, and `--yes` buys a second refusal. The remote path does not have this: `RemoteBackend.supports()` is `id in SUPPORTED`, so the capability check catches it first. | `EmbeddedBackend.supports`, and the `mutating` block in `runOnBackend` |
| 3 | **Same shape for a blank `--title`,** which no `supports()` can see: `knowledge add <body> --title ""` answers `confirmation_required` / 64 on **both** backends, and only tells you the title is the problem once `--yes` is passed. | the `mutating` block in `runOnBackend`, above both `knowledgeAdd` handlers |
| 4 | **The `app_unreachable` remedy is not backend-aware.** `--remote` with nothing listening answers "Start the desktop app, or drop `--remote` to run in-process" for every command, including the two no backend serves — the closed road `otherBackendRemedy` exists to stop offering. It is raised in `dispatch`, before a backend object exists, which is why it never reaches that helper. | the `BackendKind.REMOTE` branch of `dispatch` |

**Closed earlier on 2026-09-11, recorded so a stale-binary report is not
re-filed as a new defect.** `mcp serve --stdio` on a backend that cannot serve
it used to emit zero bytes on stdout: the stdout hand-over was decided in
`dispatch` from argv, above the capability check, so the refusal went to stderr
along with everything else. `runOnBackend` now hands stdout over as the last
step before the handler, so every pre-handler refusal renders on real stdout:
`tanvrit mcp serve --stdio --remote --endpoint <reachable> --json </dev/null`
writes its terminal `remote_unsupported` envelope to stdout and exits 5.
Rebuild before reproducing this — a launcher built before that change still
emits zero bytes, and a report of that is a report about the binary rather than
about this tree.

## Known stale citations elsewhere

Citations in files this document does not own that point at a line the code has
since moved. Recorded rather than silently left, because a wrong
`file.kt:NNN` sends a reader to confidently wrong evidence:

- `composeApp/build.gradle.kts:1282` — the comment `// appVersionName (line ~106)`.
  `appVersionName` is at `:155`.

Related, and the same failure mode: several files still spell the tarball
metavariable `tanvrit-cli-<ver>-<os>-<arch>.tar.gz` in lowercase, which is the
exact reading that produced a 404 filename in this document. The concrete
filenames are all correct and gated; the placeholders are not gated and still
invite it — four in `composeApp/build.gradle.kts` (one of them `cliDistTar`'s
own `description`, which `./gradlew tasks` prints), five in
`scripts/check-cli-artifact-names.sh`, one in `.github/workflows/ci.yml` and
one in `.claude/skills/tanvrit-ai-testing/SKILL.md`. Counted 2026-09-11 with
`grep -c '<os>'`; none of those files is this document's to edit.
