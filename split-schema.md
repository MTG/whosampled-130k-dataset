# Split artifact format

The partition notebook writes six files into the split output directory. They
are the contract between this repository and the training code in
[raraz15/sample-identification](https://github.com/raraz15/sample-identification),
and are **not** committed here -- `train.json` alone is ~44 MB. They are shared
on request; see [Data availability](README.md#data-availability).

```
splits-27052026/
├── train.json
├── val.json
├── test.json
├── audio-paths-train.txt
├── audio-paths-val.txt
└── audio-paths-test.txt
```

Per-split relation and track counts are in the [README](README.md).

## `{train,val,test}.json`

A single JSON object mapping **edge id → relation record**. The edge id is the
WhoSampled sample id, repeated inside the record as `edge_id`.

```json
{
  "220631": {
    "url": "https://www.whosampled.com/sample/220631/The-Chemical-Brothers-Superflash-Roger-Roger-Nino-Nardini-Super-Flash/",
    "dst_song": "Superflash",
    "dst_artist": ["The Chemical Brothers"],
    "dst_audio_id": "wfKG58nxKkI",
    "src_song": "Super Flash",
    "src_artist": ["Roger Roger", "Nino Nardini"],
    "src_audio_id": "HBSoC3tsULM",
    "dst_timings": {"start_times": "13", "throughout": true},
    "src_timings": {"start_times": "6", "throughout": false},
    "part_sampled": "Vocals / Lyrics",
    "edge_id": "220631",
    "component_type": "isolated_pair",
    "component_idx": 21332
  }
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `url` | str | WhoSampled page for the relation |
| `src_song`, `src_artist` | str, list[str] | The track that **was sampled** (the source) |
| `dst_song`, `dst_artist` | str, list[str] | The track that **does the sampling** (the destination) |
| `src_audio_id`, `dst_audio_id` | str | YouTube id; also the audio filename stem |
| `src_timings`, `dst_timings` | obj | `start_times` (comma-separated seconds, as a string) and `throughout` (bool) |
| `part_sampled` | str | What was sampled, e.g. `Vocals / Lyrics`, `Drums`, `Bass`, `Multiple Elements` |
| `edge_id` | str | WhoSampled sample id; same as the dict key |
| `component_type` | str | Shape of the connected component this edge belongs to (below) |
| `component_idx` | int | Index of the component; **stable across the three splits** |

Direction matters: the edge runs source → destination. `src_timings` refers to
the offset within the source track, `dst_timings` within the destination track.

### `component_type`

The relations form a directed graph over tracks. Each edge is labelled with the
shape of its weakly connected component (counts from `train.json`):

| Value | Count | Meaning |
| --- | --- | --- |
| `tree` | 54,065 | Branching, acyclic: one track sampled by several, or sampling several |
| `isolated_pair` | 23,628 | Exactly two tracks, one edge |
| `isolated_chain` | 1,061 | A → B → C ... with no branching |
| `complex` | 357 | Everything else (multiple parents, dense components) |

## `audio-paths-{train,val,test}.txt`

One absolute path per line, pointing into the 16 kHz WAV tree, sharded two
levels deep by the first two characters of the YouTube id:

```
/projects/mtg/projects/sample-identification/datasets/whosampled-wav-16khz/wE/wE4JrM8OFts.wav
```

These are 16 kHz mono WAVs. The list is the set of tracks reachable from the
relations in the corresponding `.json`,
so it is what you feed to an embedding/inference pass.

**These paths are absolute and machine-specific.** Rewrite the prefix for your
own storage, or regenerate the files from the notebook.

## Partition guarantees

The split is done at the level of **connected components**, never individual
edges, and the partition notebook verifies all four properties before writing:

1. **Mutually exclusive at the component level** -- no component is split across
   train/val/test.
2. **Complete coverage** -- every component lands in exactly one split; none is
   dropped or duplicated.
3. **No track leakage** -- checked at the node level, not just the edge level.
   This is what actually guarantees a track never appears in two splits.
4. **External eval sets held out** -- no track belonging to Sample100 or
   SamplePairs appears in train or val. Components containing them are
   constrained into test up front, and count toward the test quota.

Property 3 is the one that matters and the one an edge-level split silently
violates: two different edges can share a track, so splitting on edges leaks the
track across splits even when no edge is duplicated.

Cleaning applied before the split: cycle removal, isolated-node removal,
WhoSampled ID collision resolution (internal ids are used instead), and dropping
edges whose `part_sampled` is `Dialogue`.
