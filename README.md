<p align="center">
  <img alt="twill" src="https://raw.githubusercontent.com/twill-lang/twill/main/assets/twill-mark.png" width="96">
</p>

<h1 align="center">warp</h1>

<p align="center">
  <b>Data pipelines and dataset loaders for <a href="https://github.com/twill-lang/twill">twill</a>.</b><br>
  Written in twill.
</p>

<p align="center">
  <img alt="warp" src="https://img.shields.io/badge/warp-v0.1.0-7FE3C4?style=flat-square&labelColor=12332C">
  <img alt="written in twill" src="https://img.shields.io/badge/written%20in-twill-A8DCCB?style=flat-square&labelColor=12332C">
  <img alt="status: tests passing" src="https://img.shields.io/badge/tests-passing-D2F0E4?style=flat-square&labelColor=12332C">
  <img alt="MIT" src="https://img.shields.io/badge/license-MIT-4FB79B?style=flat-square&labelColor=12332C">
</p>

---

## It runs

`warp` is written in twill, in `.tw` files, using `mode systems`. That subset
did not exist when this library was written, so for a long time none of the code
here executed and this section said so. twill 1.6 is the release that closed it:
the 5 test suites under `tests/` pass, and CI runs them against a released
twill on every push rather than gating on the prose in this file.

```bash
twill test tests
```

You need twill 1.6.5 or newer. `docs/needs.md` is still worth reading -- it
is the list of what this library asked the language for, and it now records
which of those arrived and which are still open.

## Status

| Piece | State |
| --- | --- |
| Composable pipeline: map, filter, batch, shuffle, prefetch, repeat | written, unrun |
| A declared dtype carried to the batch, so the model builds it narrow not at f64 | written; bytes gated on twill NEEDS-111 |
| The order check that catches shuffle-after-batch | written, unrun |
| Disk cache keyed on the transformation chain | written, blocked on file IO |
| Augmentation for images and sequences, each seeded explicitly | written, unrun |
| Deterministic per-sample seeding | written, unrun |
| Streaming for data larger than memory | written, blocked on ranged file reads |
| Dataset descriptions with origin, licence and checksums | written; **the digests are not yet pinned**, see below |
| Downloading a dataset | **not written.** twill has no network, see docs/needs.md |
| Tests | written, blocked on a test runner |
| Bundled datasets | **never.** warp describes data, it does not ship it |
| Parallel loading across processes | **not planned for v0.1** |
| Anything running end to end | **no** |

## The worked example

```rust
mode systems

import "twill_modules/warp/src/pipeline.tw" as pipe
import "twill_modules/warp/src/augment.tw" as aug
import "twill_modules/warp/src/rng.tw" as rng
import "twill_modules/warp/src/cache.tw" as cache
import "twill_modules/warp/src/dtype.tw" as dt

let RUN_SEED: I64 = 20260807

fn build(src: pipe.Source, epoch: I64) -> pipe.Pipeline {
  let p = pipe.from_source(src)

  # Every stage carries a name and a version. They are not documentation: they
  # are the stage's identity in the cache key, because a function value cannot
  # be hashed. Change what the function does, change the version.
  p = pipe.map(p, "standardise", 1, fn(s: smp.Sample) -> smp.Sample = standardise(s))
  p = pipe.map(p, "crop-32-pad-4", 1,
    fn(s: smp.Sample) -> smp.Sample = aug.random_crop(s, 32, 32, 4, rng.seed_for(RUN_SEED, epoch, s.id)))

  # Scaled pixels are meant for f32, so declare it: the batch carries the dtype
  # and the model builds its tensor narrow in one step rather than at f64 and a
  # cast after, which moves twice the bytes for nothing. A declaration today, a
  # real narrow store once twill NEEDS-111 lands; either way the cache keys on
  # it, so f32 and bf16 runs never share an entry.
  p = pipe.astype(p, dt.DT_F32)

  # Shuffle, then batch. The other order gives every batch the same 32 samples
  # for the whole run. See the top of src/pipeline.tw.
  p = pipe.shuffle(p, 10000, RUN_SEED)
  p = pipe.batch(p, 128, false)
  p = pipe.prefetch(p, 2)
  p
}

fn main() {
  let p = build(cifar_source(), 0)

  # Says nothing when the order is sensible, and names both stages when it is
  # not. A warning rather than an error: there are real reasons to batch first.
  let warnings = pipe.validate(p)
  let w: I64 = 0
  while w < len(warnings) {
    write_err("warp: " + warnings[w] + "\n")
    w = w + 1
  }

  let it = pipe.iterate(p)
  while true {
    match pipe.next_batch(it) {
      Opt.None => return,
      Opt.Some(b) => train_step(b),
    }
  }
}
```

## The three things worth reading

### The order stages compose in

Stages apply in the order they were added, and the two orders people mix up
produce different training runs without either one failing.

- **shuffle then batch**: each batch is a fresh mix, and the composition of
  every batch changes each epoch.
