# What warp needs from twill

warp is written in twill and does not run yet. This file is the reason: the
language and runtime features the source uses that `mode systems` does not
provide today, with the file and function that needs each one and what warp does
in the meantime.

It is a work queue for the language, not a complaint. Every entry was reached by
writing real code and hitting the wall.

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64` with bitwise operations, `Str` with length, byte indexing
and slicing, `Arr[T]`, `Dict[Str, V]`, `struct`, `enum` with exhaustive `match`,
`Opt` and `Res`, and `read_file`. Everything below is beyond that.

## Blocking: warp cannot load anything without these

### 1. Function values with a declared type

**Needs:** `Fn(A) -> B` as a type that a struct field can hold and a parameter
can declare, and closures over the enclosing scope
**Used by:** `src/pipeline.tw` (`Stage.fn_map`, `Stage.fn_keep`, `Source.get`),
`src/datasets.tw` (`idx_source`, `csv_source`)
**Status:** numeric mode has first-class functions and closures. `mode systems`
does not say whether it keeps them, and section 1.2 mentions no function type.

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
**Status:** `docs/self-hosting.md` names `F64` once, as an enum payload in the
token example, and specifies nothing else. Section 1.2 is about `I64`.

The systems subset was designed around a compiler, where integers are the whole
job. A data loader's payload is floats. This is the same entry weft records, and
it being on both lists is the argument for it.

### 3. Float math builtins in systems mode

**Needs:** `sqrt`, `log`, `cos` on `F64`
**Used by:** `src/augment.tw` (`gaussian`, `stddev_of`)
**Status:** these exist in numeric mode as differentiable tensor operations.
Whether they exist on a systems-mode `F64` is unspecified.

Box-Muller needs `sqrt`, `log` and `cos` and there is no way around it. Gaussian
noise is not optional in an augmentation library.

### 4. Writing files, and directories

**Needs:** `write_file`, `rename`, `mkdir_all`, `path_exists`, `list_dir`,
`remove_all`, `file_size`, `mtime`
**Used by:** `src/cache.tw` (`write`, `is_fresh`, `prune`), `src/datasets.tw`
(`verify`), `src/stream.tw` (`open`, `reopen`)
**Status:** `write_file` is in section 1.2 and `list_dir` is mentioned in
passing. The rest are not, and spool records the same gap.

Two are worth naming individually. `rename` must be atomic within a directory,
because the cache writes to a temporary name and renames into place: an entry
half written when a run is interrupted is worse than no entry, since its key
says it is complete and every later run will read it. And `mtime` is what makes
`cache.is_fresh` possible, which is the safety net for the one failure the cache
design cannot rule out by construction.

### 5. Ranged reads

**Needs:** `read_file_at(path, offset, length) -> Res[Str, Str]`
**Used by:** `src/stream.tw` (`fill`)
**Status:** not in the design. `read_file` reads the whole file.

This is the entire content of "streaming". A dataset that does not fit in memory
cannot be read with a function whose only mode is to read all of it, and every
other part of `stream.tw` is written against this one call. It is the smallest
possible addition that makes out-of-core data possible: no file handles, no
seeking API, one function.

## Painful: written around, badly

### 6. Number parsing

**Needs:** `parse_i64(Str) -> Res[I64, Str]`, `parse_f64(Str) -> Res[F64, Str]`
**Used by:** `src/strutil.tw`, and through it every reader in the library
**Status:** `str` goes one way and nothing comes back.

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
**Status:** section 1.2 says shifts mask the shift count to 0..63 and says
nothing about the sign.

The generator masks everything to 32 bits after every step, so it never shifts a
negative value, which is a real cost: it throws away half the width of the type
on the hottest path in the loader. The reason is that the sequence must be
identical on every machine, and a shift whose behaviour is unstated cannot be
depended on for that. spool's `sha256.tw` is masked for the same reason and
would have the same question.

### 8. `Bytes` as distinct from `Str`

**Needs:** the `Bytes` type from section 1.2, and `read_file` returning it
**Used by:** `src/datasets.tw` (`read_idx`, `be32`), `src/stream.tw`
**Status:** `Bytes` is designed. `read_file` returning `Res[Bytes, Str]` is in
the design; warp is written against a `Str` because that is what the milestone
provides.

IDX files are binary and warp indexes them as a byte string. It works, since
`Str` in the subset is bytes that print, and it means the type says "text" about
data that is not. The distinction matters at exactly one place, which is where a
file is read and something has to decide whether to trust it as UTF-8.

### 9. Decompression

**Needs:** gzip, or a documented decision that the user decompresses first
**Used by:** `src/datasets.tw` (MNIST and Fashion-MNIST ship as `.gz`)
**Status:** not in the design, and reasonably so.

warp reads the decompressed IDX file and tells the user to gunzip it. That is an
extra step in every getting-started guide forever. Not a language feature so
much as a standard-library decision, and the honest options are a gzip reader in
twill, a process interface (spool needs one anyway) so warp can shell out, or
keeping the manual step and documenting it. Currently the third.

### 10. Networking, or a process interface

**Needs:** an HTTPS fetch, or `run(program, argv, dir)`
**Used by:** `src/datasets.tw`, which describes downloads it cannot perform
**Status:** section 1.2 stops at "no sockets". spool records the same gap and
argues for the process interface as the smaller ask.

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
**Status:** tensors are the numeric half of the language. The systems subset
says nothing about them.

A sample is a flat `Arr[F64]` plus an `Arr[I64]` shape, which is a tensor
written out longhand. Every consumer has to rebuild the tensor at the boundary,
and the shape can disagree with the buffer because nothing checks it. If the two
halves of the language can pass a tensor across the seam, `sample.tw` collapses
to two fields and the shape check comes for free from the existing checker.

This is the largest design question on the list and it is a language-level one:
`mode systems` was defined by what a compiler needs, and a data loader is the
first program that wants both halves at once.

Since this entry was written, the numeric half grew narrow dtypes (raster
`docs/dtypes.md`): seven of them, a `.to(dt)` cast, and a packed byte buffer
designed under raster NEEDS-111. warp now carries a declared dtype on the
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
**Status:** section 1.2 offers unconstrained monomorphized generics and flags
the estimate as uncertain.

Nothing warp does is exotic: no bounds, no nesting beyond `Arr[Arr[F64]]`. It is
listed so the requirement is on record from a second program, since the design
document says this is the item most likely to be underestimated.

### 13. A test runner

**Needs:** a `twill test` that collects `tests/*_test.tw`
**Used by:** everything in `tests/`
**Status:** none. Same gap spool and weft record.

Every test file is a program that calls its cases at the bottom and ends with
`report`, which exits non-zero on a failure. A new test file is invisible to CI
until someone adds it by hand.

### 14. A defined iteration protocol

**Needs:** a `for x in thing` that works on a user type
**Used by:** `src/pipeline.tw` (`Iter`), `src/stream.tw` (`next_line`)
**Status:** `for` works on lists and 1-D tensors in numeric mode. There is no
way for a user type to take part.

Every consumer of a pipeline writes the same `while true { match next_batch(it)
{ ... } }`, and the `Opt.None` arm is where someone will eventually forget to
break. Low priority, purely ergonomic, but it is the shape of every training
loop that will ever use this library.
