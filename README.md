# dgx-vllm

Automated **mirror** of the prebuilt [eugr/spark-vllm](https://hub.docker.com/r/eugr/spark-vllm)
and [eugr/spark-vllm-b12x](https://hub.docker.com/r/eugr/spark-vllm-b12x) nightly container
images (built from [eugr/spark-vllm-docker](https://github.com/eugr/spark-vllm-docker)).

Instead of rebuilding vLLM from source, this repo watches eugr's Docker Hub images by
**manifest digest** and, whenever one changes (a new or redone nightly), snapshots it
under our own permanent dated+sequence tag. eugr only retains a handful of dated tags;
we keep a permanent history plus JSON metadata for reverse lookup.

## Container Variants

| Image                                           | Upstream source                 | Description                          |
|-------------------------------------------------|---------------------------------|--------------------------------------|
| `ghcr.io/spark-arena/dgx-vllm-eugr-nightly`     | `docker.io/eugr/spark-vllm`      | Mirrored vLLM nightly                 |
| `ghcr.io/spark-arena/dgx-vllm-eugr-nightly-tf5` | `docker.io/eugr/spark-vllm`      | Deprecated back-compat alias (same content/digest as `-nightly`) |
| `ghcr.io/spark-arena/dgx-vllm-eugr-nightly-b12x`| `docker.io/eugr/spark-vllm-b12x` | Mirrored b12x variant nightly (separate upstream lineage) |
| `ghcr.io/spark-arena/dgx-vllm-eugr-nightly-wheels` | eugr GitHub releases          | Wheels carrier (`.whl` files) |

> `-tf5` is deprecated. The upstream `--tf5` build flag no longer produces a separate
> lineage, so `-nightly-tf5` is published as an identical alias of `-nightly` purely for
> backward compatibility.

> `-b12x` is a genuinely distinct upstream image, not an alias — it tracks its own digest
> and is mirrored independently of `-nightly`. The wheels carrier is variant-independent
> (it comes from eugr's GitHub release assets, not from any image).

## Tagging Scheme

Each mirror publishes 3 tags per image:

- **`YYYYMMDDNN`** — Monotonically increasing snapshot identifier (NN = 01, 02, ...),
  computed in US/Pacific. A same-day "redo" upstream produces the next `NN`.
- **`YYYYMMDD`** — Overwritten to point to the latest snapshot of that day
- **`latest`** — Always the most recent snapshot

The `YYYYMMDDNN` sequence is **shared across variants**: a run that mirrors only `-b12x`
still consumes the next sequence number, so a given tag is not guaranteed to exist on
every image. Use `latest` (or `build-index.json`) to resolve a variant's newest snapshot.

## Provenance & Reverse Lookup

The mirrored images are eugr's exact layers with our `dev.sparkrun.*` provenance labels
added. Because adding labels rewrites the image config, **our published digest
intentionally differs from eugr's** — both are recorded so you can reverse-look-up from
either.

`build-index.json` maps each published snapshot back to its fingerprints:

| Field            | Meaning                                                        |
|------------------|----------------------------------------------------------------|
| `tag`            | Our `YYYYMMDDNN` snapshot id                                    |
| `variant`        | `nightly`, `nightly-tf5`, or `nightly-b12x`                     |
| `image_digest`   | **Our** published manifest digest (has labels)                 |
| `source_image`   | eugr's Docker Hub repo this snapshot came from                  |
| `source_digest`  | eugr's Docker Hub manifest digest                              |
| `source_tag`     | eugr's dated tag at mirror time (e.g. `nightly-20260704`)       |
| `repo_commit`    | `build_script_commit` from the image's `build-metadata.yaml`    |
| `vllm_hash`      | `vllm_commit`                                                   |
| `vllm_version`   | `vllm_version`                                                  |
| `flashinfer_hash`| `flashinfer_commit`                                            |
| `pushed_at`      | UTC timestamp                                                   |

The same `dev.sparkrun.*` values are baked into each image as OCI labels
(`docker inspect`). State tracking files:

- `upstream-state.json` — last mirrored digest per source image (`last_digest`,
  `last_digest_b12x`) + eugr GitHub state (change detection)
- `sequence.txt` — last computed `YYYYMMDDNN`
- `wheels-state.json` — sha256 fingerprints of the mirrored wheels
