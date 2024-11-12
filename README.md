# CUT&Tag

This repository contains workflows and scripts for processing and aligning CUT&Tag sequencing data, with Bowtie2 configured as the default aligner for accurate read mapping. The workflow includes support for experiments that include spike-in fragments, allowing for sequential alignment of raw reads against both a primary genome (e.g., mm10 for mouse studies) and a spike-in reference genome.

Cleavage Under Targets and Tagmentation (CUT&Tag) is a chromatin profiling technique introduced in 2019 by [Hatice S Kaya-Okur et al.](https://pubmed.ncbi.nlm.nih.gov/31036827/), primarily designed to achieve high-resolution mapping of histone modifications. CUT&Tag combines antibody-guided cleavage with tagmentation, generating targeted DNA fragments with minimal background noise, making it particularly effective for histone profiling and well-defined chromatin marks.

While CUT&Tag is highly effective for histone modification mapping, it may be less ideal for transcription factors (TFs) and certain DNA-binding proteins due to their lower abundance, more transient binding, and positioning in open chromatin regions. In cases where TF profiling is essential, other methods like CUT&RUN or ChIP-seq may be preferable.

This repository provides a full tutorial for CUT&Tag data processing, including alignment, spike-in normalization, and visualization examples to support downstream analysis.

## A brief primer on CUT&Tag

CUT&Tag (Cleavage Under Targets and Tagmentation) is a highly sensitive technique for profiling chromatin post-translational modifications (PTMs), requiring only about 5 million reads to detect PTM peaks accurately. This efficiency makes CUT&Tag an ideal choice for studies focused on chromatin modifications, especially histone marks.

The method is particularly suited for PTM analysis due to its lack of cross-linking, which reduces background noise and enhances resolution. However, because CUT&Tag doesn’t cross-link chromatin components, it may be less effective for mapping transient interactions, such as those involving transcription factors (TFs) or other chromatin-binding proteins.

For reads longer than 25 bp, adapter trimming is recommended to ensure accurate downstream analysis and alignment.

## Raw Data Quality Control (QC)

Begin by inspecting the raw reads with `FastQC` to check for the presence of adapter sequences or any other quality issues. If adapters or other contaminants are detected, trim the reads using `fastp`. After trimming, run `FastQC` again on the cleaned reads to confirm successful removal of adapters and to ensure no quality flags remain.

```
# Run FastQC on raw reads
fastqc <read_1.fastq> -t $CPUS --outdir ./res_fastqc/ --extract --quiet
fastqc <read_2.fastq> -t $CPUS --outdir ./res_fastqc/ --extract --quiet

# Perform adapter trimming and quality filtering with fastp
fastp -i <read_1.fastq> -I <read_2.fastq> \
  -o <trimmed_read_1.fastq> -O <trimmed_read_2.fastq> \
  --detect_adapter_for_pe \
  --cut_front \
  --cut_tail \
  --cut_mean_quality 20 \
  --length_required 50

# Run FastQC on the trimmed reads
fastqc <trimmed_read_1.fastq> -t $CPUS --outdir ./res_fastqc/ --extract --quiet
fastqc <trimmed_read_2.fastq> -t $CPUS --outdir ./res_fastqc/ --extract --quiet
```

## Read Mapping

For CUT&Tag experiments, Bowtie2 is recommended as the default aligner due to its accuracy and sensitivity. If spike-in fragments are included, align the raw reads twice: once against the primary genome and once against the spike-in reference genome. This dual alignment approach enables better normalization and quantification.

### Mapping to the Primary Genome

For short paired-end reads (without adapter trimming) and an insert size between 10 and 700 bp, use Bowtie2 with these parameters:

```
bowtie2 -x <index_base> -1 <read_1.fastq> -2 <read_2.fastq> --end-to-end --very-sensitive --no-mixed --no-discordant --phred33 -I 10 -X 700 -S <output.sam>
```

If adapter trimming is required (for reads >25 bp), switch to local alignment to ignore residual adapter sequences at the 3' ends of reads:

```
bowtie2 -x <index_base> -1 <read_1.fastq> -2 <read_2.fastq> --local --very-sensitive --no-mixed --no-discordant --phred33 -I 10 -X 700 -S <output.sam>
```

### Mapping to the Spike-In Genome

To map reads to the spike-in reference (e.g., E. coli), build the genome index and align reads with additional parameters to prevent cross-mapping with the primary genome:

```
bowtie2-build <spikein_fasta> <spikein_index_base> 

bowtie2 -x <spikein_index_base> -1 <read_1.fastq> -2 <read_2.fastq> --end-to-end --very-sensitive --no-overlap --no-dovetail --no-mixed --no-discordant --phred33 -I 10 -X 700 -S <output.sam>
```





