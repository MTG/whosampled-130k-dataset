# WhoSampled-130k

Dataset construction for automatic music sample identification: sampling-graph
analysis and the train/val/test partition behind **91,967 annotated sample
relations over 131,539 tracks**, mined from [WhoSampled](https://www.whosampled.com).

This repository covers the **data** side only -- how the graph is built, cleaned,
analysed, and split. Model code, training, and evaluation live in
[raraz15/sample-identification](https://github.com/raraz15/sample-identification),
which consumes the split artifacts described in [split-schema.md](split-schema.md).

## Data availability

> **TODO:** link the Zenodo page here once it is live.

We share the dataset for non-commercial scientific research purposes only, upon
request, through a Zenodo page. Only the final splits are distributed there:
`train.json`, `val.json` and `test.json`, whose format is
documented in [split-schema.md](split-schema.md). The `audio-paths-*.txt` files
are not released -- they hold absolute paths from our cluster. No audio is
redistributed.

## What is here

```
notebooks/
  analysis-and-partition-whosampled.ipynb   The main pipeline: graph -> clean -> split -> verify
  analysis-Sample100.ipynb                  Sample100 eval set, and its overlap with WhoSampled
  analysis-SamplePairs.ipynb                SamplePairs eval set, same
  sample100-graph.ipynb                     Sampling-graph figures for Sample100
data/                       Small curated id mappings the notebooks depend on
split-schema.md             Format of the generated split artifacts
```

## Setup

```bash
conda env create -f environment.yml
conda activate whosampled-130k
```

The notebooks are committed as-run, with outputs, and hardcode absolute paths in
their first cells. Point those at your own copies of the data before running
anything.

## The pipeline

**1. Build the graph.** `analysis-and-partition-whosampled.ipynb` reads the
WhoSampled annotations (`whosampled_251120_pairs.clean.download_complete` --
pairs where both sides were downloaded) and builds a directed graph: an edge
runs from the track that was sampled to the track that samples it.

**2. Clean it.** Cycles are removed, along with the annotations behind them.
Isolated nodes are dropped. WhoSampled sample-id collisions are resolved in
favour of our own internal ids. Edges whose `part_sampled` is `Dialogue` are
filtered out.

**3. Partition.** The split is done over **weakly connected components**, never
individual edges. Components containing tracks from the external evaluation sets
are placed into test first and count toward its quota; the remaining components
are shuffled with a fixed seed and distributed smallest-bucket-first, with train
absorbing the remainder.

**4. Verify.** Before anything is written, the notebook checks that the result is
a clean partition -- no component split across two splits, complete coverage, no
track leakage at the node level, and no evaluation-set track reaching train or
val. The four properties are stated in full in
[split-schema.md](split-schema.md#partition-guarantees).

**5. Write.** Six files land in the split output directory:

| Split | Relations | Tracks |
| --- | --- | --- |
| train | 79,111 | 114,721 |
| val | 4,393 | 6,374 |
| test | 8,463 | 10,444 |

They are not committed here -- `train.json` alone is ~44 MB. Their format is
documented in [split-schema.md](split-schema.md), and they are available on
request (see [Data availability](#data-availability)).

## Evaluation sets

Two external sample-identification datasets are held out entirely, so that
neither can leak into training:

- **Sample100** (Sample ID Dataset 2.0) -- 106 sample relations over 144 tracks, of
  which 99 are matched to WhoSampled relations in `data/`. (Its own readme says
  105; `samples.csv` actually carries 106.)
- **SamplePairs** -- 100 sample relations used, of the 103 in its source metadata.

The two sets link to WhoSampled differently. Sample100 ships only track titles
and artists, so its relations had to be matched to WhoSampled by hand.
SamplePairs already carries the WhoSampled URL for every pair, so the work there
was curation rather than matching. Both results live in `data/`, which is what
makes the hold-out reproducible. `analysis-Sample100.ipynb` and
`analysis-SamplePairs.ipynb` document that work, including the duplicate
relations found in both sets.

## Curated data

`data/` holds the small hand-made files the notebooks depend on. None can be
regenerated automatically -- they encode manual matching and curation work.

| File | Rows | What it is |
| --- | --- | --- |
| `sample100-whosampled-edge-ids.csv` | 99 | Maps each Sample100 relation id (`S001`, ...) to its WhoSampled sample id. Matched by hand. |
| `sample100-whosampled-original-urls.tsv` | 99 | Per Sample100 relation: the WhoSampled URL plus resolved YouTube URLs for both tracks. |
| `sample100-ground-truth.json` | 106 | Sample100 ground truth as `{sample_id: {src_audio_id, dst_audio_id}}`, using Sample100's `T###` ids. |
| `samplepairs-track-ids.csv` | 103 | Every SamplePairs relation as identifiers only, plus which ones we use. |

`samplepairs-track-ids.csv` is laid out as:

```
pair_id,status,dst_track_id,dst_youtube_id,src_track_id,src_youtube_id,whosampled_id
1,used,T001,6jgBcsRCqI8,T002,1dSRzxg7p1E,1178057
```

`src` is the track that was **sampled**, `dst` the one that **does the
sampling** -- the same direction as the split records. The YouTube id is also
the audio filename stem, and `whosampled_id` is the join key to the main
dataset. `status` records our curation of the 103 source relations: 100 `used`,
2 `commented_out` (pairs 26 and 87), 1 `removed` (pair 99).

**Known inconsistency:** the partition notebook reads the working copy with
`pd.read_csv(..., delimiter=',')` and no `comment='#'`, so pairs 26 and 87 parse
as ordinary rows and their tracks are still held out of train and val. This is
conservative -- it excludes more than intended, never less -- but the two
exclusions are not actually applied. The notebooks are committed as-run, so this
is documented rather than silently corrected.

### Provenance and licensing

`sample100-whosampled-edge-ids.csv` and `sample100-whosampled-original-urls.tsv`
are our own work. `sample100-ground-truth.json` reformats ground truth from the
**Sample ID Dataset 2.0** (Sample100), distributed under the **GPL-3.0**; its
track and relation ids are Sample100's. This repository is GPL-3.0 as well, so
redistributing that derivative is compatible with the upstream terms.

`samplepairs-track-ids.csv` is deliberately limited to identifiers and our own
inclusion decisions. SamplePairs ships no license or provenance statement, so
its metadata compilation -- artist, title, year, sample start times -- is **not**
redistributed here.

Also not vendored: Sample100's `samples.csv`, `tracks.csv` and
`youtube-links.csv`, and SamplePairs' `metadata.tsv` and `samples.csv`. The
notebooks read those directly from the evaluation-set directories.

## Citation

If you use this dataset, please cite **both** the paper and WhoSampled as the
source of the data.

```bibtex
@misc{araz2026building,
  author       = {Araz, R. Oguz and Lizarraga, Xavier and Serra, Xavier and Bogdanov, Dmitry},
  title        = {Building a Dataset for Music Sample Identification},
  howpublished = {Extended Abstracts for the Late-Breaking Demo Session of the 27th Int. Society for Music Information Retrieval Conf.},
  year         = {2026}
}
```

## License

[GPL-3.0](LICENSE).
