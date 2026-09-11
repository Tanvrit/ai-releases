# Installing Tanvrit AI on macOS

> **Re-synced 2026-09-11; it had been pinned to v1.1.1 while the product
> shipped v1.5.32.** Until that date this guide offered **1.1.1**, a 200 MB
> Apple Silicon `.dmg` on the retired
> `pub-01f0ff97….r2.dev` host (404 today, as is the `dl.tanvrit.com` spelling of
> the same key), a SHA-256 for that dead file, and the sentence *"**Intel Mac?**
> Not yet — the current build only ships for Apple Silicon."* The Intel build
> has shipped since v1.5.32 and `README.md` had already started sending Intel
> users **here** for the Gatekeeper step — so the one page that told them the
> build did not exist was the page they were pointed at. Every URL, size and
> status below was checked with `curl -I` against the live host on 2026-09-11.

> **Signing:** the shipped `.dmg`s are **ad-hoc signed only** — jpackage's
> default — and are **not** Developer ID signed and **not** notarized.
> `release-macos.yml` runs `:composeApp:packageDmg` (not `notarizeDmg`) whenever
> no signing certificate is configured for the run, and warns in the job log
> when it does. macOS will therefore refuse the first launch. That is expected;
> § 3 is the one-time bypass.

## 1. Download

Both architectures ship. Pick the one that matches your Mac: **Apple menu →
About This Mac** — "Apple M1/M2/M3/M4" means Apple Silicon, "Intel Core …"
means Intel.

| Mac | Download | Size |
|---|---|---|
| **Apple Silicon** (M1 / M2 / M3 / M4) | [TanvritAI-1.5.32-macOS-arm64.dmg](https://dl.tanvrit.com/ai/manual/TanvritAI-1.5.32-macOS-arm64.dmg) | 272 MB (285,142,546 bytes) |
| **Intel** | [TanvritAI-1.5.32-macOS-x64.dmg](https://dl.tanvrit.com/ai/manual/TanvritAI-1.5.32-macOS-x64.dmg) | 278 MB (291,348,765 bytes) |

Both answered **HTTP 200** with those exact `Content-Length` values on
2026-09-11. They are distinct objects, not one file under two keys.

**The filename is a case-sensitive R2 key** and `dl.tanvrit.com` does not fall
back: `macOS` has a lowercase `m` and a capital `OS`, `Linux` and `Windows` are
capitalised. `…-macos-arm64.dmg` is a hard 404, not a redirect.

### Verifying the download

**There is no published SHA-256 to compare against.** `Tanvrit/ai` is a private
repository, so its GitHub release page is not openable, and no checksum file is
mirrored beside the installers — `…/TanvritAI-1.5.32-macOS-arm64.dmg.sha256`,
`…/SHA256SUMS` and `…/checksums.txt` all answer 404 (checked 2026-09-11). This
guide printed a SHA-256 until that date, published beside the v1.1.1 link — it
matches nothing you can download today, so it has been removed rather than
left to look authoritative.

Until checksums are published, the byte count is the only figure worth
comparing:

```bash
stat -f%z ~/Downloads/TanvritAI-1.5.32-macOS-arm64.dmg   # expect 285142546
stat -f%z ~/Downloads/TanvritAI-1.5.32-macOS-x64.dmg     # expect 291348765
```

A short file means the transfer was truncated — re-download rather than
opening it.

## 2. Install

1. Double-click the `.dmg` to mount it.
2. Drag **Tanvrit AI** into the **Applications** folder.
3. Eject the disk image (right-click the icon on your desktop → Eject).

## 3. First launch — bypass Gatekeeper once

macOS will refuse to open the app on the first try, because it is not notarized
(see the signing note at the top). **Pick one** of the two paths:

### Path A — point-and-click (one-time, per user)

1. Open **Applications**.
2. **Right-click** (or Control-click) on **Tanvrit AI** → **Open**.
3. macOS asks: *"Tanvrit AI is from an unidentified developer. Are you sure you want to open it?"* — click **Open**.

That's it. From now on you can launch it from Spotlight or the Dock normally.

### Path B — terminal (one command, no clicks)

```bash
xattr -d com.apple.quarantine "/Applications/Tanvrit AI.app"
open "/Applications/Tanvrit AI.app"
```

The first line drops the "downloaded from the internet" tag; macOS then trusts
the binary and launches it without complaint.

## 4. What you'll see on first launch

- A **splash → workspace picker** flow (you can skip if you don't have a Tanvrit account yet).
- A sidebar with **Dashboard, Project Overview, Editor, Agent, MCP, Settings**.
- The **Get started without an API key** banner on the Dashboard — click **Install Tiny Stack** to pull the five on-device specialists (~3.65 GB total). Or skip and add a cloud API key in Settings instead.

## 5. The `tanvrit` CLI is a separate download — and there is no macOS build yet

The command-line tool is **not** inside the `.dmg` — see
[../CLI.md](../CLI.md) §"Where the binary comes from" for the packaging reason
and the two corrections that section carries. It ships as its own tarball — and as of v1.5.32 **only for Linux**.
`tanvrit-cli-1.5.32-macOS-arm64.tar.gz` and `-macOS-x64.tar.gz` are 404 on
`dl.tanvrit.com` (the macOS release leg was added on 2026-09-11 and has never
run). On a Mac, build it from source; see [../CLI.md](../CLI.md).

## 6. Getting unstuck

| Symptom | Fix |
|---|---|
| *"Tanvrit AI is damaged and can't be opened."* | Re-download — the file was probably truncated mid-transfer. Compare the byte count (see § 1). |
| *"Tanvrit AI can't be opened because Apple cannot check it for malicious software."* | You're on first launch — follow § 3 Path A or B above. |
| The app will not launch at all, and § 3 does not help | Check you took the right architecture in § 1 — an Apple Silicon build is a native arm64 binary and an Intel Mac cannot run it. (The reverse, an Intel build on Apple Silicon, is the case macOS offers to handle for you.) |
| App opens, then closes immediately | Run from Terminal so the panic surfaces: `"/Applications/Tanvrit AI.app/Contents/MacOS/Tanvrit AI"`. Send the output to the team. |
| "Engine not running" warning on first launch | Expected — the optional local inference engine ships separately. The app uses your cloud provider or the bundled tiny models instead. |
| Network indicator says "no connectivity" | Tanvrit AI runs fully offline by design; only the cloud providers and the optional hosted gateway need network. |

## 7. Uninstall

```bash
rm -rf "/Applications/Tanvrit AI.app"
rm -rf ~/.tanvrit-ai      # workspace + indexed projects + cached models
rm -rf ~/.tanvrit         # auth profile + hooks.json + annotations
```

## 8. Got feedback?

Send a screenshot + a one-line description to the team. Bonus points if you include:

- The Console.app entry for `Tanvrit AI` (if it crashed)
- The output of `xcode-select -p` and `sysctl -n machdep.cpu.brand_string` if a build setting is suspect
- Whether the issue reproduces on a fresh `~/.tanvrit-ai` (delete it, relaunch — see § 7)
