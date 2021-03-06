---
feature_text: |
  ## Precision Medicine Bioinformatics
  Introduction to bioinformatics for DNA and RNA sequence analysis
title: Developer Notes
categories:
    - Module-10-Appendix
feature_image: "assets/genvis-dna-bg_optimized_v1a.png"
date: 0010-02-01
---

This module is used to document background details that are generally considered too obscure for use in the main workshop but are helpful for the course developers to keep track of certain details.

### Set up reference genome files and store on genomedata.org
```

# set up genome references dir
cd /workspace/
mkdir -p references/genome/
cd references/genome/

# download original references file
wget -c ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/GRCh38_full_analysis_set_plus_decoy_hla.fa
wget -c ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/GRCh38_full_analysis_set_plus_decoy_hla.dict
wget -c ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/20150713_location_of_centromeres_and_other_regions.txt
wget -c ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/GRCh38_full_analysis_set_plus_decoy_hla-extra.fa
wget -c ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/README.20150309.GRCh38_full_analysis_set_plus_decoy_hla
wget -c ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/GRCh38_full_analysis_set_plus_decoy_hla.fa.fai
mkdir all
mkdir chr6
mkdir chr17
mkdir chr6_and_chr17

# store original full genome and rename files for consistency
mv 20150713_location_of_centromeres_and_other_regions.txt all/location_of_centromeres_and_other_regions.txt
mv GRCh38_full_analysis_set_plus_decoy_hla-extra.fa all/ref_genome-extra.fa
mv GRCh38_full_analysis_set_plus_decoy_hla.dict all/ref_genome.dict
mv GRCh38_full_analysis_set_plus_decoy_hla.fa all/ref_genome.fa
mv GRCh38_full_analysis_set_plus_decoy_hla.fa.fai all/ref_genome.fa.fai
mv README.20150309.GRCh38_full_analysis_set_plus_decoy_hla all/README.txt

# split ref genome into pieces
mkdir split/
faSplit byname all/ref_genome.fa split/
mv split/chr6.fa chr6/ref_genome.fa
mv split/chr17.fa chr17/ref_genome.fa
cat chr6/ref_genome.fa chr17/ref_genome.fa > chr6_and_chr17/ref_genome.fa
rm -fr split

# create .fai files for each version of the reference
samtools faidx chr6/ref_genome.fa
samtools faidx chr17/ref_genome.fa
samtools faidx chr6_and_chr17/ref_genome.fa

# create .dict files for each version of the reference
java -jar /usr/local/bin/picard.jar CreateSequenceDictionary R=chr6/ref_genome.fa O=chr6/ref_genome.dict
java -jar /usr/local/bin/picard.jar CreateSequenceDictionary R=chr17/ref_genome.fa O=chr17/ref_genome.dict
java -jar /usr/local/bin/picard.jar CreateSequenceDictionary R=chr6_and_chr17/ref_genome.fa O=chr6_and_chr17/ref_genome.dict

# compress the reference files for storage on the file server
gzip all/ref_genome.fa
gzip chr6/ref_genome.fa
gzip chr17/ref_genome.fa
gzip chr6_and_chr17/ref_genome.fa

# create tarballs for convenient downloading
cd /workspace/references/genome/all
tar -cf ref_genome.tar *
rm -f README.txt location_of_centromeres_and_other_regions.txt ref_genome-extra.fa ref_genome.dict ref_genome.fa.gz ref_genome.fa.fai

cd /workspace/references/genome/chr6
tar -cf ref_genome.tar *
rm -f ref_genome.dict ref_genome.fa.gz ref_genome.fa.fai

cd /workspace/references/genome/chr17
tar -cf ref_genome.tar *
rm -f ref_genome.dict ref_genome.fa.gz ref_genome.fa.fai

cd /workspace/references/genome/chr6_and_chr17
tar -cf ref_genome.tar *
rm -f ref_genome.dict ref_genome.fa.gz ref_genome.fa.fai

```

### Create reference transcriptome files and store on genomedata.org for use in the course
Download transcriptome annotations (GTF files) from Ensembl. Make sure the version used matches the version of VEP installed on the AMI! Be sure to fix the chromosome names in the GTF to be compatible with our reference genome. Finally create sub-setted GTF files for the regions of interest used in the course.

```bash
# create dir for these annotation files
cd /home/ubuntu/workspace/inputs/references/
mkdir transcriptome
cd transcriptome

# download the GTF file for the desired version of Ensembl
wget ftp://ftp.ensembl.org/pub/release-93/gtf/homo_sapiens/Homo_sapiens.GRCh38.93.gtf.gz
gunzip Homo_sapiens.GRCh38.93.gtf.gz

# create a version of the GTF with the chr names fixed for our reference genome (i.e. with chr names, etc.)
# in order to do this we need a .dict file for the whole reference genome
wget http://genomedata.org/pmbio-workshop/references/genome/all/ref_genome.dict
convertEnsemblGTF.pl ref_genome.dict /opt/vep_cache/homo_sapiens/93_GRCh38/chr_synonyms.txt Homo_sapiens.GRCh38.93.gtf > Homo_sapiens.GRCh38.93.namefixed.gtf
rm -f ref_genome.dict

# produce GTF files of various subsets
mkdir all chr6 chr17 chr6_and_chr17
mv Homo_sapiens.GRCh38.93.gtf all/ref_transcriptome_nochrs.gtf
mv Homo_sapiens.GRCh38.93.namefixed.gtf all/ref_transcriptome.gtf
cat all/ref_transcriptome.gtf | grep --color=never -w "^chr6" > chr6/ref_transcriptome.gtf
cat all/ref_transcriptome.gtf | grep --color=never -w "^chr17" > chr17/ref_transcriptome.gtf
cat chr6/ref_transcriptome.gtf chr17/ref_transcriptome.gtf > chr6_and_chr17/ref_transcriptome.gtf

# get the reference genome files we will need at least temporarily
cd ~/workspace/inputs/references/
mkdir temp
cd temp
wget http://genomedata.org/pmbio-workshop/references/genome/all/ref_genome.tar
tar -xvf ref_genome.tar
gunzip ref_genome.fa.gz


# produce transcriptome (cDNA) fasta files of various subsets using our GTF and genome Fasta files
cd ~/workspace/inputs/references/
gtf_to_fasta transcriptome/all/ref_transcriptome.gtf temp/ref_genome.fa transcriptome/all/ref_transcriptome.fa
gtf_to_fasta transcriptome/chr6/ref_transcriptome.gtf temp/ref_genome.fa transcriptome/chr6/ref_transcriptome.fa
gtf_to_fasta transcriptome/chr17/ref_transcriptome.gtf temp/ref_genome.fa transcriptome/chr17/ref_transcriptome.fa
gtf_to_fasta transcriptome/chr6_and_chr17/ref_transcriptome.gtf temp/ref_genome.fa transcriptome/chr6_and_chr17/ref_transcriptome.fa

```

