[![Static Badge](https://img.shields.io/badge/LICENSE-MIT-yellow)](https://opensource.org/license/mit/) [![Static Badge](https://img.shields.io/badge/FASTQC-v.0.12.0-blue)](https://github.com/s-andrews/FastQC) [![Static Badge](https://img.shields.io/badge/fastp-v.0.20.1-blue)](https://github.com/OpenGene/fastp/releases)  [![Static Badge](https://img.shields.io/badge/bwa-v.0.7.17-blue)](https://github.com/lh3/bwa)  [![Static Badge](https://img.shields.io/badge/java-%3E%3Dv.17-pink)
](https://openjdk.org/projects/jdk/17/) [![Static Badge](https://img.shields.io/badge/gatk-%3E%3Dv.4.4.0.0-teal)
](https://hub.docker.com/r/broadinstitute/gatk/) [![Static Badge](https://img.shields.io/badge/snpEff%26SnpSift-%3Dv.5.2-purple)
](https://pcingola.github.io/SnpEff/)    [![Static Badge](https://img.shields.io/badge/InterVar-%3E%3Dv.2.2.2-green)](https://github.com/WGLab/InterVar)


## Introduction
Clinical genomics analysis refers to the application of genomic technologies and computational methods to understand the genetic basis of diseases and inform clinical decision-making. It involves the analysis of genomic data obtained from patients to identify genetic variations, mutations, and other molecular features that may contribute to disease development or progression. The goal is to use this information to guide diagnosis, treatment decisions, and personalized medicine approaches.

## Prerequisites:
Before we begin, please have all the files from the [GATK Resource Bundle](https://console.cloud.google.com/storage/browser/genomics-public-data/resources/broad/hg38/v0;tab=objects?prefix=&forceOnObjectsSortingFiltering=false) and keep all the files downloaded in one folder (let's name the folder resource). We can create a folder using the `mkdir` command.


## A step-by-step guide to Clinical Genomics analysis:

### 1. Pre-processing reads: 
Pre-processing the reads involves checking the quality (Phred-scores) using tools like [FASTQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) or [MultiQC](https://github.com/MultiQC/MultiQC). After that, we need to remove adapter contamination and trim quality reads using either one of the following tools : [fastp](https://github.com/OpenGene/fastp?tab=readme-ov-file), [cutadapt](https://cutadapt.readthedocs.io/en/stable/), [TrimGalore](https://www.bioinformatics.babraham.ac.uk/projects/trim_galore/) or [Trimmomatic](/http://www.usadellab.org/cms/?page=trimmomatic).

The following code can be used to check the quality of reads obtained from the sequencer.

#### Running FastQC

```
mkdir fastq_bt  # bt stands for before trimming
#Running FASTQC
fastqc -o fastq_bt gatk_demo1.fastq.gz gatk_demo2.fastq.gz

```
#### Running FASTP
```
fastp -i gatk_demo1.fastq.gz -I gatk_demo2.fastq.gz -o gatk_demo1_trimmed.fastq.gz -O gatk_demo2_trimmed.fastq.gz \
       -R gatk_demo -h gatk_demo.html -j gatk_demo.json --detect_adapter_for_pe

In case you want to use cutadapt or trimgalore ( which is essentially a wrapper around cutadapt, 
```
#### Running FastQC again after Adapter and Quality Trimming
```
mkdir fastq_at  # at stands for after trimming
#Running FASTQC
fastqc -o fastq_at gatk_demo1_trimmed.fastq.gz gatk_demo2_trimmed.fastq.gz

```
### 2. Aligning reads to reference genome:
Aligning sequencing reads to a reference genome is a crucial step in many bioinformatics workflows, including variant calling and downstream genomic analyses. The Burrows-Wheeler Aligner [BWA](https://github.com/lh3/bwa) is a popular tool for aligning short DNA sequences to a large reference genome efficiently. Before performing the alignment, we first need to index the reference genome.
#### Indexing the reference genome
```
cd resource
bwa index Homo_sapiens_assembly38.fasta gatk_demo1_trimmed.fastq.gz gatk_demo2_trimmed.fastq.gz > demo.sam

```
Note: Since the Homo spaiens reference genome is very huge in size (3GB), the indexing process can take anywhere between 45min to 2hrs, depending on your system configurations. Alternatively, if you have downloaded all the files in the GATK resource bundle folder, then the respective index files from BWA are already downloaded, and you do not need to index your reference genome again.

#### Alignment to the indexed reference genome
```
bwa mem Homo_sapiens_assembly38.fasta
```

### 3. Alignment Post-Processing:
After aligning your sequencing reads to a reference genome using BWA, there are several post-processing steps you may want to perform to analyze and manipulate the alignment results. For this purpose, we will be utilising the [Picard](https://broadinstitute.github.io/picard/) tool. Picard is a collection of command-line tools for manipulating high-throughput sequencing (HTS) data files, such as SAM and BAM files. These tools are developed by the Broad Institute and are widely used in genomics and bioinformatics workflows. Picard tools are often employed for quality control, data preprocessing, and various manipulations of sequence data. Here are some common post-processing steps:
- **3.1 Adding one or more read groups to your SAM file:** The [AddOrReplaceReadGroups](https://gatk.broadinstitute.org/hc/en-us/articles/360037226472-AddOrReplaceReadGroups-Picard-) tool in Picard is used to add or replace read group information in a SAM or BAM file. This is important for downstream applications that require proper grouping and identification of reads, especially when dealing with data from multiple sequencing libraries or samples. Even though for this demo data purpose, we have a single sample bam file, still this step is absolutely mandatory for the GATK pipeline! The GATK Pipeline requires at least one ReadGroup id to be specified in the  Here's an example of how to use the AddOrReplaceReadGroups tool:
  ```
  java -jar picard.jar AddOrReplaceReadGroups \
         I=input.bam \
         O=output.bam \
         RGID=ID123 \
         RGLB=library1 \
         RGPL=illumina \
         RGPU=unit1 \
         RGSM=sample1

  ```

  Note: For those of you who are running [GATK in Docker](https://gatk.broadinstitute.org/hc/en-us/articles/360035889991--How-to-Run-GATK-in-a-Docker-container), you dont need to install picard separately. All the functionalities of picard are already present within GATK, check out the script (clinical_genomics_demo.sh) for clarity. However, those of you who will have downloaded the GATK https://github.com/broadinstitute/gatk/releases/download/4.4.0.0/gatk-4.4.0.0.zip. Make sure you have Java (v.>= 17) installed, and then you can execute the Picard command as shown in the example. Additionally, specify the full path to the picard.jar file in the command, depending on your system configuratio
- **3.2 SAM to BAM conversion:** The [SamFormatConverter](https://gatk.broadinstitute.org/hc/en-us/articles/360037058992-SamFormatConverter-Picard-) tool in Picard can be used to convert a SAM file to BAM format. Here's an example of how to use SamFormatConverter for this purpose:
  ```
  java -jar picard.jar SamFormatConverter \
         I=input.sam \
         O=output.bam

  ```
- **3.3 Sorting BAM file:** The [SortSam](https://gatk.broadinstitute.org/hc/en-us/articles/360037594291-SortSam-Picard-) tool in Picard is used to sort a SAM or BAM file by coordinate order. Sorting is a necessary step for downstream analyses and visualization tools that require data to be ordered by genomic coordinates. Here's an example of how to use SortSam:
  ```
  java -jar picard.jar SortSam \
         I=input.bam \
         O=sorted_output.bam \
         SORT_ORDER=coordinate

  ```
- **3.4 Marking (and optionally deleting) duplicates:** In Picard, the [MarkDuplicates](https://gatk.broadinstitute.org/hc/en-us/articles/360037052812-MarkDuplicates-Picard-) tool is used to identify and mark duplicate reads in a SAM or BAM file. Marking duplicates is a common step in the preprocessing of sequencing data, especially when dealing with PCR-amplified libraries. Here's an example of how to use MarkDuplicates:
  ```
  java -jar picard.jar MarkDuplicates \
      I=input.bam \
      O=marked_duplicates.bam \
      M=marked_dup_metrics.txt
  ```

### 4. Base Quality score recalibration (not mandatory, but highly recommended) :
### 5. Variant Calling:
### 6. Variant Filtering:
- **6.1 Splitting variants into SNPs and INDELs:**
- **6.2 Variant Quality Score Recalibration:**
- **6.3 Hard Filtering Variants:**
### 7. Variant Annotation:
### 8. Clinical Interpretation:
   



