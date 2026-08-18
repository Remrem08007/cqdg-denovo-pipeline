# CQDG De Novo Variant Pipeline

A reusable **Nextflow DSL2 workflow** for processing family-level gVCF data to support downstream **de novo variant identification**.

The pipeline groups gVCFs by family, performs joint genotyping, applies sequencing-type-specific variant quality filtering, normalizes variants, and annotates the resulting VCFs with Ensembl VEP.

> **Scope:** this workflow prepares and annotates family-level variant data. It does not itself determine inheritance or call de novo variants.

## Workflow overview

```text
Family gVCFs
    │
    ├─ Exclude MNP-like records / normalize input
    │
    ├─ Group samples by family
    │
    ├─ GATK CombineGVCFs
    │
    ├─ GATK GenotypeGVCFs
    │
    ├─ Variant quality filtering
    │      ├─ WGS → VQSR
    │      └─ WES → hard filtering
    │
    ├─ Split multiallelic variants / normalize
    │
    ├─ Ensembl VEP annotation
    │
    └─ bgzip-compressed VCF + tabix index
```

## Main tools

The workflow is implemented in **Nextflow DSL2** and uses containerized versions of:

- **GATK 4.5.0.0** — gVCF combination, joint genotyping, VQSR, and hard filtering
- **bcftools 1.19** — filtering and multiallelic normalization
- **Ensembl VEP 111** — functional variant annotation
- **htslib / tabix 1.19** — VCF indexing
- **nf-schema 2.0.0** — parameter validation and help output

## Input

### Family sample file

The workflow accepts a tab-delimited sample file through `--sampleFile`. Two formats are supported.

#### V1 — sequencing type supplied globally

```text
FAMILY_ID    SAMPLE_1_GVCF    SAMPLE_2_GVCF    SAMPLE_3_GVCF
```

Example:

```tsv
FAMILY-001	/path/FAMILY-001-01.g.vcf.gz	/path/FAMILY-001-02.g.vcf.gz	/path/FAMILY-001-03.g.vcf.gz
FAMILY-002	/path/FAMILY-002-01.g.vcf.gz	/path/FAMILY-002-02.g.vcf.gz	/path/FAMILY-002-03.g.vcf.gz
```

Use:

```text
--sampleFileFormat V1
--sequencingType WGS
```

`V1` is the default sample-file format and `WGS` is the default sequencing type.

#### V2 — sequencing type supplied per family

```text
FAMILY_ID    SEQUENCING_TYPE    SAMPLE_1_GVCF    SAMPLE_2_GVCF    SAMPLE_3_GVCF
```

Example:

```tsv
FAMILY-001	WES	/path/FAMILY-001-01.g.vcf.gz	/path/FAMILY-001-02.g.vcf.gz	/path/FAMILY-001-03.g.vcf.gz
FAMILY-002	WGS	/path/FAMILY-002-01.g.vcf.gz	/path/FAMILY-002-02.g.vcf.gz	/path/FAMILY-002-03.g.vcf.gz
```

Use:

```text
--sampleFileFormat V2
```

For each gVCF path, the workflow also expects the corresponding index to be available alongside the file.

## WGS vs WES filtering

The filtering path depends on sequencing type:

- **WGS:** Variant Quality Score Recalibration (**VQSR**)
- **WES:** configurable **hard filtering**

The default hard-filter expressions are defined in `nextflow.config` and can be overridden through the `hardFilters` parameter.

## Reference resources

The workflow requires the following reference inputs.

### Reference genome

- `referenceGenome` — directory containing the reference genome files
- `referenceGenomeFasta` — reference FASTA filename within that directory

The workflow was developed around **GRCh38** resources.

### Broad/GATK resources

- `broad` — directory containing the interval list and VQSR resources
- `intervalsFile` — interval-list filename within that directory

The current VQSR implementation expects the following resource filenames in the configured Broad resource directory:

- `hapmap_3.3.hg38.vcf.gz`
- `1000G_omni2.5.hg38.vcf.gz`
- `1000G_phase1.snps.high_confidence.hg38.vcf.gz`
- `Homo_sapiens_assembly38.dbsnp138.vcf.gz`

