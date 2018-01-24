![](logo.png)

# STREAMclean - Download and filter reads from the SRA in a streaming way

## The Challenge
- The *SRA* is full of sequencing data. 🎉
- Tons of
  - sequencing platforms
  - experiment types (genomic, transcriptomic, metagenomic, younameit)
  - read qualities
- Great, lots of data to play around with, but…
    - often you don't want all the data from an experiment
        - saving 100s of read sets takes lots of space
        - files contain contaminants 😭
        - you only want individual genomes out of a metagenome
- The big question: How can we easily get only the interesting parts of *SRA* sets?

## Our solution
- Get reference genomes of interest or contaminants out of refseq to create a reference database
- Streaming the data right out of the *SRA* and use `magicblast` to compare to our reference database
- only save those reads you actually want!

![](workflow.png)

## How to use it: Bash wrapper script  
`./mapper_wrapper.sh -d test1 -i bacteria -s SRR4420340`

or more specifically

`./mapper_wrapper.sh -d test1 -i "-t 199310 bacteria" -s SRR4420340`


This will:
1. Download specified reference genomes using the [ncbi-genome-download package](https://github.com/kblin/ncbi-genome-download).
2. Create a [magic-blast](https://ncbi.github.io/magicblast/) database of the collected reference genomes.
3. Map the SRA accessions against the whitelist/blacklist reference database.


## Comparing bwa and magicblast

We downloaded the read sets and reference genomes for E. coli ST131 (`SRR5629778` and `HG941718.1`) and Homo sapiens (`SRR2848544` and `GRCh38.p7 Primary Assembly`). Each read set was aligned against its own reference genome and the opposing reference using `bwa 0.7.12-r1039` and `ncbi-magicblast-1.3.0`.

### Self-comparison

#### E.coli

**total reads**
```
grep "@SRR5629778" SRR5629778.ecoli.st131.fastq |wc
10964
```

`bwa`:

**mapped reads**
```
samtools view -bS SRR5629778_bwa.sam |samtools view -F 4 -|cut -f 1|sort|uniq |wc -l
7581
```

**unmapped reads**
```
samtools view -bS SRR5629778_bwa.sam |samtools view -f 4 -|cut -f 1|sort|uniq |wc -l
3383
```

`magicblast`:

**mapped reads**
```
samtools view -bS SRR5629778_magicblast.sam |samtools view -F 4 -|cut -f 1|sort|uniq |wc -l
9919
```

**unmapped reads**
```
samtools view -bS SRR5629778_magicblast.sam |samtools view -f 4 -|cut -f 1|sort|uniq |wc -l
1045
```


#### Homo sapiens

**total reads**
```
grep "@SRR2848544"  ~/bastian/reference_data/SRR2848544.fastq |wc
2912
```
`bwa`:

**mapped reads**
```
samtools view -bS SRR2848544_bwa.sam |samtools view -F 4 -|cut -f 1|sort|uniq |wc -l
2910
```

**unmapped reads**
```
samtools view -bS SRR2848544_bwa.sam |samtools view -f 4 -|cut -f 1|sort|uniq |wc -l
2
```

`magicblast`:

**mapped reads**
```
samtools view -bS SRR2848544_magicblast_default.sam |samtools view -F 4 -|cut -f 1|sort|uniq |wc -l
2912
```

**unmapped reads**
```
samtools view -bS SRR2848544_magicblast_default.sam |samtools view -f 4 -|cut -f 1|sort|uniq |wc -l
3
```

### Cross comparison

#### E. coli reads against Homo sapiens
`bwa`:

**mapped reads**
```
samtools view -bS SRR5629778.ecoli.st131_bwa.sam |samtools view -F 4 -|cut -f 1|sort|uniq |wc -l
0
```

**unmapped reads**
```
samtools view -bS SRR5629778.ecoli.st131_bwa.sam |samtools view -f 4 -|cut -f 1|sort|uniq |wc -l
10964
```

`magicblast`:

**mapped reads**
```
samtools view -bS SRR5629778.ecoli.st131_magicblast.sam |samtools view -F 4 -|cut -f 1|sort|uniq |wc -l
10955
```

**unmapped reads**
```
samtools view -bS SRR5629778.ecoli.st131_magicblast.sam |samtools view -f 4 -|cut -f 1|sort|uniq |wc -l
9
```

#### Homo sapiens reads against E. coli

`bwa`:

**mapped reads**
```
samtools view -bS SRR2848544_bwa_he.sam |samtools view -F 4 -|cut -f 1|sort|uniq |wc -l
0
```

**unmapped reads**
```
samtools view -bS SRR2848544_bwa_he.sam |samtools view -f 4 -|cut -f 1|sort|uniq |wc -l
2912
```

`magicblast`:

**mapped reads**
```
samtools view -bS SRR2848544_magicblast_he.sam |samtools view -F 4 -|cut -f 1|sort|uniq |wc -l
292
```

**unmapped reads**
```
samtools view -bS SRR2848544_magicblast_he.sam |samtools view -f 4 -|cut -f 1|sort|uniq |wc -l
2899
```

## How-to run `bwa`

Create index of the reference genome:

`bwa index reference_genome.fasta reference_genome.fasta`

Align reads

`bwa mem reference_genome.fasta read_set.fastq`
