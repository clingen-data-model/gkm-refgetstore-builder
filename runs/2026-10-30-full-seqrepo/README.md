# A SeqRepo instance mirroring the full store

`/Volumes/RefgetStores/seqrepo/full-2026-10-29` is a biocommons SeqRepo
instance holding the sequences and sequence aliases of the full RefgetStore
`/Volumes/RefgetStores/full-2026-10-29`
([`../2026-10-29-biological-region/`](../2026-10-29-biological-region/README.md)),
apart from 178 Ensembl proteins that SeqRepo cannot file under their digest.

| | SeqRepo instance | RefgetStore |
| --- | ---: | ---: |
| sequences | 1,909,742 | 1,909,920 |
| residues | 52,499,564,896 | 52,499,735,021 |
| sequence aliases (accession and assembly namespaces) | 18,134,407 | 18,139,427 |
| digest aliases (`VMC`, `MD5`, `SHA1`, `SEGUID`) | 7,638,968 | — |
| files | 1,292 | |
| bytes | 13,307,693,514 | |

## How it was built

    gks-refgetstore seqrepo --config sources.toml --lock build.lock.json \
        --store-dir /Volumes/RefgetStores/full-2026-10-29 \
        --target-dir /Volumes/RefgetStores/seqrepo --instance-name full-2026-10-29

with `biocommons.seqrepo` 0.6.11, `bioutils` 0.6.2, `pysam` 0.24.1, htslib
`bgzip` 1.24, and gtars 0.11.0 from `theferrit32/gtars@7831e09`. The run
(`build-3`) took 9,594 s to store every sequence and 104 s to read them all
back.

- **Store and lock.** The store holds exactly the 239 collections `sources.toml`
  selects from `build.lock.json` (strict check), with the locked sizes, and no
  other sequence.
- **Writes.** Each store sequence was passed once to the seqrepo library's
  `SeqRepo.store(seq, aliases)` with every alias the store holds for it, then
  `commit()`. Seqrepo computed every `seq_id`, wrote the bgzip FASTA files with
  their `.fai`/`.gzi` indexes, inserted the alias rows, and added its four
  digest aliases per sequence. Every `seq_id` equals the store's sha512t24u
  digest.
- **Namespaces.** In `aliases.sqlite3`, the store's `refseq` is `NCBI`
  (1,045,010), `ensembl` is `Ensembl` (1,052,528) and `lrg` is `LRG` (5,905).
  `insdc`, the 42 release namespaces `ensembl-75`–`ensembl-116`, and the 25
  assembly namespaces are stored under the store's names.
- **Assembly names.** `seqrepo add-assembly-names` was not run. For all 25
  assembly namespaces the store shares with bioutils' assembly data, no
  bioutils name is missing or points at a different sequence; that command would
  add only 29 `NCBI34`/`NCBI35` names the store does not have.

## Sequences left out

Before each `store()`, the builder passes the sequence to the function
`store()` computes its `seq_id` with (`bioutils`' `sha512t24u`, after seqrepo's
`normalize_sequence`). A sequence that function renames, or rejects, is left
out of the instance together with its aliases and recorded with its characters
outside A–Z. The rule is seqrepo's own, so it covers any future input of the
same kind without a list of names.

In this store it left out 178 sequences with 5,020 aliases, all of them renamed
and none rejected. Every one is an Ensembl protein (`ENSP…`) containing `*`:
76 have a single `*`, 102 more than one. Seqrepo strips `*` before hashing, so
it would file each under the digest of the star-less sequence rather than the
store's digest, and its aliases would resolve to a sequence other than the one
the store holds for them. 113 of the 178 are in Ensembl 116 (`ensembl`). For 12
the star-less sequence is itself a store sequence, which the instance holds
under its own digest and aliases. The store keeps all 178.

[`artifacts/left-out.tsv`](artifacts/left-out.tsv) lists each one: the store
digest, the `seq_id` seqrepo would give it, whether that `seq_id` is a store
sequence, length, number of `*`, and its aliases.

[`artifacts/full-2026-10-29.provenance.json`](artifacts/full-2026-10-29.provenance.json)
records the manifest, lock and store digests, the tool versions, the counts,
the left-out sequences, the verification result, and the size and SHA-256 of
every file in the instance. The same file sits beside the instance.

## Attempts

The record holds three runs of the same command. The first two were stopped
(exit 143) and their partial instances discarded; each restart ran changed code.

| label | stored before stopping | code | why stopped |
| --- | --- | --- | --- |
| `build` | 1,195,629 sequences, 33.50 Gbp, 6,271 s | left-out rule only | memory |
| `build-2` | 394,213 sequences, 13.51 Gbp, 1,953 s | + scope check on its own store | to add the alias page cache |
| `build-3` | all, 9,594 s | + alias page cache | finished |

- **Memory.** Activity Monitor showed 17.8 GB for `build`. The scope check loads
  every collection to compare it with the lock, and gtars keeps a loaded
  collection for the life of the store object: 10.8 GB on this store, on top of
  the 4.1 GB `open_local` takes. The write reused that store object and never
  reads collections. The check now gets a store of its own, dropped when it
  returns.
- **Speed.** The write slows as `aliases.sqlite3` grows (6.3 GB at the end):
  each alias row updates four indexes in no useful key order. The builder sets
  `pragma cache_size` on seqrepo's alias connection to 4 GiB (sqlite's default
  is 2 MB); seqrepo still writes every row. `build-3` reached 33.50 Gbp in
  5,565 s, 11% sooner than `build`. The remaining cost is about 3 ms per
  sequence of seqrepo's sqlite work plus 60–100 ms per Mbp, so the Mbp/s in the
  log follows the mix of sequence lengths.

The manifest's `git` block was recorded at `init`, before the scope-check and
page-cache changes. [`artifacts/build-3-code.diff`](artifacts/build-3-code.diff)
is the working-tree diff against `338b507` that `build-3` ran (SHA-256
`c7cfbab4…efe7`).

## Validation

- **Built-in verification** (`build-3`): every sequence read back from SeqRepo
  re-hashes to its `seq_id` and matches the store's length and MD5. The store
  sequences missing from SeqRepo are exactly the ones the left-out rule selects;
  verify finds them by applying the rule to each such sequence, not from the
  write's record. SeqRepo holds no sequence the store lacks. The aliases are
  equal in both directions apart from the 5,020 on left-out sequences, and an
  alias only in SeqRepo would still fail. Each sequence has one alias in each
  digest namespace; no alias row is non-current.
- **Data proxies and VRS samples** were not run on this instance.

`tests`: the repository's 286 tests pass, including those for the left-out rule
and for a sequence filed under another digest stopping the build.
