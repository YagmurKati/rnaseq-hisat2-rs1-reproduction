# RNA-seq analysis workflow (HISAT2, RS1) — Kubernetes reproduction guide

This repository documents **how to run** the *RNA-seq analysis workflow
(HISAT2, RS1)* on a Kubernetes cluster. It does **not** contain the workflow
itself — the workflow code lives upstream and is only reproduced here.

> **Cite the original workflow, not this repository.**
> The workflow is [`Nine-s/nextflow_RS1_hisat2_new`](https://github.com/Nine-s/nextflow_RS1_hisat2_new)
> by **Ninon De Mecquenem** (CRC 1404 **FONDA**, subproject **A2 — Energy-Aware
> Optimization of Workflows in Bioinformatics**). This repository adds only the
> deployment/run instructions that the upstream repository does not include, so
> that the run can be reproduced. VIVO run record:
> <https://vivo-fonda.hu-berlin.de/vivo/individual?uri=http%3A%2F%2Fexample.org%2Fvivo-import%2Frun-metadata%2Fworkflow%2Frna-seq-analysis-hisat2-rs1>

The upstream repository ships `main.nf`, the `modules/`, and helper `bin/`
scripts, but **no `README`, no `nextflow.config`, and no run instructions**.
Reproducing it therefore requires the details collected below.

---

## 1. What the workflow does

A DSL2 Nextflow RNA-seq pipeline for *Drosophila melanogaster*:

```
CHECK_STRANDNESS ─┐
FASTP ────────────┤
EXTRACT_EXONS ─────┐
EXTRACT_SPLICE_SITES┤→ HISAT2_INDEX_REFERENCE → HISAT2_ALIGN → SAMTOOLS → CUFFLINKS
```

Per sample: strandedness inference, adapter/quality trimming (fastp),
splice-aware HISAT2 alignment against a GTF-derived index, BAM sort (samtools),
and transcript assembly/quantification (Cufflinks).

## 2. Details missing from the upstream repo (read this first)

### 2.1 `params.mode` is **mandatory**
`main.nf` runs the alignment branch only if `params.mode` is set. There is **no
default**, so if you forget it the pipeline finishes without aligning anything
(no BAMs, `SAMTOOLS`/`CUFFLINKS` get no input). Choose one:

| `params.mode` | Behaviour |
| --- | --- |
| `exon_splice_site` | Splice-aware index from GTF exons + splice sites; aligns **FASTP-trimmed** reads. Standard RNA-seq. **Used for the reference run.** |
| `minimum_genome_build` | Plain genome index; aligns **raw** reads. Lighter, not splice-aware. |

### 2.2 Required parameters
`reads`, `reference_genome`, `reference_cdna`, `reference_annotation`,
`reference_annotation_ensembl`, `mode`, `threads`, `outdir`, and `baseDir`
(the `python`-labelled helpers call `python ${params.baseDir}/bin/*.py`, so
`baseDir` must point at the checked-out source). All are set in
[`nextflow.config`](nextflow.config).

### 2.3 Container images (pinned by digest)
| Process / label | Image |
| --- | --- |
| `CHECK_STRANDNESS`, `python` helpers | `ninedem/check_strandedness@sha256:1a8e1b00…8dcdde` |
| `fastp` | `biocontainers/fastp@sha256:36c38416…179d0` |
| `hisat2` | `quay.io/biocontainers/hisat2@sha256:21be9c91…82ec6` |
| `samtools` | `quay.io/biocontainers/samtools@sha256:557b8fd7…08beca` |
| `cufflinks` | `pgcbioinfo/cufflinks@sha256:833b5af0…b52d` |

The `check_strandedness` image ships Python 3 as `python3`; the config adds a
`beforeScript` symlink so the repo's `python …` helper calls resolve.

### 2.4 Inputs (read set **RS1**)
Three paired-end ENA/SRA accessions and the Ensembl **BDGP6.32, release 106**
reference:

- Reads: `SRR1509507`, `SRR14197369`, `SRR14404397` (`_1`/`_2` FASTQ.gz)
- Reference: `Drosophila_melanogaster.BDGP6.32.dna.toplevel.fa`,
  `…BDGP6.32.cdna.all.fa`, `…BDGP6.32.106.gtf`

SHA-256 of the exact staged bytes: [`input-SHA256SUMS`](input-SHA256SUMS).

---

## 3. Prerequisites

- A Kubernetes cluster and `kubectl` context with:
  - a namespace you can create Jobs/ConfigMaps in (reference run: `yagmur`);
  - a **ReadWriteMany** PVC mounted at `/workspace` (reference run: `rnaseq-pvc`);
  - a ServiceAccount the Nextflow head can use to create task pods
    (reference run: `nextflow-sa`, bound to a role allowing pod create/get/list/
    delete — the Nextflow Kubernetes executor launches one pod per task).
- **≥ ~450 GB free** on the PVC (see §6 — the largest sample expands to a
  ~111 GB intermediate SAM).
- The input data staged on the PVC (see §4).

## 4. Stage the inputs

Place the six FASTQ files under `/workspace/data/reads/` and the three reference
files under `/workspace/data/reference/`, matching the paths in
`nextflow.config`. Any Job/pod that mounts the PVC can download them (e.g. the
FASTQs from the ENA Portal API and the reference from the Ensembl release-106
FTP). Verify against `input-SHA256SUMS`.

## 5. Run it

```bash
# 1) Create the run config as a ConfigMap consumed by the driver Job
kubectl -n <ns> create configmap rnaseq-hisat2-rs1-run01-config \
  --from-file=nextflow.config=nextflow.config

# 2) Launch the driver Job (an initContainer git-clones the pinned upstream
#    commit onto the PVC, then Nextflow runs with the Kubernetes executor)
kubectl -n <ns> apply -f k8s/driver-job.yaml

# 3) Follow progress
kubectl -n <ns> logs -f job/rnaseq-hisat2-rs1-run01 -c nextflow
```

Outputs are published to `/workspace/results/HISAT2-RS1-RUN01/`:
`*.sam.sorted.bam` (one per sample), `transcripts.gtf`, the HISAT2 index,
fastp reports, and the Nextflow `trace`/`report`/`timeline`/`dag`.

### Resuming
If the head Job dies (e.g. the node restarts or the PVC fills), fix the cause and
resume — cached tasks are skipped:

```bash
kubectl -n <ns> apply -f k8s/driver-job-resume.yaml
```

Two gotchas learned reproducing this run:
- **Free PVC space before resuming** if a previous attempt filled it, or the
  first uncached run of the large sample will fail mid-sort.
- If a crashed attempt left `trace/report/timeline` files in the results dir,
  Nextflow will **not** overwrite the trace on the next run, leaving it stale.
  Delete those files before re-running, or regenerate a complete trace from the
  finished session with `nextflow log <run-name> -f task_id,hash,native_id,name,status,exit,submit,start,complete,duration,realtime,peak_rss,peak_vmem,rchar,wchar`.

## 6. Runtime & resource notes (reference run)

- Wall-clock ≈ **9.5 h** (18 tasks; `exon_splice_site`).
- The largest sample (`SRR1509507`) produces a **~111 GB uncompressed SAM**;
  `samtools sort` of it (2 CPUs) is the long pole (~3 h). Give samtools more
  CPU/RAM in `nextflow.config` if you have it.
- Peak concurrent memory ≈ 14.9 GB; CPU ≈ 20 CPU-hours; CPU-only (no GPU).

## 7. Reproducibility record

| | |
| --- | --- |
| Upstream repo | `Nine-s/nextflow_RS1_hisat2_new` |
| Pinned commit | `6b7688c7cab3c0bdb39e0e228ceab2bac31e2caa` |
| Mode | `exon_splice_site` |
| Nextflow | 25.04.8 |
| Result | 18/18 tasks succeeded, 0 failed |
| VIVO run record | see link at top |

## 8. Publishing run metadata to VIVO

Energy/CPU/memory/carbon metadata for a run is collected from Prometheus/Kepler
and published to FONDA VIVO with the
[`fonda-kubernetes-vivo-publisher`](https://github.com/YagmurKati/fonda-kubernetes-vivo-publisher)
toolkit (profile `examples/rnaseq-hisat2-rs1/`).

## Attribution & license

The **workflow** is © its upstream authors — see
[`Nine-s/nextflow_RS1_hisat2_new`](https://github.com/Nine-s/nextflow_RS1_hisat2_new)
and cite it. Only the **reproduction materials** in this repository (this README,
`nextflow.config`, and the `k8s/` manifests) are provided under the MIT License
([`LICENSE`](LICENSE)); see also [`CITATION.cff`](CITATION.cff).
