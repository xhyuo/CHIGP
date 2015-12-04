#CHiGP

CHiGP stands for **C**apture **Hi**-C **G**ene **P**rioritisation. It's purpose is to take a bunch of p-vals from a genome-wide association study and integrate these with capture Hi-C data to prioritise genes. The computational engine is written in R and whilst it should be possible to process on a laptop, things will go considerably faster if you have access to a compute cluster.

#Prerequisites

  * Linux or Mac OS X (preferably the former)
  * R (developed with 3.2.2)
    * data.table
    * snpStats (bioconductor)
    * GenomicRanges (bioconductor)
    * reshape2
  * (optional) make
  * (optional) PERL
    * Macd queue software and dependencies.
  * [htslib](https://github.com/samtools/htslib) for tabix


#Data files

  * capture Hi-C data (we support a few tab delimited formats)
    * PeakMatrix format
    * WashU format
  * 1000 Genomes VCF files.
  * coding SNP bed file
  
#Initial setup

This assumes that you have R installed and have the ability to install packages. Firstly head to [bioconductor](https://www.bioconductor.org/install/) and read how to install snpStats and GenomicRanges dependencies. After you have these install data.table

```
install.packages("data.tables")
```
Next grab the test file bundle. This contains a cut down set of real data taken from Misfud et al. and GWAS from Okada et al.
```
cd CHIGP
curl -s http://www.immunobase.org/Downloads/CHIGP/test_data.tgz
tar -xvzf test_data.tgz
```

The above command should have created a DATA directory 

#Test run
Once you have all of the above dependencies and data installed you should be able to take CHIGP for a spin. Running CHICGP consists of three steps
  1. Convert p-vals to posterior probabilities using a reference set of genotypes.
  2. Generate support files for gene score algorithm.
  3. Integrate CHiC interaction and other functional data to prioritise genes.
For a given GWAS it should only be neccessary to run step 1 only once (unless of course you want to fiddle with parameters). You might want to run step 2 multiple times depending on what capture HiC datasets are available to you.

## Converting p-vals to posterior probabilities.

For a detailed discussion on how we do this please see Wakefield(2009), Giombartelli(2014) and Pickrell(2014). In fact the code used here is adapted from Giombartelli. Briefly, we split the genome into blocks based on a recombination frequency of 0.1cM, for each region we then estimate the minor allele frequency (MAF) of each SNP within that region available in the reference genotyping dataset. **Note that if a SNP in your GWAS is not in your reference genotyping set then it will be ommited from future steps**. We use this to estimate an approximate Bayes factor for that SNP to be causal given the data, we can then for a given region estimate the posterior probability (PPi) that a SNP is causal. 


By way of illustration using our test dataset we can compute PPi for all variants on chr22
```
cd CHIGP/sh
./test_ppi.sh
# results are returned in CHIGP/data/out/
```

If you have access to high performance computing the above step is easily parallelised, see section below on how to do this.

## Generating support files.

We next need to do some housekeeping to generate files that the algorithm can use to compute the gene scores and thus prioritise the genes. Again we illustrate this process using data from :-
  1. Interactions Mifsud et al.(2015)
  2. Recombination freq data from HapMap project  **need URL**
  3. Functional annotatation taken from Ensembl e75 using VEP **Need citation**.

```
cd CHICGP/sh ## if not already there from previous step
./test_gen_resource_files.sh
```

## Computing gene scores

Next we compute final gene scores using chr22 as an example using data taken from Okada et al. 2014

```
cd CHICGP/sh ## if not already there from previous step
./test_compute_gene_scores.sh
```                                                                                  

## Interpreting the results

Here is a results table (split over two tables for readability) of the top results by all_gene_score

|disease         |ensg            |name    |biotype        |strand | baitChr|
|:---------------|:---------------|:-------|:--------------|:------|-------:|
|0.1cM_chr22.imp |ENSG00000100321 |SYNGR1  |protein_coding |+      |      22|
|0.1cM_chr22.imp |ENSG00000161180 |CCDC116 |protein_coding |+      |      22|
|0.1cM_chr22.imp |ENSG00000161179 |YDJC    |protein_coding |-      |      22|
|0.1cM_chr22.imp |ENSG00000100023 |PPIL2   |protein_coding |+      |      22|
|0.1cM_chr22.imp |ENSG00000128228 |SDF2L1  |protein_coding |+      |      22|

|ensg            |name    | all_gene_score| CD34_gene_score| coding_gene_score| GM12878_gene_score| promoter_gene_score|
|:---------------|:-------|--------------:|---------------:|-----------------:|------------------:|-------------------:|
|ENSG00000100321 |SYNGR1  |      0.6919344|       0.6919344|          0.00e+00|          0.6917125|           0.6917124|
|ENSG00000161180 |CCDC116 |      0.5201607|       0.5062241|          1.07e-05|          0.5128092|           0.3676899|
|ENSG00000161179 |YDJC    |      0.5064901|       0.4987388|                NA|          0.5064809|           0.3825915|
|ENSG00000100023 |PPIL2   |      0.3698667|       0.0007144|                NA|          0.3698667|           0.0006832|
|ENSG00000128228 |SDF2L1  |      0.3680642|       0.3679398|                NA|          0.3680642|           0.0003168|

This is just an example analysis for true analysis we would probably want to set a cut off of 0.5 on all_gene_score (things below this are likely to be highly speculative) for manual interpretation.

For this example we can infer that the SYNGR1 prioritisation is driven by a signal within the promoter region of this gene. CCDC116 and YDJC genes have some evidence for non tissue specific interaction (we see similar scores across cell types). However the promoter region seem to be a major driver of these prioritisations. *PPIL2* appears to be driven by a tissue specific interaction in GM12878 cell line. Finally the *SDF2L1* appears to be driven by a non tissue specific interaction.

## TODO

  * Add in part for parallelising over HPC cluster.
  * Add an option to ppi computation to impute using PMI method.
  * Describe set based and hierachical Bayes factor comparison method.
  * Add in software to compute suitable 'nulls' that are required for paper.