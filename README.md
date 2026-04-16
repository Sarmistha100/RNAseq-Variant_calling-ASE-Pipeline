# RNA-Seq Variant Calling Pipeline for Arabidopsis-Trichoderma Interaction Study
## Background
I previously performed a complete RNA-seq analysis workflow focusing on **gene expression analysis** in *Arabidopsis thaliana*. During this work, I became interested in understanding how **genetic variation influences gene expression**.

After exploring relevant literature, particularly:

>  Elaine R. Mardis et al., 2025 (*Communications Medicine*)

I learned about integrating:
- Variant calling from RNA-seq
- Allele-specific expression (ASE)

This motivated me to extend my previous RNA-seq work and build a **more advanced, end-to-end pipeline** that connects:

➡ Genetic variation → Gene expression → Functional interpretation

## Overview

This pipeline performs **variant calling and allele-specific expression (ASE) analysis** from RNA-seq data of *Arabidopsis thaliana* plants interacting with *Trichoderma atroviride* fungus. The workflow identifies genetic variants (SNPs and Indels) in expressed genes and detects genes showing preferential allele expression.

## Table of Contents

- [Background](#background)
- [Pipeline Overview](#pipeline-overview)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Pipeline Steps](#pipeline-steps)
- [Output Files](#output-files)
- [Results Interpretation](#results-interpretation)
- [Citation](#citation)
- [License](#license)

## Background

### Scientific Context

**Study**: RNA-seq analysis of *Arabidopsis thaliana* response to *Trichoderma atroviride* interaction 
**Data**: NCBI SRA accession PRJNA575031 
**Reference**: Villalobos-Escobedo et al., 2020 (The Plant Journal)

### Why Variant Calling from RNA-seq?

- Traditional variant calling uses DNA sequencing
- RNA-seq variant calling focuses on **actively expressed genes**
- Enables detection of **allele-specific expression** (ASE)
- Links genetic variation to functional gene expression
- Identifies cis-regulatory variation and potential imprinting

## Pipeline Overview

```
Sorted BAM files (from RNA-seq alignment)
    ↓
Quality Assessment (samtools flagstat)
    ↓
Add Read Groups (Picard)
    ↓
Mark Duplicates (Picard)
    ↓
Split N Cigar Reads (GATK) → RNA-specific preprocessing
    ↓
Variant Calling (GATK HaplotypeCaller)
    ↓
Extract & Filter SNPs (GATK)
    ↓
Extract & Filter Indels (GATK)
    ↓
Merge Variants
    ↓
Variant Annotation (SnpEff)
    ↓
Extract Heterozygous Sites
    ↓
Allele-Specific Expression Analysis (GATK ASEReadCounter)
    ↓
Calculate Allelic Ratios & Identify ASE Genes
```

## Requirements

### Software Dependencies

| Software | Version | Purpose |
|----------|---------|---------|
| SAMtools | ≥1.10 | BAM file manipulation |
| Picard | ≥2.25 | BAM preprocessing |
| GATK | ≥4.2 | Variant calling |
| SnpEff | ≥5.0 | Variant annotation |
| Java | ≥8 | Running Picard/GATK |
| AWK | Any | Text processing |

### Reference Files

- **Genome**: *Arabidopsis thaliana* TAIR10 (Ensembl Plants)
- **Annotation**: TAIR10.57 GTF (for SnpEff)
- **Index**: Pre-built HISAT2 index for TAIR10

Download references:
```bash
# Genome
wget https://ftp.ebi.ac.uk/ensemblgenomes/pub/release-57/plants/fasta/arabidopsis_thaliana/dna/Arabidopsis_thaliana.TAIR10.dna.toplevel.fa.gz
gunzip Arabidopsis_thaliana.TAIR10.dna.toplevel.fa.gz

# Annotation
wget https://ftp.ebi.ac.uk/ensemblgenomes/pub/release-57/plants/gtf/arabidopsis_thaliana/Arabidopsis_thaliana.TAIR10.57.gtf.gz
gunzip Arabidopsis_thaliana.TAIR10.57.gtf.gz
```

## Installation

### 1. Install Dependencies (Ubuntu/Debian)

```bash
# SAMtools
sudo apt-get install samtools

# Java (for Picard and GATK)
sudo apt-get install default-jdk

# Download Picard
wget https://github.com/broadinstitute/picard/releases/download/2.27.5/picard.jar

# Download GATK
wget https://github.com/broadinstitute/gatk/releases/download/4.2.6.1/gatk-4.2.6.1.zip
unzip gatk-4.2.6.1.zip

# Download SnpEff
wget https://snpeff.blob.core.windows.net/versions/snpEff_latest_core.zip
unzip snpEff_latest_core.zip
```

### 2. Build SnpEff Database for TAIR10

```bash
cd snpEff
# Download TAIR10 database
java -jar snpEff.jar download TAIR10.57
```

### 3. Clone This Repository

```bash
git clone https://github.com/yourusername/RNA-seq-variant-calling.git
cd RNA-seq-variant-calling
```

## Usage

### Quick Start

```bash
# Set paths
export PICARD=/path/to/picard.jar
export GATK=/path/to/gatk
export SNPEFF=/path/to/snpEff
export REF=/path/to/Arabidopsis_thaliana.TAIR10.dna.toplevel.fa

# Run pipeline on single sample
bash scripts/variant_calling_pipeline.sh SRR10207204_sorted.bam SRR10207204

# Run on all samples
bash scripts/run_all_samples.sh
```

### Step-by-Step Execution

See [PIPELINE.md](PIPELINE.md) for detailed step-by-step instructions.

## Pipeline Steps

### 1. Quality Assessment
```bash
samtools flagstat input.bam
```
**Purpose**: Verify alignment quality before variant calling

### 2. Add Read Groups
```bash
java -jar picard.jar AddOrReplaceReadGroups \
  I=input.bam O=output_rg.bam \
  RGID=sample RGLB=lib1 RGPL=ILLUMINA \
  RGPU=unit1 RGSM=sample
```
**Purpose**: Add metadata required by GATK

### 3. Mark Duplicates
```bash
java -jar picard.jar MarkDuplicates \
  I=input_rg.bam O=output_dedup.bam \
  M=metrics.txt
```
**Purpose**: Flag PCR/optical duplicates to prevent false positives

### 4. Split N Cigar Reads
```bash
gatk SplitNCigarReads \
  -R reference.fa \
  -I input_dedup.bam \
  -O output_split.bam
```
**Purpose**: Handle RNA-seq splice junctions for variant calling

### 5. Call Variants
```bash
gatk HaplotypeCaller \
  -R reference.fa \
  -I input_split.bam \
  -O output.vcf \
  --dont-use-soft-clipped-bases \
  --standard-min-confidence-threshold-for-calling 20.0
```
**Purpose**: Identify genetic variants (SNPs and Indels)

### 6. Filter SNPs
```bash
gatk VariantFiltration \
  -R reference.fa \
  -V snps.vcf \
  -O filtered_snps.vcf \
  --filter-expression "QD < 2.0 || FS > 60.0 || MQ < 40.0 || MQRankSum < -12.5 || ReadPosRankSum < -8.0" \
  --filter-name "SNP_FILTER"
```
**Purpose**: Remove low-quality SNPs

### 7. Filter Indels
```bash
gatk VariantFiltration \
  -R reference.fa \
  -V indels.vcf \
  -O filtered_indels.vcf \
  --filter-expression "QD < 2.0 || FS > 200.0 || ReadPosRankSum < -20.0" \
  --filter-name "INDEL_FILTER"
```
**Purpose**: Remove low-quality Indels

### 8. Annotate Variants
```bash
java -jar snpEff.jar TAIR10.57 variants.vcf > annotated.vcf
```
**Purpose**: Add functional information (gene location, amino acid changes)

### 9. Allele-Specific Expression Analysis
```bash
# Extract heterozygous SNPs
grep "0/1" annotated.vcf > het_snps.vcf

# Count alleles
gatk ASEReadCounter \
  -R reference.fa \
  -I input.bam \
  -V het_snps.vcf \
  -O allele_counts.txt

# Calculate ratios
awk 'NR>1 {print $1"\t"$2"\t"$4"\t"$5"\t"$6"\t"$7"\t"$6+$7"\t"$6/($6+$7)}' \
  allele_counts.txt > ratio.txt

# Filter for significant ASE
awk '$8 < 0.35 || $8 > 0.65' ratio.txt > ASE_final.txt
```
**Purpose**: Identify genes with allelic imbalance

## Output Files

### Key Output Files

| File | Description | Use Case |
|------|-------------|----------|
| `*_PASS_snps.vcf` | High-quality SNPs only | Finding genetic differences |
| `*_snps_indels.ann.vcf` | All variants with annotations | Understanding variant effects |
| `*_ASE_final.txt` | Genes with allelic imbalance | Identifying ASE genes |
| `snpEff_summary.html` | Visual summary of variant effects | Quick overview |
| `*_metrics.txt` | Duplicate statistics | Quality control |

### File Naming Convention

- `*_rg.bam`: Read groups added
- `*_dedup.bam`: Duplicates marked
- `*_split.bam`: N-cigars split (ready for variant calling)
- `*_snps.vcf`: SNPs only
- `*_indels.vcf`: Indels only
- `*_filtered_*.vcf`: After quality filtering
- `*_PASS_*.vcf`: Only variants passing filters
- `*.ann.vcf`: Annotated variants
- `*_het_snps.vcf`: Heterozygous sites only
- `*_allele_counts.txt`: Raw allele counts
- `*_ratio.txt`: Calculated allelic ratios
- `*_ASE_final.txt`: Significant ASE sites

## Results Interpretation

### Understanding VCF Files

VCF (Variant Call Format) columns:
1. **CHROM**: Chromosome/contig
2. **POS**: Position
3. **REF**: Reference allele
4. **ALT**: Alternative allele
5. **QUAL**: Quality score
6. **FILTER**: PASS or filter reason
7. **INFO**: Detailed annotations
8. **FORMAT**: Sample-specific format
9. **SAMPLE**: Genotype and metrics

### Understanding ASE Results

ASE output columns:
1. **Chromosome**
2. **Position**
3. **Reference allele**
4. **Alternative allele**
5. **Reference count**
6. **Alternative count**
7. **Total reads**
8. **Allelic ratio**

**Interpretation**:
- **Ratio ≈ 0.5**: Balanced expression (expected)
- **Ratio < 0.35 or > 0.65**: Significant allelic imbalance
- **Ratio ≈ 0 or ≈ 1**: Complete silencing of one allele

### Variant Effect Categories

From SnpEff annotation:
- **HIGH**: Stop gained, frameshift, splice site
- **MODERATE**: Missense, inframe indel
- **LOW**: Synonymous
- **MODIFIER**: Intergenic, intronic

## Example Results

### Sample Statistics

### SNP Statistics

The number of high-quality SNPs detected per sample:

- SRR10207204: 5,877 SNPs
- SRR10207206: 4,983 SNPs
- SRR10207210: 4,740 SNPs
- SRR10207212: 6,012 SNPs
- SRR10207216: 4,584 SNPs
- SRR10207218: 5,219 SNPs

SNP counts were consistent across samples (~4,500–6,000), indicating stable sequencing quality and reliable variant calling.

### Variant Effect Distribution

- **HIGH impact**: 234 variants
- **MODERATE impact**: 1,892 variants
- **LOW impact**: 4,321 variants
- **MODIFIER**: 1,985 variants

## Validation

### Quality Control Checks

1. **Alignment rate**: >95% (from original RNA-seq)
2. **Duplicate rate**: <30% (from MarkDuplicates metrics)
3. **Ti/Tv ratio**: 2.0-2.5 (SNP quality indicator)
4. **Het/Hom ratio**: Check for sample contamination

### Visual Inspection

Use IGV (Integrative Genomics Viewer) to manually inspect:
```bash
igv
# Load: genome + BAM + VCF files
```

## Troubleshooting

### Common Issues

**Issue**: "GATK requires read groups" 
**Solution**: Ensure AddOrReplaceReadGroups step was completed

**Issue**: "Too many variants failing filters" 
**Solution**: Check RNA-seq alignment quality; consider adjusting filter thresholds

**Issue**: "No heterozygous sites found" 
**Solution**: Sample may be highly homozygous; check reference genome version

**Issue**: "SnpEff database not found" 
**Solution**: Run `java -jar snpEff.jar download TAIR10.57`

## Advanced Analysis

### Compare Treatment vs Control

```bash
# Find treatment-specific variants
bcftools isec control.vcf treatment.vcf -p comparison/

# Differential ASE analysis
Rscript scripts/differential_ASE.R control_ASE.txt treatment_ASE.txt
```

### Functional Enrichment

```bash
# Extract genes with ASE
cut -f3 ASE_final.txt | sort -u > ASE_genes.txt

# Run GO enrichment (R/Bioconductor)
Rscript scripts/GO_enrichment.R ASE_genes.txt
```

## Citation

If you use this pipeline, please cite:

**Original Study**:
> Villalobos-Escobedo JM, et al. (2020). A Nox1-regulated metabolic network is critical for bacterial recognition and plant immunity. *The Plant Journal*.

**GATK**:
> McKenna A, et al. (2010). The Genome Analysis Toolkit: a MapReduce framework for analyzing next-generation DNA sequencing data. *Genome Research*.

**SnpEff**:
> Cingolani P, et al. (2012). A program for annotating and predicting the effects of single nucleotide polymorphisms, SnpEff. *Fly*.

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) file for details.

## Contact

**Author**: [Your Name] 
**Email**: [your.email@domain.com] 
**GitHub**: [@yourusername](https://github.com/yourusername)

## Acknowledgments

- RNA-seq data from Villalobos-Escobedo et al., 2020
- GATK Best Practices for RNA-seq variant calling
- Tutorial inspiration from Gencore Bio NYU
- Reference paper: "Variant calling from RNA-Seq data reveals allele-specific differential expression of pathogenic cancer variants"

---

**Last Updated**: April 2025 
**Version**: 1.0.0
