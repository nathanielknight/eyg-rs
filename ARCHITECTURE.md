# eyg-rs Architecture

Guidance for agents working on this repository. Read this before writing code.
When this document and the reference interpreter disagree, the reference
interpreter wins. Record the disagreement in an issue note.

## Goals

- A conforming EYG interpreter in Rust, suitable for **embedding**, including
  hosts that define their own effects.
- **Performance is a goal from the start.** Don't defer the decisions in
  "Performance" below.
- Behaviour matches the reference interpreter (`eyg_interpreter`, driven by the
  pinned `eyg` CLI), **including its quirks**. Don't "fix" semantics toward what
  is natural in Rust.

## Source of truth, in order

1. The pinned oracle (`eyg eval -`).
2. The JSON spec suites (`spec/evaluation/*.json`, `ir_suite.json`).
3. This document.

## Workspace layout

```
crates/
  eyg-ir/           AST types + JSON decoder. No evaluation.
  eyg-interpreter/  Lowering, machine, values, builtins, embedding API. Pure: no I/O.
  eyg-cli/          `eval -`, standard effect implementations, process concerns.
```

- `eyg-interpreter` must not depend on `eyg-cli`, do I/O, read env vars or
  spawn threads.
- The CLI implements standard effects **only through the public embedding API**.
  If an effect can't be written that way, fix the API, not the CLI.
- `#![forbid(unsafe_code)]` in every crate.

## IR and decoding (`eyg-ir`)

Nodes are JSON objects tagged by key `"0"`. Tags confirmed from the spec suites:

| Tag  | Node            | Fields                           |
|------|-----------------|----------------------------------|
| `v`  | variable        | `l` name                         |
| `f`  | lambda          | `l` param, `b` body              |
| `a`  | apply           | `f` function, `a` argument       |
| `l`  | let             | `l` name, `v` value, `t` then    |
| `i`  | integer         | `v`                              |
| `s`  | string          | `v`                              |
| `b`  | builtin         | `l` name                         |
| `ta` | list tail (nil) |                                  |
| `c`  | cons            |                                  |
| `u`  | empty record    |                                  |
| `t`  | tag             | `l` label                        |
| `m`  | match case      | `l` label                        |
| `p`  | perform         | `l` effect label                 |
| `h`  | handle          | `l` effect label                 |

Remaining tags (record select/extend/overwrite, no-cases, binary, vacant,
reference) must be confirmed from `ir_suite.json` or the oracle before they are
implemented. Don't guess at them.

Requirements:

- **No recursion limit failures.** IR trees can be arbitrarily deep. Decode
  iteratively, or with `serde_json` plus its recursion limit disabled and
  `serde_stacker`. Include a test with a 100,000-deep expression.
- **Iterative `Drop`** for any deep tree or linked structure (AST, environments,
  continuations, lists), so that dropping doesn't overflow the stack.
- Unknown tags produce a decode error that names the tag. Never panic.
- Reference nodes must **decode** successfully even though resolution is
  deferred (see "References").

## Primitives are curried values, not syntax

`perform`, `handle`, `tag`, `match`, `cons`, the record operations and builtins
are **function values** applied one argument at a time:

- `perform(Label)(arg)`
- `tag(Label)(value)`
- `match(Label)(branch)(otherwise)(value)`
- `handle(Label)(handler)(exec)`, where `handler` is `value -> resume -> result`
  and `exec` is a thunk

The machine must therefore represent **partially applied primitives** cheaply.
The lowering pass should detect saturated applications and emit direct
operations (see "Performance").

## Lowering

Raw IR is never evaluated. A lowering pass converts `eyg-ir` trees into an
internal representation:

- Variables become slot indices (de Bruijn-style), not name lookups.
- Labels (record fields, tags, effect names) and builtin names are interned to
  `u32` symbols. A symbol table travels with the lowered program so values can
  be rendered back to names.
- Saturated primitive and builtin applications are fused into single nodes.
- Unbound variables are found at lowering time but **reported at evaluation
  time**, with whatever error the oracle produces. Check what the oracle does
  before choosing.

## Machine

- **CEK-style, with the continuation as heap data.** Rust recursion depth must
  not grow with EYG evaluation depth.
- **Multi-shot continuations are required.** A handler may call `resume` more
  than once (spec case "multiple resumption"). Captured continuations must be
  cheap to clone: use a persistent stack, such as an `Rc`-linked frame list.
  Don't capture a continuation by copying a `Vec`.