- **batch then shuffle**: samples are grouped in file order and then the groups
  are reordered. The same samples travel together for the whole run.

If the data is sorted by class, which is the usual layout on disk, the second
gives batches that are each a single class. Batch normalisation then computes
its statistics over one class at a time, the gradient is systematically wrong,
and the run does not crash. It converges to something worse, and the loss curve
looks plausible.

warp does not reorder the stages behind your back, because then the code says
one thing and does another. `pipe.validate` reports the suspicious order,
naming both stages, and the caller decides. The full argument is at the top of
[`src/pipeline.tw`](src/pipeline.tw), including the two other orderings that
matter: map before filter, and repeat before shuffle.

### The cache key

The cache is keyed on the **whole transformation chain**, not on the output and
not on the source:

```
key = sha256(cache format version
             + source name, version and size
             + every stage that can change a sample's value, in order)
```

Five decisions are packed into that, and each is argued in the header of
[`src/cache.tw`](src/cache.tw). The two with consequences:

**A stage that changes behaviour must change its name or raise its version.**
There is no way for a program to hash the behaviour of a function value it was
handed. Two different functions are indistinguishable to anything except running
them on every input. So the obligation is on the caller, and warp makes it
unavoidable by requiring a name and a version on every `map` and `filter` rather
than defaulting them. A library that let you write `.map(f)` would be silently
wrong the first time anyone edited `f`, and it would look like it worked.
`cache.is_fresh` is the safety net: an entry older than the source files that
define the pipeline is treated as a miss even when the key matches.

**Batching, prefetching and repeating are excluded from the key.** None of them
changes what a sample contains, so changing the batch size does not throw away
an hour of decoding. Shuffle is included, with its seed, because a cached
artefact is a sequence and its order is part of it.

`cache.explain` returns the exact text that was hashed, so "why did my cache
miss" is answered with a diff rather than a hex string.

### Seeding

Every augmentation takes a seed, not a generator:

```rust
aug.random_crop(s, 32, 32, 4, rng.seed_for(RUN_SEED, epoch, s.id))
```

`seed_for` mixes the run seed, the epoch and the sample's own index, so sample
4,013 of epoch 2 gets the same crop on every machine, in any order, with any
number of workers. Mixing rather than adding matters: added, epoch 1 sample 2
and epoch 2 sample 1 are the same seed, and the resulting correlation across an
epoch is invisible in a loss curve.

## Datasets

warp bundles no data. Each dataset is a description: origin, citation, licence,
files, expected sizes and digests, plus the code to read the format once the
file is on disk.

| Dataset | Origin | Licence |
| --- | --- | --- |
| MNIST | LeCun and Cortes, 1998 | CC BY-SA 3.0 |
| Fashion-MNIST | Zalando Research, 2017 | MIT |
| CIFAR-10 | Krizhevsky, University of Toronto, 2009 | not stated by the authors; cite the technical report |
| Iris | Fisher, 1936, via UCI | CC BY 4.0 as distributed by UCI |

**The digests are not pinned yet, and warp refuses to use a dataset whose digest
it has not pinned.** A hash copied from somewhere without checking it against the
real file is worse than no hash, because it looks like verification and is not.
`verify` prints the digest it computed so it can be recorded. The open item is
in [`docs/datasets.md`](docs/datasets.md).

Refusing rather than warning is the same bias twill takes everywhere else: when
the signal is ambiguous, assume less. A warning in a training log is not read.

## Streaming

`src/stream.tw` is for data that does not fit in memory, and it is a different
shape from the indexable pipeline rather than a mode of it. What is given up is
stated in the file: no length, no random access and therefore no true shuffle
(the reservoir in `pipeline.tw` is the approximation), and one pass unless the
file can be reopened unchanged. A reopen of a file that changed size or mtime
during the run is refused, because half a run trained on one file and half on
another is a result nobody can interpret.

## Layout

```
src/
  pipeline.tw   map, filter, batch, shuffle, prefetch, repeat, and the order check
  cache.tw      disk cache keyed on the transformation chain
  augment.tw    image and sequence transforms, each taking an explicit seed
  rng.tw        deterministic seeding, splitting and permutation
  datasets.tw   descriptions, verification, and the IDX and CSV readers
  stream.tw     chunked reading for data larger than memory
  sample.tw     what moves through a pipeline
  strutil.tw    parsing, because the subset has none
tests/          one file per source file, harness.tw is the runner
docs/needs.md   what the language still owes this code
docs/datasets.md  the digests, and what is still to verify
```

## The sibling repositories

- [twill](https://github.com/twill-lang/twill), the language.
- [spool](https://github.com/twill-lang/spool), the package manager. warp
  depends on it for `src/sha256.tw` rather than writing a second digest that
  would have to agree with it byte for byte.
- [weft](https://github.com/twill-lang/weft), plotting. warp loads it, weft
  draws it.

## Licence

MIT.
