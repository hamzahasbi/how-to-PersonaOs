# Dependency Security

PersonaOS is Bun-first, so dependency installs use Bun's native package-age guard plus Socket's Bun-native scanner:

```bash
bun run install:safe
```

The root `bunfig.toml` also sets:

```toml
[install]
minimumReleaseAge = 604800

[install.security]
scanner = "@socketsecurity/bun-security-scanner"
```

The release-age guard rejects package versions published less than seven days ago, which reduces exposure to fresh supply-chain attacks. The Socket scanner checks Bun installs for malware, typosquatting, and supply-chain risk. It can run in free mode, or with Socket org settings when `SOCKET_API_KEY` is set.

CI also runs `bun audit --audit-level=high` as a blocking advisory check. Semgrep Community Edition scans the source with its automatic community ruleset and fails the workflow when it finds an issue. The scanner is open source, requires no account, and runs from a versioned container image pinned by digest.

Docker builds copy `bunfig.toml` before install and pass the release-age guard explicitly with `bun install --minimum-release-age 604800`.

## Socket Tools

References:

- [Socket Bun Security Scanner](https://github.com/SocketDev/bun-security-scanner)
- [Socket Firewall Free](https://docs.socket.dev/docs/socket-firewall-free)

Socket Firewall Free runs as an `sfw` command prefix and is still useful for npm, yarn, pnpm, Python, and Rust workflows:

```bash
npm i -g sfw
sfw npm install
```

For this repo's Bun installs, use the Bun scanner instead of `sfw bun install`. If a deployment path switches to npm, yarn, or pnpm, run that install through `sfw`, for example:

```bash
sfw npm ci
```

Socket Firewall Free blocks confirmed malware, warns on AI-detected malware, does not block unknown packages, does not support private registries, and always sends anonymous telemetry. Use Socket Firewall Enterprise if you need policy-based blocking, private registry support, or Bun-compatible centralized enforcement.
