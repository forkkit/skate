# skate

Fast sorted key extraction and extra utilities.

## Problem

Handling a TB of JSON and billions of documents without big data tooling;
especially the following use case:

* deriving a key from a document
* sort documents by (that) key

One use case is match candidate generation for deduplication.

## Transformation

We take jsonlines as input and extract id and derive the key. The resulting
file will be a TSV of the shape:

```
ID    KEY    DOC
```

The key will be sorted (optionally, but typical for the use case).

## Why an extra command?

We had a python program for this, which we parallelized with the great [GNU
parallel](https://www.gnu.org/software/parallel/) - however, when sharding the
input with parallel the program worked on each chunk; hence probably miss
clusters (not a problem of parallel, but our code, but still).

## Usage

```
$ skate-sorted-keys < release_entities.jsonl | skate-cluster > cluster.jsonl
```

A few options:

```
$ skate-sorted-keys -h
Usage of skate-sorted-keys:
  -S    skip sorting
  -b int
        batch size (default 50000)
  -compress-program string
        compress program, passed to sort (default "zstd")
  -f string
        key function name (default "tsand")
  -verbose
        show progress
  -w int
        number of workers (default 8)
```

Clusters are json lines;

* a single string as key `k`
* a list of documents as values `v`

The reason to include the complete documents is performance - for simplicity
and (typically) sequential reads, a "file" seems to be a good option.

```json
{
  "k": "植字手引",
  "v": [
    {
      "abstracts": [],
      "refs": [],
      "contribs": [
        {
          "index": 0,
          "raw_name": "大久保, 猛雄",
          "given_name": "大久保, 猛雄",
          "role": "author"
        }
      ],
      "language": "ja",
      "publisher": "広島植字研究会",
      "ext_ids": {
        "doi": "10.11501/1189671"
      },
      "release_year": 1929,
      "release_stage": "published",
      "release_type": "article-journal",
      "webcaptures": [],
      "filesets": [],
      "files": [],
      "work_id": "aaaab7poljf25dg4322ebsgism",
      "title": "植字手引",
      "state": "active",
      "ident": "bc5mykteevcy3masrst3zjqgwq",
      "revision": "97846ea8-41e5-40aa-9d41-e8c4b45f67e4",
      "extra": {
        "jalc": {}
      }
    }
  ]
}
```

Options:

```
$ skate-cluster -h
Usage of skate-cluster:
  -d int
        which column contains the doc (default 3)
  -k int
        which column contains the key (one based) (default 2)
```

## Performance notes

* key extraction at about 40k/s, 2B docs would take 13h
* having pipes in Go, on the shell or not at all seems to make little difference
* using jsoniter, parallel decode is at around 50MB/s

Note: need to debug performance at some point; e.g.

```
$ zstdcat -T0 refs_titles.tsv.zst | TMPDIR=/fast/tmp LC_ALL=C sort -S20% | \
    LC_ALL=C uniq -c | zstd -c9 > refs_titles_unique.tsv.zst
```

takes 46min, and we can iterate of 2-5M lines/s.
