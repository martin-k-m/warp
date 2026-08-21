# Datasets

warp ships no data. Every dataset below is a description: where it came from,
who made it, under what terms, which files it consists of, and how to read the
format once a file is on disk. Fetching is the user's step, and so is agreeing
to whatever the publisher asks.

## The digests are not pinned

`src/datasets.tw` carries a `sha256` field per file and every one of them is
currently the empty string, which `verify` treats as a failure. That is the
honest state of this repository and not an oversight.

A digest copied from a third-party page without checking it against the real
file is worse than no digest: it looks like verification, it will match whatever
that third party is serving, and it silences the one check that would have
caught a substitution. Pinning them needs the files in hand and a machine that
downloaded them from the publisher.

Until then `verify` refuses and prints the digest it computed, in the form:

```
train-images-idx3-ubyte.gz has no pinned digest in warp. Its sha256 is <hex>.
Record it in src/datasets.tw and in docs/datasets.md before training on it.
```

Refusing rather than warning is deliberate. A warning in a training log is not
read, and the cost of a false refusal is a minute of someone's attention while
the cost of a false pass is an experiment run on the wrong bytes.

**Open item:** download each file from the URL in `src/datasets.tw`, verify it
against the publisher's own checksum where one is published, and record the
digest in both places.

Running `verify` and pasting back what it printed does not close this. That
records what warp read off whatever the mirror served, which is the thing the
digest is supposed to be independent of. The README shows the refusal, digest
and all, precisely because that number is not yet worth pinning.

Expect it to be slow. `std/hash` is SHA-256 written in twill, and MNIST's
9.9 MB training file took about four minutes on the machine that produced the
line in the README. `verify` checking the size first is what keeps that cost
off the common failure.

## The size check comes first

`verify` compares the file size before it computes a digest. Reading 170
megabytes to discover that the download was an HTML error page is a slow way to
learn it, and a wrong size is the overwhelmingly common failure.

## MNIST

- **Origin.** Yann LeCun and Corinna Cortes. Derived from NIST Special
  Databases 1 and 3.
- **Cite.** LeCun, Bottou, Bengio and Haffner, Gradient-based learning applied
  to document recognition, Proc. IEEE 86(11), 1998.
- **Licence.** CC BY-SA 3.0, as stated on the distribution page. The underlying
  NIST databases are US government work; the split and normalisation are the
  authors'.
- **Format.** IDX, gzipped. 60,000 training and 10,000 test images, 28 by 28,
  one channel, unsigned bytes. Labels 0 to 9.
- **Note.** The original `yann.lecun.com` host has been unreliable for years.
  warp points at the CVDF mirror on Google Cloud Storage, which serves the same
  four files. The digest, once pinned, is what makes that mirror safe to use.

## Fashion-MNIST

- **Origin.** Zalando Research.
- **Cite.** Xiao, Rasul and Vollgraf, Fashion-MNIST: a novel image dataset for
  benchmarking machine learning algorithms, arXiv:1708.07747, 2017.
- **Licence.** MIT.
- **Format.** Identical to MNIST, deliberately, so the same reader handles both.
- **Note.** Ten clothing categories, not digits. It exists because MNIST is too
  easy to distinguish between methods, and swapping one for the other is a
  one-line change here for exactly that reason.

## CIFAR-10

- **Origin.** Alex Krizhevsky, Vinod Nair and Geoffrey Hinton, University of
  Toronto. A labelled subset of the 80 Million Tiny Images collection.
- **Cite.** Krizhevsky, Learning Multiple Layers of Features from Tiny Images,
  technical report, University of Toronto, 2009.
- **Licence.** **Not stated by the authors.** They ask that the technical report
  be cited. That is a request, not a licence, and warp says so rather than
  inventing one. Check the terms before redistributing anything derived from it.
- **Format.** Binary, in a tar archive. 50,000 training and 10,000 test images,
  32 by 32, three channels, one label byte per record.
- **Note.** The parent collection, 80 Million Tiny Images, was withdrawn by its
  authors in 2020 over offensive labels and imagery in the unfiltered set.
  CIFAR-10's ten classes are hand-verified and are not implicated, but anyone
  citing the lineage should know it.

## Iris

- **Origin.** Ronald Fisher, 1936, from measurements by Edgar Anderson.
  Distributed by the UCI Machine Learning Repository.
- **Cite.** Fisher, The use of multiple measurements in taxonomic problems,
  Annals of Eugenics 7(2), 1936.
- **Licence.** CC BY 4.0 as distributed by UCI.
- **Format.** CSV, 150 rows, four features, three classes.
- **Note on the download.** The URL in `src/datasets.tw` is UCI's zip, and the
  file warp names and sizes is `iris.data`, the CSV inside it. Unzip first, the
  same manual step the gzipped IDX files need and for the same reason: twill
  cannot decompress. See entry 9 of `docs/needs.md`.
- **Note.** Published in a eugenics journal by an author who was a proponent of
  it. It is included because it is everywhere and small enough to be useful in a
  test, and the provenance is recorded because people should know what they are
  citing.

## Adding a dataset

Four things, and none of them is the data:

1. A `Dataset` in `src/datasets.tw` with the origin, the citation and the
   licence position filled in. An empty licence field fails the tests.
2. A `File` per artefact with its URL and expected size.
3. A digest, verified against a file you downloaded yourself.
4. A section here, including anything a user should know before publishing a
   result on it.
