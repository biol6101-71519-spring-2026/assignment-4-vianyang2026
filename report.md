# Assignment 4 Report: De Novo Genome Assembly of Escherichia coli (K-12)


## Introduction

Escherichia coli K-12 MG1655 is a well-characterised, non-pathogenic laboratory strain with a compact genome of approximately 4.6 Mb and a GC content of ~50.8% (Blattner et al., 1997). Its small size, the availability of a high-quality reference sequence (GenBank NC_000913.3), and a publicly available Illumina dataset with deep, even coverage (~220×, SRA: SRR2584863, paired-end 2×150 bp, HiSeq 2500) make it an ideal benchmark for evaluating assembly workflows. Assembly was performed with SPAdes, using its --careful mode to minimise substitution errors via a post-processing MismatchCorrector step (Bankevich et al., 2012). Assembly quality was then evaluated using QUAST and BUSCO, which together capture complementary aspects of assembly performance, contiguity and gene-space completeness, respectively.



## Methods

### Data acquisition and QC
Reads were downloaded from the NCBI SRA (accession SRR2584863) using `fasterq-dump`. Raw read quality was then assessed with FastQC v0.12.1 and summarised with MultiQC v1.14. Next reads were then trimmed with fastp v0.23.4 using the following filters: minimum read length 50 bp, minimum base quality phred ≥ 20, and a maximum of 40% low-quality bases per read. Adapter sequences were detected and removed automatically. Post-trimming quality was re-assessed with FastQC and MultiQC to to confirm that reads met quality thresholds before assembly..

### Genome assembly
Trimmed paired-end reads were assembled with SPAdes v3.15.5 using `--careful` mode and 8 threads, with a memory cap of 16 GB. SPAdes performs iterative assembly across multiple k-mer lengths selected automatically based on read length. The `--careful` flag enables MismatchCorrector, which maps reads back to the initial assembly and corrects single-nucleotide discrepancies, producing a more accurate final sequence. No additional polishing was applied, as the Illumina data are sufficiently accurate.

### Assembly quality assessment
Assembly statistics (contig count, total length, N50/N75/N90, largest contig, GC content) were computed with QUAST v5.2.0 using an estimated reference size of 4.6 Mb. Gene-space completeness was assessed with BUSCO v5.4.7 against the `bacteria_odb10` lineage dataset (genome mode). As an additional quality metric, trimmed reads were mapped back to the assembly using minimap2 v2.26 (short-read preset `-ax sr`), sorted and indexed with samtools v1.17, and per-contig coverage statistics were extracted with `samtools coverage`. A contig length distribution plot was generated with a custom Python script using matplotlib v3.7.1 and BioPython v1.81.



## Results

### Raw read quality
Raw sequencing yielded 3,106,518 reads (1,553,259 pairs) at 150 bp, giving  approximately 101× theoretical coverage of the 4.6 Mb genome. Pre-trimming, R1 and R2 showed Q20 rates of 89.2% and Q30 rates of 81.6%, with slight quality degradation toward read ends, as expected for HiSeq 2500 data. No significant adapter contamination was detected by FastQC. After fastp trimming (min length 50 bp, phred ≥ 20, ≤ 40% low-quality bases), 2,538,296 reads (81.7% of input) were retained, with a mean length of 144 bp, Q20 rate 96.4%, and Q30 rate 89.4%. 526,316 reads were removed for low quality, 40,782 for being too short, and 1,124 for excess N bases. The trimmed MultiQC report confirmed clean, adapter-free reads with uniform quality across positions.

### Assembly statistics

The SPAdes assembly produced 69 contigs ≥ 500 bp with a total length of 4.55 Mb, within 1% of the expected 4.6 Mb. The N50 was 151,564 bp, meaning half the assembly is contained in contigs of at least 152 kb. The largest contig was 348 kb. GC content was 50.72%, consistent with the E. coli K-12 reference (50.8%). The 139 contigs smaller than 500 bp account for only 24,164 bp combined and likely represent repetitive or low-complexity regions that SPAdes could not fully resolve.

