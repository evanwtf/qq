# qq

## Overview
`qq` is a single-binary CLI that transcodes between structured data formats and
queries them with `jq` syntax (via the pure-Go `gojq`). Every supported format
is decoded into a JSON-compatible Go value (`map[string]any` / `[]any` / scalars),
the query runs against that, and the result is re-encoded to the requested output
format. There is no server or persistent state; it is a stdin/stdout/file tool.

## Tech Stack
- **Go 1.25** (module `github.com/JFryy/qq`; `go.mod` declares `go 1.25.0`).
- **Query engine:** `github.com/itchyny/gojq`.
- **CLI:** `github.com/spf13/cobra`.
- **Interactive TUI:** `charmbracelet/bubbletea` + `bubbles` + `lipgloss`.
- **Highlighting:** `github.com/alecthomas/chroma`.
- **Format libs:** `goccy/go-json`, `go.yaml.in/yaml/v4`, `BurntSushi/toml`,
  `hashicorp/hcl` + `tmccombs/hcl2json`, `clbanning/mxj` (XML), `apache/arrow`
  (parquet), `hamba/avro`, `fxamacker/cbor`, `vmihailenco/msgpack`, `gopkg.in/ini.v1`.
- Distributed as static binaries (release workflow) and a distroless Docker image.

## Key Concepts & Terminology
- **Codec** — a per-format `Codec` struct exposing `Unmarshal([]byte, any) error`
  and/or `Marshal(any) ([]byte, error)`. Lives in `codec/<name>/`.
- **EncodingType** — the int enum in `codec/codec.go` identifying each format; its
  iota order, the `String()` name array, and the `Codecs` map must stay in sync.
- **Binary formats** — parquet, msgpack, cbor, avro. `codec.IsBinaryFormat` gates
  raw-byte stdout writes; tests pass these as files, not piped text.
- **Read-only formats** — `.proto` decodes but cannot marshal back to proto
  format (output uses the JSON marshaler). `.line`/`.txt` also marshal via JSON.
- **Structurally constrained formats** — `.env` and `.properties` marshal only
  flat `map[string]string` structures; `.csv`/`.tsv` are tabular-only;
  `.parquet` and `.avro` only interoperate with specific formats (see
  `should_skip_conversion` in `tests/test.sh`).
- **Stream mode** (`--stream`) emits jq-compatible path-value pairs; output is
  always compact JSON regardless of `-o`. Supported inputs: json, jsonl, yaml,
  csv, tsv, line, txt.
- **Slurp mode** (`-s`) reads multiple inputs into a single array.

## Environment & Dependencies
- Go toolchain (build pulls modules from the network on first run; no vendor dir).
- `jq` must be on `PATH` to run `tests/test.sh` (it's a prerequisite check + oracle).
- `bin/qq` must be built before `tests/test.sh` runs (`make test` builds first).
- Docker multi-arch builds require `buildx` + QEMU.

## Commands
```sh
make build                       # go build -o bin/qq .
go build -o bin/qq .             # equivalent direct build
make test                        # builds, runs ./tests/test.sh, then go test ./... -v -cover
go test ./... -v -cover          # Go unit tests only
go test ./cli -run TestVersionFlag -v   # single Go test
./tests/test.sh                  # conversion matrix + behaviour tests (needs bin/qq + jq)
gofmt -l .                       # list unformatted files (must be empty)
gofmt -w .                       # format
make install                     # build + test, then copy to ~/.local/bin
make clean                       # remove bin/qq, coverage, test cache
make docker-push                 # buildx multi-arch push to jfryy/qq:latest
go run . '.' tests/test.yaml -o json    # run without installing
```
No separate linter is configured; `gofmt` is the only enforced formatter and all
files are currently gofmt-clean.

## Project Layout
- `main.go` — entrypoint; installs the SIGPIPE→exit-0 handler, runs the cobra cmd.
- `cli/` — cobra command, flag parsing, input/file resolution, and the query,
  streaming, and slurp execution paths. **Version string lives here**
  (`cli/qq.go`, `v := "v0.3.x"`).
- `codec/` — the format registry (`codec.go`), pretty-printer/highlighter
  (`stdout.go`), streaming parser (`stream.go`), and one subpackage per format.
  See `codec/AGENTS.md`.
- `internal/tui/` — interactive REPL (`-I`/`--interactive`) built on bubbletea.
- `tests/` — `test.sh` plus one `test.<ext>` fixture per format.
- `docs/` — demo assets. `.github/workflows/` — CI (`go.yml`, `docker-image.yml`,
  `build.yml` release).

## Code Style & Patterns
- Internal data is always JSON-shaped Go values; new codecs convert to/from that,
  often by delegating to `goccy/go-json` (see `codec/proto/proto.go`).
- Codec methods use pointer receivers (`func (c *Codec) ...`); registry vars are
  plain `Codec{}` values.
- Errors are wrapped with `fmt.Errorf("...: %v", err)` and returned up; the CLI
  layer prints to stdout/stderr and calls `os.Exit` with meaningful codes
  (0 ok, 1 error, and `-e` exit-status: 1 = false/null, 4 = no output).
- Mutually-exclusive flag combos are rejected explicitly in `handleCommand`
  (`--stream`+`--interactive`, `--slurp`+`--stream`, `--slurp`+`--interactive`).
- Tests are table/round-trip style; Go tests sit beside code (`*_test.go`),
  end-to-end coverage is the shell conversion matrix.

## Making Changes
- Make minimal, focused changes; preserve the codec-registry architecture.
- Don't add new dependencies without justification — the dependency surface is the
  point of the tool, but each addition widens the binary and CVE surface.
- When you add or change a format, add/update its `tests/test.<ext>` fixture and,
  if the conversion is structurally restricted, the skip rules in `tests/test.sh`.
- Update `README.md`'s supported-formats list when adding a format.
- On release, bump the hardcoded version in `cli/qq.go`; releases are cut by
  creating a GitHub Release (triggers `build.yml`).

## Guardrails
### Always
- Run `gofmt -w .` before committing; keep `gofmt -l .` empty.
- Run `make test` (not just `go test`) — the shell matrix catches transcoding
  regressions the Go tests don't.
- Keep the three structures in `codec/codec.go` in sync: the `EncodingType` iota
  block, the `String()` name array (same order), and the `Codecs` map.

### Never
- Don't break the SIGPIPE handling in `main.go` — it's deliberate so piped use
  under `pipefail` exits 0.
- Don't make `--stream` honor `-o`; stream output is intentionally compact JSON.
- Don't assume every format round-trips: tabular (csv/tsv), parquet, avro, env,
  properties, toml, and ini have structural input/output constraints encoded in
  `tests/test.sh`'s `should_skip_conversion`.

### Use Extra Caution
- `go.mod` / `go.sum` — change only via `go get`/`go mod tidy`.
- `Dockerfile` builds with `CGO_ENABLED=1` and static linking (arrow/parquet);
  the local `make build` does not set CGO — keep both paths working.
- `.github/workflows/` and release tooling.

## Agent Notes
This file is symlinked to `CLAUDE.md` and `GEMINI.md`; keep all instructions
tool-neutral. Component-specific guidance lives in `codec/AGENTS.md`.
