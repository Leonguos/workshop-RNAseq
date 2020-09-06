---
layout: default
title:  'Exercise RNAseq assembly'
---

# Preparing RNAseq data for assembly

This exercise is meant to get you acquainted with the type of data you would normally encounter in an annotation project. We will for all exercises use data for the fruit fly, Drosophila melanogaster, as that is one of the currently best annotated organisms and there is plenty of high quality data available.

You can create the folders where you want but I would suggest a folder organisation, if you do not follow this organisation remmember to put the correct path to your data

```
cd
mkdir RNAseq_assembly_annotation
cd RNAseq_assembly_annotation

```
Here I will need you to clone our github indeed this github contain NBIS annotation team scripts and will be used during this pratical

```
git clone https://github.com/NBISweden/GAAS.git
```
```
ln -s /proj/uppstore2017171/courses/RNAseqWorkshop/downloads/assembly_annotation .

mkdir RNAseq_assembly

cd RNAseq_assembly
```


## Assembling transcripts based on RNA-seq data

Rna-seq data is in general very useful in annotation projects as the data usually comes from the actual organism you are studying and thus avoids the danger of introducing errors caused by differences in gene structure between your study organism and other species.

Important remarks to remember before starting working with RNA-seq:

- Check if RNAseq are paired or not. Last generation of sequenced short reads (since 2013) are almost all paired. Anyway, it is important to check that information, which will be useful for the tools used in the next steps.

- Check if RNAseq are stranded. Indeed this information will be useful for the tools used in the next steps. (In general way we recommend to use stranded RNAseq to avoid transcript fusion during the transcript assembly process. That gives more reliable results. )

- Left / L / forward / 1 are identical meaning. It is the same for Right / R /Reverse / 2


### Checking encoding version and fastq quality score format

To check the technology used to sequences the RNAseq and get some extra information we have to use fastqc tool.

```
module load bioinfo-tools
module load FastQC/0.11.5

mkdir fastqc_reports

fastqc ~/RNAseq_assembly_annotation/assembly_annotation/raw_computes/ERR305399_1.fastq.gz -o fastqc_reports/
```
scp the html file resulting of fastqc, what kind of result do you have?

```
scp login@rackham.uppmax.uu.se:/home/login/RNAseq_annotation/YOURFILE .
```
Checking the fastq quality score format :

As we will be using the scripts libraries available in the git gaas you need first to export the libraries :

```
export PERL5LIB=$PERL5LIB:~/RNAseq_assembly_annotation/GAAS/annotation/
```
Then :
```
~/RNAseq_assembly_annotation/GAAS/annotation/Tools/bin/fastq_guessMyFormat.pl -i ~/RNAseq_assembly_annotation/assembly_annotation/raw_computes/ERR305399_1.fastq.gz

```

In the normal mode, it differentiates between Sanger/Illumina1.8+ and Solexa/Illumina1.3+/Illumina1.5+.
In the advanced mode, it will try to pinpoint exactly which scoring system is used.

