# METAHICT

METAHICT is a Nextflow DSL2 workflow for genome-resolved analysis of short- or
long-read shotgun metagenomes with paired-end metagenomic Hi-C data. It
performs read preprocessing, assembly, Hi-C alignment, coverage and contact
analysis, binning, bin reassembly, scaffolding, taxonomic annotation, and
detection of mobile genetic elements (MGEs) and candidate MGE–host pairs.

![METAHICT workflow overview](images/METAHICT_Overview.png)

## Requirements

METAHICT 1.2.0 supports and tests on Linux systems. The host must
provide Conda, `curl`, `tar`, and Git. Python, Nextflow, and the scientific
programs are installed into METAHICT-managed environments.

Default allocations reach 16 threads and 64 GB RAM. METAHICT caps requests to
available local resources, but large datasets may still require a high-memory
server. Allow sufficient storage for environments, reference databases,
results, and Nextflow work files.

## Installation

### Recommended: GitHub release

```bash
git clone https://github.com/dyxstat/METAHICT.git
cd METAHICT
chmod +x metahict nextflow/bin/nextflow
```

This installation keeps the workflow, locked scientific environments,
documentation, and example dataset together. Typical installation time is 5–10 minutes, excluding database installation.

### Optional: Bioconda

Bioconda provides the METAHICT workflow layer as an alternative. Install it at
an explicit location on a volume with sufficient space:

```bash
conda create -p /path/to/metahict-bioconda-v1.2.0 \
  --strict-channel-priority \
  -c conda-forge \
  -c bioconda \
  metahict=1.2.0
conda activate /path/to/metahict-bioconda-v1.2.0
```

Replace `/path/to/metahict-bioconda-v1.2.0` with an absolute path. The
Bioconda environment and the scientific environments created by
`metahict install` will remain below this location. Reference databases are
installed or linked separately.

### Run METAHICT from any directory

The GitHub launcher may be called by its absolute path, so analyses do not
need to run inside the source directory:

```bash
/path/to/METAHICT/metahict run \
  --samplesheet /path/to/analysis/samplesheet.csv \
  --config /path/to/analysis/metahict_configuration.yaml \
  --outdir /path/to/analysis/results
```

Use absolute paths for analysis inputs and outputs when running this way. For
the Bioconda installation, activate its environment and run `metahict` from
any directory.

## Quick start

The commands below use the recommended GitHub installation. For a Bioconda
installation, replace `./metahict` with `metahict`.

### 1. Install and test the software

```bash
./metahict doctor
./metahict install
./metahict test workflow
```

`doctor` checks the operating system, architecture, required commands, and
distributed lock files. `install` creates and verifies the locked scientific
environments. `test workflow` runs the core workflow and standalone
scaffolding entry with stub tasks, without reference databases or sequencing
reads. Typical installation time is 10–20 minutes, excluding databases.

### 2. Install and validate the reference databases

```bash
./metahict database all
./metahict doctor --runtime --databases
```

This installs the CheckM, CheckM2, GTDB-Tk, and geNomad databases under
`databases/`. Existing or shared databases can be linked or supplied by path;
see [Installation and databases](docs/installation.md).

### 3. Create the configuration

```bash
./metahict config
```

This creates `metahict_configuration.yaml`. Keep the scientific defaults for
a first analysis and review its `resources` section for the available server.

### 4. Create the samplesheet

Use the real FASTQ paths and the restriction enzyme or enzyme combination used
to prepare the Hi-C library:

```bash
./metahict samplesheet \
  --sample sample_01 \
  --sg-r1 /data/sample_01/shotgun_R1.fastq.gz \
  --sg-r2 /data/sample_01/shotgun_R2.fastq.gz \
  --hic-r1 /data/sample_01/hic_R1.fastq.gz \
  --hic-r2 /data/sample_01/hic_R2.fastq.gz \
  --enzyme Sau3AI,MluCI \
  --output samplesheet.csv
```

Replace the paths, sample name, and enzymes with the study metadata. For a
single-file long-read shotgun library, omit `--sg-r2` and add its type, for
example `--long-read-type nano-hq`. Accepted values are `pacbio-raw`,
`pacbio-corr`, `pacbio-hifi`, `nano-raw`, `nano-corr`, and `nano-hq`.