### BUSCO completeness


BUSCO analysis against `bacteria_odb10` (124 conserved orthologs) showed 100% complete single-copy genes, with zero fragmented or missing BUSCOs. This is the maximum possible score and indicates that the assembly captures the complete gene space of *E. coli* K-12 without detectable gaps or duplications.

### Coverage analysis
Read mapping with minimap2 showed a mean depth of 78× across the major contigs (the largest contig, 348 kb, had 78.0× mean depth; the next three largest had 77.2×, 76.8×, and 80.4× respectively), consistent with the expected ~80× empirical coverage after trimming. In total, 99.97% of assembled bases were covered (4,575,014 / 4,576,410 bp), with no contig-level drop-out regions. Coverage was largely uniform across all major contigs, indicating no systematic assembly gaps or misassembly artefacts. A small number of very short contigs (<500 bp) showed zero or near-zero coverage and likely represent assembly noise.

### Contig length distribution
The contig length distribution plot shows that most assembly length is concentrated in a small number of large contigs (consistent with a nearly complete bacterial genome), while a tail of short contigs (<1 kb) represents repetitive or low-complexity regions that SPAdes could not resolve fully.


## Discussion

The *E. coli* K-12 assembly is of high quality by all measured metrics. The total assembly length (4.55 Mb) recovers 98.9% of the expected genome size, and the 100% BUSCO score confirms that no annotated gene content is missing or fragmented. The N50 of ~152 kb is strong for a short-read-only Illumina assembly of a bacterial genome, indicating that SPAdes successfully resolved most of the chromosome into large, contiguous segments. Coverage analysis corroborates assembly integrity: the ~78× mean depth is consistent with theoretical expectations, and near-complete breadth (99.97%) rules out large assembly gaps.

The primary limitation of this assembly is fragmentation due to repetitive sequences. Short 150 bp reads cannot span repeat elements longer than ~300 bp (e.g., rRNA operons, insertion sequences), which is why the genome is split across 69 contigs rather than assembled into a single circular chromosome. The *E. coli* K-12 genome contains 7 rRNA operons, each ~5.5 kb, which are likely responsible for most of the remaining breaks. A hybrid approach combining these Illumina reads with long-read data (e.g., PacBio HiFi or Oxford Nanopore) would bridge these repeats and likely yield a complete, closed assembly. Alternatively, scaffolding the contigs against the reference (NC_000913.3) using tools such as RaGOO or ABACAS could produce a chromosome-scale pseudomolecule without additional sequencing.


## References

1. Bankevich A et al. (2012) SPAdes: A new genome assembly algorithm and its applications to single-cell sequencing. *J Comput Biol* 19(5):455–477. doi:10.1089/cmb.2012.0021
2. Blattner FR et al. (1997) The complete genome sequence of *Escherichia coli* K-12. *Science* 277(5331):1453–1462. doi:10.1126/science.277.5331.1453
3. Ewels P et al. (2016) MultiQC: Summarize analysis results for multiple tools and samples in a single report. *Bioinformatics* 32(19):3047–3048. doi:10.1093/bioinformatics/btw354
4. Gurevich A et al. (2013) QUAST: Quality assessment tool for genome assemblies. *Bioinformatics* 29(8):1072–1075. doi:10.1093/bioinformatics/btt086
5. Li H (2018) Minimap2: Pairwise alignment for nucleotide sequences. *Bioinformatics* 34(18):3094–3100. doi:10.1093/bioinformatics/bty191
6. Manni M et al. (2021) BUSCO update: Novel and streamlined workflows along with broader and deeper phylogenetic coverage for scoring of eukaryotic, prokaryotic, and viral genomes. *Mol Biol Evol* 38(10):4647–4654. doi:10.1093/molbev/msab199
7. Chen S et al. (2018) fastp: An ultra-fast all-in-one FASTQ preprocessor. *Bioinformatics* 34(17):i884–i890. doi:10.1093/bioinformatics/bty560
8. SRA dataset: SRR2584863. NCBI BioProject PRJNA295606.