More test can be made and should be made on RNA-seq data before doing the assembly, we have not time to do all of them during this course. have a look [here](https://en.wikipedia.org/wiki/List_of_RNA-Seq_bioinformatics_tools)

## Genome-Guided Transcriptome Assembly

### Trimmomatic/Hisat2/Stringtie

#### Trimmomatic

[Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) performs a variety of useful trimming tasks for illumina paired-end and single ended data.The selection of trimming steps and their associated parameters are supplied on the command line.

```
mkdir trimmomatic
module load bioinfo-tools
module load trimmomatic/0.36
```

The following command line will perform the following:
	• Remove adapters (ILLUMINACLIP:TruSeq3-PE.fa:2:30:10)
	• Remove leading low quality or N bases (below quality 3) (LEADING:3)
	• Remove trailing low quality or N bases (below quality 3) (TRAILING:3)
	• Scan the read with a 4-base wide sliding window, cutting when the average quality per base drops below 15 (SLIDINGWINDOW:4:15)
	• Drop reads below the 36 bases long (MINLEN:36)

```
java -jar /sw/apps/bioinfo/trimmomatic/0.36/milou/trimmomatic-0.36.jar PE -threads 5 -phred33 ~/RNAseq_assembly_annotation/assembly_annotation/raw_computes/ERR305399_1.fastq.gz ~/RNAseq_assembly_annotation/assembly_annotation/raw_computes/ERR305399_2.fastq.gz trimmomatic/ERR305399.left_paired.fastq.gz trimmomatic/ERR305399.left_unpaired.fastq.gz trimmomatic/ERR305399.right_paired.fastq.gz trimmomatic/ERR305399.right_unpaired.fastq.gz ILLUMINACLIP:/sw/apps/bioinfo/trimmomatic/0.36/milou/adapters/TruSeq3-PE.fa:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36

```


#### Hisat2

Once the reads have been trimmed, we use [hisat2](https://ccb.jhu.edu/software/hisat2/index.shtml) to align the RNA-seq reads to a genome in order to identify exon-exon splice junctions.
HISAT2 is a fast and sensitive alignment program for mapping next-generation sequencing reads (whole-genome, transcriptome, and exome sequencing data) against a reference genome.

First you need to build an index of your chromosome 4

```
mkdir index

module load HISAT2
module load samtools/1.8
hisat2-build ~/RNAseq_assembly_annotation/assembly_annotation/chromosome/chr4.fa index/chr4_index
```

Then you can run Hisat2 :
--phred33 Input qualities are ASCII chars equal to the Phred quality plus 33. This is also called the "Phred+33" encoding, which is used by the very latest Illumina pipelines (you checked it with the script fastq_guessMyFormat.pl)
--rna-strandness <string> For single-end reads, use F or R. 'F' means a read corresponds to a transcript. 'R' means a read corresponds to the reverse complemented counterpart of a transcript. For paired-end reads, use either FR or RF. (RF means fr-firststrand see [here](https://github.com/NBISweden/GAAS/blob/master/annotation/CheatSheet/rnaseq_library_types.md) for more explanation).
--novel-splicesite-outfile <path> In this mode, HISAT2 reports a list of splice sites in the file :
chromosome name <tab> genomic position of the flanking base on the left side of an intron <tab> genomic position of the flanking base on the right <tab> strand (+, -, and .) '.' indicates an unknown strand for non-canonical splice sites.

```
hisat2 --phred33 --rna-strandness RF --novel-splicesite-outfile hisat2/splicesite.txt -S hisat2/accepted_hits.sam -p 5 -x index/chr4_index -1 trimmomatic/ERR305399.left_paired.fastq.gz -2 trimmomatic/ERR305399.right_paired.fastq.gz
```

Finally you need to change the sam into bam file and to sort it in order for stringtie to use the bam file to assemble the read into transcripts.

```
samtools view -bS -o hisat2/accepted_hits.bam hisat2/accepted_hits.sam

samtools sort -o hisat2/accepted_hits.sorted.bam hisat2/accepted_hits.bam
```


#### Stringtie

[StringTie](https://ccb.jhu.edu/software/stringtie/) is a fast and highly efficient assembler of RNA-Seq alignments into potential transcripts. It uses a novel network flow algorithm as well as an optional de novo assembly step to assemble and quantitate full-length transcripts representing multiple splice variants for each gene locus.
You can add as input an annotation from gtf/gff3 file to calculate TPM and FPKM values.


```
module load bioinfo-tools
module load StringTie

stringtie hisat2/accepted_hits.sorted.bam -o stringtie/transcripts.gtf
```

When done you can find your results in the directory ‘outdir’. The file transcripts.gtf includes your assembled transcripts.

You could now also visualise all this information using a genome browser, such as IGV. IGV requires a genome fasta file and any number of annotation files in GTF or GFF3 format (note that GFF3 formatted file tend to look a bit weird in IGV sometimes).

Transfer the gtf files to your computer using scp:

```
scp __YOURLOGIN__@rackham.uppmax.uu.se:/home/__YOURLOGIN__/RNAseq_assembly_annotation/RNAseq_assembly/stringtie/transcripts.gtf .
```

Looking at your results, are you happy with the default values of Stringtie (which we used in this exercise) or is there something you would like to change?


### De-novo Transcriptome Assembly

[Trinity](https://github.com/trinityrnaseq/trinityrnaseq/wiki) assemblies can be used as complementary evidence, particularly when trying to polish a gene build with Pasa. Before you start, check how big the raw read data is that you wish to assemble to avoid unreasonably long run times.

```
module load bioinfo-tools
module load trinity/2.4.0
module load samtools

Trinity --seqType fq --max_memory 32G --left ~/RNAseq_assembly_annotation/assembly_annotation/raw_computes/ERR305399_1.fastq.gz --right ~/RNAseq_assembly_annotation/assembly_annotation/raw_computes/ERR305399_2.fastq.gz --CPU 5 --output trinity --SS_lib_type RF
```

Trinity takes a long time to run (like hours), you can stop the program when you start it and have a look at the results, look in ~/RNAseq_assembly_annotation/assembly_annotation/RNAseq/trinity the output is Trinity.fasta


In order to compare the output of stringtie and the output of trinity we need to map the trinity transcript to the chr4 of Drosophila.

We'll use the GMAP software to align the Trinity transcripts to our reference genome. Trinity contains a utility that facilitates running GMAP, which first builds an index for the target genome followed by running the gmap aligner:

```
module load gmap-gsnap

mkdir gmap

/sw/apps/bioinfo/trinity/2.4.0/rackham/util/misc/process_GMAP_alignments_gff3_chimeras_ok.pl --genome ~/RNAseq_assembly_annotation/assembly_annotation/chromosome/chr4.fa --transcripts ~/RNAseq_assembly_annotation/assembly_annotation/RNAseq/trinity/Trinity.fasta > gmap/transcript_trinity.gff
```


## Assessing quality of you RNAseq assembly

There are different ways of assessing the quality of your assembly, you will find some of them [here](https://github.com/trinityrnaseq/trinityrnaseq/wiki/Transcriptome-Assembly-Quality-Assessment).

We will run busco to check the the quality of the assembly.
[BUSCO](https://busco.ezlab.org/) provides measures for quantitative assessment of genome assembly, gene set, and transcriptome completeness (what we are going to do here). Genes that make up the BUSCO sets for each major lineage are selected from orthologous groups with genes present as single-copy orthologs in at least 90% of the species in the chosen branch of tree of life.

For the trinity results :

```
module load BUSCO/3.0.2b

/sw/apps/bioinfo/BUSCO/3.0.2b/rackham/bin/run_BUSCO.py -i ~/RNAseq_assembly_annotation/assembly_annotation/RNAseq/trinity/Trinity.fasta -o busco_trinity -l $BUSCO_LINEAGE_SETS/arthropoda_odb9 -m tran -c 5
```

Busco will take 30 to run so you can check the results in ~/RNAseq_assembly_annotation/assembly_annotation/RNAseq/busco_trinity


For the guided assembly results

You need first to extract the transcript sequences from the gtf transcript file :

```
module load BioPerl
export PERL5LIB=$PERL5LIB:~/RNAseq_assembly_annotation/GAAS/annotation/
~/RNAseq_assembly_annotation/GAAS/annotation/Tools/bin/gff3_sp_extract_sequences.pl --cdna -g transcripts.gtf -f ~/RNAseq_assembly_annotation/assembly_annotation/chromosome/chr4.fa -o transcripts_stringtie.fa

```
Then you can run busco again :

```
/sw/apps/bioinfo/BUSCO/3.0.2b/rackham/bin/run_BUSCO.py -i stringtie/transcripts_stringtie.fa -o busco_stringtie -l $BUSCO_LINEAGE_SETS/arthropoda_odb9 -m tran -c 5

```

Compare the two busco, what do you think happened for stringtie?


## What's next?

Now you are ready either to annotate your RNAseq or you can use then to do the genome annotation.

For the de-novo assembly you can use the Trinity.fasta file obtained.
For the genome-guided assembly you can either use the Stringtie results transcripts.gtf but you will often need to reformat it into a gff file.