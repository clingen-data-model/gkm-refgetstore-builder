# Publish the full SeqRepo as a single archive

`seqrepo/full-2026-10-29.tar.gz` is the SeqRepo instance built from the full
store ([`../2026-10-30-full-seqrepo/`](../2026-10-30-full-seqrepo/README.md))
as one gzipped tar, published to GCS and R2. It adds no data.

| | |
| --- | --- |
| GCS | <https://storage.googleapis.com/clingen-public/seqrepo/full-2026-10-29.tar.gz> (`gs://clingen-public/seqrepo/`, us-east1) |
| R2 | <https://static.ferriter.dev/seqrepo/full-2026-10-29.tar.gz> (`theferrit32-public/seqrepo/`) |
| size | 8,114,076,347 bytes (the instance is 13,307,693,514) |
| SHA-256 | `07682b24416145576fa5775bd8ff8d4777d3fea29c5026d1b300cb17be9d6d96` |

Beside the archive at both locations are `full-2026-10-29.tar.gz.sha256` and the
build's `full-2026-10-29.provenance.json` (verification counts, the 178
sequences left out, versions, every file's SHA-256). The archive holds the
single directory `full-2026-10-29/` with the instance's 1,292 files and
`aliases.sqlite3` at its top. clinvar-ingest's `vrsify --seqrepo` accepts the
archive URL: it verifies the `.sha256`, extracts it and finds the instance one
level down.

## How it was made

With the same flags as the earlier archives: no AppleDouble entries or extended
attributes, `root:root` ownership.

    COPYFILE_DISABLE=1 tar --no-mac-metadata --no-xattrs --no-acls \
        --uid 0 --gid 0 --uname root --gname root \
        -C /Volumes/RefgetStores/seqrepo -cf - full-2026-10-29 | pigz -6

It took 44 s. The archive is 61% of the instance: the sequences are already
bgzip-compressed, but the 6.3 GB `aliases.sqlite3` compresses well.

- **GCS** (`upload-gcs`): `rclone copy --checksum` of the three files through a
  GCS remote defined only by `RCLONE_CONFIG_CLINGENGCS_*` environment variables
  (gcloud application-default credentials). 27.5 minutes.
- **R2** (`upload-r2-2`): `rclone copyto --checksum` of each file to its full
  key under `seqrepo/`. The bucket also holds a zero-byte object whose key is
  exactly `seqrepo`, so `rclone copy … theferrit32-public/seqrepo` resolves the
  destination to that object and fails ("is a file not a directory";
  `upload-r2`). That object was left in place.

## Validation

- Extracting the archive gives a tree identical to the instance directory
  (`diff -rq`), with one top-level directory, 1,292 files and no `._` entries
  (`verify-archive`).
- GCS (`check-gcs`): `rclone check`, 0 differences across the three files.
- R2 (`check-r2`): each file's size and MD5 by key (`rclone size`,
  `rclone md5sum`) equal the local file's; `rclone check` cannot open the
  `seqrepo` path for the reason above.
- Unauthenticated HTTPS serves all three at full length from both locations,
  and each served `.sha256` holds the archive's digest.