These filenames are currently part of the workflow assumptions and should be considered when preparing a reusable installation.

### VEP cache

- `vepCache` — directory containing an Ensembl VEP cache compatible with the configured VEP container and GRCh38 reference

VEP is run in **offline/cache mode**.

## Required parameters

| Parameter | Required | Description |
| --- | --- | --- |
| `sampleFile` | Yes | Tab-delimited family/sample input file |
| `outputDir` | Yes | Output directory |
| `referenceGenome` | Yes | Directory containing the reference genome |
| `broad` | Yes | Directory containing interval and VQSR resources |
| `vepCache` | Yes | Ensembl VEP cache directory |
| `intervalsFile` | Yes | Interval-list filename |
| `hardFilters` | Yes by schema | Hard-filter definitions used for WES |
| `sampleFileFormat` | No | `V1` or `V2`; default `V1` |
| `sequencingType` | No | `WGS` or `WES`; default `WGS` for V1 inputs |
| `vepCpu` | No | Number of VEP forks; default `4` |
| `TSfilterSNP` | No | VQSR SNP truth-sensitivity threshold |
| `TSfilterINDEL` | No | VQSR INDEL truth-sensitivity threshold |

For the complete parameter schema, see [`nextflow_schema.json`](nextflow_schema.json).

## Running the workflow

Display parameter help with:

```bash
nextflow run main.nf --help
```

A typical run can be configured through a JSON parameter file:

```bash
nextflow run main.nf -params-file params.json
```

A minimal parameter file will need to provide the required paths for the sample file, output directory, reference genome, Broad/GATK resources, VEP cache, interval file, and hard-filter configuration.

## Stub testing

The workflow includes Nextflow `stub` blocks for its processes. This allows the pipeline structure and channel wiring to be exercised without running the full bioinformatics tools:

```bash
nextflow run main.nf -stub-run -params-file params.json
```

## Output

Final annotated family-level VCFs are published to `outputDir` with names of the form:

```text
variants.<FAMILY_ID>.vep.vcf.gz
variants.<FAMILY_ID>.vep.vcf.gz.tbi
```

The Nextflow configuration also writes execution metadata under:

```text
<outputDir>/pipeline_info/
```

including:

- execution timeline
- execution report
- execution trace
- pipeline DAG

## Reproducibility and execution

Process containers are defined in `nextflow.config`. The workflow also defines per-process CPU, memory, disk, time, retry, and executor settings to support reproducible execution in managed compute environments.

Because reference data are external to the containers, reproducible deployment also depends on using compatible versions of the reference genome, GATK/VQSR resources, interval lists, and VEP cache.

## Current implementation notes

This repository is intended to be reusable, but several assumptions remain specific to the current implementation:

- the workflow is built around GRCh38 resources;
- several VQSR resource filenames are currently expected by name;
- `referenceGenome` and `referenceGenomeFasta` are separate parameters;
- compatibility between the VEP cache, VEP container, and reference FASTA must be maintained by the user.

These are deployment constraints rather than hidden dependencies and are documented here to make reuse easier.

## References

- [Nextflow documentation](https://www.nextflow.io/docs/latest/)
- [bcftools documentation](https://samtools.github.io/bcftools/bcftools.html)
- [GATK CombineGVCFs](https://gatk.broadinstitute.org/hc/en-us/articles/360037593911-CombineGVCFs)
- [GATK GenotypeGVCFs](https://gatk.broadinstitute.org/hc/en-us/articles/360037057852-GenotypeGVCFs)
- [GATK Variant Quality Score Recalibration](https://gatk.broadinstitute.org/hc/en-us/articles/360035531612-Variant-Quality-Score-Recalibration-VQSR)
- [GATK VariantFiltration](https://gatk.broadinstitute.org/hc/en-us/articles/360041850471-VariantFiltration)
- [Ensembl VEP documentation](https://www.ensembl.org/info/docs/tools/vep/index.html)
