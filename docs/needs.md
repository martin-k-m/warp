# What warp needs from twill

warp is written in twill and it runs: `twill test tests` passes five suites on
twill 1.7.1. This file is what warp asked the language for on the way there,
with the file and function that needed each one, what warp did in the meantime,
and whether it has arrived.

It is a work queue for the language, not a complaint. Every entry was reached by
writing real code and hitting the wall.

**Where this stands on twill 1.7.1.** Delivered: 1 function values and closures,
2 `F64`, 3 float math, 4 writing files and directories, 5 ranged reads, 7 the
sign of `shr`, 12 generic containers, 13 a test runner. Open: 6 number parsing,
8 `Bytes` distinct from `Str`, 9 decompression, 10 networking or a process
interface, 11 a tensor across the seam, 14 an iteration protocol, and 15 below,
which this repository found while writing the getting-started section. Each
entry carries the check that settled it.

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64` with bitwise operations, `Str` with length, byte indexing
and slicing, `Arr[T]`, `Dict[Str, V]`, `struct`, `enum` with exhaustive `match`,
`Opt` and `Res`, and `read_file`. Everything below is beyond that.

## Was blocking: warp could not load anything without these

All five arrived. Kept, rather than deleted, because the argument for each one is
why it was asked for and the next library to want it can point here.

### 1. Function values with a declared type

**Needs:** `Fn(A) -> B` as a type that a struct field can hold and a parameter
can declare, and closures over the enclosing scope
**Used by:** `src/pipeline.tw` (`Stage.fn_map`, `Stage.fn_keep`, `Source.get`),
`src/datasets.tw` (`idx_source`, `csv_source`)
**Status: delivered.** `mode systems` has `Fn(A) -> B` as a declarable type and
closures over the enclosing scope, checked with `twill check` and run. Every
`Source.get` and every `map` stage in this library is one, and the pipeline
tests exercise both.

This is the whole shape of the library. A pipeline is a list of user functions,
and a source is a function from an index to a sample. Without function values
warp would have to become a fixed set of built-in transforms with no way to add
one, which is a different and much worse library.

The `Source.get` field is a closure over the loaded buffer, so closures are
needed and not only function pointers.

### 2. F64 as a first-class type in the systems subset

**Needs:** `F64` values, literals, arithmetic, comparison, and `f64(I64)` /
`i64(F64)`
**Used by:** `src/sample.tw`, `src/augment.tw`, `src/strutil.tw`, and every
sample warp has ever seen
**Status: delivered.** `F64` literals, arithmetic, comparison and `f64(I64)` /
`i64(F64)` all work in `mode systems`. `src/sample.tw` and the augmentation
suite are float end to end.

The systems subset was designed around a compiler, where integers are the whole
job. A data loader's payload is floats. This is the same entry weft records, and
it being on both lists is the argument for it.

### 3. Float math builtins in systems mode

**Needs:** `sqrt`, `log`, `cos` on `F64`
**Used by:** `src/augment.tw` (`gaussian`, `stddev_of`)
**Status: delivered.** `sqrt`, `log` and `cos` take a systems-mode `F64` and
return one. `aug.gaussian` runs, and `tests/augment_test.tw` covers the jitter
that depends on it.

Box-Muller needs `sqrt`, `log` and `cos` and there is no way around it. Gaussian
noise is not optional in an augmentation library.

### 4. Writing files, and directories

**Needs:** `write_file`, `rename`, `mkdir_all`, `path_exists`, `list_dir`,
`remove_all`, `file_size`, `mtime`
**Used by:** `src/cache.tw` (`write`, `is_fresh`, `prune`), `src/datasets.tw`
(`verify`), `src/stream.tw` (`open`, `reopen`)
**Status: delivered.** `write_file`, `rename`, `mkdir_all`, `path_exists`,
`list_dir`, `remove_all`, `file_size` and `mtime` all exist, and `temp_dir` came
with them, which is what lets a test write to a scratch directory instead of a
hard-coded `/tmp`. `cache.write`, `cache.read`, `cache.has` and `cache.prune`
now run against real files in `tests/cache_test.tw`.

One shape to watch: `list_dir` returns a `Res`, not an `Arr[Str]`. `cache.prune`
was written against the bare array and called `len` on the `Res`, which is a
runtime error and not a checker one, so it survived until something ran it.
`mtime` returns -1 rather than an error for a path it cannot read, which is the
convention `file_size` follows.

Two are worth naming individually. `rename` must be atomic within a directory,
because the cache writes to a temporary name and renames into place: an entry
half written when a run is interrupted is worse than no entry, since its key
says it is complete and every later run will read it. And `mtime` is what makes
`cache.is_fresh` possible, which is the safety net for the one failure the cache
design cannot rule out by construction.

### 5. Ranged reads

**Needs:** `read_file_at(path, offset, length) -> Res[Str, Str]`
**Used by:** `src/stream.tw` (`fill`)
**Status: delivered.** `read_file_at(path, offset, length)` returns
`Res[Str, Str]`, which is the signature this entry asked for. `src/stream.tw`
needed no change, and `tests/stream_test.tw` now reads a file larger than one
`CHUNK` through it.

This is the entire content of "streaming". A dataset that does not fit in memory
cannot be read with a function whose only mode is to read all of it, and every
other part of `stream.tw` is written against this one call. It is the smallest
possible addition that makes out-of-core data possible: no file handles, no
seeking API, one function.

## Painful: written around, badly

### 6. Number parsing

**Needs:** `parse_i64(Str) -> Res[I64, Str]`, `parse_f64(Str) -> Res[F64, Str]`
**Used by:** `src/strutil.tw`, and through it every reader in the library
**Status: open on twill 1.7.1.** `parse_i64` and `parse_f64` are still unknown
names, so `src/strutil.tw` still carries warp's own parser and the reasoning
below still applies.

warp ships its own decimal and exponent parser. It is a hundred lines that every
program reading a CSV will otherwise write again, and it will be subtly
different every time: this one accumulates the fraction as an integer and
divides once, because adding `digit / 10^k` as it goes rounds at every step and
the error shows up in the eighth digit, which is exactly where a cached value
gets compared against a freshly computed one and fails.

Correct float parsing is genuinely hard and belongs in the runtime, next to
whatever prints them, so that printing and parsing round trip.

### 7. `shr` on a negative I64 is unspecified

**Needs:** a statement of whether `shr` is arithmetic or logical, or two
operators
**Used by:** `src/rng.tw` (`mix`, `next`)
**Status: delivered.** twill's language guide now states that `shr` is
arithmetic, shifting the sign bit in, and `shr(-8, 1)` is `-4`. There is no
`ushr` operator; the guide gives the idiom for building one.

`src/rng.tw` still masks to 32 bits, and that is now a choice rather than a
constraint. Widening it would double the state on the hottest path in the
loader, and it would also change every seeded sequence this library has ever
produced, so it is a version bump and a cache-key change, not a cleanup.

The generator masks everything to 32 bits after every step, so it never shifts a
negative value, which is a real cost: it throws away half the width of the type
on the hottest path in the loader. The reason is that the sequence must be
identical on every machine, and a shift whose behaviour is unstated cannot be
depended on for that. spool's `sha256.tw` is masked for the same reason and
would have the same question.

### 8. `Bytes` as distinct from `Str`

**Needs:** the `Bytes` type from section 1.2, and `read_file` returning it
**Used by:** `src/datasets.tw` (`read_idx`, `be32`), `src/stream.tw`
**Status: half delivered.** `Bytes` is a type the checker accepts. `read_file`
and `read_file_at` still hand back a `Str`, so warp still indexes IDX files as a
byte string and the type still says "text" about data that is not.

IDX files are binary and warp indexes them as a byte string. It works, since
`Str` in the subset is bytes that print, and it means the type says "text" about
data that is not. The distinction matters at exactly one place, which is where a
file is read and something has to decide whether to trust it as UTF-8.

### 9. Decompression

**Needs:** gzip, or a documented decision that the user decompresses first
**Used by:** `src/datasets.tw` (MNIST and Fashion-MNIST ship as `.gz`)
**Status: open on twill 1.7.1**, and reasonably so.

warp reads the decompressed IDX file and tells the user to gunzip it. That is an
extra step in every getting-started guide forever. Not a language feature so
much as a standard-library decision, and the honest options are a gzip reader in
twill, a process interface (spool needs one anyway) so warp can shell out, or
keeping the manual step and documenting it. Currently the third.

### 10. Networking, or a process interface

**Needs:** an HTTPS fetch, or `run(program, argv, dir)`
**Used by:** `src/datasets.tw`, which describes downloads it cannot perform
**Status: open on twill 1.7.1.** There is no socket and no process builtin, so
neither half of this is available. spool records the same gap and argues for the
process interface as the smaller ask.

warp prints the URL and the expected size and asks the user to fetch the file.
This is the entry warp is least unhappy about: a data-loading library that
silently downloads 170 megabytes is a library that surprises people, and the
verification step is more valuable than the fetching step. But it does mean
`warp.get("cifar-10")` cannot exist.

## Would improve it

### 11. A tensor type in systems mode

**Needs:** `Tensor` usable from `mode systems`, or a stated conversion at the
boundary
**Used by:** `src/sample.tw` (the whole file)
**Status: open on twill 1.7.1.** twill's language guide is explicit now:
`mode systems` has no tensor type at all. So the seam is still a seam and
`src/sample.tw` is still a tensor written out longhand.

A sample is a flat `Arr[F64]` plus an `Arr[I64]` shape, which is a tensor
written out longhand. Every consumer has to rebuild the tensor at the boundary,
and the shape can disagree with the buffer because nothing checks it. If the two
halves of the language can pass a tensor across the seam, `sample.tw` collapses
to two fields and the shape check comes for free from the existing checker.

This is the largest design question on the list and it is a language-level one:
`mode systems` was defined by what a compiler needs, and a data loader is the
first program that wants both halves at once.

Since this entry was written, the numeric half grew narrow dtypes (twill
`docs/dtypes.md`): seven of them, a `.to(dt)` cast, and a packed byte buffer
designed under twill NEEDS-111. warp now carries a declared dtype on the
pipeline and the batch (`src/dtype.tw`, `pipe.astype`), so a consumer building
the tensor at the boundary builds it at the width the data was meant for, f32
for scaled pixels, i32 for labels, rather than at f64 and narrowing after, which
moves twice the bytes through the pipeline and the cache for nothing. That is a
declaration today, not a storage change, for the same reason this entry is open:
without a tensor across the seam, and without NEEDS-111's buffer landed in the
runtime, warp still holds `Arr[F64]`. When both land, the dtype already threads
to the point a narrow store hangs off, and the cache already keys on it.

### 12. Generic containers over user types

**Needs:** `Arr[T]` where `T` is a user struct, which the design implies but does
not state
**Used by:** `src/pipeline.tw` (`Arr[Stage]`), `src/datasets.tw` (`Arr[File]`),
`src/cache.tw` (`Arr[smp.Batch]`)
**Status:** done, and the estimate this entry worried about was wrong in the
useful direction. `Arr[T]` over a user struct works and is checked, and twill
1.7 went further and closed NEEDS-4, so a declaration here could take its own
type parameters if it wanted any.

The uncertain estimate was for monomorphization, which turned out not to be
needed at all: twill's runtime is a tree walker over dynamically typed values,
so the same code runs whatever `T` is and the parameters only have to reach the
checker. Nothing warp does is exotic -- no bounds, no nesting beyond
`Arr[Arr[F64]]` -- and none of it ever needed the expensive half.

Also done, on twill 1.7: `datasets.verify` returned a `Str` that was empty on
success and returns `Res[Unit, Str]`, which is the last error channel in warp
that was not already a `Res`.

### 13. A test runner

**Needs:** a `twill test` that collects `tests/*_test.tw`
**Used by:** everything in `tests/`
**Status: delivered.** `twill test tests` collects the suites and reports them,
and CI runs it against a pinned release.

The harness stayed. Every test file is still a program that calls its cases at
the bottom and ends with `report`, because that is what makes a file runnable on
its own, and `twill test` runs it either way. A new test file is picked up
without anyone adding it to a list. What is still done by hand is the call at
the bottom of the file, and CI checks the last line is `t.report(` for exactly
that reason.

### 14. A defined iteration protocol

**Needs:** a `for x in thing` that works on a user type
**Used by:** `src/pipeline.tw` (`Iter`), `src/stream.tw` (`next_line`)
**Status: open on twill 1.7.1.** `for x in v` over a user struct passes the
checker and fails at run time with "value is not iterable", so there is still no
way for a user type to take part.

Every consumer of a pipeline writes the same `while true { match next_batch(it)
{ ... } }`, and the `Opt.None` arm is where someone will eventually forget to
break. Low priority, purely ergonomic, but it is the shape of every training
loop that will ever use this library.

## Found since

### 15. Which directory a relative path is relative to

**Needs:** a statement of what a relative path passed to `read_file`,
`path_exists` or `write_file` resolves against
**Used by:** `examples/train.tw`, and any program a user writes
**Status:** open on twill 1.7.1, and undocumented rather than decided.

`examples/train.tw` asks for `data/mnist`. Run as `twill run examples/train.tw`
from the root of a clone, that reads `examples/data/mnist`: the path resolves
against the directory of the file being run and not against the process's
working directory. `cwd()` in the same program reports the working directory,
so the two disagree and nothing in the language guide says which one file IO
uses.

Resolving against the script is defensible, and it is the choice that makes an
example carry its own fixtures. The problem is only that it is unwritten, so
the first thing a user does with this repository is put a 47 MB file in the
wrong directory and get "missing" for a file that is plainly there. warp's
README now says where the data goes; the language should say why.
