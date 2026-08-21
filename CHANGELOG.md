# Changelog

Every entry below is a commit in this repository. Nothing is listed that I
cannot point at.

## Unreleased

Fifteen commits since `v0.1.0`. Almost all of them are either following twill's
releases or fixing what following them exposed, which is the shape of a library
written before the subset it is written in existed.

### It runs

- The 5 suites under `tests/` pass. `twill test tests` on 1.7.1 reports
  `5 file(s): 5 passed, 0 failed`, and CI runs that rather than gating on the
  prose in the README.
- `twill check` is clean on every `.tw` file in the repository.
- `examples/train.tw` runs as far as the dataset digest check and refuses
  there, which is the behaviour it is written to have. Pinning the digests is
  the open item in `docs/datasets.md`.

### The language floor moved twice

- Onto twill 1.6 (`ee7845d`): a pipeline stage became an enum with an
  exhaustive `match` instead of an integer tag, and paths are built with
  `path_join` instead of string concatenation.
- Onto twill 1.7 (`7b79606`). The suite passed unchanged: nothing here used the
  pattern language or the type parameters 1.7 adds, and both are additions at
  positions that were previously syntax errors. CI has been pinned through
  1.6.0-rc2, 1.6.0 to 1.6.6, 1.7.0 and now 1.7.1, one commit per release.
- `docs/needs.md` was corrected each time rather than left to rot. Entry 12
  asked for generic containers over user types and flagged the estimate as
  uncertain because of monomorphization; that half turned out never to be
  needed, because twill's runtime is a tree walker over dynamically typed
  values and the parameters only ever had to reach the checker.

### Fixed

- `datasets.verify` returned a `Str` that was empty on success. It returns
  `Res[Unit, Str]`, and the example takes the `Err` arm instead of testing a
  length (`6fd6dce`). It was the last error channel in warp that was not
  already a `Res`.
- `examples/train.tw` called `main()` itself, so it built the pipeline twice.
  `twill run` executes a systems-mode file's top level and then calls `main()`,
  which it has done since twill 1.6.1 (`6fd6dce`).

### CI

- Checks every `.tw` file in the repository rather than `src tests examples`
  (`6c4ec79`). Two sibling repositories had real programs outside those three
  directories and both were broken because the gate could not see them.
- Records in the workflow why it does not run the examples (`5639493`):
  `examples/train.tw` needs a dataset download the job deliberately does not
  make.

### Moved

- Repository URLs and the module path point at the `twill-lang` organisation
  (`ca2c230`).

## v0.1.0

The first cut: the pipeline, the cache, the augmentations, the seeding, the
streaming reader, the dataset descriptions and the tests, none of which ran,
because `mode systems` did not yet have the function values, floats and file IO
they are written against. `docs/needs.md` was written alongside the code and is
the list of what was missing.

- `549bb2f` warp: data pipelines and dataset loaders for twill, written in
  twill.
- `01ffac8` A pipeline and a batch carry a declared dtype, so a consumer builds
  its tensor at the width the data was meant for rather than at f64 and a
  narrowing after. A declaration, not a storage change: the buffer is still
  `Arr[F64]` and a narrow store waits on twill NEEDS-111.
- `d3607d6` Port to twill 1.5.