### 5. Run the complete workflow

```bash
./metahict run \
  --samplesheet samplesheet.csv \
  --config metahict_configuration.yaml \
  --outdir results
```

METAHICT writes each sample under `results/<sample>/` and run records under
`results/nextflow_reports/`. Scaffolding is not run automatically; it remains
available as a selected-module analysis for MAGs chosen by the user.

After correcting a failed task, repeat the command with `--resume` and retain
`results/nextflow_work/` so Nextflow can reuse successful tasks.

### 6. Run an individual module (optional)

Available entry names are `preprocessing`, `assembly`, `alignment`, `coverage`,
`contact`, `binning`, `reassembly`, `scaffolding`, `annotation`, and `mge`.
Display the required inputs, parameters, outputs, and example command before
running a module:

```bash
./metahict run --entry-module binning --help
```

Replace `binning` with the required entry name. The [module
reference](docs/modules/README.md) provides browsable guides for all stages.

## Test with the example dataset

This functional test runs the actual scientific programs and can take
approximately 20–30 minutes, depending on the system. Install or link the
reference databases before starting it.

### GitHub installation

The example FASTQ files are included in the GitHub release:

```bash
./metahict test example --outdir results
```

### Bioconda installation

The example FASTQ files are not installed by Bioconda. Add them once, then run
the test:

```bash
curl -fL https://github.com/dyxstat/METAHICT/archive/refs/tags/v1.2.0.tar.gz | \
  tar -xzf - --strip-components=1 \
    -C "$CONDA_PREFIX/share/metahict" \
    METAHICT-1.2.0/example_dataset
metahict test example --outdir results
```

The command extracts only `example_dataset/` into the installed METAHICT
directory. It creates neither a source checkout nor a symbolic link. This
Bioconda-specific step does not affect `metahict test workflow` or analyses
using a user samplesheet.

See [Testing METAHICT](docs/test_dataset.md) for expected behavior and failure
diagnostics.

## Main results

| Result | Location below `results/sample_01/` |
| --- | --- |
| Cleaned shotgun and Hi-C reads | `1_preprocessing/sg/`; `1_preprocessing/hic/` |
| Metagenome assembly | `2_assembly/final_assembly.fasta` |
| Filtered Hi-C alignment | `3_alignment/sorted_map.bam` |
| Shotgun depth | `4_coverage/coverage.txt` |
| Normalized contact matrix | `5_contact/denoised_contact_matrix_normcc.npz` |
| Consolidated MAGs | `6_binning/metahict/final_bins/` |
| Reassembled MAGs (paired short reads only) | `7_reassembly/reassembled_bins/` |
| Scaffolded MAGs (optional standalone module) | `8_scaffolding/<BIN_ID>/scaffolded_bin.fa` |
| GTDB-Tk taxonomy | `9_annotation/classify/gtdbtk.*.summary.tsv` |
| MGE calls, circular contigs, and candidate MGE–host pairs | `10_MGE/` |

## Documentation and help

| Goal | Documentation |
| --- | --- |
| First complete analysis | [Command-by-command tutorial](docs/quickstart.md) |
| Understand the biological stages | [Concepts](docs/concepts.md) |
| Install or reuse environments and databases | [Installation and databases](docs/installation.md) |
| Change resources or algorithms | [Configuration reference](docs/configuration.md) |
| Run one module | [Module reference](docs/modules/README.md) |
| Understand results, resume, or run outside the checkout | [Workflow execution](docs/nextflow.md) |
| Inspect logs and provenance | [Logging](docs/logging.md) |
| Diagnose a failure | [Troubleshooting](docs/troubleshooting.md) |

Command-line help is generated from the current interface:

```bash
./metahict --help
./metahict run --help
./metahict run --entry-module binning --help
```

The complete documentation index is [docs/README.md](docs/README.md).

## Third-party software

Versions, licenses, sources, and citations for external programs are listed in
[Third-party software](docs/third_party.md).

## License

METAHICT is distributed under the GNU General Public License; see
[LICENSE](LICENSE). The METAHICT workflow layer is available through Bioconda.
The `metahict install` command creates the locked scientific software
environments, while reference databases are installed separately.
