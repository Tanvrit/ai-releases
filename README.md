# Tanvrit AI — Downloads & Docs

Public home for **downloading Tanvrit AI** and its command-line interface. This
repository holds documentation and the Homebrew formula only — **no source code**.
The application source lives in a separate private repository; releases are
published here so anyone can install without private access.

- Product site: https://ai.tanvrit.com
- All releases: https://github.com/Tanvrit/ai-releases/releases

## Install the desktop app

| Platform | Download |
|---|---|
| macOS (Apple Silicon) | [TanvritAI-macOS-arm64.dmg](https://dl.tanvrit.com/ai/manual/) |
| macOS (Intel) | [TanvritAI-macOS-x64.dmg](https://dl.tanvrit.com/ai/manual/) |
| Windows (x64) | [TanvritAI-Windows-x64.msi](https://dl.tanvrit.com/ai/manual/) |
| Linux (amd64) | [TanvritAI-Linux-amd64.deb](https://dl.tanvrit.com/ai/manual/) |
| Linux (arm64) | [TanvritAI-Linux-arm64.deb](https://dl.tanvrit.com/ai/manual/) |
| iOS (framework) | see the [latest release](https://github.com/Tanvrit/ai-releases/releases/latest) |

Each release page also carries the exact, versioned asset URLs.

## Install the CLI (`tanvrit`)

### Homebrew (macOS & Linux)

```sh
brew tap tanvrit/tap
brew install tanvrit
tanvrit status --json
```

The tap lives at https://github.com/Tanvrit/homebrew-tap.

### Manual (tarball)

```sh
# pick your platform: Linux-x64, Linux-arm64, macOS-arm64, macOS-x64
curl -fLO https://dl.tanvrit.com/ai/manual/tanvrit-cli-<version>-<platform>.tar.gz
tar xzf tanvrit-cli-<version>-<platform>.tar.gz
./tanvrit-cli-<version>/bin/tanvrit status --json
```

The CLI is a self-contained runtime image (bundled JRE) — no separate Java install
is required. It reaches the same functionality as the desktop app: attach to a
running app over the local API when one is up, otherwise run in-process. See
[docs/CLI.md](docs/CLI.md).

## Sign in

`tanvrit auth login` opens a browser for OAuth 2.0 (authorization code + PKCE), the
same flow as `gcloud`/`wrangler`, and shares the desktop app's token store.

## Documentation

- [CLI reference](docs/CLI.md)
- [macOS install notes](docs/install/macos.md)

## License

MIT © Tanvrit Private Limited. See [LICENSE](LICENSE).