### Create exome target region .bed and interval files for various subsets starting with the manufacturer file
```bash

# get exome file from nimblegen
mkdir -p /workspace/inputs/references/exome
cd /workspace/inputs/references/exome
wget -c https://sequencing.roche.com/content/dam/rochesequence/worldwide/resources/SeqCapEZ_Exome_v3.0_Design_Annotation_files.zip
unzip SeqCapEZ_Exome_v3.0_Design_Annotation_files.zip
rm -f SeqCapEZ_Exome_v3.0_Design_Annotation_files.zip

# perform liftover, sort, and merge
wget -c http://hgdownload.cse.ucsc.edu/goldenPath/hg19/liftOver/hg19ToHg38.over.chain.gz
liftOver SeqCapEZ_Exome_v3.0_Design_Annotation_files/SeqCap_EZ_Exome_v3_hg19_primary_targets.bed  hg19ToHg38.over.chain.gz SeqCap_EZ_Exome_v3_hg38_primary_targets.bed unMapped.bed
cut -f 1-3 SeqCap_EZ_Exome_v3_hg38_primary_targets.bed > SeqCap_EZ_Exome_v3_hg38_primary_targets.v2.bed
bedtools sort -i SeqCap_EZ_Exome_v3_hg38_primary_targets.v2.bed > SeqCap_EZ_Exome_v3_hg38_primary_targets.v2.sort.bed
bedtools merge -i SeqCap_EZ_Exome_v3_hg38_primary_targets.v2.sort.bed > SeqCap_EZ_Exome_v3_hg38_primary_targets.v2.sort.merge.bed

# create the subsets
mkdir all chr6 chr17 chr6_and_chr17
cp SeqCap_EZ_Exome_v3_hg38_primary_targets.v2.sort.merge.bed all/exome_regions.bed
grep -w -P "^chr6" SeqCap_EZ_Exome_v3_hg38_primary_targets.v2.sort.merge.bed > chr6/exome_regions.bed
grep -w -P "^chr17" SeqCap_EZ_Exome_v3_hg38_primary_targets.v2.sort.merge.bed > chr17/exome_regions.bed
grep -w -P "^chr6|^chr17" SeqCap_EZ_Exome_v3_hg38_primary_targets.v2.sort.merge.bed > chr6_and_chr17/exome_regions.bed

# get the .dict files for each subset
wget http://genomedata.org/pmbio-workshop/references/genome/all/ref_genome.dict -O all/ref_genome.dict
wget http://genomedata.org/pmbio-workshop/references/genome/chr6/ref_genome.dict -O chr6/ref_genome.dict
wget http://genomedata.org/pmbio-workshop/references/genome/chr17/ref_genome.dict -O chr17/ref_genome.dict
wget http://genomedata.org/pmbio-workshop/references/genome/chr6_and_chr17/ref_genome.dict -O chr6_and_chr17/ref_genome.dict

# create the matching interval lists
java -jar /usr/local/bin/picard.jar BedToIntervalList I=all/exome_regions.bed O=all/exome_regions.bed.interval_list SD=all/ref_genome.dict
java -jar /usr/local/bin/picard.jar BedToIntervalList I=chr6/exome_regions.bed O=chr6/exome_regions.bed.interval_list SD=chr6/ref_genome.dict
java -jar /usr/local/bin/picard.jar BedToIntervalList I=chr17/exome_regions.bed O=chr17/exome_regions.bed.interval_list SD=chr17/ref_genome.dict
java -jar /usr/local/bin/picard.jar BedToIntervalList I=chr6_and_chr17/exome_regions.bed O=chr6_and_chr17/exome_regions.bed.interval_list SD=chr6_and_chr17/ref_genome.dict

# clean up and store files on genomedata.org
rm -fr SeqCap* unMapped.bed
rm -f */ref_genome.dict
tar -cf transfer.tar *

```

### Prepare original starting data

The plan is to provide students with raw down-sampled fastq files as starting point. These notes document our original source of data files and any transformations needed.

