# Spec — `std.fs` (filesystem access over affine `substrate` handles)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-fs` (M-528, #169, Batch P5-B; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.fs` · Ring 2 (RFC-0016 §4.2) · Tier B (RFC-0016 §4.4) |
| **Tracks** | `M-528` (#169) — the Phase-5 task this spec delivers |
| **Scope** | Filesystem access — path resolution, open/read/write/append, `stat`/metadata, directory `list`, `create`/`remove`/`rename` — over **affine `substrate` handles** (a handle is consumed exactly once, LR-8), layered on the `io` byte source/sink surface (M-514). Every path/permission/IO failure is an explicit, traceable error (G2); there is **no silent create-on-write and no silent truncation**. |
| **Boundary** | Out of scope: the **byte source/sink + (de)serialization abstraction** itself — that is `io`/`serialize` (M-514); `fs` *builds on* it (it adds paths, handles, permissions, and directory structure *over* io's byte streams), and does **not** re-define byte transfer. Clocks / timestamps as values are `time` (M-529); `fs` carries an `mtime` only as opaque metadata it gets from the OS, it does not interpret it. The raw OS syscalls themselves are confined to an audited `wild` block (ADR-014), inventoried below (§4 / §6); whether that floor lives in `std` or a separate `std-sys` phylum is **FLAGGED** (§7 Q1, RFC-0016 §8-Q6). |
| **Depends on** | RFC-0016 §4.1 (the C1–C6 contract), §4.4 (the `fs` Tier-B row + honesty crux), §4.5 (the guarantee-matrix obligation), §8-Q6 (the `wild`/FFI floor / `std-sys` split); ADR-014 (the permitted-but-warned `unsafe`/`wild` policy — the audited OS-facility escape hatch); RFC-0001 (the value model — `Value`/`Repr`/`Meta`, the guarantee lattice §4.3, content-addressing §4.6); RFC-0006 LR-8 (the affine `substrate` `Resource` kind) / LR-9 (leaks structurally excluded; `wild` denied-by-default, inventoried); RFC-0013 (the structured diagnostic record); RFC-0014 (declared bounded effects). |
| **Grounds on** | `std.io` (**M-514**, #155 — *Draft — landed* in this same wave) — the byte source/sink + the affine `substrate` handle `fs` layers paths over (its exact signatures are M-514's to own; **FLAGGED**, §7 Q2). `std.core` (M-515) `Option`/`Result`/error values + the guarantee-lattice types. The `wild`-audit tooling that inventories every `wild` block (`crates/mycelium-sec`, `docs/spec/Security-Checks-Contract.md` §4 — LR-9/S6/ADR-014). KC-3: above the kernel — but see C5 below, where the audited `wild` floor **narrows** the "no new trusted code" claim honestly. |

---

## 1. Summary

`std.fs` is the filesystem surface — open a path to a `substrate` handle, read/write/append bytes through
it, `stat` metadata, `list` a directory, and `create`/`remove`/`rename` entries — layered on the `io`
byte-stream surface (M-514). Its **honesty crux** is C1/G2 at the OS boundary: **every** path, permission,
or IO failure (a missing path, a permission denial, a disk-full, a read of a directory-as-file) is an
explicit `Result::Err` carrying an RFC-0013 diagnostic record that names *the path* and *the errno-class
cause* — there is **no silent create-on-write** (opening for write does not conjure a file unless the
caller declared `create`), **no silent truncation** (an existing file is not zeroed unless the caller
declared `truncate`), and **no silent partial write** (a short write is surfaced, never swallowed). Its
second crux is LR-8: a `substrate` handle is **affine** — consumed exactly once; a use-after-consume is an
explicit error, not undefined behaviour. Ring 2, Tier B. Unlike the pure commons, `fs` cannot be wholly
`wild`-free: the OS syscalls bottom out in an **audited `wild` block** (ADR-014), so its C5 "no new trusted
code" claim is *narrowed* (§5 C5) — that surface is inventoried (§4 / §6) and may be quarantined in a
separate `std-sys` phylum so the rest of `std` stays leak-free by construction (LR-9; **FLAGGED §7 Q1**).

## 2. Scope & module boundary

- **In scope:** path values and resolution (`Path`, `join`, `parent`, `exists`); opening a path to an
  affine `substrate` handle under an explicit `OpenOptions` (read / write / append / `create` /
  `create_new` / `truncate` — each a *declared* intent, never a default); reading + writing + flushing
  bytes through that handle (delegating the byte transfer to the `io` surface); `stat`/`metadata`
  (size, kind, permissions, timestamps as opaque metadata); directory `list`/`read_dir`; `create_dir`,
  `remove_file`/`remove_dir`, `rename`/`copy`; closing/consuming a handle. The `fs`-specific concern is
  *which file, with what permission, under what declared intent* — over io's *how bytes move*.
- **Out of scope (and who owns it):**
  - **Byte transfer + (de)serialization** — `std.io`/`serialize` (M-514). `fs` opens and names; `io`
    moves the bytes and owns the `substrate` handle's read/write/flush surface and its single-consumption
    enforcement. `fs` does not re-define `Read`/`Write` or a serialization format; it threads them.
  - **Time / clocks as values** — `std.time` (M-529). A file's `mtime`/`atime` is carried as opaque OS
    metadata; `fs` does not interpret it into a monotonic/wall distinction (that typed distinction is
    `time`'s, M-529).
  - **The OS-facility floor / FFI** — the raw syscalls (`open`/`read`/`write`/`stat`/`unlink`/…) are an
    audited `wild` block (ADR-014). Whether that floor is *inside* `std.fs`, shared with `io`, or hoisted
    into a separate `std-sys` phylum is the RFC-0016 §8-Q6 question — **FLAGGED §7 Q1**, not decided here.
  - **Random / entropy** (`/dev/urandom` as a source) — `std.rand` (M-531), which reifies nondeterminism
    (RT3); `fs` does not silently become an entropy source.
- **Ring & layering:** Ring 2 (RFC-0016 §4.2). `fs` is **new library code written to the §4.1 contract
  over Ring 0/1 + the io surface**: the path/permission/intent logic is safe Mycelium over the value
  model and over io's byte streams; **only** the OS syscalls are unsafe, and they are confined to an
  audited `wild` block (ADR-014) — inventoried (LR-9/S6), justified by an adjacent `// SAFETY:` (the
  `myc-sec` `wild`-audit, `docs/spec/Security-Checks-Contract.md` §4). This is the honest *narrowing* of
  the blanket "no new trusted code" KC-3 claim (see §5 C5).

## 3. Exported-op surface (design sketch)

A design sketch — enough to fix the surface and feed the guarantee matrix, not a committed grammar.
Value-semantic, immutable-by-default. A `Path` is a plain immutable value (content-addressable, ADR-003).
A `File`/`Dir` is an affine `substrate` handle — **consumed exactly once** (LR-8); the type system /
linearity hook makes a second use a compile-time or explicit runtime error, never UB. Every fallible op
returns `Result<_, FsErr>`; an effectful op declares `effects: io` on its signature (C6). The byte-moving
ops (`read`/`write`/`flush`) are **owned by `io` (M-514)** and shown here only to fix the seam — their
exact signatures are **not invented here** (FLAGGED §7 Q2).

```
// illustrative signatures (not a committed surface)

// --- paths: pure, total, value-semantic ---
Path::new(s: Text)            -> Path                 // Exact, total
join(p: Path, c: Text)        -> Path                 // Exact, total (lexical; no syscall)
parent(p: Path)               -> Option<Path>         // Exact, total (None at root)

// --- declared open intent: NEVER a silent default (C1) ---
//     every create/truncate is opt-in; absent the flag, the op refuses rather than mutating.
struct OpenOptions { read, write, append, create, create_new, truncate : bool }   // all default false

open(p: Path, o: OpenOptions) -> Result<File, FsErr> // effects: io
  // Err(NotFound)     when the path is absent AND create/create_new not set (NO silent create)
  // Err(AlreadyExists)when create_new set AND the path exists
  // Err(PermDenied)   when the OS denies access
  // a File is an AFFINE substrate handle (LR-8) — consumed exactly once

// --- byte transfer: OWNED BY io (M-514); shown only to fix the seam (FLAGGED §7 Q2) ---
read (f: &File, buf)          -> Result<usize, FsErr> // effects: io — io's Read; short read explicit
write(f: &File, buf)          -> Result<usize, FsErr> // effects: io — io's Write; short write explicit (NO silent partial)
flush(f: &File)               -> Result<(), FsErr>    // effects: io — surfaces a deferred write error
close(f: File)                -> Result<(), FsErr>    // effects: io — CONSUMES the handle (LR-8); a close error is surfaced, not dropped

// --- metadata: read-only; permissions/timestamps are OPAQUE (time interp is M-529) ---
stat(p: Path)                 -> Result<Metadata, FsErr> // effects: io — Err(NotFound|PermDenied)
exists(p: Path)               -> Result<bool, FsErr>     // effects: io — distinct from open; still fallible (PermDenied)

// --- directory + namespace mutation: each effect explicit + fallible ---
read_dir(p: Path)             -> Result<DirIter, FsErr>  // effects: io — Err(NotFound|NotADirectory|PermDenied)
create_dir(p: Path)           -> Result<(), FsErr>       // effects: io — Err(AlreadyExists|PermDenied)
remove_file(p: Path)          -> Result<(), FsErr>       // effects: io — Err(NotFound|PermDenied)
remove_dir(p: Path)           -> Result<(), FsErr>       // effects: io — Err(NotEmpty|NotFound|PermDenied)
rename(from: Path, to: Path)  -> Result<(), FsErr>       // effects: io — Err(NotFound|PermDenied|CrossDevice)
copy(from: Path, to: Path)    -> Result<u64,  FsErr>     // effects: io — Err on any leg; NO silent overwrite unless declared

// --- the explicit, traceable error set (RFC-0013 record: path + errno-class + why) ---
enum FsErr {
  NotFound, PermDenied, AlreadyExists, NotADirectory, IsADirectory,
  NotEmpty, DiskFull, CrossDevice, WouldBlock, Interrupted, Loop,
  UseAfterConsume,           // LR-8: a consumed substrate handle was used again
  Io(IoErr),                 // a lower-level io (M-514) failure, threaded not swallowed
  Os{ errno_class, raw },    // the audited wild-floor result, classified (NEVER a bare code)
}
```

> **Note (the never-silent defaults, C1/G2).** `OpenOptions` fields **all default `false`**: opening a
> non-existent path *without* `create` is `Err(NotFound)`, never a silent file creation; opening an
> existing file *without* `truncate` preserves its bytes, never a silent zero-out. The mutation is the
> caller's *declared* intent or it does not happen — this is the §4.4 `fs` honesty crux made structural.

## 4. Guarantee matrix (the load-bearing deliverable — RFC-0016 §4.5)

Rows = exported ops. To be encoded as a checked table (the RFC-0003 §4 template) and asserted in tests
once code lands — never prose only. **Tag legend:** `Exact` = exact result, no accuracy/approximation
semantics (`fs` carries *none* — it has no ε/δ; its honesty is C1 fallibility + C6 effects, not C2
precision). The **`wild`?** column marks which ops bottom out in the audited `wild`/syscall floor
(ADR-014 / §6 inventory).

| Op | Guarantee tag | Fallibility (explicit error set) | Declared effects | EXPLAIN-able? | `wild`? |
|---|---|---|---|---|---|
| `Path::new` / `join` / `parent` | `Exact` | total (`parent`: `Option`, `None` at root) | none | n/a | no (pure lexical) |
| `exists` | `Exact` | `Err(PermDenied)` | io | yes (diagnostic record) | yes (`stat`/`access`) |
| `open` | `Exact` | `Err(NotFound \| AlreadyExists \| PermDenied \| IsADirectory \| Loop)` | io | yes (the open-intent + refusal record) | yes (`open`/`openat`) |
| `read` *(io M-514)* | `Exact` | `Err(Io \| PermDenied \| Interrupted \| WouldBlock)`; short read explicit | io | yes (diagnostic record) | yes (`read`) |
| `write` *(io M-514)* | `Exact` | `Err(Io \| DiskFull \| PermDenied)`; short write explicit (no silent partial) | io | yes (diagnostic record) | yes (`write`) |
| `flush` *(io M-514)* | `Exact` | `Err(Io \| DiskFull)` | io | yes (diagnostic record) | yes (`fsync`/`write`) |
| `close` | `Exact` | `Err(Io \| DiskFull)`; **consumes the handle (LR-8)** | io | yes (diagnostic record) | yes (`close`) |
| `stat` / `metadata` | `Exact` | `Err(NotFound \| PermDenied)` | io | yes (diagnostic record) | yes (`stat`/`fstat`) |
| `read_dir` / `list` | `Exact` | `Err(NotFound \| NotADirectory \| PermDenied)` | io | yes (diagnostic record) | yes (`opendir`/`readdir`) |
| `create_dir` | `Exact` | `Err(AlreadyExists \| PermDenied \| NotFound)` | io | yes (diagnostic record) | yes (`mkdir`) |
| `remove_file` | `Exact` | `Err(NotFound \| PermDenied \| IsADirectory)` | io | yes (diagnostic record) | yes (`unlink`) |
| `remove_dir` | `Exact` | `Err(NotEmpty \| NotFound \| PermDenied)` | io | yes (diagnostic record) | yes (`rmdir`) |
| `rename` | `Exact` | `Err(NotFound \| PermDenied \| CrossDevice)` | io | yes (diagnostic record) | yes (`rename`/`renameat`) |
| `copy` | `Exact` | `Err(NotFound \| PermDenied \| DiskFull)`; no silent overwrite unless declared | io | yes (diagnostic record) | yes (`open`+`read`+`write`) |
| *(any op on a consumed handle)* | `Exact` | `Err(UseAfterConsume)` (LR-8) | none | yes (the affine-violation record) | no (caught above the floor) |

**Tag justification (VR-5 — downgrade rather than overclaim):**

- **Every `fs` op tags `Exact`.** `fs` carries **no** accuracy/approximation/probability semantics — there
  is no ε, no δ, no `BoundBasis` to resolve a strength against (contrast `math` M-525). "Exact" here means
  the operation has no *precision* dimension to be honest about; its honesty is borne entirely by C1
  (fallibility) and C6 (declared effects), **not** by a C2 precision tag. An op either performs its named
  filesystem effect and returns, or it returns an explicit `FsErr` — there is no third, silently-degraded
  outcome. The `Exact` tag is therefore the *honest* tag, not an upgrade: nothing about a filesystem
  result is approximated by `fs` itself. (The *underlying state* is the OS's; `fs` reports the OS's answer
  faithfully, it does not certify the filesystem.)
- **Fallibility is the load-bearing column, not the tag.** Each row's `FsErr` set is the never-silent
  guarantee (C1/G2): a missing path → `NotFound`, a permission denial → `PermDenied`, a full disk →
  `DiskFull`, a directory opened as a file → `IsADirectory`, a re-used consumed handle → `UseAfterConsume`
  (LR-8). **No row returns a sentinel, a silent clamp, a `0`-for-error, or a silent partial result.** A
  short read/write returns the *actual* count and the caller must reconcile it — the shortfall is never
  swallowed (RFC-0013 / I1 — recovered or re-propagated, never silently dropped).
- **`wild`? marks the trusted floor.** Every io-effecting row bottoms out in the audited `wild` syscall
  block (ADR-014); the pure-lexical path ops (`Path::new`/`join`/`parent`) and the affine-violation catch
  (`UseAfterConsume`) do **not**. This column **is** the `wild` inventory the acceptance criterion demands
  (§6); it narrows C5 honestly (§5 C5) rather than hiding the FFI floor.
- **EXPLAIN-able everywhere it can fail.** Every fallible row carries an RFC-0013 diagnostic record — the
  attempted path, the declared `OpenOptions` intent, the classified errno (`Os{errno_class}`, never a bare
  raw code), and a one-line *why* — so a failure is robust **and** legible (C3 / the maintainer's
  failure-semantics directive): the trace names the path **and** the cause, is actionable, and is
  recovered or re-propagated (I1), never silently swallowed.

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2).** The module's crux. Every path/permission/IO failure is an explicit `FsErr`
  with an RFC-0013 record (path + errno-class + why). **No silent create-on-write**: `OpenOptions.create`
  defaults `false`, so opening an absent path for write is `Err(NotFound)`, not a conjured file. **No
  silent truncation**: `truncate` defaults `false`, so an existing file's bytes survive unless the caller
  declared the zero-out. **No silent partial write**: a short write returns the real count; the shortfall
  is surfaced, never swallowed. **No silent overwrite**: `copy`/`rename` to an existing target without a
  declared intent is an error, not a clobber. A disk-full, a missing path, a permission denial, and a
  use-after-consume of a `substrate` handle are each an explicit, *traceable* error naming the path and
  the cause.
- **C2 — honest per-op tag (VR-5).** `fs` carries **no** accuracy semantics, so every op tags `Exact` —
  and this is a *downgrade-safe* claim, not an upgrade: there is no precision being asserted, so there is
  nothing to over-claim. `fs`'s honesty lives in C1 (fallibility) and C6 (effects), not in a precision
  tag. The matrix never marks an op `Proven`/`Empirical`/`Declared`, because `fs` establishes no bound.
- **C3 — no black boxes / EXPLAIN (SC-3/G11).** Every fallible op emits an inspectable RFC-0013 diagnostic
  record: the path attempted, the `OpenOptions` intent (so *why* a `create`/`truncate` did or did not
  fire is visible), the classified errno-class, and a human *why*. The `wild`-floor calls are themselves
  inventoried + justified (`// SAFETY:`) and surfaced by the `myc-sec` `wild`-audit (EXPLAIN-able at the
  tooling layer — `docs/spec/Security-Checks-Contract.md` §4). No opaque heuristic decides a user-visible
  filesystem outcome.
- **C4 — content-addressed, value-semantic (ADR-003 / RFC-0001).** A `Path` is an immutable, content-
  addressable value (two equal paths are the same value). `Metadata` is a value snapshot, and **metadata
  is not identity** (ADR-003): a file's `mtime`/permissions/inode are *not* part of a path's or a read
  value's identity — re-reading the same bytes yields the same value regardless of timestamp. The affine
  `substrate` `File`/`Dir` *handle* is the one non-value object (it is an effect capability, LR-8), and it
  is explicitly typed as such, not smuggled as a value.
- **C5 — above the small kernel (KC-3) — NARROWED, honestly.** `fs`'s path/permission/intent logic adds
  no trusted code. **But** its filesystem effects bottom out in real OS syscalls, which require an audited
  `wild` block (ADR-014) — so the blanket "no new trusted code" KC-3 claim **does not hold whole** for
  `fs`. This is stated, not hidden (the honesty rule applied to the contract itself): the `wild` surface
  is **inventoried** (the §4 `wild`? column + §6) and **justified** (each block carries an ADR-014
  `// SAFETY:`, checked by the `myc-sec` `wild`-audit). That audited floor is the minimum unsafe needed
  for filesystem access; everything above it is safe Mycelium. Whether the floor is quarantined into a
  separate `std-sys` phylum so the *rest* of `std` stays leak-free **by construction** (LR-9) is RFC-0016
  §8-Q6 — **FLAGGED §7 Q1**, not silently decided here.
- **C6 — declared, bounded effects (RFC-0014).** Every filesystem op declares `effects: io` on its
  signature; only the pure lexical path ops (`Path::new`/`join`/`parent`) and the `UseAfterConsume` catch
  are `effects: none`. fs IO is a **declared effect** — a deterministic-fragment program cannot touch the
  filesystem silently. The `substrate` handle is **affine** (LR-8): consumed exactly once; `close`
  consumes it, and any subsequent use is `Err(UseAfterConsume)`, structurally closing the only leak vector
  (an unreleased external resource — LR-9). Unbounded directory walks (`read_dir` over a deep tree) are
  surfaced as an explicit iterator, not an ambient recursion (a budget, if needed for a recursive
  `walk_dir`, is a declared bound — FLAGGED §7 Q4).

## 6. Grounding

- The `fs` scope + honesty crux ("every path/permission failure explicit; no silent create-on-write"):
  **RFC-0016 §4.4** (the Tier-B `fs` row, M-528) and §4.1 (the C1–C6 contract lifted to library scope),
  §4.5 (the guarantee-matrix obligation as a checked table, never prose).
- The byte source/sink + `substrate` handle `fs` layers on: **`std.io` (M-514)** — RFC-0016 §4.4 io row
  ("input/output over `substrate` affine handles (consumed exactly once)"). `fs` *builds on* it and does
  **not** restate or re-own its signatures; those are M-514's (**FLAGGED §7 Q2**).
- The affine `substrate` `Resource` kind (consumed exactly once; the only leak vector, closed): **RFC-0006
  LR-8** (the affine `Resource` hook) and **LR-9** (leaks structurally excluded; `wild` denied-by-default,
  lexically marked, inventoried); the `substrate` lexeme is Ratified (DN-02 §2; `docs/Glossary.md` §2.18).
- The audited OS-facility floor: **ADR-014** (Accepted — `unsafe`/`wild` permitted-but-warned, every block
  carries a mandatory `// SAFETY:` justification) and the **`wild`-audit** that inventories every `wild`
  block (`docs/spec/Security-Checks-Contract.md` §4, enacted by `crates/mycelium-sec`; LR-9/S6/DN-02 §5).
- The robust-and-legible failure record (path + errno-class + why, recovered/re-propagated): **RFC-0013**
  (structured diagnostics + reified error policy) and the I1 propagation floor; **RFC-0014** (declared,
  bounded effects — fs IO as a declared effect).
- The value model + honesty lattice: **RFC-0001** (`Value`/`Repr`/`Meta`, the lattice §4.3, content-
  addressing §4.6); **ADR-003** (metadata is not identity); **G2** (never-silent), **VR-5** (honest tags),
  **G11** (dual projection), **KC-3** (small kernel — narrowed here, §5 C5).

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) The `wild`/FFI floor and the `std-sys` split.** `fs` cannot be wholly `wild`-free — its syscalls
  need an audited `wild` block (ADR-014). The open question (RFC-0016 §8-Q6) is the *minimal* audited FFI
  floor and **where it lives**: inside `std.fs`, shared with `io`/`time`/`rand` in one `std-sys`
  capability layer, or hoisted into a separate `std-sys` **phylum** so the pure-safe rest of `std` stays
  *certified leak-free by construction* (LR-9 — a `phylum` with no `wild` blocks earns the leak-free
  badge). This spec strongly *prefers* the `std-sys` quarantine (it preserves the leak-free badge for pure
  `std`) but does **not** decide it. — *Disposition: FLAGGED; ties directly to RFC-0016 §8-Q6; the
  maintainer's call at ratification.*
- **(Q2) The exact `io` (M-514) surface `fs` builds on.** `fs` layers paths/handles/permissions over io's
  byte streams, but io's concrete `Read`/`Write`/`flush` signatures and its `substrate` single-consumption
  mechanism are **M-514's to own** — they are *sketched* here only to fix the seam, not invented.
  — *Disposition: FLAGGED; thread whatever M-514 commits; do not pre-commit io's surface from `fs`.*
- **(Q3) Path model + portability.** Is `Path` a `Text` newtype, a structured component list, or an
  OS-string that may not be UTF-8 (the classic portability hazard)? A non-UTF-8 path is itself a
  never-silent concern (C1 — a lossy path decode must be an explicit error, not a replacement char). The
  separator / normalization / case-fold policy is OS-dependent and must be reified (C3), not a hidden
  default. — *Disposition: FLAGGED; default to an explicit, fallible path-decode; coordinate with
  `text`/`string` (M-524) on the lossy-decode error. Ties to RFC-0016 §8-Q3 (ergonomics vs the contract).*
- **(Q4) Atomicity, symlinks, and recursive walks.** Cross-platform atomicity of `rename`/`copy`, symlink
  vs hardlink handling (follow vs nofollow as a *declared* intent, never silent), and a recursive
  `walk_dir` (which, if added, carries a declared traversal/`alloc` budget per C6/RFC-0014) are unsettled.
  TOCTOU between `exists`/`stat` and a subsequent `open` is real — the spec prefers the open-and-handle-
  the-error idiom over check-then-act, but the guidance is not yet ratified. — *Disposition: FLAGGED;
  default to declared-intent + open-then-handle; budget any recursive walk.*
- **(Q5) Capability scoping of the filesystem effect.** Is the `io` filesystem effect ambient, or is it a
  scoped capability (a "filesystem root" handle a program is granted, à la WASI preopens) so a
  deterministic-fragment program's reach is bounded and inspectable? This is the C6/RT3 ergonomics
  question for `fs` specifically. — *Disposition: FLAGGED; ties to RFC-0016 §8-Q3 and the §8-Q6 `std-sys`
  decision (Q1); prefer a scoped capability but do not decide here.*

## Meta — changelog

- **2026-06-17 — Draft (needs-design).** Stands up the `std.fs` (M-528, #169) module spec under RFC-0016
  (Draft): Ring 2 / Tier B filesystem access over **affine `substrate` handles** (consumed exactly once,
  LR-8), layered on the `io` byte-stream surface (M-514) — `fs` adds paths/permissions/declared-intent
  *over* io's bytes and does not re-own them (FLAGGED). Fixes the scope + boundary (io owns byte transfer
  M-514; `time` owns clocks M-529; the OS floor is an audited `wild` block ADR-014), the exported-op
  surface sketch (path values; `OpenOptions` with every create/truncate **opt-in**; open/read/write/
  flush/close/stat/list/create/remove/rename/copy; the explicit `FsErr` set), and — the load-bearing
  deliverable — the per-op **guarantee matrix**: every op tags `Exact` (fs carries *no* accuracy
  semantics; its honesty is C1 fallibility + C6 effects, never a C2 precision over-claim), every
  path/permission/IO failure is an explicit RFC-0013 diagnostic naming the path + errno-class (**no silent
  create-on-write, no silent truncation, no silent partial write**, C1/G2), and a **`wild`? column**
  inventories which ops bottom out in the audited syscall floor. §4.1 conformance (C1–C6) stated
  concretely — C5 **honestly narrowed**: `fs` cannot be wholly `wild`-free, so the "no new trusted code"
  KC-3 claim is qualified, the `wild` surface inventoried (§6) + justified (ADR-014 `// SAFETY:`,
  `myc-sec` audit), and the affine handle (LR-8) closes the only leak vector (LR-9). Grounding traces to
  RFC-0016 §4.1/§4.4/§4.5/§8-Q6, ADR-014, RFC-0006 LR-8/LR-9, RFC-0013/0014, M-514, RFC-0001, G2/VR-5/
  KC-3. Five questions FLAGGED (the `wild`/FFI floor + `std-sys` phylum split per §8-Q6; the exact io
  surface owned by M-514; the path model + non-UTF-8 portability; atomicity/symlinks/recursive-walk budget
  and TOCTOU; capability-scoped filesystem effect) — the io surface and the `std`-vs-`std-sys` placement are
  *not* invented here. No code; no kernel change beyond the FLAGGED, inventoried `wild` floor (KC-3,
  narrowed). Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).
