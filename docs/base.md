# Obtaining the base data file
The first step in Polygenic Risk Score (PRS) analyses is to generate or obtain the base data (GWAS summary statistics). Ideally these will correspond to the most powerful GWAS results available on the phenotype under study. In this example, we will use a modified version of the Height GWAS summary statistics generated by the [GIANT consortium](https://portals.broadinstitute.org/collaboration/giant/index.php/GIANT_consortium_data_files#GWAS_Anthropometric_2014_Height). You can download the summary statistic file [here](https://github.com/choishingwan/PRS-Tutorial/raw/master/resources/GIANT.height.gz) or you can use the following bash command:
``` bash

curl https://github.com/choishingwan/PRS-Tutorial/raw/master/resources/GIANT.height.gz -L -O

```

which will create a file called **GIANT.height.gz** in your working directory. 

!!! warning
    If you download the summary statistics without using the bash command, and are using a MAC machine, the gz file will be decompressed automatically, resulting in a **GIANT.height** file instead. 
    
    To maintain consistency, we suggest compressing the **GIANT.height** file with 
    ```
    gzip GIANT.height
    ```
    before starting the tutorial
    


# Reading the base data file
**GIANT.height.gz** is compressed. To read its content, you can type:

```bash
gunzip -c GIANT.height.gz | head
```

which will display the first 10 lines of the file

!!! note
    Working with compressed files reduces storage space requirements

The **GIANT.height.gz** file contains the following columns:

|SNP|CHR|BP|A1|A2|MAF|SE|P|N|INFO|OR|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|rs2073813	|1	|753541	|A	|G	|0.125	|0.0083	|0.68	|69852	|0.866425782879888	|0.996605773454898|
rs12562034	|1	|768448	|A	|G	|0.092	|0.0088	|0.55	|88015	|0.917520990188678	|0.994714020220009|
rs2980319	|1	|777122	|A	|T	|0.125	|0.006	|0.65	|148975	|0.847126999058955	|0.997303641721713|

The column headers correspond to the following: 

- **SNP**: SNP ID, usually in the form of rs-ID
- **CHR**: The chromosome in which the SNP resides
- **BP**: Chromosomal co-ordinate of the SNP
- **A1**: The effect allele of the SNP
- **A2**: The non-effect allele of the SNP
- **MAF**: The minor allele frequency (MAF) of the SNP
- **SE**: The standard error (SE) of the effect size esimate
- **P**: The P-value of association between the SNP genotypes and the base phenotype
- **N**: Number of samples used to obtain the effect size estimate
- **INFO**: The imputation information score
- **OR**: The effect size estimate of the SNP, if the outcome is binary/case-control. If the outcome is continuous or treated as continuous then this will be the BETA

# QC checklist: Base data

Below we perform QC on these base data according to the 'QC checklist' in the guide paper, which we recommend that users follow each time they perform a PRS analysis:

# \# Heritability check
We recommend that PRS analyses are performed on base data with a chip-heritability estimate $h_{snp}^{2} > 0.05$. The chip-heritability of a GWAS can be estimated using e.g. LD Score Regression (LDSC). Our GIANT height GWAS data are known to have a chip-heritability much greater than 0.05 and so we can move on to the next QC step. 

# \# Effect allele
The GIANT consortium report which is the effect allele and which is the non-effect allele in their results, critical for PRS association results to be in the correction direction.

!!! Important
    Some GWAS results files do not make clear which allele is the effect allele and which the non-effect allele.
    If the incorrect assumption is made in computing the PRS, then the effect of the PRS in the target data will be in the wrong direction.

    To avoid misleading conclusions the effect allele from the base (GWAS) data must be known.


# \# File transfer

A common problem is that the downloaded base data file becomes 
corrupted during download, which can cause PRS software to crash 
or to produce errors in results. However, a `md5sum` hash is 
generally included in files so that file integrity can be checked. 
The following command performs this `md5sum` check: 

```bash tab="Linux"
md5sum GIANT.height.gz
```

```bash tab="OS X"
md5 GIANT.height.gz
```


if the file is intact, then `md5sum` generates a string of characters, which in this case should be: `80e48168416a2fdbe88d68cdfebd4ca2`. 
If a different string is generated, then the file is corrupted.

# \# Genome build
These base data are on the same genome build as the target data that we will be using. You must check that your base and target data are on the same genome build, and if they are not then use a tool such as LiftOver to make the builds consistent across the data sets.

# \# Standard GWAS QC
As described in the paper, both the base and target data should be subjected to the standard stringent QC steps performed in GWAS. 
If the base data have been obtained as summary statistics from a public source, then the typical QC steps that you will be able to perform on them are to filter the SNPs according to INFO score and MAF. 
SNPs with low minor allele frequency (MAF) or imputation information score (INFO) are more likely to generate false positive results due to their lower statistical power (and higher probability of genotyping errors in the case of low MAF). 
Therefore, SNPs with low MAF and INFO are typically removed before performing downstream analyses.
We recommend removing SNPs with MAF < 1% and INFO < 0.8 (with very large base sample sizes these thresholds could be reduced if sensitivity checks indicate reliable results).
These SNP filters can be acheived using the following code:

```bash
gunzip -c GIANT.height.gz |\
awk 'NR==1 || ($6 > 0.01) && ($10 > 0.8) {print}' |\
gzip  > Height.gz
```

The bash code above does the following:
1. Decompresses and reads the **GIANT.height.gz** file
2. Prints the header line (`NR==1`)
3. Prints any line with MAF above 0.01 (`$6` because the sixth column of the file contains the MAF information)
4. Prints any line with INFO above 0.8 (`$10` because the tenth column of the file contains the INFO information)
5. Compresses and writes the results to **Height.gz**

# \# Ambiguous SNPs
If the base and target data were generated using different genotyping chips and the chromosome strand (+/-) for either is unknown, then it is not possible to match ambiguous SNPs (i.e. those with complementary alleles, either C/G or A/T) across the data sets, because it will be unknown whether the base and target data are referring to the same allele or not. Ambiguous SNPs can be removed in the base data and then there will be no such SNPs in the subsequent analyses, since analyses are performed only on SNPs that overlap between the base and target data.

Nonambiguous SNPs can be retained using the following:
```bash
gunzip -c Height.gz |\
awk '!( ($4=="A" && $5=="T") || \
        ($4=="T" && $5=="A") || \
        ($4=="G" && $5=="C") || \
        ($4=="C" && $5=="G")) {print}' |\
    gzip > Height.noambig.gz
```

??? note "How many non-ambiguous SNPs were there?"
    There are `609,041` ambiguous SNPs


# \# Mismatching genotypes
If there is a non-ambiguous mismatch in allele coding between the base and target data sets, such as A/C in the base and G/T in the target data, then this can be resolved by ‘flipping’ the alleles in either data set to their complementary alleles. However, since we need the target data to know which SNPs have mismatching genotypes across the data sets, then we will perform this 'allele flipping' in the target data.

# \# Duplicate SNPs
If an error has occurred in the generation of the base data then there may be duplicated SNPs in the base data file.
Most PRS software do not allow duplicated SNPs in the base data input and thus they should be removed, using a command such as the one below: 

```bash
gunzip -c Height.noambig.gz |\
awk '{ print $1}' |\
sort |\
uniq -d > duplicated.snp
```

The above command does the following:

1. Decompresses and reads the **Height.noambig.gz** file
2. Prints out the first column of the file (which contains the SNP ID; change `$1` to another number if the SNP ID is located in another column, e.g. `$3` if the SNP ID is located on the third column)
3. Sort the SNP IDs. This will put duplicated SNP IDs next to each other
4. Print out any duplicated SNP IDs using the uniq command and print them to the *duplicated.snp* file


??? note "How many duplicated SNPs are there?"
    There are a total of `10` duplicated SNPs

Duplicated SNPs can then be removed using the `grep` command:
```bash
gunzip -c Height.noambig.gz  |\
grep -vf duplicated.snp |\
gzip - > Height.QC.gz
```

The above script does the following:

1. Decompresses and reads the **Height.noambig.gz** file 
2. Establishes whether any row contains entries observed in `duplicated.snp` and removes them if so
3. Compresses and writes the results to **Height.QC.gz**

# \# Sex chromosomes 
Previously performed QC on these data removed individuals with mismatching (inferred) biological and reported sex, while the sex chromosomes are not included. Please refer to the corresponding section in the paper for details relating QC performed in relation to the sex chromosomes. 

# \# Sample overlap
In this tutorial the target data are simulated and thus there must be no sample overlap. However, users should ensure that the possibility of sample overlap between the base and target data is minimised. 

# \# Relatedness
In this tutorial the target data are simulated and thus there must be no closely related individuals across the base and target data. However, users should ensure that the possibility of closely related individuals between the base and target data is minimised. 

The **Height.QC.gz** base data are now ready for using in downstream analyses.

    