Download (using wget) the WGS/WES data for HCC1395/BL data from: [https://github.com/genome/gms/wiki/HCC1395-WGS-Exome-RNA-Seq-Data](https://github.com/genome/gms/wiki/HCC1395-WGS-Exome-RNA-Seq-Data).

```bash
cd ~/data
mkdir unaligned_bams
cd unaligned_bams
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_D1VCPACXX_6.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_D1VCPACXX_6.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_D1VCPACXX_7.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_D1VCPACXX_8.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_D1VCPACXX_1.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_D1VCPACXX_2.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_D1VCPACXX_3.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_D1VCPACXX_4.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_D1VCPACXX_5.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_C1TD1ACXX_7_CGATGT.bam
wget https://xfer.genome.wustl.edu/gxfer1/project/gms/testdata/bams/hcc1395/gerald_C1TD1ACXX_7_ATCACG.bam
```

For newer RNAseq data (See [https://confluence.gsc.wustl.edu/pages/viewpage.action?spaceKey=CI&title=Cancer+Informatics+Test+Data](https://confluence.gsc.wustl.edu/pages/viewpage.action?spaceKey=CI&title=Cancer+Informatics+Test+Data)) download from inside MGI filesystem.

```bash
scp -i ~/PMB.pem /gscmnt/gc2764/cad/HCC1395/rna_seq/GRCh38/alignments/H_NJ-HCC1395-HCC1395_RNA.bam ubuntu@18.217.114.211:data/unaligned_bams/
scp -i ~/PMB.pem /gscmnt/gc2764/cad/HCC1395/rna_seq/GRCh38/alignments/H_NJ-HCC1395-HCC1395_BL_RNA.bam ubuntu@18.217.114.211:data/unaligned_bams/
```

Rename bam files for easier reference.

```bash
cd /home/ubuntu/data/unaligned_bams
mv gerald_D1VCPACXX_6.bam WGS_Norm_Lane1_gerald_D1VCPACXX_6.bam
mv gerald_D1VCPACXX_7.bam WGS_Norm_Lane2_gerald_D1VCPACXX_7.bam
mv gerald_D1VCPACXX_8.bam WGS_Norm_Lane3_gerald_D1VCPACXX_8.bam
mv gerald_D1VCPACXX_1.bam WGS_Tumor_Lane1_gerald_D1VCPACXX_1.bam
mv gerald_D1VCPACXX_2.bam WGS_Tumor_Lane2_gerald_D1VCPACXX_2.bam
mv gerald_D1VCPACXX_3.bam WGS_Tumor_Lane3_gerald_D1VCPACXX_3.bam
mv gerald_D1VCPACXX_4.bam WGS_Tumor_Lane4_gerald_D1VCPACXX_4.bam
mv gerald_D1VCPACXX_5.bam WGS_Tumor_Lane5_gerald_D1VCPACXX_5.bam
mv gerald_C1TD1ACXX_7_CGATGT.bam Exome_Norm_gerald_C1TD1ACXX_7_CGATGT.bam
mv gerald_C1TD1ACXX_7_ATCACG.bam Exome_Tumor_gerald_C1TD1ACXX_7_ATCACG.bam
mv H_NJ-HCC1395-HCC1395_BL_RNA.bam RNAseq_Norm_H_NJ-HCC1395-HCC1395_BL_RNA.bam
mv H_NJ-HCC1395-HCC1395_RNA.bam RNAseq_Tumor_H_NJ-HCC1395-HCC1395_RNA.bam
```

Save read group information from each bam file (already separated by read group) for later use

```bash
cd /home/ubuntu/data/unaligned_bams
ls -1 | perl -ne 'chomp; print "samtools view -H $_ | grep -H --label=$_ \@RG\n"' | bash > readgroup_info.txt
cat readgroup_info.txt | perl -ne 'my ($bam, $id, $pl, $pu, $lb, $sm); if ($_=~/(\S+\.bam)\:/){$bam=$1} if ($_=~/(ID\:\d+)/){$id=$1} if ($_=~/(PL\:\w+)/){$pl=$1} if ($_=~/(PU\:\S+)/){$pu=$1} if($_=~/LB\:\"(.+)\"/){$lb=$1} if ($_=~/(SM\:\S+)/){$sm=$1} print "$bam\t$id\t$pl\t$pu\t$lb\t$sm\n";' > readgroup_info.clean.txt
```

Revert bams before conversion to fastq (best practice with MGI bams)
- Run times: ~41-318min
- Memory: 2.4-6.3GB

```bash
cd ~/data
mkdir reverted_bams
cd reverted_bams
mkdir Exome_Norm Exome_Tumor WGS_Norm WGS_Tumor RNAseq_Norm RNAseq_Tumor
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/Exome_Norm_gerald_C1TD1ACXX_7_CGATGT.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/Exome_Norm/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/Exome_Tumor_gerald_C1TD1ACXX_7_ATCACG.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/Exome_Tumor/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/WGS_Norm_Lane1_gerald_D1VCPACXX_6.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/WGS_Norm/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/WGS_Norm_Lane2_gerald_D1VCPACXX_7.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/WGS_Norm/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/WGS_Norm_Lane3_gerald_D1VCPACXX_8.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/WGS_Norm/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/WGS_Tumor_Lane1_gerald_D1VCPACXX_1.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/WGS_Tumor/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/WGS_Tumor_Lane2_gerald_D1VCPACXX_2.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/WGS_Tumor/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/WGS_Tumor_Lane3_gerald_D1VCPACXX_3.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/WGS_Tumor/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/WGS_Tumor_Lane4_gerald_D1VCPACXX_4.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/WGS_Tumor/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/WGS_Tumor_Lane5_gerald_D1VCPACXX_5.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/WGS_Tumor/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/RNAseq_Norm_H_NJ-HCC1395-HCC1395_BL_RNA.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/RNAseq_Norm/
java -Xmx16g -jar $PICARD RevertSam I=/home/ubuntu/data/unaligned_bams/RNAseq_Tumor_H_NJ-HCC1395-HCC1395_RNA.bam OUTPUT_BY_READGROUP=true O=/home/ubuntu/data/reverted_bams/RNAseq_Tumor/
```

Convert bams to fastq files

- Run times: 18-58min

```bash
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/Exome_Norm/2891351068.bam F=/data/fastqs/Exome_Norm/2891351068_1.fastq F2=/data/fastqs/Exome_Norm/2891351068_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/Exome_Tumor/2891351066.bam F=/data/fastqs/Exome_Tumor/2891351066_1.fastq F2=/data/fastqs/Exome_Tumor/2891351066_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/WGS_Norm/2891323123.bam F=/data/fastqs/WGS_Norm/2891323123_1.fastq F2=/data/fastqs/WGS_Norm/2891323123_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/WGS_Norm/2891323124.bam F=/data/fastqs/WGS_Norm/2891323124_1.fastq F2=/data/fastqs/WGS_Norm/2891323124_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/WGS_Norm/2891323125.bam F=/data/fastqs/WGS_Norm/2891323125_1.fastq F2=/data/fastqs/WGS_Norm/2891323125_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/WGS_Tumor/2891322951.bam F=/data/fastqs/WGS_Tumor/2891322951_1.fastq F2=/data/fastqs/WGS_Tumor/2891322951_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/WGS_Tumor/2891323147.bam F=/data/fastqs/WGS_Tumor/2891323147_1.fastq F2=/data/fastqs/WGS_Tumor/2891323147_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/WGS_Tumor/2891323150.bam F=/data/fastqs/WGS_Tumor/2891323150_1.fastq F2=/data/fastqs/WGS_Tumor/2891323150_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/WGS_Tumor/2891323174.bam F=/data/fastqs/WGS_Tumor/2891323174_1.fastq F2=/data/fastqs/WGS_Tumor/2891323174_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/WGS_Tumor/2891323175.bam F=/data/fastqs/WGS_Tumor/2891323175_1.fastq F2=/data/fastqs/WGS_Tumor/2891323175_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/RNAseq_Norm/2895625992.bam F=/data/fastqs/RNAseq_Norm/2895625992_1.fastq F2=/data/fastqs/RNAseq_Norm/2895625992_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/RNAseq_Norm/2895626097.bam F=/data/fastqs/RNAseq_Norm/2895626097_1.fastq F2=/data/fastqs/RNAseq_Norm/2895626097_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/RNAseq_Tumor/2895626107.bam F=/data/fastqs/RNAseq_Tumor/2895626107_1.fastq F2=/data/fastqs/RNAseq_Tumor/2895626107_2.fastq
java -Xmx16g -jar $PICARD SamToFastq I=/data/reverted_bams/RNAseq_Tumor/2895626112.bam F=/data/fastqs/RNAseq_Tumor/2895626112_1.fastq F2=/data/fastqs/RNAseq_Tumor/2895626112_2.fastq
```

gzip all fastq files

```bash
gzip -v /data/fastqs/*/*.fastq
```

Create tarball of all fastq files to host for later use
```bash
cd /data/
tar -cvf fastqs.tar fastqs/
```

### Create downsampled data sets to allow faster analysis a teaching setting
Starting with aligned data, the following steps were used to create down-sampled data files, example shown below for chr6 exome data:
Our current version of downsampled data hosted on genomedata.org was generated with Exome_Norm_sorted_mrkdup.bam/Exome_Tumor_sorted_mrkdup.bam (without BQSR). This should be changed later such that the Exome_Norm_sorted_mrkdup_bqsr.bam/Exome_Tumor_sorted_mrkdup_bqsr.bam are used for generating the downsized data.
Note that if you were to merge data from multiple chromosomes, at the step prior to filtering sam reads, you would need to concat the readname files from all chromosomes and sort and uniq them.

```bash
#Starting off with slices of bam file with chr6 as the region specified. This contains all mapped reads that fall/overlap(?) with this region. We want to ensure that if read 1 is in this region that when reconstructing the fastq, we would also like to include read 2, thus we have the following approach.
cd <directory containing Exome alignment>

sambamba slice -o chr6_Exome_Norm_sorted_mrkdup_bqsr.bam Exome_Norm_sorted_mrkdup_bqsr.bam chr6

sambamba slice -o chr6_Exome_Tumor_sorted_mrkdup_bqsr.bam Exome_Tumor_sorted_mrkdup_bqsr.bam chr6

samtools view chr6_Exome_Norm_sorted_mrkdup_bqsr.bam | cut -d$'\t' -f1 > chr6_Exome_Norm_readnames.txt
samtools view chr6_Exome_Tumor_sorted_mrkdup_bqsr.bam | cut -d$'\t' -f1 > chr6_Exome_Tumor_readnames.txt

java -jar /usr/local/bin/picard.jar FilterSamReads I=Exome_Tumor_sorted_mrkdup_bqsr.bam O=chr6_Exome_Tumor_all_read_pairs.bam READ_LIST_FILE=chr6_Exome_Tumor_readnames.txt FILTER=includeReadList
java -jar /usr/local/bin/picard.jar FilterSamReads I=../final/Exome_Norm_sorted_mrkdup_bqsr.bam O=chr6_Exome_Norm_all_read_pairs.bam READ_LIST_FILE=chr6_Exome_Norm_readnames.txt FILTER=includeReadList

ls -1 | grep chr6_Exome_Tumor_all_read_pairs.bam | perl -ne 'chomp; print "samtools view -H $_ | grep -H --label=$_ \@RG\n"' | bash > chr6_Exome_Tumor_readgroup_info.txt
cat chr6_Exome_Tumor_readgroup_info.txt | perl -ne 'my ($bam, $id, $pl, $pu, $lb, $sm); if ($_=~/(\S+\.bam)\:/){$bam=$1} if ($_=~/(ID\:\d+)/){$id=$1} if ($_=~/(PL\:\w+)/){$pl=$1} if ($_=~/(PU\:\S+)/){$pu
=$1} if($_=~/LB\:\"(.+)\"/){$lb=$1} if ($_=~/(SM\:\S+)/){$sm=$1} print "$bam\t$id\t$pl\t$pu\t$lb\t$sm\n";' > chr6_Exome_Tumor_readgroup_info.clean.txt

ls -1 | grep chr6_Exome_Norm_all_read_pairs.bam | perl -ne 'chomp; print "samtools view -H $_ | grep -H --label=$_ \@RG\n"' | bash > chr6_Exome_Norm_readgroup_info.txt
cat chr6_Exome_Norm_readgroup_info.txt | perl -ne 'my ($bam, $id, $pl, $pu, $lb, $sm); if ($_=~/(\S+\.bam)\:/){$bam=$1} if ($_=~/(ID\:\d+)/){$id=$1} if ($_=~/(PL\:\w+)/){$pl=$1} if ($_=~/(PU\:\S+)/){$pu
=$1} if($_=~/LB\:\"(.+)\"/){$lb=$1} if ($_=~/(SM\:\S+)/){$sm=$1} print "$bam\t$id\t$pl\t$pu\t$lb\t$sm\n";' > chr6_Exome_Norm_readgroup_info.clean.txt

#reverting bams
mkdir -p reverted_bams
mkdir -p reverted_bams/Exome_Norm reverted_bams/Exome_Tumor

java -Xmx8g -jar /usr/local/bin/picard.jar RevertSam I=chr6_Exome_Tumor_all_read_pairs.bam OUTPUT_BY_READGROUP=true O=reverted_bams/Exome_Tumor/
java -Xmx8g -jar /usr/local/bin/picard.jar RevertSam I=chr6_Exome_Norm_all_read_pairs.bam OUTPUT_BY_READGROUP=true O=reverted_bams/Exome_Norm/

#Bam to Fastq:
mkdir -p fastqs
mkdir -p fastqs/Exome_Norm fastq/Exome_Tumor

java -Xmx8g -jar /usr/local/bin/picard.jar SamToFastq I=reverted_bams/Exome_Norm/2891351068.bam F=fastqs/Exome_Norm/2891351068_1.fastq F2=fastqs/Exome_Norm/2891351068_2.fastq
- java -Xmx8g -jar /data/bin/picard.jar SamToFastq I=reverted_bams/Exome_Tumor/2891351066.bam F=fastqs/Exome_Tumor/2891351066_1.fastq F2=fastqs/Exome_Tumor/2891351066_2.fastq

# You may want to move all the above data generated into a designated space for that specific chromosome e.g. mkdir chr6_subset_data

#After moving the data you would need to gzip files and tar ball them for uploading
```
The following steps are examples shown for chr6+chr17 with whole genome data:
```bash
cd <directory containing your WGS data>
sambamba slice -o chr6_WGS_Norm_merged_sorted_mrkdup.bam WGS_Norm_merged_sorted_mrkdup.bam chr6
sambamba slice -o chr6_WGS_Tumor_merged_sorted_mrkdup.bam WGS_Tumor_merged_sorted_mrkdup.bam chr6
sambamba slice -o chr17_WGS_Norm_merged_sorted_mrkdup.bam WGS_Norm_merged_sorted_mrkdup.bam chr17
sambamba slice -o chr17_WGS_Tumor_merged_sorted_mrkdup.bam WGS_Tumor_merged_sorted_mrkdup.bam chr17

# move all generated files to a subset directory such as chr6+chr17
# cd into that Directory

samtools view chr6_WGS_Norm_merged_sorted_mrkdup.bam | cut -d$'\t' -f1 > chr6_WGS_Norm_readnames.txt
samtools view chr6_WGS_Tumor_merged_sorted_mrkdup.bam | cut -d$'\t' -f1 > chr6_WGS_Tumor_readnames.txt
samtools view chr17_WGS_Norm_merged_sorted_mrkdup.bam | cut -d$'\t' -f1 > chr17_WGS_Norm_readnames.txt
samtools view chr17_WGS_Tumor_merged_sorted_mrkdup.bam | cut -d$'\t' -f1 > chr17_WGS_Tumor_readnames.txt

cat chr17_WGS_Tumor_readnames.txt chr6_WGS_Tumor_readnames.txt | sort | uniq > chr17_chr6_WGS_Tumor_readnames.txt
cat chr6_WGS_Norm_readnames.txt chr17_WGS_Norm_readnames.txt | sort | uniq > chr6_chr17_WGS_Norm_readnames.txt

java -Xmx12g -jar /usr/local/bin/picard.jar FilterSamReads I=../all/WGS_Norm_merged_sorted_mrkdup.bam O=chr6_chr17_WGS_Norm_merged_all_read_pairs.bam READ_LIST_FILE=chr6_chr17_WGS_Norm_readnames.txt FILTER=includeReadList
java -Xmx12g -jar /usr/local/bin/picard.jar FilterSamReads I=../all/WGS_Tumor_merged_sorted_mrkdup.bam O=chr6_chr17_WGS_Tumor_merged_all_read_pairs.bam READ_LIST_FILE=chr6_chr17_WGS_Tumor_readnames.txt FILTER=includeReadList

#reverting bams
mkdir -p reverted_bams
mkdir -p reverted_bams/WGS_Norm reverted_bams/WGS_Tumor

java -Xmx24g -Djava.io.tmpdir='/workspace/tmp/' -jar /usr/local/bin/picard.jar RevertSam I=chr6_chr17_WGS_Tumor_merged_all_read_pairs.bam OUTPUT_BY_READGROUP=true O=reverted_bams/WGS_Tumor/
mv 2891322951.bam WGS_Tumor_Lane1.bam
mv 2891323147.bam WGS_Tumor_Lane5.bam
mv 2891323150.bam WGS_Tumor_Lane4.bam
mv 2891323174.bam WGS_Tumor_Lane2.bam
mv 2891323175.bam WGS_Tumor_Lane3.bam

java -Xmx12g -jar /usr/local/bin/picard.jar RevertSam I=chr6_chr17_WGS_Norm_merged_all_read_pairs.bam OUTPUT_BY_READGROUP=true O=reverted_bams/WGS_Norm/
mv 2891323123.bam WGS_Norm_Lane1.bam
mv 2891323124.bam WGS_Norm_Lane2.bam
mv 2891323125.bam WGS_Norm_Lane3.bam

#Bam to Fastq:
mkdir -p fastqs
mkdir -p fastqs/WGS_Norm fastqs/WGS_Tumor

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/data/DNA_alignments/chr6+chr17/reverted_bams/WGS_Norm/WGS_Norm_Lane1.bam F=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Norm/WGS_Norm_Lane1_R1.fastq F2=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Norm/WGS_Norm_Lane1_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/data/DNA_alignments/chr6+chr17/reverted_bams/WGS_Norm/WGS_Norm_Lane2.bam F=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Norm/WGS_Norm_Lane2_R1.fastq F2=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Norm/WGS_Norm_Lane2_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/data/DNA_alignments/chr6+chr17/reverted_bams/WGS_Norm/WGS_Norm_Lane3.bam F=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Norm/WGS_Norm_Lane3_R1.fastq F2=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Norm/WGS_Norm_Lane3_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/data/DNA_alignments/chr6+chr17/reverted_bams/WGS_Tumor/WGS_Tumor_Lane1.bam F=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane1_R1.fastq F2=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane1_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/data/DNA_alignments/chr6+chr17/reverted_bams/WGS_Tumor/WGS_Tumor_Lane2.bam F=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane2_R1.fastq F2=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane2_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/data/DNA_alignments/chr6+chr17/reverted_bams/WGS_Tumor/WGS_Tumor_Lane3.bam F=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane3_R1.fastq F2=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane3_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/data/DNA_alignments/chr6+chr17/reverted_bams/WGS_Tumor/WGS_Tumor_Lane4.bam F=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane4_R1.fastq F2=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane4_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/data/DNA_alignments/chr6+chr17/reverted_bams/WGS_Tumor/WGS_Tumor_Lane5.bam F=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane5_R1.fastq F2=~/workspace/data/DNA_alignments/chr6+chr17/fastqs/WGS_Tumor/WGS_Tumor_Lane5_R2.fastq
```
### Creating the subset data from RNAseq alignments
```bash
cd <directory containing your WGS data>
sambamba slice -o chr6_RNAseq_Norm_lane1.bam HCC1395BL_RNA_H3MYFBBXX_4_CTTGTA.bam chr6
sambamba slice -o chr6_RNAseq_Norm_lane2.bam HCC1395BL_RNA_H3MYFBBXX_5_CTTGTA.bam chr6
sambamba slice -o chr17_RNAseq_Norm_lane1.bam HCC1395BL_RNA_H3MYFBBXX_4_CTTGTA.bam chr17
sambamba slice -o chr17_RNAseq_Norm_lane2.bam HCC1395BL_RNA_H3MYFBBXX_5_CTTGTA.bam chr17

sambamba slice -o chr6_RNAseq_Tumor_lane1.bam HCC1395_RNA_H3MYFBBXX_4_GCCAAT.bam chr6
sambamba slice -o chr6_RNAseq_Tumor_lane2.bam HCC1395_RNA_H3MYFBBXX_5_GCCAAT.bam chr6
sambamba slice -o chr17_RNAseq_Tumor_lane1.bam HCC1395_RNA_H3MYFBBXX_4_GCCAAT.bam chr17
sambamba slice -o chr17_RNAseq_Tumor_lane2.bam HCC1395_RNA_H3MYFBBXX_5_GCCAAT.bam chr17


# move all generated files to a subset directory such as chr6+chr17
# cd into that Directory

samtools view chr6_RNAseq_Norm_lane1.bam | cut -d$'\t' -f1 > chr6_RNA_Norm_lane1_readnames.txt
samtools view chr6_RNAseq_Norm_lane2.bam | cut -d$'\t' -f1 > chr6_RNA_Norm_lane2_readnames.txt
samtools view chr17_RNAseq_Norm_lane1.bam | cut -d$'\t' -f1 > chr17_RNA_Norm_lane1_readnames.txt
samtools view chr17_RNAseq_Norm_lane2.bam | cut -d$'\t' -f1 > chr17_RNA_Norm_lane2_readnames.txt

samtools view chr6_RNAseq_Tumor_lane1.bam | cut -d$'\t' -f1 > chr6_RNA_Tumor_lane1_readnames.txt
samtools view chr6_RNAseq_Tumor_lane2.bam | cut -d$'\t' -f1 > chr6_RNA_Tumor_lane2_readnames.txt
samtools view chr17_RNAseq_Tumor_lane1.bam | cut -d$'\t' -f1 > chr17_RNA_Tumor_lane1_readnames.txt
samtools view chr17_RNAseq_Tumor_lane2.bam | cut -d$'\t' -f1 > chr17_RNA_Tumor_lane2_readnames.txt

cat chr6_RNA_Norm_lane1_readnames.txt chr17_RNA_Norm_lane1_readnames.txt | sort | uniq > chr6_chr17_RNA_Norm_lane1_readnames.txt
cat chr6_RNA_Norm_lane2_readnames.txt chr17_RNA_Norm_lane2_readnames.txt | sort | uniq > chr6_chr17_RNA_Norm_lane2_readnames.txt
cat chr6_RNA_Tumor_lane1_readnames.txt chr17_RNA_Tumor_lane1_readnames.txt | sort | uniq > chr6_chr17_RNA_Tumor_lane1_readnames.txt
cat chr6_RNA_Tumor_lane2_readnames.txt chr17_RNA_Tumor_lane2_readnames.txt | sort | uniq > chr6_chr17_RNA_Tumor_lane2_readnames.txt

java -Xmx12g -jar /usr/local/bin/picard.jar FilterSamReads I=HCC1395BL_RNA_H3MYFBBXX_4_CTTGTA.bam O=chr6_chr17_RNA_Norm_lane1_all_read_pairs.bam READ_LIST_FILE=chr6_chr17_RNA_Norm_lane1_readnames.txt FILTER=includeReadList
java -Xmx12g -jar /usr/local/bin/picard.jar FilterSamReads I=HCC1395BL_RNA_H3MYFBBXX_5_CTTGTA.bam O=chr6_chr17_RNA_Norm_lane2_all_read_pairs.bam READ_LIST_FILE=chr6_chr17_RNA_Norm_lane2_readnames.txt FILTER=includeReadList
java -Xmx12g -jar /usr/local/bin/picard.jar FilterSamReads I=HCC1395_RNA_H3MYFBBXX_4_GCCAAT.bam O=chr6_chr17_RNA_Tumor_lane1_all_read_pairs.bam READ_LIST_FILE=chr6_chr17_RNA_Tumor_lane1_readnames.txt FILTER=includeReadList
java -Xmx12g -jar /usr/local/bin/picard.jar FilterSamReads I=HCC1395_RNA_H3MYFBBXX_5_GCCAAT.bam O=chr6_chr17_RNA_Tumor_lane2_all_read_pairs.bam READ_LIST_FILE=chr6_chr17_RNA_Tumor_lane2_readnames.txt FILTER=includeReadList

#reverting bams
mkdir -p reverted_bams
mkdir -p reverted_bams/RNA_Norm reverted_bams/RNA_Tumor

java -Xmx12g -Djava.io.tmpdir='/workspace/tmp/' -jar /usr/local/bin/picard.jar RevertSam I=chr6_chr17_RNA_Norm_lane1_all_read_pairs.bam OUTPUT_BY_READGROUP=true O=reverted_bams/RNA_Norm/
java -Xmx12g -Djava.io.tmpdir='/workspace/tmp/' -jar /usr/local/bin/picard.jar RevertSam I=chr6_chr17_RNA_Norm_lane2_all_read_pairs.bam OUTPUT_BY_READGROUP=true O=reverted_bams/RNA_Norm/

java -Xmx12g -Djava.io.tmpdir='/workspace/tmp/' -jar /usr/local/bin/picard.jar RevertSam I=chr6_chr17_RNA_Tumor_lane1_all_read_pairs.bam OUTPUT_BY_READGROUP=true O=reverted_bams/RNA_Tumor/
java -Xmx12g -Djava.io.tmpdir='/workspace/tmp/' -jar /usr/local/bin/picard.jar RevertSam I=chr6_chr17_RNA_Tumor_lane2_all_read_pairs.bam OUTPUT_BY_READGROUP=true O=reverted_bams/RNA_Tumor/

mv 2895625992.bam RNAseq_Norm_Lane1.bam
mv 2895626097.bam RNAseq_Norm_Lane2.bam
mv 2895626107.bam RNAseq_Tumor_Lane1.bam
mv 2895626112.bam RNAseq_Tumor_Lane2.bam

#Bam to Fastq:
mkdir -p fastqs
mkdir -p fastqs/RNAseq_Norm fastqs/RNAseq_Tumor

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/rnaseq/alignments/reverted_bams/RNA_Norm/RNAseq_Norm_Lane1.bam F=~/workspace/rnaseq/alignments/fastqs/RNAseq_Norm/RNAseq_Norm_Lane1_R1.fastq F2=~/workspace/rnaseq/alignments/fastqs/RNAseq_Norm/RNAseq_Norm_Lane1_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/rnaseq/alignments/reverted_bams/RNA_Norm/RNAseq_Norm_Lane2.bam F=~/workspace/rnaseq/alignments/fastqs/RNAseq_Norm/RNAseq_Norm_Lane2_R1.fastq F2=~/workspace/rnaseq/alignments/fastqs/RNAseq_Norm/RNAseq_Norm_Lane2_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/rnaseq/alignments/reverted_bams/RNA_Tumor/RNAseq_Tumor_Lane1.bam F=~/workspace/rnaseq/alignments/fastqs/RNAseq_Tumor/RNAseq_Tumor_Lane1_R1.fastq F2=~/workspace/rnaseq/alignments/fastqs/RNAseq_Tumor/RNAseq_Tumor_Lane1_R2.fastq

java -Xmx24g -jar /usr/local/bin/picard.jar SamToFastq I=~/workspace/rnaseq/alignments/reverted_bams/RNA_Tumor/RNAseq_Tumor_Lane2.bam F=~/workspace/rnaseq/alignments/fastqs/RNAseq_Tumor/RNAseq_Tumor_Lane2_R1.fastq F2=~/workspace/rnaseq/alignments/fastqs/RNAseq_Tumor/RNAseq_Tumor_Lane2_R2.fastq

```


### Obtain 1000 genomes exome bam files for joint genotyping and VQSR steps

Additional 1000 Genomes samples and their variants will be used to improve the overall accuracy of variant calling and subsequent filtering steps. Recall that the cell line being used in this tutorial ([HCC1395](https://www.atcc.org/Products/All/CRL-2324.aspx)) was derived from a 43 year old caucasian female (by a research group in Dallas Texas). Therefore, for a "matched" set of reference alignments we might limit to those of European descent.

#### Get a list of 1000 Genome samples

- First, view a summary of samples/ethnicities available from the [1000 Genomes Populations page](http://www.internationalgenome.org/data-portal/population)
- Now, we are guessing, but perhaps GBR (British in England and Scotland) would be the best match.
- Go to the [1000 Genomes Samples page](http://www.internationalgenome.org/data-portal/sample)
- Filter by population -> GBR
- Filter by analysis group -> Exome
- Filter by data collection -> 1000 genomes on GRCh38
- Download the list to get sample details (e.g., save as igsr_samples_GBR.tsv)

#### Get a list of data files for GBR samples with exome data

- Go to back to the [1000 Genomes Populations page](http://www.internationalgenome.org/data-portal/population)
- Select GBR
- Scroll down to 'Data collections for GBR'
- Choose '1000 Genomes on GRCh38' tab
- Select 'Data types' -> 'Alignment'
- Select 'Analysis groups' -> 'Exome'
- 'Download the list' (e.g., save as igsr_GBR_GRCh38_exome_alignment.tsv).

#### Download exome cram and crai files for 5 female GBR cases

Using the information obtained above, we will download the already aligned exome data for several 1KGP individuals.

```bash
mkdir -p /workspace/inputs/data/1KGP
cd /workspace/inputs/data/1KGP

#We have made the above sample and data details files available for download
wget http://genomedata.org/pmbio-workshop/references/1KGP/igsr_samples_GBR.tsv
wget http://genomedata.org/pmbio-workshop/references/1KGP/igsr_GBR_GRCh38_exome_alignment.tsv

#Limit to the first 5, female samples in the file. Use the ids from that file filter for the correct cram/crai files and create download commands
grep "female" igsr_samples_GBR.tsv | cut -f 1 | head -5 > GBR_female_5_samples.txt
grep -f GBR_female_5_samples.txt igsr_GBR_GRCh38_exome_alignment.tsv | cut -f 1 | grep ".cra" | perl -ne 'chomp; print "wget $_\n";' > wget_files.sh

#Download the files
bash wget_files.sh
```

Note: These cram files were created in a generally compatible way with the anlysis done so far in this tutorial. The were aligned with BWA-mem to GRCh38 with alternative sequences, plus decoys and HLA. This was followed by GATK BAM improvement steps as in the 1000 Genomes phase 3 pipeline (GATK IndelRealigner, BQSR, and Picard MarkDuplicates). Of course, there will be batch effects related to potentially different sample preparation, library construction, exome capture reagent and protocol, sequencing pipeline, etc. For more details see:

- http://www.internationalgenome.org/analysis
- https://media.nature.com/original/nature-assets/nature/journal/v526/n7571/extref/nature15393-s1.pdf

#### Convert 1KGP cram files to bam file - this was necessary to subset the data to chr6/17 with sambamba for which I could not get sambamba slice to work on cram (no way to specify reference genome?)

```bash
#Download complete version of genome for cram to bam conversion
cd /workspace/inputs/references/genome/
wget -c ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/GRCh38_full_analysis_set_plus_decoy_hla.fa
wget -c ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/GRCh38_full_analysis_set_plus_decoy_hla.dict

#Convert cram to bam
cd /workspace/inputs/data/1KGP
samtools view -@ 16 -b -T /workspace/inputs/references/genome/GRCh38_full_analysis_set_plus_decoy_hla.fa HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.cram > HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam
samtools view -@ 16 -b -T /workspace/inputs/references/genome/GRCh38_full_analysis_set_plus_decoy_hla.fa HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.cram > HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam
samtools view -@ 16 -b -T /workspace/inputs/references/genome/GRCh38_full_analysis_set_plus_decoy_hla.fa HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.cram > HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam
samtools view -@ 16 -b -T /workspace/inputs/references/genome/GRCh38_full_analysis_set_plus_decoy_hla.fa HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.cram > HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam
samtools view -@ 16 -b -T /workspace/inputs/references/genome/GRCh38_full_analysis_set_plus_decoy_hla.fa HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.cram > HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam

#Index bam files - necessary for sambamba slice
samtools index -@ 16 HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam
samtools index -@ 16 HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam
samtools index -@ 16 HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam
samtools index -@ 16 HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam
samtools index -@ 16 HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam

#Slice files down to just chr6 and chr17 - apparently this can only be done one chr at a time?
sambamba slice -o HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr6
sambamba slice -o HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr6
sambamba slice -o HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr6
sambamba slice -o HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr6
sambamba slice -o HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr6

sambamba slice -o HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr17
sambamba slice -o HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr17
sambamba slice -o HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr17
sambamba slice -o HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr17
sambamba slice -o HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.bam chr17

#Merge chr6 and chr7 subsets
samtools merge -@ 16 HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam
samtools merge -@ 16 HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam
samtools merge -@ 16 HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam
samtools merge -@ 16 HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam
samtools merge -@ 16 HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam

#Index subsetted bam files - necessary for gatk
samtools index -@ 8 HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam
samtools index -@ 8 HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam
samtools index -@ 8 HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam
samtools index -@ 8 HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam
samtools index -@ 8 HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam

samtools index -@ 8 HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam
samtools index -@ 8 HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam
samtools index -@ 8 HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam
samtools index -@ 8 HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam
samtools index -@ 8 HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam

samtools index -@ 8 HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam
samtools index -@ 8 HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam
samtools index -@ 8 HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam
samtools index -@ 8 HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam
samtools index -@ 8 HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam
```

Host bam/bai files on genomedata.org for student use. Note - converting back to cram files is problematic. If done, then subsequence GATK steps will fail unless provided with the full reference genome. Simpler to just leave as bam files.

#### Perform germline variant calling on 1KGP exomes with GATK in GVCF mode

Run the GATK HaplotypeCaller GVCF workflow commands as above.

```bash
mkdir -p /workspace/germline/1KGP/chr6_chr17
cd /workspace/germline/1KGP/chr6_chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam -O /workspace/germline/1KGP/chr6_chr17/HG00099_HC_calls.g.vcf -L chr6 -L chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam -O /workspace/germline/1KGP/chr6_chr17/HG00102_HC_calls.g.vcf -L chr6 -L chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam -O /workspace/germline/1KGP/chr6_chr17/HG00104_HC_calls.g.vcf -L chr6 -L chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam -O /workspace/germline/1KGP/chr6_chr17/HG00150_HC_calls.g.vcf -L chr6 -L chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6_chr17.bam -O /workspace/germline/1KGP/chr6_chr17/HG00158_HC_calls.g.vcf -L chr6 -L chr17

mkdir -p /workspace/germline/1KGP/chr6
cd /workspace/germline/1KGP/chr6
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam -O /workspace/germline/1KGP/chr6/HG00099_HC_calls.g.vcf -L chr6
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam -O /workspace/germline/1KGP/chr6/HG00102_HC_calls.g.vcf -L chr6
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam -O /workspace/germline/1KGP/chr6/HG00104_HC_calls.g.vcf -L chr6
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam -O /workspace/germline/1KGP/chr6/HG00150_HC_calls.g.vcf -L chr6
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr6.bam -O /workspace/germline/1KGP/chr6/HG00158_HC_calls.g.vcf -L chr6

mkdir -p /workspace/germline/1KGP/chr17
cd /workspace/germline/1KGP/chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00099.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam -O /workspace/germline/1KGP/chr17/HG00099_HC_calls.g.vcf -L chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00102.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam -O /workspace/germline/1KGP/chr17/HG00102_HC_calls.g.vcf -L chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00104.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam -O /workspace/germline/1KGP/chr17/HG00104_HC_calls.g.vcf -L chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00150.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam -O /workspace/germline/1KGP/chr17/HG00150_HC_calls.g.vcf -L chr17
gatk --java-options '-Xmx60g' HaplotypeCaller -ERC GVCF -R /workspace/inputs/references/genome/ref_genome.fa -I /workspace/inputs/data/1KGP/HG00158.alt_bwamem_GRCh38DH.20150826.GBR.exome.chr17.bam -O /workspace/germline/1KGP/chr17/HG00158_HC_calls.g.vcf -L chr17
```

Host g.vcf and idx files on genomedata.org as well
