# Genome-scale detection of positive selection in 9 primates

This page contains scripts, analyses and procedures developed in:

> **Genome-scale detection of positive selection in 9 primates predicts human-virus evolutionary conflicts**
> Robin van der Lee, Laurens Wiel, Teunis J.P. van Dam, Martijn A. Huynen  
> bioRxiv 131680; https://doi.org/10.1101/131680

> Centre for Molecular and Biomolecular Informatics - Radboud Institute for Molecular Life Sciences<br/>
> Radboud university medical center<br/>
> Nijmegen, The Netherlands

Please consider citing the paper if you found this resource useful.


## Issues & Contact

Note that these scripts were not designed to function as a fully-automated pipeline, but rather as a series of steps with extensive manual quality control between them. It will therefore not be straightforward to run all steps smoothly in one go. Feel free to contact me if you run into any issue with individual steps.

> **robinvanderlee AT gmail DOT com**<br/><br/>
> [Google Scholar](https://scholar.google.co.uk/citations?user=ISYCcUUAAAAJ)<br/>
> [ORCID](http://orcid.org/0000-0001-7391-9438)<br/>
> [Twitter](https://twitter.com/robinvdlee)<br/>
> [LinkedIn](http://nl.linkedin.com/in/robinvdlee)


## Requirements

Scripts depend on various programs and modules to run. Refer to the paper for which versions were used.

###### Perl
- Perl 5: https://www.perl.org/
- Install various modules using `cpan` or `cpanm`
	- `cpanm DBI`
	- `cpanm DBD::mysql`  (requires a working mysql installation, https://www.mysql.com/)
- Download the various helper scripts that are also part of this GitHub repository
	- `functions.pl`
	- Scripts in `Ensembl_API_subroutines`

###### BioPerl & Ensembl API
- BioPerl `cpanm Bio::Perl` (http://bioperl.org/)
- Ensembl API (http://www.ensembl.org/info/docs/api/index.html)

###### R
- R: https://www.r-project.org/
- Install the following packages:
	- ``



R
	+ modules
	?? gebruik ik dit script? ./Biological_correlations_PSR/basic_statistics_of_data/aa_freqs/analyze_aa_freqs.r:require(lattice)
library("Biostrings")
library("biomaRt")
library("ggplot2")
library("phangorn")
library("session")
library(GenomicRanges)
library(MASS)
library(data.table)
library(dplyr)
library(ggplot2)
library(gplots)
library(gridExtra)
library(jsonlite)
library(parallel)
library(plotrix)
library(rtracklayer)
library(scales)
library(vioplot)


###### Command line tools
GNU Parallel (https://www.gnu.org/software/parallel/)

###### Aligners and alignment analysis tools
- PRANK multiple sequence aligner (http://wasabiapp.org/software/prank/)
- GUIDANCE (http://guidance.tau.ac.il/)<br/>
*Note that a bug fix is required for GUIDANCE (version 1.5 - 2014, August 7) to work with PRANK, see [GUIDANCE_source_code_fix_for_running_PRANK](Supplementary_data_and_material/GUIDANCE_source_code_fix_for_running_PRANK/)*
- t_coffee, which includes `TCS` (http://www.tcoffee.org/Projects/tcoffee/)

###### PAML
PAML software package, which includes `codeml` (http://abacus.gene.ucl.ac.uk/software/paml.html)


Jalview
pal2nal?
muscle
mafft





## Steps

Please see the `Materials and Methods` section of the paper for theory and detailed explanations. Analyses presented in the paper are based on Ensembl release 78, December 2014 (http://dec2014.archive.ensembl.org/).

### 1. One-to-one orthologs
Obtain one-to-one ortholog clusters for nine primates with high-coverage whole-genome sequences. These scripts can be edited to obtain orthology clusters for (i) a different set of species than the ones we use here, and (ii) different homology relationships than the one-to-one filter we use.<br/>
Two methods, same result:

###### 1a. Ensembl API method
1. Fetch orthology information using the Ensembl API: `get_one2one_orthologs__Ensembl_API.pl`
2. Clean the results using: `get_one2one_orthologs__Ensembl_API__process_orthologs.r`

###### 1b. Ensembl BioMart method 
1. From BioMart, first get orthology information for all species of interest
![alt text](Images/Step1b__1.png)
![alt text](Images/Step1b__2.png)
2. Combine the acquired ortholog information using `get_one2one_orthologs__combine_biomart_orthology_lists.r`


### 2. Sequences
For all one-to-one orthologs, get the coding DNA (cds) and the corresponding protein sequences from the Ensembl Compara gene trees (as these trees are the basis for the orthology calls).

###### 2a. Get sequences
1. Run `start_parallel_sequence_fetching.pl`, which reads the ortholog cluster information, divides the genes in batches of 1000 (`$batchsize` can be changed in the script) and prints the instructions for the next step:
2. `get_sequences_for_one2one_orthologs_from_gene_tree_pipeline.pl`. This step fetches the sequences. E.g.
```
STARTING PARALLEL INSTANCE 11
from 10001 to 11000
1000 genes
screen -S i11
perl get_sequences_for_one2one_orthologs_from_gene_tree_pipeline.pl -p -f sequences/parallel_instance_11.txt
```
Note that these steps also fetch the alignments underlying the Compara gene trees, filtered for the species of interest. These are not required for later steps.

###### 2b. Check sequences
`check_compatibility_protein_cds_sequences.pl`. Tends to only complain at annotated selenocysteine residues.


### 3. Alignments
Produce codon-based nucleotide sequence alignments for all the one-to-one ortholog clusters. Then assess the confidence in the alignments using two indepedent approaches. Low confidence scores of either method led us to remove entire alignments from our analysis or mask unreliable individual columns and codons.

###### 3a. PRANK codon-based multiple alignment
This step collects all fasta files containing cDNA sequences for the species of interest, and for each of them runs the `PRANK` in codon mode (`-codon`) to align them. Jobs are executed and monitored in parallel using `GNU Parallel` (set number of cores with `--max-procs`).
```
find sequences/ -type f -name "*__cds.fa" | parallel --max-procs 4 --joblog parallel_prank-codon.log --eta 'prank +F -codon -d={} -o={.}.prank-codon.aln.fa -quiet > /dev/null'
```
This effectively execute the following commands:
```
find sequences/ -type f -name "*__cds.fa" | parallel 'echo prank +F -codon -d={} -o={.}.prank-codon.aln.fa -quiet'
```
```
...
prank +F -codon -d=sequences//cds/ENSG00000274211__cds.fa -o=sequences//cds/ENSG00000274211__cds.prank-codon.aln.fa -quiet
prank +F -codon -d=sequences//cds/ENSG00000274523__cds.fa -o=sequences//cds/ENSG00000274523__cds.prank-codon.aln.fa -quiet
prank +F -codon -d=sequences//cds/ENSG00000019549__cds.fa -o=sequences//cds/ENSG00000019549__cds.prank-codon.aln.fa -quiet
...
```

###### 3b. GUIDANCE - assessment and masking
1. Run GUIDANCE to assess the sensitivity of the alignment to perturbations of the guide tree.<br/>
*Note that this requires a bug fix in GUIDANCE (version 1.5 - 2014, August 7), see [GUIDANCE_source_code_fix_for_running_PRANK](Supplementary_data_and_material/GUIDANCE_source_code_fix_for_running_PRANK/)*
```
find sequences/ -type f -name "*__cds.fa" | parallel --max-procs 4 --nice 10 --joblog parallel_guidance-prank-codon.log --eta 'mkdir -p sequences/guidance-prank-codon/{/.}; guidance.pl --program GUIDANCE --seqFile {} --seqType nuc --msaProgram PRANK --MSA_Param "\+F \-codon" --outDir sequences/guidance-prank-codon/{/.} &> sequences/guidance-prank-codon/{/.}/parallel_guidance-prank-codon.output'
```

2. `perl mask_msa_based_on_guidance_results.pl`. Analyze and parse the GUIDANCE results, and mask the alignments based on the scores.

###### 3c. TCS - assessment and masking
Run T-Coffee TCS to assess alignment stability by independently re-aligning all possible pairs of sequences. Note that we ran TCS on translated PRANK codon alignments.<br/>

1. Translate the PRANK alignments to protein. Note that we use the PRANK alignments generated through GUIDANCE (Step 3b) to ensure we are masking the same alignments with both GUIDANCE and TCS!
```
find sequences/prank-codon-masked/ -type f -name "*__cds.prank-codon.aln.fa" | parallel --max-procs 4 --nice 10 --joblog parallel_translate-prank-codon-alignments.log --eta 't_coffee -other_pg seq_reformat -in {} -action +translate -output fasta_aln > sequences/tcs-prank-codon/{/.}.translated.fa'
```

2. Run TCS on the translated PRANK alignments:
```
find . -type f -name "*prank-codon.aln.translated.fa" | parallel --max-procs 4 --nice 10 --joblog ../../parallel_tcs-t-coffee.log --eta 't_coffee -infile {} -evaluate -method proba_pair -output score_ascii, score_html -quiet  > /dev/null'
```

3. `perl mask_msa_based_on_tcs_results.pl`. Analyze and parse the TCS results, and mask the alignments based on the scores. Note that we mask the original PRANK codon-based alignments based on the TCS results on the translated alignment!




**********************
TO TEST 3b2 and 3c1-3
**********************




###### 3d. Translate alignments
Steps are all CDS/codon based, but this is handy for checking alignments, analyzing results etc


###### 3e. View alignments




Some useful commands to monitor progress:
```
XX
```
find sequences/cds/ | grep prank | wc -l
tail -f parallel_guidance-prank-codon.log
find sequences/guidance-prank-codon/ -type f | grep PRANK.std$ | xargs cat | grep Writing -A 1






## Supplementary data and material

Additional data and material can be found at:
- This GitHub page: [Supplementary data and material](Supplementary_data_and_material)<br/>

and
- http://www.cmbi.umcn.nl/~rvdlee/positive_selection/

