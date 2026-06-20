# qq

[![Go](https://github.com/JFryy/qq/actions/workflows/go.yml/badge.svg)](https://github.com/JFryy/qq/actions/workflows/go.yml)
[![Docker Build](https://github.com/JFryy/qq/actions/workflows/docker-image.yml/badge.svg)](https://github.com/JFryy/qq/actions/workflows/docker-image.yml)
[![Go Version](https://img.shields.io/github/go-mod/go-version/JFryy/qq)](https://golang.org/)
[![License](https://img.shields.io/github/license/JFryy/qq)](https://github.com/JFryy/qq/blob/main/LICENSE)
[![Latest Release](https://img.shields.io/github/v/release/JFryy/qq)](https://github.com/JFryy/qq/releases)
[![Docker Pulls](https://img.shields.io/docker/pulls/jfryy/qq)](https://hub.docker.com/r/jfryy/qq)

`qq` is a multi-format transcoder and query tool powered by `jq` syntax. It lets you query and convert between configuration and data formats without needing separate tools for each one. Single binary, no persistent state — reads stdin or files, writes to stdout.

## Supported Formats

**Read/write:** `.json`, `.yaml`/`.yml`, `.toml`, `.xml`, `.hcl`/`.tf`, `.csv`, `.tsv`, `.ini`, `.gron`, `.html`, `.jsonl`/`.ndjson`/`.jsonlines`, `.jsonc`, `.parquet`, `.msgpack`/`.mpk`, `.cbor`, `.avro`, `.base64`/`.b64`, `.txt`/`.text`, `.line`, `.env`, `.properties`

**Read-only:** `.proto` (output uses JSON marshaler)

Not all format pairs round-trip freely. Some formats have structural constraints:
- **csv/tsv** — tabular; only interoperable with each other
- **parquet** — only interoperable with itself
- **avro** — requires array-of-records input
- **env/properties** — flat `map[string]string` only
- **toml/ini** — require a top-level map (no arrays at root)

## Installation

**Homebrew:**
```sh
brew install jfryy/tap/qq
```

**AUR (Arch Linux):**
```sh
yay qq-git
```

**From source** (requires Go 1.25+):
```sh
make install
```

**Download binaries:** [GitHub Releases](https://github.com/JFryy/qq/releases)

**Docker:**
```sh
docker pull jfryy/qq
echo '{"foo":"bar"}' | docker run -i jfryy/qq '.foo = "bazz"' -o tf
```

## CLI Usage

```
qq [expression] [file] [flags]
cat [file] | qq [expression] [flags]
qq -I file
```

When a file argument is provided, the input format is inferred from the extension. When reading from stdin, use `-i` to specify the input format (defaults to JSON).

### Flags

| Flag | Short | Purpose | Default |
| ---- | ----- | ------- | ------- |
| `--input` | `-i` | Input format (required for stdin) | `json` |
| `--output` | `-o` | Output format | `json` |
| `--raw-output` | `-r` | Output strings without escapes and quotes | `false` |
| `--interactive` | `-I` | Interactive query builder with autocomplete | `false` |
| `--stream` | | Emit path-value pairs (supports: json, jsonl, yaml, csv, tsv, line, txt) | `false` |
| `--slurp` | `-s` | Read all inputs into an array | `false` |
| `--exit-status` | `-e` | Set exit code based on output value | `false` |
| `--monochrome-output` | `-M` | Disable colored output | `false` |
| `--version` | `-v` | Print version | |
| `--help` | `-h` | Print help | |

Flag combinations `--stream`+`--interactive`, `--slurp`+`--stream`, and `--slurp`+`--interactive` are mutually exclusive and will error.

### Exit Codes

| Code | Meaning |
| ---- | ------- |
| `0` | Success |
| `1` | Error, or with `-e`: last output value is `false` or `null` |
| `4` | With `-e`: no output produced |

### Examples

**Transcoding between formats:**
```sh
# YAML → JSON
qq '.' config.yaml -o json

# TOML → YAML
qq '.' pyproject.toml -o yaml

# CSV → Parquet
qq '.' data.csv -o parquet

# XML → TOML
qq '.root' config.xml -o toml

# Terraform HCL → JSON
qq '.' infra.tf -o json

# Pipe with explicit input format
cat file.msgpack | qq -i msgpack -o yaml
```

**Querying data:**
```sh
# JSON is default input/output
cat file.json | qq '.name'

# File extension is auto-detected
qq '.servers[0].host' config.yaml

# Filter and transform
qq '[.[] | select(.age > 30)]' users.json

# Use gron for grep-friendly output
qq file.xml -o gron | grep -vE "sweet.potatoes" | qq -i gron
```

**Streaming mode** — process large files with constant memory:
```sh
# Emit path-value pairs
qq --stream '.' large.json

# Filter stream elements
qq --stream 'select(length == 2)' large.json
```

**Slurp mode** — combine multiple values into an array:
```sh
echo -e '{"id":1}\n{"id":2}' | qq -s 'map(.id)'
```

**Exit status** — use in shell conditionals:
```sh
echo '{"active":true}' | qq -e '.active' && echo "is active"
```

**Interactive mode** — query builder with autocomplete and live preview:
```sh
qq . file.json --interactive
```

![Demo GIF](docs/demo.gif)

**Fetch and query HTML:**
```sh
curl example.com | qq -i html '.html.body.ul.li[0]'
```

## Git Diff Integration

Use `qq` for human-readable diffs of configuration files. Add to your `.git/config`:

```
[diff "csv"]
  textconv = "f(){ qq --monochrome-output --output gron --input csv  \"$1\" 2>/dev/null | sort || cat \"$1\"; }; f"
[diff "env"]
  textconv = "f(){ qq --monochrome-output --output gron --input env  \"$1\" 2>/dev/null | sort || cat \"$1\"; }; f"
[diff "html"]
  textconv = "f(){ qq --monochrome-output --output gron --input html \"$1\" 2>/dev/null | sort || cat \"$1\"; }; f"
[diff "ini"]
  textconv = "f(){ qq --monochrome-output --output gron --input ini  \"$1\" 2>/dev/null | sort || cat \"$1\"; }; f"
[diff "toml"]
  textconv = "f(){ qq --monochrome-output --output gron --input toml \"$1\" 2>/dev/null | sort || cat \"$1\"; }; f"
```

And to `.gitattributes`:

```
*.csv diff=csv
*.env diff=env
*.html diff=html
*.ini diff=ini
*.toml diff=toml
```

## Repository Map

| Path | Purpose |
| ---- | ------- |
| `main.go` | Entrypoint; SIGPIPE handler, cobra command execution |
| `cli/` | Cobra command, flag parsing, query/streaming/slurp execution |
| `codec/` | Format registry and one subpackage per format — see [codec/AGENTS.md](codec/AGENTS.md) |
| `codec/codec.go` | `EncodingType` enum, `Codecs` map, `Unmarshal`/`Marshal` dispatch |
| `codec/stdout.go` | Terminal pretty-printing and chroma syntax highlighting |
| `codec/stream.go` | Streaming parser (channel-based, path-value pairs) |
| `internal/tui/` | Interactive REPL (`-I`/`--interactive`) built on bubbletea |
| `tests/` | `test.sh` (conversion matrix + behavior tests) and `test.<ext>` fixtures |
| `docs/` | Demo GIF and VHS tape for recording it |
| `.github/workflows/` | CI: `go.yml` (build+test), `docker-image.yml` (Docker push), `build.yml` (release binaries) |
| `Makefile` | Build, test, install, clean, Docker push targets |

## Tech Stack

- **Go 1.25** (module `github.com/JFryy/qq`)
- **Query engine:** [gojq](https://github.com/itchyny/gojq) (pure-Go jq implementation)
- **CLI:** [cobra](https://github.com/spf13/cobra)
- **Interactive TUI:** [bubbletea](https://github.com/charmbracelet/bubbletea) + bubbles + lipgloss
- **Highlighting:** [chroma](https://github.com/alecthomas/chroma)
- **Format libraries:** goccy/go-json, go.yaml.in/yaml/v4, BurntSushi/toml, hashicorp/hcl + hcl2json, clbanning/mxj (XML), apache/arrow (parquet), hamba/avro, fxamacker/cbor, vmihailenco/msgpack, gopkg.in/ini.v1

## Development

### Prerequisites

- Go 1.25+ toolchain
- `jq` on `PATH` (required for `tests/test.sh`)

### Common Commands

| Task | Command | Notes |
| ---- | ------- | ----- |
| Build | `make build` | Produces `bin/qq` |
| Run without building | `go run . '.' file.yaml -o json` | |
| Full test suite | `make test` | Builds first, runs shell tests then Go tests |
| Go unit tests only | `go test ./... -v -cover` | |
| Single Go test | `go test ./cli -run TestVersionFlag -v` | |
| Shell tests only | `./tests/test.sh` | Requires `bin/qq` built first |
| Format code | `gofmt -w .` | |
| Check formatting | `gofmt -l .` | Must be empty before committing |
| Install locally | `make install` | Builds, tests, copies to `~/.local/bin` |
| Clean | `make clean` | Removes `bin/qq`, coverage, test cache |
| Docker multi-arch push | `make docker-push` | Pushes `jfryy/qq:latest` via buildx |

### Testing

Tests live in two layers:

1. **Go unit tests** (`*_test.go` beside source) — codec round-trip tests, CLI flag tests, streaming/slurp tests
2. **Shell test matrix** (`tests/test.sh`) — converts every format to every other format, tests jq query functionality, streaming, slurp, and exit-status behavior

The shell matrix is the primary regression safety net. Always run `make test` (not just `go test`) before submitting changes.

### Making Changes

- Run `gofmt -w .` before committing; `gofmt -l .` must produce no output.
- When adding a format, follow the checklist in [codec/AGENTS.md](codec/AGENTS.md#adding-a-format).
- Update this README's supported-formats list when adding a format.
- Bump the version string in `cli/qq.go` on release.

### Build and Release

- **CI** runs on push/PR to `main` and `develop` branches (`go.yml`): builds, runs `go test`, runs `make test`.
- **Docker images** are built and pushed on push to `main`/`develop` (`docker-image.yml`).
- **Release binaries** (linux/darwin/windows × amd64/arm64) are built when a GitHub Release is created (`build.yml`).
- The Dockerfile uses `CGO_ENABLED=1` with static linking for arrow/parquet support; local `make build` does not set CGO.

## Background

`qq` is inspired by `fq` and `jq`. `jq` is a powerful and succinct query tool, sometimes I would find myself needing to use another bespoke tool for another format than json, whether its something dedicated with json query built in or a simple converter from one configuration format to json to pipe into jq. `qq` aims to be a handly utility on the terminal or in shell scripts that can be used for most interaction with structured formats in the terminal. It can transcode configuration formats interchangeably between one-another with the power of `jq` and it has an `an interactive repl (with automcomplete)` to boot so you can have an interactive experience when building queries optionally. Many thanks to the authors of the libraries used in this project, especially `jq`, `gojq`, `gron` and `fq` for direct usage and/or inspiration for the project.

## Contributions

All contributions are welcome to `qq`, especially for upkeep/optimization/addition of new encodings.

## Thanks and Acknowledgements / Related Projects

This tool would not be possible without the following projects, this project is arguably more of a composition of these projects than a truly original work, with glue code, some dedicated encoders/decoders, and the interactive mode being original work.
Nevertheless, I hope this project can be useful to others, and I hope to contribute back to the community with this project.

* [gojq](https://github.com/itchyny/gojq): `gojq` is a pure Go implementation of jq. It is used to power the query engine of qq.
* [fq](https://github.com/wader/fq) : fq is a `jq` like tool for querying a wide array of binary formats.
* [jq](https://github.com/jqlang/jq): `jq` is a lightweight and flexible command-line JSON processor.
* [gron](https://github.com/tomnomnom/gron): gron transforms JSON into discrete assignments that are easy to grep.
* [yq](https://github.com/mikefarah/yq): yq is a lightweight and flexible command-line YAML (and much more) processor.
* [goccy](https://github.com/goccy/go-json): goccy has quite a few encoders and decoders for various formats, and is used in the project for some encodings.
* [go-toml](https://github.com/BurntSushi/toml): go-toml is a TOML parser for Golang with reflection.