- **Deep handlers.** A handler stays installed for the computation after
  `resume` (spec case "handler will be applied after resumption").
- **Tail calls** must run in constant continuation space.

## Values

- A compact `Value` enum. Immediates (integers) are inline. Heap types
  (strings, binaries, lists, records, tagged values, closures, partial
  applications, continuations) sit behind `Rc`.
- Records are keyed by interned symbols. Start with a sorted small vector and
  move to a persistent map only if benchmarks justify it.
- Lists are persistent cons lists, so `cons` and `list_pop` are O(1).
- Provide an encoder from `Value` to the spec's JSON value encoding
  (`{"integer": n}`, `{"string": s}`, `{"tagged": {"label", "value"}}`,
  `{"record": {...}}`, `{"list": [...]}`, and so on). The conformance harness
  compares in this encoding, never by pretty-printed text.

## Semantics to match (oracle quirks)

The oracle is Gleam compiled to JavaScript. Verify each of these against the
oracle before you implement it, and add a test for each:

- **Integers.** Gleam `Int` on the JS target is a JS number. Match the oracle
  around ±2^53, for overflow, for `int_divide` with negative operands, and for
  `int_to_string`/`int_parse` of large values. Use a differential test; don't
  reason from first principles.
- **Strings.** `string_length` counts **graphemes**. Use
  `unicode-segmentation`. Check case conversion of non-ASCII text and the
  empty-separator behaviour of `string_split` against the spec.
- **Equality** (`equal`) on closures, continuations and partial applications:
  match the oracle.
- **Error kinds.** Break and error variants mirror the spec's `break`
  encodings (e.g. `{"UndefinedBuiltin": "name"}`).

## Embedding API (`eyg-interpreter`)

The primitive is **break/resume**:

```rust
let mut m = Machine::new(&program);
match m.run() {
    Ok(value) => ...,
    Err(Break::Effect { label, lift }) => m.resume(reply),
    Err(Break::UnresolvedReference(cid)) => ...,
    Err(Break::Error(e)) => ...,
}
```

- An unhandled `perform` pauses the machine and surfaces `label` and `lift`. The
  host replies with `resume(value)`. This is exactly the shape of the spec's
  `effects` cases, which list `{label, lift, reply}` in order, so the harness
  can drive them directly.
- On top of that, provide a convenience runner that takes a map from label to
  `impl Handler` (`fn handle(&mut self, lift: Value) -> Result<Value, HostError>`)
  and loops until it gets a value or an unhandled break.
- Break/resume keeps async hosts, sandboxed hosts and step debugging possible
  without changing the API. Don't add async to the library.

## References

The resolver is deferred. The interpreter evaluates self-contained
programs only.

- Reference nodes decode normally.
- Evaluating one returns `Break::UnresolvedReference(..)`.
- Hashing and CIDs are a later milestone in `eyg-ir`. Don't start them without
  an issue.

## Performance

- The lowering pass, interned symbols, slot environments and fused
  applications are required from the first working version.
- Benchmarks use `criterion` with a small set of representative programs:
  deep `fix` recursion, record-heavy code, string building, and effect-heavy
  handler loops. CI records results but doesn't gate on them yet.
- No global allocator in the library. An allocator choice belongs in the CLI
  only.
- Don't trade conformance for speed. A faster wrong answer is a failing test.

## Testing

- The conformance harness reads the spec suites: `source` → run (feeding
  `effects` replies in order) → compare the result with `value` or `break` in
  the spec JSON encoding.
- Hidden tests exist, generated from the oracle. Passing the visible suites is
  necessary but not sufficient. Implement the semantics; don't special-case
  test inputs.
- You may add and edit your own unit tests freely. You may not edit the
  conformance harness, spec files, toolchain files or CI configuration.
- Never add `#[ignore]`. Never gate tests behind `cfg` tricks.

## Dependencies

These are approved: `serde`, `serde_json`, `serde_stacker`, `thiserror`,
`unicode-segmentation`, and, for dev, `criterion` and `proptest`. Any other
dependency change goes to human review. Say why you need it in the MR.

## Deferred, not in scope without an issue

Reference resolution and the module system, DAG-JSON hashing and CIDs, type
inference, `touch_grass` effect typing, compilation to other targets, step
budgets ("fuel") for interrupting long-running programs.

