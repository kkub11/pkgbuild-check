# pkgbuild-check

A static security analyzer for Arch Linux PKGBUILD files. Checks for common
malicious patterns and verifies that recommended best practices are in place —
useful before running `makepkg` on an AUR package you haven't reviewed before.

```
  pkgbuild-check  ─  PKGBUILD
────────────────────────────────────────────────────────────────────
  Score  100/100   CLEAN
  0 critical  0 warnings  8/10 best practices

  GOOD PRACTICES FOUND
────────────────────────────────────────────────────────────────────
  ✓ Strong checksum present (b2/sha256/sha512)
  ✓ package() uses $pkgdir
  ✓ DESTDIR=$pkgdir used with make install
  ✓ HTTPS or VCS source URL
  ...
```

## What it checks

### Critical issues (red)

Patterns that strongly indicate malicious intent. A hit here means you should
not install the package without thorough manual review.

| Check | Why it matters |
|---|---|
| `curl`/`wget` piped to `sh`/`bash` | Executes untrusted remote code at build time |
| `base64` + `eval` | Classic obfuscation — hides the real payload from a quick read |
| `eval` on dynamic variables | Can silently execute arbitrary commands |
| Reverse shell patterns (`/dev/tcp`, `nc -e`, …) | Opens a backdoor to an attacker |
| Cron/systemd persistence injection | Establishes malicious persistence outside the package |
| SSH `authorized_keys` modification | Grants persistent remote access |
| `/etc/passwd` or `/etc/shadow` tampering | Creates backdoor accounts |
| `rm -rf` on system paths outside `$pkgdir` | Can destroy the host filesystem |
| Cryptocurrency miner signatures | xmrig, stratum+tcp, cpuminer, etc. |
| `LD_PRELOAD` injection outside `$pkgdir` | Hijacks library loading system-wide |
| Binary download + immediate execution | Bypasses `source()` checksum verification |

### Warnings (yellow)

Suspicious patterns that aren't necessarily malicious but warrant a closer look.

| Check | Why it matters |
|---|---|
| `SKIP` in checksum arrays | Disables integrity verification; only acceptable for VCS sources |
| `md5sums` or `sha1sums` | Cryptographically broken — prefer `b2sums`, `sha256sums`, `sha512sums` |
| Plain `http://` source URL | MITM-vulnerable even with checksums present |
| External script sourced at build time | Bypasses `source()` integrity checking |
| VCS source on `master`/`main`/`HEAD` | Fetches unreviewed, unpinned code on every build |
| `.install` script present | Runs as root post-install; review it separately |
| `chmod 777` or world-writable `o+w` | World-writable installed files are a privilege escalation risk |
| `sudo`/`su` inside build functions | Build functions must not self-escalate |

### Best practices (green / blue)

Things a well-written PKGBUILD should have. Missing items are noted but don't
affect the verdict unless combined with other issues.

- Strong checksum (`b2sums`, `sha256sums`, `sha512sums`)
- `$pkgdir` used in `package()`
- `$srcdir` or `$pkgname` used in `build()`/`prepare()`
- `DESTDIR="$pkgdir" make install` pattern
- HTTPS or VCS source URLs (no plain HTTP)
- `validpgpkeys` for upstream PGP verification
- `license=` field declared
- `$pkgver` used in source URL
- `install -Dm` for file placement
- `# Maintainer:` comment present

## Installation

No dependencies beyond Python 3 (3.6+), which is already on any Arch system.

```bash
# Clone and install
git clone https://github.com/yourname/pkgbuild-check.git
cd pkgbuild-check
chmod +x pkgbuild-check
sudo cp pkgbuild-check /usr/local/bin/
```

Or without cloning:

```bash
curl -O https://raw.githubusercontent.com/yourname/pkgbuild-check/main/pkgbuild-check
chmod +x pkgbuild-check
sudo mv pkgbuild-check /usr/local/bin/
```

## Usage

```bash
pkgbuild-check PKGBUILD              # check a file
pkgbuild-check /path/to/PKGBUILD    # any path
cat PKGBUILD | pkgbuild-check        # read from stdin
pkgbuild-check --no-color PKGBUILD  # disable colour output (for logs/CI)
pkgbuild-check --help
```

### Exit codes

| Code | Meaning |
|---|---|
| `0` | Clean — no issues found |
| `1` | Warnings only |
| `2` | Critical issues found |
| `3` | File not found or permission error |

Exit codes make it easy to gate on results in scripts:

```bash
# Only build if the PKGBUILD passes
pkgbuild-check PKGBUILD && makepkg -si
```

### Use with an AUR helper

Most AUR helpers let you drop into a shell to review the PKGBUILD before
building. You can run `pkgbuild-check` there, or add it as a pre-build hook.

Example with `yay` — use the `--editmenu` flag to open the PKGBUILD in your
editor before each build, where you can exit and run `pkgbuild-check` manually:

```bash
yay -S --editmenu <package>
```

To make this the default (edit `~/.config/yay/config.json`):

```json
{
  "editmenu": true
}
```

## Limitations

This tool does **static pattern matching only**. It cannot:

- Detect obfuscation it hasn't seen before
- Follow indirect execution paths or variable expansion chains
- Sandbox or execute the build script
- Replace a careful manual read

It is a first-pass filter, not a guarantee. Always read the PKGBUILD yourself,
especially for packages with complex build logic or multiple sources.

## Contributing

Bug reports and new check ideas are welcome as issues. If you're adding a
detection pattern, please include a minimal PKGBUILD snippet that triggers it
and one that shouldn't.

## License

MIT — see [LICENSE](LICENSE).
