
# Week 10 Practicals

# Introduction
In this practical, we will use a RNA-Seq dataset from a model plant *Arabidopsis thaliana* (Wang et al, 2017, https://onlinelibrary.wiley.com/doi/10.1111/tpj.13481). This dataset was initially used for identification of long non-coding RNAs by transcriptome assembly of RNA-Seq.   

## Dataset
Due to the the limitation of computing resources in our VM, we will use a subset of raw reads (reads from chromomose 2 of Arabidopsis) from one of three biological replicates sampled from leaves tissue. Here are some informations for Arabidopsis and this RNA-Seq datatset:

- Reference genome build: TAIR10 (https://www.arabidopsis.org/index.jsp)
- Number of chromosomes: 5 chromosomes + Chloroplast + Mitochondria
- Genome size: ~135 Mb

- Number of raw reads (pairs): 2,991,799
- Sequencing type: PE125

## Tools and pipeline
The pipeline of today's Prac is shown in following flowchart

Tools usded in this Prac:

| Tool/Package| Version     | URL |
| ----------- | ----------- | ----------- |
| fastQC      | v0.11.9     |   https://www.bioinformatics.babraham.ac.uk/projects/fastqc/   |
| cutadapt    | 1.18        |   https://cutadapt.readthedocs.io/en/stable/   |
| Trinity     | v2.8.5      |   https://github.com/trinityrnaseq/trinityrnaseq/wiki   |
| GMAP        | 2020-10-14  |   http://research-pub.gene.com/gmap/   |
| STAR        | 2.7.9a      |   https://github.com/alexdobin/STAR   |
| StringTie   | v2.1.7      |   https://ccb.jhu.edu/software/stringtie/   |
| BUSCO       | 2.0         |   https://busco.ezlab.org/   |
| IGV         | web         |   https://software.broadinstitute.org/software/igv/ |


## Running time estimate (based on teaching VM)

The following table shows the estimated run time in VM for the major steps:

| Step        | Tool/Package| Estimated run time |
| ----------- | ----------- | ----------- |
| QC                            | fastQC      | 3 mins  |
| Sequence trimming             | cutadapt    | 4 mins  |
| De novo assembly              | Trinity     | 2.5 hrs |
| Genome mapping for transcripts| GMAP        | 3 mins  |
| Genome mapping for short reads| STAR        | 10 mins |
| Genome-guided assembly        | StringTie   | 1 mins  |
| BUSCO for Trinity             | BUSCO       | 3 mins  |
| BUSCO for StringTie           | IGV         | N/A     |

## What you will learn in this Practicl

- Practice bash commands you have learned
- Practice QC for NGS data you have learned
- Learn how to do de novo transcriptome assembly
- Learn how to do genome-guided transcriptome assembly

# Practicals for transcriptome assembly
There will be three parts to do the transcriptome assembly in this Prac. Please follow the order in following instructions, because some commands will rely on the results from previous commands. Feel free to talk to tutors/instructors when you have any issue.

## Part 1, Setup and data preparation
In this part, we will activate the conda environment and install tools/packages required in this project. We will also create some folders to keep files (including input RNA-Seq raw reads, databases and output files) organised.

### 1.1 Activate conda environment and install needed tools
The first step is to activate the conda environment. This conda environment have most of the tools that we are going to use pre-installed.

As you have learned from previous Pracs, after you login your VM and open the terminal you will see the prompt in the terminal window. This is the prompt in my VM.

```bash
(base) student@bioinf-2021-s2-student-34:~$
```

After you are in the terminal window, we will activte the conda environment using following command:

```bash
conda activate 3000
```

And you will see that the prompt of your terminal becomes:

```bash
(3000) student@bioinf-2021-s2-student-34:~$
```

Did you see the difference. The `(3000)` at the beginning of your prompt indicates that you are in the `3000` conda environment.

Then we need to install several tools/packages via conda. Type the following commands in your terminal to install `Trinity`, `GMAP`, `STAR`, `BUSCO` respectively.

```bash
conda install -c bioconda trinity
conda install -c bioconda gmap
conda install -c bioconda star
conda install -c bioconda busco
```

You may need to press `enter` to confirm the installation.

Once you have all tools installed, you can use following commands to check whether all required tools/packages have been successfully installed in your conda environment.

```bash
conda list
```

After all required tools/pakcages are successfully installed in your conda environment, we will create some folders to store files used in this Prac.

### 1.2 Prepare folder structure for the project
For each project, I normally store files at different processing stages into different folders. I put initial input data into `data` folder, and output files from different processing stages into separate folders under `results` folder. If there are databases involved, I also create a `DB` folder. I store all scirpts in a separate `scripts` folder (we won't use this folder in this Prac).

```bash
mkdir practical_week10
cd practical_week10
mkdir data DB results
cd results
mkdir 1_QC 2_clean_data 3_denovo_assembly 4_genome_guided_assembly 5_final_assembly
```

The final folder structure for this project will be like this:

```
practical_week10/
├── DB
├── data
└── results
    ├── 1_QC
    ├── 2_clean_data
    ├── 3_denovo_assembly
    ├── 4_genome_guided_assembly
    └── 5_final_assembly
```

### 1.3 upload raw data and DB

The initial RNA-Seq raw data will be downloaded and stored in `data` folder.

```bash
cd ~/practical_week10/data
wget ...
```

The databases, including Arabidopsis reference genome (TAIR10_chrALL.fa), annotated genes (TAIR10_GFF3_genes.gtf), and BUSCO lineages file (viridiplantae_odb10.2020-09-10.tar.gz) will be downloaded and stored in folder `DB`.

```bash
cd ~/practical_week10/DB
wget ...
tar -zxvf viridiplantae_odb10.2020-09-10.tar.gz
```

Now, all the setup work is done. Let's move to the part 2.

## Part 2, QC
In This part, we will use the skills that we learned from the `NGS_practicals` to do QC and trim adaptor and low-quality sequences for our RNA-Seq raw data. 

### 2.1 QC for raw reads
The first step is to do QC for the raw reads using fastQC

```bash
cd ~/practical_week10/results/1_QC
fastqc -t 2 -o ./ ~/practical_week10/data/Col_leaf_chr2_R1.fastq.gz
fastqc -t 2 -o ./ ~/practical_week10/data/Col_leaf_chr2_R2.fastq.gz
```

Then we can check the QC report of raw reads.

### 2.2 Adaptor and low-quality sequence trimming

After we finish the QC for raw reads, we need to trim adaptor and low-quality sequences from the raw reads. The adaptors for this RNA-Seq dataset are Illumina TrueSeq adaptors as `AGATCGGAAGAGCACACGTCTGAACTCCAGTCA` and `AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT`.

```bash
cd ~/practical_week10/results/2_clean_data
cutadapt -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -o Col_leaf_chr2_R1.clean.fastq.gz -p Col_leaf_chr2_R2.clean.fastq.gz --minimum-length 25 --quality-cutoff 20 ~/practical_week10/data/Col_leaf_chr2_R1.fastq.gz ~/practical_week10/data/Col_leaf_chr2_R2.fastq.gz
```

Then we can do QC for clean reads.

### 2.3 QC for clean reads

```bash
cd ~/practical_week10/results/2_clean_data
fastqc -t 2 -o ./ Col_leaf_chr2_R1.clean.fastq.gz
fastqc -t 2 -o ./ Col_leaf_chr2_R2.clean.fastq.gz
```

We can check the QC reports for clean reads and compare them with the raw reads.

## Part 3, De novo transcriptome assembly

So far we have trimmed the adaptor and low-quality sequences from the reads. Next we will do de novo transcriptome assembly using Trinity. 

### 3.1 De novo assembly using Trinity

We will use the clean reads as input.

```bash
cd ~/practical_week10/results/3_denovo_assembly
Trinity --seqType fq --left ~/practical_week10/results/2_clean_data/Col_leaf_chr2_R1.clean.fastq.gz --right ~/practical_week10/results/2_clean_data/Col_leaf_chr2_R2.clean.fastq.gz --output Col_leaf_chr2_trinity --CPU 2 --max_memory 8G --no_salmon
```

This step will take more than 2 hours, so we will finish here today and let the VM finish the job, and we will come back to run remaining analyses in Thursday's Prac.

### 3.2 Get Trinity assembly statistics

All results from Trinity will be stored in the folder of "Col_leaf_chr2_trinity" under directory of "~/practical_week10/results/3_denovo_assembly". It includes a lot of intermediate files and log files during the Trinity assembly processes. The file inlcudes all final assembled transcripts is named "Trinity.fasta".  

Trinity package provide a useful utility to summarise the basic assembly statitics.

```bash
cd ~/practical_week10/results/3_denovo_assembly
TrinityStats.pl ./Col_leaf_chr2_trinity/Trinity.fasta
```
And you will get output like this:

```
################################
## Counts of transcripts, etc.
################################
Total trinity 'genes':  8194
Total trinity transcripts:      11689
Percent GC: 41.86

########################################
Stats based on ALL transcript contigs:
########################################

        Contig N10: 4472
        Contig N20: 3186
        Contig N30: 2618
        Contig N40: 2177
        Contig N50: 1829

        Median contig length: 699
        Average contig: 1109.66
        Total assembled bases: 12970774


#####################################################
## Stats based on ONLY LONGEST ISOFORM per 'GENE':
#####################################################

        Contig N10: 3642
        Contig N20: 2743
        Contig N30: 2186
        Contig N40: 1818
        Contig N50: 1502

        Median contig length: 438
        Average contig: 836.52
        Total assembled bases: 6854460
```

This report gives us some basic statistics of assembled transcripts from Trinity, such as number of genes and transcripts, length of transcripts. 

Question? Difference between transcripts and genes? Why we got more transcripts than genes?

### 3.3 Assessing the assembly quality using BUSCO

After we get the assembled transcripts, we need to find some ways to assess the quality of the assembly, such as the completeness of genes/transcripts. On of a popular ways that we can check the assembly quality is using BUSCO (Benchmarking Universal Single-Copy Orthologs, https://busco.ezlab.org/). BUSCO provides quantitative measures for the assessment of genome assembly, gene set, and transcriptome completeness, based on evolutionarily-informed expections of gene content from near-universal single-copy orthologs selected from different lineages.

```bash
cd ~/practical_week10/results/3_denovo_assembly
busco -i ./Col_leaf_chr2_trinity/Trinity.fasta -l ~/practical_week10/DB/viridiplantae_odb10 -o BUSCO_Trinity_viridiplantae -m transcriptome --cpu 2
```

BUSCO will output a bunch of files including the information of predicted ORFs (Open Reading Frames) from assembled transcripts, output files from blast search against orthologs. Among of these output files, the most important one is this text file called "short_summary_BUSCO_Trinity_viridiplantae.txt". You can find one summary line which looks like this `C:28.0%[S:20.2%,D:7.8%],F:8.5%,M:63.5%,n:42`. This line summarises the completeness of assembled transcripts, and explanations of these number can be found in the same text file after this one line summary.

```
        C:28.0%[S:20.2%,D:7.8%],F:8.5%,M:63.5%,n:425

        119     Complete BUSCOs (C)
        86      Complete and single-copy BUSCOs (S)
        33      Complete and duplicated BUSCOs (D)
        36      Fragmented BUSCOs (F)
        270     Missing BUSCOs (M)
        425     Total BUSCO groups searched
```

### 3.4 Align assembled transcripts to genome (Only appliable when there is a reference genome)

Arabidopsis has a reference genome, therefore we can map the assembled transcirpts to the reference genome to get a genomic coordinates file (GTF or GFF format). We will use an aligner called `GMAP` (http://research-pub.gene.com/gmap/) to do genome mapping.

To use GMAP, we first need to build the genome index.

```bash
cd ~/practical_week10/DB
gmap_build -D ./ -d TAIR10_GMAP TAIR10_chrALL.fa
```

After building the genome index, we can map the assembled transcripts to the reference genome using following commands:

```bash
cd ~/practical_week10/results/3_denovo_assembly
gmap -D ~/practical_week10/DB/TAIR10_GMAP -d TAIR10_GMAP -t 2 -f 3 -n 1 ./Col_leaf_chr2_trinity/Trinity.fasta > Trinity.gff3
```

This will generate a GFF3 format file, including genomic coordinates for our de novo assembled transcripts, which we can import it into IGV for visualisation.

## Part 4, Genome guided transcriptome assembly (Only appliable when there is a reference genome)

In this section, we will do genome guided transcriptome assembly using STAR and Stringie.

### 4.1 Genome mapping using STAR
The first step for genome guided transcriptome assembly is to map RNA-Seq reads to a reference genome. We will use STAR to do this job in this practical (You can also choose other short RNA aligners to do the mapping).

For genome mapping using STAR, we first need to build the genome index.

```bash
cd ~/practical_week10/DB
STAR --runThreadN 2 --runMode genomeGenerate --genomeDir ~/practical_week10/DB/TAIR10_STAR125 --genomeFastaFiles TAIR10_chrALL.fa --sjdbGTFfile TAIR10_GFF3_genes.gtf --sjdbOverhang 125
```

And then, we map the clean reads to the reference genome.

```bash
cd ~/practical_week10/results/4_genome_guided_assembly
STAR --genomeDir ~/practical_week10/DB/TAIR10_STAR125 --readFilesIn ~/practical_week10/results/2_clean_data/Col_leaf_chr2_R1.clean.fastq.gz ~/practical_week10/results/2_clean_data/Col_leaf_chr2_R2.clean.fastq.gz --readFilesCommand zcat --runThreadN 2 --outSAMstrandField intronMotif --outSAMattributes All --outFilterMismatchNoverLmax 0.03 --alignIntronMax 10000 --outSAMtype BAM SortedByCoordinate --outFileNamePrefix Col_leaf_chr2. --quantMode GeneCounts
```

STAR will output multiple files with prefix `Col_leaf_chr2`. To get an idea about the mapping info, you can check the `Col_leaf_chr2.final.Log.out`. The mapped reads were stored in a bam file called `Col_leaf_chr2.Aligned.sortedByCoord.out.bam`. To view this file in IGV, we need to create an index file.

```bash
cd ~/practical_week10/results/4_genome_guided_assembly
samtools index Col_leaf_chr2.Aligned.sortedByCoord.out.bam
```

In next step, we will use this mapped bam file to do genome guided transcriptome assembly.

### 4.2 Assembly using StringTie

The command for genome guided transcriptome assembly is as follows:

```bash
cd ~/practical_week10/results/4_genome_guided_assembly
stringtie Col_leaf_chr2.Aligned.sortedByCoord.out.bam -o StringTie.gtf -p 2
```

StringTie only output the genome coordinates file of assembled trasncripts. We can get the sequences of assembled transcripts using a command called "gffread".

```bash
cd ~/practical_week10/results/4_genome_guided_assembly
gffread -w StringTie.fasta -g ~/practical_week10/DB/TAIR10_chrALL.fa StringTie.gtf
```

### 4.3 Assessing the assembly quality using BUSCO
As we did before, we can use BUSCO to assess the assembly quality.

```bash
cd ~/practical_week10/results/4_genome_guided_assembly
busco -i StringTie.fasta -l ~/practical_week10/DB/viridiplantae_odb10 -o BUSCO_StringTie_viridiplantae -m transcriptome --cpu 2
```

Questions? 
- What is the short summary of BUSCO report for StringTie assembly?
- How many complete orthologs?
- How many duplicated and single copy orthologs?
- Which assembly is better, compared to Trinity assembly?

## Part 5, Visualsing your assembled transcripts in IGV

After we finish the de novo transcriptome assembly and genome guided transcriptome assembly, we can visualise out results in IGV to check the assembly quality and look for interested genes/transripts.

First, we will put all important final files into one folder.

```bash
cd ~/practical_week10/results/5_final_assembly
cp ~/practical_week10/results/3_denovo_assembly/Col_leaf_chr2_trinity/Trinity.fasta ./
cp ~/practical_week10/results/3_denovo_assembly/Trinity.gff3 ./
cp ~/practical_week10/results/4_genome_guided_assembly/StringTie.fasta ./
cp ~/practical_week10/results/4_genome_guided_assembly/StringTie.gtf ./
cp ~/practical_week10/results/4_genome_guided_assembly/Col_leaf_chr2.Aligned.sortedByCoord.out.bam ./
cp ~/practical_week10/results/4_genome_guided_assembly/Col_leaf_chr2.Aligned.sortedByCoord.out.bam.bai ./
```
Then we can check the assembled transcripts by loading the `Trinity.gff3` and `StringTie.gtf` into IGV. We can also load the bam file into IGV to check the reads supporting assembled transcripts.
