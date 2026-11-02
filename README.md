<div align="center">

# revanity-go

**Vanity address generator for [Reticulum](https://reticulum.network) and LXMF networks.**

[![Go 1.20+](https://img.shields.io/badge/go-1.20+-00ADD8.svg)](https://go.dev)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status: Stable](https://img.shields.io/badge/status-stable-brightgreen.svg)](https://github.com/ratspeak/revanity-go)

</div>

---

Searches in parallel for a Reticulum destination hash matching your pattern. X25519 public-key computation uses libsodium through cgo. The default destination type is `lxmf.delivery`; the resulting LXMF address differs from the identity hash.

## Install

Requires [libsodium](https://doc.libsodium.org/installation), Go 1.20 or newer, Git and a C compiler. On macOS, install the Xcode command-line tools; on Debian/Ubuntu, install `build-essential` along with libsodium.
### Step 1
#### macOS
```bash
brew install libsodium
```
#### Linux
```bash
sudo apt install build-essential libsodium-dev
```
### Step 2
```bash
git clone https://github.com/ratspeak/revanity-go.git
cd revanity-go
CGO_ENABLED=1 go build -o revanity-go
```

The source includes Homebrew paths for Apple Silicon. If libsodium is elsewhere, provide its include and library paths through `CGO_CFLAGS` and `CGO_LDFLAGS`. A successful build is separate from measuring search speed on that machine.

## Usage

Choose one of `-prefix`, `-suffix`, `-contains` or `-regex`. The first three accept hexadecimal text; use `-dry-run` to check the pattern without starting a search.

```bash
# Find address starting with "dead"
./revanity-go -prefix dead

# Other match modes
./revanity-go -suffix cafe
./revanity-go -contains beef
./revanity-go -regex "^(dead|beef)"

# Options
./revanity-go -prefix dead -workers 8                # set worker count
./revanity-go -prefix dead -dest nomadnetwork.node   # different dest type
./revanity-go -prefix deadbeef -dry-run              # estimate difficulty only
./revanity-go -prefix dead -quiet                    # output address only
```

## Multi-Pattern Search

Search for multiple patterns with comma-separated values. Commas also split `-regex` patterns, so a regex containing a comma cannot be passed as one pattern. Without `-loop`, the search stops after each pattern has produced a match. A result is assigned to the first matching pattern, so use distinct, non-overlapping patterns: an earlier broad pattern can prevent a later one from ever being counted. Repeated matches for a pattern are saved by default.

```bash
./revanity-go -prefix aa,bb,cc
./revanity-go -prefix dead,beef,cafe,babe       # find all four
./revanity-go -prefix dead,beef,cafe -no-dupe   # exactly one result per pattern
```

`-no-dupe` saves only the first match assigned to each pattern in that run. It does not remove results from earlier runs. Check any reported file-write errors before assuming all results were saved.

## Loop Mode

Search continuously, collecting every match instead of stopping at the first one.

```bash
./revanity-go -prefix dead -loop
./revanity-go -prefix dead -loop -output my_results   # custom output directory
```

Both multi-pattern and loop mode save results to `results/` (or the directory specified by `-output`):

- `<mode>_<patterns>.jsonl` — one JSON object per line with dest hash, identity hash, matched pattern, private key (hex/base32/base64), all destination hashes, and metadata. Append-safe across runs.
- `<dest_hash>.identity` — individual 64-byte raw Reticulum identity keys. Application import procedures vary.

## Output

With one pattern and no `-loop`, two files are saved on match:

- `<address>.identity` — 64-byte raw Reticulum identity key
- `<address>.txt` — human-readable info with private key formats and import instructions

`-output` is a file prefix in single-result mode and a directory in multi-pattern or loop mode. The `.identity`, `.txt` and `.jsonl` outputs contain private keys in plaintext. Keep them private and back them up; an address alone cannot restore an identity. Stop the receiving application and preserve its existing identity before following its supported import procedure.

## How It Works

Each worker creates an Ed25519 signing key, then varies the X25519 key for each attempt. It hashes the combined public keys to obtain an identity hash, then hashes that with the destination name hash. Both hashes are truncated to 16 bytes. In multi-pattern and loop mode, the worker starts with fresh key material after each match.

For one exact prefix or suffix of `n` hexadecimal characters, the expected work is `16^n` attempts. Each extra character multiplies that expectation by 16. Contains and regular-expression searches have different probabilities.

| Prefix or suffix length | Expected attempts |
| --- | ---: |
| 4 characters | 65,536 |
| 6 characters | 16,777,216 |
| 8 characters | 4,294,967,296 |

Search time depends on measured throughput and chance. Divide the expected attempts by the displayed checks per second for a rough expectation; it is not a deadline or guarantee. The built-in difficulty label is a rough heuristic. For long patterns, use `-quiet` to skip the built-in estimate: its attempt counter can overflow and its formatting can fail before the search starts. Calculate `16^n` separately for prefixes and suffixes.

## License

MIT
