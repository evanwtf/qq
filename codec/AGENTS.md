# codec

## Overview
The format subsystem. `codec.go` is the central registry mapping each
`EncodingType` to its `Unmarshal`/`Marshal` funcs and file extensions; every
other file under `codec/<name>/` implements one format's `Codec`.

## Key Files
- `codec.go` — `EncodingType` enum, `String()` name array, `Codecs` map, and the
  top-level `Unmarshal`/`Marshal`/`GetEncodingType`/`IsBinaryFormat` helpers.
- `stdout.go` — `PrettyFormat` + chroma syntax highlighting (`tokenColorMap`).
- `stream.go` — `StreamParser`, the channel-based streaming decoder.
- `<name>/<name>.go` — per-format codec (package name = format, e.g. `package yaml`;
  the proto package is named `package codec` but imported with the `proto` alias).

## Conventions
- A `Codec` is a struct with pointer-receiver methods
  `Unmarshal(input []byte, v any) error` and/or `Marshal(v any) ([]byte, error)`.
- Decode to / encode from JSON-shaped Go values (`map[string]any`, `[]any`,
  scalars). The common pattern is to convert through `goccy/go-json` rather than
  building bespoke trees.
- A format may supply only one direction; the registry can mix codecs (e.g. HTML
  decodes via `html.Codec` but marshals via `xmlCodec`; proto/line/txt marshal via
  the JSON codec).
- Marshal output is trimmed of trailing whitespace (see `json/json.go`).

## Adding a Format
1. Create `codec/<name>/<name>.go` with a `Codec` implementing the needed methods.
2. In `codec.go`: add an `EncodingType` const **at the end** of the iota block,
   append its canonical name to the `String()` array in the **same position**, add
   a package-level `<name>Codec` var, and add a `Codecs` map entry with extensions.
3. If it's a binary format, add it to `IsBinaryFormat` and the binary-handling
   branches in `tests/test.sh`.
4. Add a `tests/test.<ext>` fixture. If the format can't round-trip with others,
   add skip rules to `should_skip_conversion` in `tests/test.sh`.
5. For highlighting, ensure `lexers.Get(fileType.String())` resolves, or add the
   format to the json-lexer fallback list in `stdout.go`.

## Commands
```sh
go test ./codec/... -v -cover            # all codec unit tests
go test ./codec/json -run TestX -v       # single codec test
```

## Guardrails
### Always
- Keep iota order, `String()` array order, and `Codecs` keys consistent — index
  math in `String()` depends on it.
### Never
- Don't reorder or insert into the middle of the `EncodingType` enum; append only.
- Don't make `Marshal` emit a trailing newline that `PrettyFormat` doesn't expect.

## Agent Notes
Symlinked to `CLAUDE.md` and `GEMINI.md`; keep guidance tool-neutral. Cross-cutting
build/test/release rules live in the root `AGENTS.md` — don't duplicate them here.
