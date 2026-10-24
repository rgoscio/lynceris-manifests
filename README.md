# Lynceris Gulf Chokepoint Reader - daily integrity manifests

This repository contains one manifest per day for the
[Gulf Chokepoint Reader](https://lynceris.com/reader/).

A manifest records **the state of the archive on that day**: which source
artefact was in force, what the reading contained, and what the register
recorded. It exists so that a third party can confirm that the published
data has not been altered after the fact.

Manifests contain no credentials. With the one exception described in
the correction below, they contain no source content and no local paths.
Each manifest names the sources and gives their status and, for each
source captured, its checksum and, where it applies, the number of items.
It lists the address of any public document newly archived that day and
states the reason for any known gap.

Correction, 7 October 2026. This file previously said that manifests
contain "no source content, no local paths and no credentials". That is
not accurate for the manifest of 2026-10-05. Its `anchored_late` section
holds 30 full registry lines - 29 documents and the sitemap they were
listed in - captured on 25 September 2026 and anchored later. Those
lines include local file paths, the capture host name, storage paths
and capture-method fields. They also include document titles and, in
some lines, short fragments extracted from the document page, for
example the wording of an effective date. They hold no full source text
and no credentials. The manifest is anchored and has not been altered.

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
day's reading, not of the published HTML page. The published HTML is
not byte-identical to the build output: the hosting layer rewrites font
references when it serves a page, and the order of the font rules it
inserts varies between requests. Two downloads of the same page can
therefore have different hashes although the reading has not changed.

Correction, 7 October 2026. This section previously said that the
hosting layer injects an analytics beacon. The beacon was switched off
on 5 October 2026. Until 7 October 2026 the hosting layer also added a
bot-detection script to pages of lynceris.com, which this file did not
mention. It was switched off on that day.

---

## The chain

Every manifest carries the hash of the previous day's manifest. Changing
any past day invalidates every manifest after it.

Each manifest is committed to this repository when it is generated. Two
further anchors are added in separate, later steps:

- We ask the Wayback Machine to capture the manifest, so that a third
  party holds a copy with its own capture time. That step can be delayed
  and it can fail, and this repository does not record its outcome.
- We timestamp the manifest file with OpenTimestamps and publish the proof
  in this repository, next to the manifest. See the next section.

Until a manifest, or a later one, has been captured by the Wayback Machine
or has a complete OpenTimestamps proof, its only anchor is its commit. A
later anchor counts because each manifest carries the hash of the one
before it.

The chain covers days earlier than the first anchor. The anchor itself
begins where it really begins - see below.

Correction, 7 October 2026. This section previously said that the most
recent manifest is anchored in a commit here and in the Wayback Machine.
That did not always hold: on 7 October 2026 the captures of the manifests
of 2026-10-06 and 2026-10-07 had not been confirmed.

---

## OpenTimestamps proofs

We have timestamped manifest files with OpenTimestamps since 7 October
2026, starting with the manifests of 6 and 7 October 2026. Since 21
October 2026 the manifest of each day is timestamped automatically on the
day it is published.

The proof of a manifest is published here, next to it:

```
2026/10/2026-10-11.json        manifest
2026/10/2026-10-11.json.ots    OpenTimestamps proof of that file
```

A proof covers the SHA-256 of the manifest file exactly as it is in this
repository. A complete proof ties that hash to a block of the Bitcoin
blockchain and shows that the file existed no later than about the time
of that block. It is a statement about time only: it does not show that
what the manifest records about a source is correct. Block times are set
by miners and can differ from the real time by up to about two hours,
which is why we write "about".

A new proof starts as pending: it refers to OpenTimestamps calendar
servers and does not yet point to a Bitcoin block. When it is completed,
usually by the next day, the file here is replaced by the completed
version. Each replacement contains everything the previous version did,
and the history of this repository keeps every version.

A day without a proof of its own is covered by the next later day that has
a complete proof, because each manifest carries the hash of the one before
it. These days have no proof of their own:

```
2026-09-18 to 2026-10-05   no later than about 2026-10-07 12:07 UTC   proof of 2026-10-06 (block 970337)
2026-10-08 to 2026-10-10   no later than about 2026-10-11 14:10 UTC   proof of 2026-10-11 (block 970941)
2026-10-16 to 2026-10-17   no later than about 2026-10-18 07:31 UTC   proof of 2026-10-18 (block 971956)
```

If a later day is missed, the same rule applies.

To check a proof, install the OpenTimestamps client
(`pip install opentimestamps-client`) and run, in a clone of this
repository:

```
ots verify 2026/10/2026-10-11.json.ots
```

The client compares the proof with the file next to it. Checking the
Bitcoin part needs access to a Bitcoin node; without one,
`ots info 2026/10/2026-10-11.json.ots` shows which Bitcoin block the proof
points to.

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

## Historical notes

### Reading of 18 September 2026

The content of that day's reading is in Polish; later readings are in
English. It is preserved unchanged, because its checksum is recorded in
the manifest for that day. The page itself carries a line stating this.

### UN Security Council Consolidated List: days without stored bytes

On the days listed below, our collector compared the UN list it had just
retrieved with the last stored copy of the list, ignoring the generation
date inside the file. Because the rest of the content was the same, it
did not store the bytes it had retrieved. The manifest for each of these
days records the SHA-256 of that day's retrieval and names the earlier
stored file (`effective_day`), which has its own SHA-256. For these
days, the checksum recorded in the manifest cannot be checked against
any stored file. According to the comparison made at the time of
retrieval, the content of the list on these days was the same as in the
earlier stored file, apart from the generation date.

Day of retrieval -> earlier stored file:

```
2026-09-22 -> 2026-09-20
2026-09-23 -> 2026-09-20
2026-09-24 -> 2026-09-20
2026-09-25 -> 2026-09-20
2026-09-26 -> 2026-09-20
2026-09-27 -> 2026-09-20
2026-09-28 -> 2026-09-20
2026-10-02 -> 2026-10-01
2026-10-03 -> 2026-10-01
2026-10-04 -> 2026-10-01
2026-10-05 -> 2026-10-01
2026-10-06 -> 2026-10-01
2026-10-08 -> 2026-10-07
2026-10-09 -> 2026-10-07
2026-10-10 -> 2026-10-07
2026-10-11 -> 2026-10-07
2026-10-12 -> 2026-10-07
2026-10-14 -> 2026-10-13
```

Days on which the retrieved bytes were identical to the stored file,
such as 21 September 2026 (identical to the file of 20 September 2026),
are not listed: their checksum can be checked against that file.

For this list, since 15 October 2026 we keep the bytes of every day's
retrieval.

---

## Contact

Errors in a manifest, or a verification that does not reproduce:
contact@lynceris.com
