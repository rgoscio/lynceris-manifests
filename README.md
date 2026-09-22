# Lynceris Gulf Chokepoint Reader - daily integrity manifests

This repository contains one manifest per day for the
[Gulf Chokepoint Reader](https://lynceris.com/reader/).

A manifest records **the state of the archive on that day**: which source
artefact was in force, what the reading contained, and what the register
recorded. It exists so that a third party can confirm that the published
data has not been altered after the fact.

Manifests contain hashes and metadata only. They contain no source
content, no local paths and no credentials.

---

## What a manifest answers

**What did the source say on day D.**

This is not the same as "which files were written on day D". When a
source publishes nothing new, the collector records a `dedup` entry and
the manifest points to the day of the artefact still in force. The chain
stays continuous even on days when nothing was written.

---

## What is hashed

**The raw bytes of the artefact as retrieved.** Not the compressed
version.

This is the first thing to get wrong: the register stores both
`sha256` (raw) and `sha256_gz` (stored, gzip). Manifests carry the raw
hash. If you receive an artefact from us it will be gzip-compressed for
transport; decompress it before hashing.

```
gunzip -c artefact.gz | sha256sum
```

---

## What is inside the hash, and what is not

A manifest has two parts.

**`signature`** carries `generated_at`, the generator version and
`content_sha256`. It is **not** covered by the hash, because the moment
of generation is not a property of the day being described.

**Everything else is inside the hash**: source and reading hashes,
`previous.content_sha256`, `known_gap`, `reconstructed_on`, `anchored`
and `anchor_note`.

The last three are deliberate. `reconstructed_on`, `anchored` and
`known_gap` are statements about the provenance of the document, not
metadata about the run that produced it. If they sat in `signature`,
`anchored` could later be switched to `true`, or a gap removed, without
changing the hash - which is the exact thing this repository exists to
prevent.

`reconstructed_on` is the date of the first reconstruction, written once.
It is not regenerated on each run, so it is stable and can live inside
the hash.

---

## Canonical form

A manifest is JSON, serialised so that the same state always produces the
same hash:

- keys sorted alphabetically at every level
- `sources` sorted by `list_name`, `artifacts` sorted by `source_url`
- separators `,` and `:` with no spaces
- UTF-8, non-ASCII characters written literally, not escaped
- LF line endings
- exactly one trailing newline
- no variable fields inside the hashed content

The Python equivalent:

```python
json.dumps(manifest, sort_keys=True, ensure_ascii=False,
           separators=(",", ":")) + "\n"
```

---

## How to verify

**1. Verify a manifest against itself.**

One rule: remove the `signature` key, canonicalise what remains, hash it.

```python
import json, hashlib
m = json.load(open("2026/09/2026-09-22.json", encoding="utf-8"))
sig = m.pop("signature")
body = json.dumps(m, sort_keys=True, ensure_ascii=False,
                  separators=(",", ":")) + "\n"
print(hashlib.sha256(body.encode("utf-8")).hexdigest()
      == sig["content_sha256"])
```

If this prints `False`, the file has changed since publication.

**2. Verify a manifest against the chain.**

Each manifest carries `previous.content_sha256` - the `content_sha256`
of the previous day's manifest. Compute it as above for the earlier day
and compare. If they differ, one of the two days has been altered.

The chain does not depend on when a manifest was generated, only on what
it states.

**3. Verify an artefact against a manifest.**

Find the source in `sources` by `list_name`. Take its `sha256`.

```
gunzip -c <artefact>.gz | sha256sum
```

The result must equal the `sha256` field. If `store_action` is `dedup`,
the artefact in force is the one from the day given in `effective_day`,
not from the day of the manifest.

**4. Verify the reading.**

`reading_markdown_sha256` is the hash of the Markdown source of that
day's reading, not of the published HTML page. The HTML differs: the
hosting layer injects an analytics beacon and rewrites font references
after publication.

---

## The chain

Every manifest carries the hash of the previous day's manifest. Changing
any past day invalidates every manifest after it, and the most recent one
is anchored in a commit here and in the Wayback Machine.

The chain covers days earlier than the first anchor. The anchor itself
begins where it really begins - see below.

---

## Fields

`reconstructed_on` - present when the manifest was produced later than
the day it describes, giving the date of that reconstruction. Days before
2026-09-23 were reconstructed on 2026-09-22.

`anchored` - `false` means that day has no commit or Wayback anchor of
its own. It is still part of the chain, because the first anchored
manifest carries the hash of the reconstructed one before it.

`known_gap` - states what the evidence for that day establishes and what
it does not. Each entry carries `establishes` and `does_not_establish`
as separate fields. These describe the scope of the evidence; they are
not an assessment of how much weight a reader should give it.

`addresses_registered` counts distinct circular addresses observed in the
register within the stated range. An address seen on more than one day is
counted once.

`artifacts_registered_not_fetched` - the number of documents whose
existence was recorded but whose content was not retrieved. The archive
keeps the content of what the reading cites; it records the existence of
the rest.

`anomaly_duplicate_source_entries` - present when a source produced more
than one entry for the same step on the same day. Two-stage sources have
a page entry and an artefact entry by design; those are not duplicates.
The key is the pair of source and step, so only a genuine repetition is
reported.

---

## Known gap, 18-22 September 2026

The collector did not write a page entry to the register on successful
retrieval, so the state of the P&I club listing pages was not archived
for those days. The circular artefacts themselves were archived and are
covered by the manifests.

This is recorded in `known_gap` in each affected manifest, per source,
with the date range and the number of addresses registered. The readings
published on those days carry a line stating the same.

The defect was fixed on 2026-09-22. Manifests from 2026-09-23 onward
carry no gap of this kind.

---

## Contact

Errors in a manifest, or a verification that does not reproduce:
contact@lynceris.com
