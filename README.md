# MetaWorks

MetaWorks consists of a Conda environment and Snakemake pipeline that is meant to be run at the command line to bioinformatically processes Illumina paired-end metabarcodes from raw reads through to taxonomic assignments. MetaWorks currently supports a number of popular marker gene amplicons and metabarcodes: COI (eukaryotes), rbcL (eukaryotes, diatoms), ITS (fungi), 16S (prokaryotes), 18S (eukaryotes, diatoms), 12S (fish), and 28S (fungi).  Taxonomic assignments are made using the RDP classifier that uses a naive Bayesian method to produce taxonomic assignments with a measure of statistical support at each rank. 

This data flow will be updated on a regular basis so check for the latest version at https://github.com/terrimporter/MetaWorks/releases .

## How to cite

If you use this dataflow or any of the provided scripts, consider citing the MetaWorks preprint:  
Porter, T.M., Hajibabaei, M. 2020.  METAWORKS: A flexible, scalable bioinformatic pipeline for multi-marker biodiversity assessments.  BioRxiv, doi: https://doi.org/10.1101/2020.07.14.202960.

If you use this dataflow for making COI taxonomic assignments, consider citing the COI classifier publication:  
Porter, T. M., & Hajibabaei, M. (2018). Automated high throughput animal CO1 metabarcode classification. Scientific Reports, 8, 4226.  

If you use the RDP classifier, please cite the publication:  
Wang, Q., Garrity, G. M., Tiedje, J. M., & Cole, J. R. (2007). Naive Bayesian Classifier for Rapid Assignment of rRNA Sequences into the New Bacterial Taxonomy. Applied and Environmental Microbiology, 73(16), 5261–5267. doi:10.1128/AEM.00062-07  

## Outline

[Overview](#overview)  

[Pipeline details](#pipeline-details)  

[Prepare your environment to run the pipeline](#prepare-your-environment-to-run-the-pipeline)   

[Implementation notes](#implementation-notes)  

[Tutorial](#tutorial)

[References](#references)  

## Overview

MetaWorks comes with a conda environment file MetaWorks_v1 that should be activated before running the pipeline.  Conda is an environment and package manager (Anaconda, 2016).  The environment file contains most of the programs and dependencies needed to run MetaWorks.  An additional program, the RDP classifier v2.12 should also be installed to make the taxonomic assignments.  If pseudogene filtering will be used, then the NCBI ORFfinder program will also need to be installed.  Additional RDP-trained reference sets may need to be downloaded if the reference set needed is not already built in to the RDP classifier (see Table 1 below).

Snakemake is a python-based workflow manager (Koster and Rahmann, 2012) and it requires three sets of files to run (Fig 1).


**Fig 1.  Using a conda environment helps to quickly gather programs and dependencies used by MetaWorks.**. 

<img src="/images/conda_env.png" width="500">


The configuration file is edited by the user to specify directory names, indicate the sample and read fields from the sequence filenames, and specify other required pipeline parameters such as primer sequences, marker name, and whether or not pseudogene filtering should be run.

The snakefile describes the pipeline itself and normally does not need to be edited in any way (Fig 2).  


**Fig 2. The snakefile describes the programs used, the settings, and the order in which to run commands.**  
The pipeline takes care of any reformatting needed when moving from one step to another.  In addition to the final results file, a summary of statistics and log files are also available for major steps in the pipeline.

<img src="/images/dataflow.png" width="500">


The pipeline begins with raw paired-end Illumina MiSeq fastq.gz files.  Reads are paired.  Primers are trimmed.  All the samples are pooled for a global analysis.  Reads are dereplicated, denoised, and chimeric sequences are removed producing a reference set of denoised exact sequence variants (ESVs). At this step, the pipeline diverges into several paths:  an ITS specific dataflow, a regular dataflow, and a pseudogene filtering dataflow.  For ITS sequences, flanking rRNA gene regions are removed then they are taxonomically assigned.  For the regular pipeline, the denoised ESVs are taxonomically assigned using the RDP classifier.  If a protein coding marker is being processed but there is no HMM profile available (yet), such as with rbcL, then the denoised ESVs are translated and the longest open reading frames (ORFs) are retained.  Obvious pseudogenes, or sequences with errors, are identified as outliers with unusually short or long sequence lengths.  If a HMM profile is available, such as with COI, then denoised ESVs are translated and the longest ORFs are subjected to hidden Markov model (HMM) profile analysis.  Obvious pseudogenes, or sequences with errors, are identified as outliers with unusually low HMM scores.  The result is a report containing a list of ESVs for each sample, with read counts, and taxonomic assignments with a measure of bootstrap support (Fig 3).


**Fig 3. The RDP classifier produces a measure of confidence for taxonomic assignments at each rank.**  
Results can be filtered by bootstrap support values to reduce false-positive assignments.  The appropriate cutoffs to use for filtering will depend on the marker/classifier used, query length, and taxonomic rank.  See links in Table 1 for classifier-specific cutoffs to ensure 95-99% accuracy.

<img src="/images/taxonomic_assignments.png" width="500">


## Pipeline details

Raw paired-end reads are merged using SEQPREP v1.3.2 from bioconda (St. John, 2016).  This step looks for a minimum Phred quality score of 20 in the overlap region, requires at least a 25bp overlap.

Primers are trimmed in two steps using CUTADAPT v2.6 from bioconda (Martin, 2011).  This step looks for a minimum Phred quality score of at least 20 at the ends, the forward primer is trimmed first based on its sequence, no more than 3 N's allowed, trimmed reads need to be at least 150 bp, untrimmed reads are discarded.  The output from the first step, is used as input for the second step.  This step looks for a minimum Phred quality score of at least 20 at the ends, the reverse primer is trimmed based on its sequence, no more than 3 N's allowed, trimmed reads need to be at least 150 bp, untrimmed reads are discarded.

Files are reformatted and samples are combined for a global analysis.

Reads are dereplicated (only unique sequences are retained) using VSEARCH v2.14.1 from bioconda (Rognes et al., 2016).

Zero-radius OTUS (Zotus) are generated using VSEARCH with the unoise3 algorithm (Edgar, 2016).  They are similar to OTUs delimited by 100% sequence similarity, except that they are also error-corrected and rare clusters are removed.  Here, we define rare sequences to be sequence clusters containing only one or two reads (singletons and doubletons) and these are also removed as 'noise'.  We refer to these sequence clusters as exact sequence variants (ESVs) for clarity.  For comparison, the error-corrected sequence clusters generated by DADA2 are referred to as either amplicon sequence variants (ASVs) or ESVs in the literature (Callahan et al., 2016).  Putative chimeric sequences are then removed using the uchime3_denovo algorithm in VSEARCH.

An ESV x sample table that tracks read number for each ESV is generated with VSEARCH using --search_exact .  Note that this in this pipeline this is just an intermediate file and that the final retained set of ESVs and mapped read numbers should be retrieved from the final output file (results.csv).

For ITS, the ITSx extractor is used to remove flanking rRNA gene sequences so that subsequent analysis focuses on just the ITS1 or ITS2 spacer regions (Bengtsson-Palme et al., 2013).

For the standard pipeline (ideal for rRNA genes) performs taxonomic assignments using the Ribosomal Database classifier v2.12 (RDP classifier).  Instructions on how to install the RDP classifier is available below under [Prepare your environment to run the pipeline](#prepare-your-environment-to-run-the-pipeline).  We have provided a list of RDP-trained classifiers that can be used with MetaWorks (Table 1). 

**Table 1.  Where to download trained RDP classifiers for a variety of popular marker gene/metabarcoding targets.**

| Marker | Target taxa | Classifier availability |
| ------ | ----------- | ----------------------- |
| COI | Eukaryotes | https://github.com/terrimporter/CO1Classifier |
| rbcL | Diatoms | https://github.com/terrimporter/rbcLdiatomClassifier |
| rbcL | Eukarytoes | https://github.com/terrimporter/rbcLClassifier |
| 12S | Fish | https://github.com/terrimporter/12SfishClassifier |
| SSU (18S) | Diatoms | https://github.com/terrimporter/SSUdiatomClassifier |
| SSU (18S) | Eukaryotes | https://github.com/terrimporter/18SClassifier |
| SSU (16S) | Prokaryotes | Built-in to the RDP classifier |
| ITS | Fungi (Warcup) | Built-in to the RDP classifier |
| ITS | Fungi (UNITE) | Built-in to the RDP classifier |
| LSU | Fungi | Built-in to the RDP classifier |

If you are using the pipeline on a protein coding marker that does not yet have a HMM.profile, such as rbcL, then ESVs are translated into every possible open reading frame (ORF) on the plus strand using ORFfinder v0.4.3 .  The longest ORF (nt) is reatined for each ESV.  Putative pseudogenes are removed as outliers with unusually small/large ORF lengths.  Outliers are calcualted as follows:  Sequence lengths shorter than the 25th percentile - 1.5\*IQR (inter quartile range) are removed as putative pseudogenes (ore sequences with errors that cause a frame-shift).  Sequence lengths longer than the 75th percentile + 1.5\*IQR are also removed as putative pseudogenes.

If you use the pipeline on a protein coding marker that has a HMM.profile, such as COI, then the ESVs are translated into nucleotide and amino acid ORFs using ORFfinder, the longest orfs are retained and consolidated.  Amino acid sequences for the longest ORFs are used for profile hidden Markov model sequence analysis using HMMER v3.3.  Sequence bit score outliers are identified as follows: ORFs with scores lower than the 25th percentile - 1.5\*IQR (inter quartile range) are removed as putative pseudogenes.  This method should remove sequences that don't match well to a profile HMM based on arthropod COI barcode sequences mined from public data at BOLD.

The final output file is results.csv and it has been formatted to specify ESVs from each sample, read counts, ESV/ORF sequences, and column headers.  Additional statistics and log files are also provided for each major bioinformatic step.

## Prepare your environment to run the pipeline

1. This pipeline includes a conda environment that provides most of the programs needed to run this pipeline (SNAKEMAKE, SEQPREP, CUTADAPT, VSEARCH, etc.).

```linux
# Create the environment from the provided environment.yml file.  Only need to do this step once.
conda env create -f environment.yml

# Activate the environment
conda activate MetaWorks_v1

# On the GPSC activate using source
source ~/miniconda/bin/activate MetaWorks_v1
```

2. Download and install the RDP Classifier v2.12

The pipeline also requires the RDP classifier for the taxonomic assignment step.  Although the RDP classifier v2.2 is available through conda, a newer v2.12 is available form SourceForge.  Go to https://sourceforge.net/projects/rdp-classifier/ and hit the 'Download' button to save the file to your computer.  On a mac, the file will be automatically downloaded to your Downloads/ folder.  Then, you can use wget or drag and drop to move the file to your home directory or wherever else you want it.  

```linux
# decompress the file
unzip rdp_classifier_2.12.zip
```

Make a note of where the application is saved so this can be added to the config.yaml file.

```linux
RDP:
    jar: "/path/to/rdp_classifier_2.12/dist/classifier.jar"
```

The RDP classifier comes with the training sets to classify 16S, fungal ITS, and fungal LSU rDNA sequences.  To classify other markers using custom-trained RDP sets, obtain these from GitHub using Table 1 as a guide .  Take note of where the rRNAclassifier.properties file is as this needs to be added to the config.yaml .

```linux
RDP:
    jar: "/path/to/rdp_classifier_2.12/dist/classifier.jar"
    t: "/path/to/CO1Classifier/v4/NCBI_BOLD_merged/mydata/mydata_trained/rRNAClassifier.properties"
```

3. If doing pseudogene filtering, then download and install the NCBI ORFfinder

The pipeline requires ORFfinder 0.4.3 available from the NCBI at ftp://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/ORFfinder/linux-i64/ .  This program should be downloaded, made executable, and put in your path.

```linux
# download
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/ORFfinder/linux-i64/ORFfinder.gz

# decompress
gunzip ORFfinder.gz

# make executable
chmod 755 ORFfinder

# put in your PATH (ex. ~/bin)
mv ORFfinder ~/bin/.
```

Run the program to test that it works:

```linux
ORFfinder
```

If you get an error that requries a GLIBC_2.14 libc.so.6 library, then follow the instructions at [Use conda's libc library for NCBI's ORFfinder](#use-condas-libc-library-for-ncbis-orffinder).

4. In most cases, your raw paired-end Illumina reads can go into a directory called 'data' which should be placed in the same directory as the other files that come with this pipeline.

```linux
# Create a new directory to hold your raw data
mkdir data
```

5. Please go through the config.yaml file and edit directory names, filename patterns, etc. as necessary to work with your filenames.

6. Run Snakemake by indicating the number of jobs or cores that are available to run the whole pipeline.  

```linux
snakemake --jobs 24 --snakefile snakefile --configfile config.yaml
```

7. When you are done, deactivate the conda environment:

```linux
conda deactivate
```

## Implementation notes

### Installing Conda

Conda is an open source package and environment management system.  Miniconda is a lightweight version of conda that only contains conda, python, and their dependencies.  Using conda and the environment.yml file provided here can help get all the necessary programs in one place to run this pipeline.  Snakemake is a Python-based workflow management tool meant to define the rules for running this bioinformatic pipeline.  There is usually no need to edit the snakefile file directly.  Changes to select parameters can be made in the config.yaml file.  If you install conda and activate the environment provided, then you will also get the correct versions of the open source programs used in this pipeline including Snakemake.

Install miniconda as follows:

```linux
# Download miniconda3
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh

# Install miniconda3 and initialize
sh Miniconda3-latest-Linux-x86_64.sh

# Add conda to your PATH, ex. to ~/bin
cd ~/bin
ln -s ~/miniconda3/bin/conda conda

# Activate conda method 1 (working in a container)
source ~/miniconda3/bin/activate MetaWorks_v1

# Activate conda method 2
conda activate MetaWorks_v1
```

### Checking program versions

Ensure the program versions in the environment are being used.

```linux
# Create conda environment from file.  Only need to do this once.
conda env create -f environment.yml

# activate the environment
conda activate MetaWorks_v1

# list all programs available in the environment at once
conda list > programs.list

# you can check that key programs in the conda environment are being used (not locally installed versions)
which SeqPrep
which cutadapt
which vsearch
which perl

# you can also check their version numbers one at a time instead of running 'conda list'
cutadapt --version
vsearch --version
```

Version numbers are also tracked in the snakefile.

### Use conda's libc library for NCBI's ORFfinder

The glibc 2.14 library is already available in the MetaWorks_v1 environment.  The LD_LIBRARY_PATH environment variable will need to be activated (and deactivated) by adding the following scripts as follows:

Create the shell script file LD_PATH.sh in the following location to set the environment variable: ~/miniconda3/envs/MetaWorks_v1/etc/conda/activate.d/LD_PATH.sh

Put the following text in the LD_PATH.sh file:

```linux
export LD_LIBRARY_PATH_CONDA_BACKUP=$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib64:$LD_LIBRARY_PATH
```

Create the file LD_PATH.sh in the following location to unset the environment variable: ~/miniconda3/envs/MetaWorks_v1/etc/conda/deactivate.d/LD_PATH.sh

Put the following text in the LD_PATH.sh file:

```linux
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH_CONDA_BACKUP
```

Create a symbolic link to the library:

```linux
cd ~/miniconda3/envs/MetaWorks_v1/lib64
ln -s ../glibc-2.14/lib/libc.so.6 libc.so.6
```

Deactivate then reactivate the environment.

### Use the bold.hmm with the pseudogene removal pipeline

Ensure that bold.hmm as well as the indexed files (bold.hmm.h3p, bold.hmm.h3m, bold.hmm.h3i, and bold.hmm.h3f) are available in the same directory as you snakefile.

### Batch renaming of files

Sometimes it necessary to rename raw data files in batches.  I use Perl-rename (Gergely, 2018) that is available at https://github.com/subogero/rename not linux rename.  I prefer the Perl implementation so that you can easily use regular expressions.  I first run the command with the -n flag so you can review the changes without making any actual changes.  If you're happy with the results, re-run without the -n flag.

```linux
rename -n 's/PATTERN/NEW PATTERN/g' *.gz
```

### Symbolic links

Symbolic links are like shortcuts or aliases that can also be placed in your ~/bin directory that point to files or programs that reside elsewhere on your system.  So long as those scripts are executable (e.x. chmod 755 script.plx) then the shortcut will also be executable without having to type out the complete path or copy and pasting the script into the current directory.

```linux
ln -s /path/to/target/directory shortcutName
ln -s /path/to/target/directory fileName
ln -s /path/to/script/script.sh commandName
```

### Prevent errors caused by lost network connections

Large jobs with many fastq.gz files can take a while to run.  To prevent unexpected network connection interruptions from stopping the MetaWorks pipeline, set the job up to keep running even if you disconnect from your session:

1. You can use nohup (no hangup).

```linux
nohup snakemake --jobs 24 --snakefile snakefile --configfile config.yaml
```

2. You can use screen.

```linux
# to start a screen session
screen
ctrl+a+c
conda activate MetaWorks_v1
snakemake --jobs 24 --snakefile snakefile --configfile config.yaml
ctrl+a+d

# to list all screen session ids, look at top of the list
screen -ls

# to re-enter a screen session to watch job progress or view error messages
screen -r session_id
```

3. You can submit your job to the queuing system if you use one.

## Tutorial

We have provided a small set of COI paired-end Illumina MiSeq files for this tutorial.  These sequence files contain reads for several pooled COI amplicons, but here we will focus on the COI-BR5 amplicon (Hajibabaei et al., 2012, Gibson et al., 2014).

**Step 1.  Prepare your environment for the pipeline.**

Begin by downloading the latest MetaWorks release available at https://github.com/terrimporter/MetaWorks/releases/tag/v1.1.0 by using wget from the command line:

```linux
# download the pipeline
wget https://github.com/terrimporter/MetaWorks/releases/download/v1.1.0/v1.1.0.zip

# unzip the pipeline
unzip v1.1.0.zip
```

If you don't already have conda on your system, then you will need to install it:

```linux
# Download miniconda3
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh

# Install miniconda3 and initialize
sh Miniconda3-latest-Linux-x86_64.sh

# Add conda to your PATH
# If you do not have a bin folder in your home directory, then create one first.
mkdir ~/bin

# Enter bin folder
cd ~/bin

# Create symbolic link to conda
ln -s ~/miniconda3/bin/conda conda
```

Create then activate the MetaWorks_v1 environment:

```linux
# Move into the MetaWorks folder
cd v1.1.0

# Create the environment from the provided environment.yml file .  Only need to do this step once.
conda env create -f environment.yml

# Activate the environment.  Do this everytime before running the pipeline.
conda activate MetaWorks_v1
```

If you do not already have the RDP classifier installed on your system, download and install the RDP Classifier v2.12.  Go to https://sourceforge.net/projects/rdp-classifier/ and hit the 'Download' button to save the file to your computer.  On a mac, the file will be automatically downloaded to your Downloads/ folder.  Then, you can use wget or drag and drop to move the file to your home directory or wherever else you want it.  

```linux
# decompress the file
unzip rdp_classifier_2.12.zip
```

Make a note of where the application is saved so this can be added to the config_testing_COI_data.yaml file.

```linux
RDP:
    jar: "/path/to/rdp_classifier_2.12/dist/classifier.jar"
```

You will also need to install the COI Classifier from https://github.com/terrimporter/CO1Classifier/releases/tag/v4 .  You can do this at the command line using wget.

```linux
# download the COIv4 classifier
wget https://github.com/terrimporter/CO1Classifier/releases/download/v4/CO1v4_trained.tar.gz

# decompress the file
tar -xvzf CO1v4_trained.tar.gz

# Note the full path to the rRNAClassifier.properties file, ex. mydata_trained/rRNAClassifier.properties
```

If you do not already have the NCBI ORFfinder installed on your system, then download it from the NCBI at ftp://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/ORFfinder/linux-i64/ .  You can download it using wget then make it executable in your path:

```linux
# download
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/ORFfinder/linux-i64/ORFfinder.gz

# decompress
gunzip ORFfinder.gz

# make executable
chmod 755 ORFfinder

# If you do not have a bin folder in your home directory, then create one first.
mkdir ~/bin

# put in your PATH (ex. ~/bin).
mv ORFfinder ~/bin/.
```

**Step 2.  Run MetaWorks using the COI testing data provided.**

The config_testing_COI_data.yaml file has been 'preset' to work with the COI_data files in the testing folder.  You will, however, still need to update your path to the RDP classifier and the trained COI classifier and save your changes.

```linux
RDP:
# enter the path to the RDP classifier 'classifier.jar' file here:
    jar: "/path/to/rdp_classifier_2.12/dist/classifier.jar"

# If you are using a custom-trained reference set 
# enter the path to the trained RDP classifier rRNAClassifier.properties file here:
    t: "/path/to/CO1Classifier/v4/mydata_trained/rRNAClassifier.properties"
```    
  
Then you should be ready to run the MetaWorks pipeline on the testing data.
    
```linux
# You may need to edit the number of jobs you would like to run, ex --jobs 1 or --jobs 4, according to how many cores you have available
snakemake --jobs 2 --snakefile snakefile --configfile config_testing_COI_data.yaml
```

**Step 3.  Analyze the output.**
The final output file is called results.csv .  The results are for the COI-BR5 amplicon.  This can be imported into R for bootstrap support filtering, pivot table creation, normalization, vegan analysis, etc.  There are also a number of other output files in the stats directory showing the total number of reads processed at each step as well as the sequence lengths.  Log files are also available for the dereplication, denoising, and chimera removal steps.

If you are done with MetaWorks, deactivate the conda environment:

```linux
conda deactivate
```

## References

Anaconda (2016).  Anaconda Software Distribution.  Available: https://anaconda.com 

Bengtsson-Palme J, Ryberg M, Hartmann M, Branco S, Wang Z, Godhe A, et al. (2013) Improved software detection and extraction of ITS1 and ITS2 from ribosomal ITS sequences of fungi and other eukaryotes for analysis of environmental sequencing data.  Methods in Ecology and Evolution, 4: 914–919. doi:10.1111/2041-210X.12073  

Callahan, B. J., McMurdie, P. J., Rosen, M. J., Han, A. W., Johnson, A. J. A., & Holmes, S. P. (2016). DADA2: High-resolution sample inference from Illumina amplicon data. Nature Methods, 13(7), 581–583. doi: 10.1038/nmeth.3869

Edgar, R. C. (2016). UNOISE2: improved error-correction for Illumina 16S and ITS amplicon sequencing. BioRxiv. doi:10.1101/081257  

Gergely, S. (2018, January). Perl-rename. Retrieved from https://github.com/subogero/rename  

Gibson, J., Shokralla, S., Porter, T. M., King, I., Konynenburg, S. van, Janzen, D. H., … Hajibabaei, M. (2014). Simultaneous assessment of the macrobiome and microbiome in a bulk sample of tropical arthropods through DNA metasystematics. Proceedings of the National Academy of Sciences, 111(22), 8007–8012. doi: 10.1073/pnas.1406468111

Hajibabaei, M., Spall, J. L., Shokralla, S., & van Konynenburg, S. (2012). Assessing biodiversity of a freshwater benthic macroinvertebrate community through non-destructive environmental barcoding of DNA from preservative ethanol. BMC Ecology, 12, 28. doi: 10.1186/1472-6785-12-28

Iwasaki W, Fukunaga T, Isagozawa R, Yamada K, Maeda Y, Satoh TP, et al.( 2013) MitoFish and MitoAnnotator: A Mitochondrial Genome Database of Fish with an Accurate and Automatic Annotation Pipeline. Molecular Biology and Evolution, 30:2531–40. 

Koster, J., Rahmann, S. (2012) Snakemake - a scalable bioinformatics workflow engine.  Bioinformatics, 29(19): 2520-2522. 

Martin, M. (2011). Cutadapt removes adapter sequences from high-throughput sequencing reads. EMBnet. Journal, 17(1), pp–10.  

Porter, T. M., & Hajibabaei, M. (2018). Automated high throughput animal CO1 metabarcode classification. Scientific Reports, 8, 4226.  

Pruesse E, Quast C, Knittel K, Fuchs BM, Ludwig W, Peplies J, et al.(2007) SILVA: a comprehensive online resource for quality checked and aligned ribosomal RNA sequence data compatible with ARB. Nucleic Acids Research, 35:7188–96.	

Rimet F, Chaumeil P, Keck F, Kermarrec L, Vasselon V, Kahlert M, et al. (2015) R-Syst::diatom: a barcode database for diatoms and freshwater biomonitoring - data sources and curation procedure. INRA Report. 

Rognes, T., Flouri, T., Nichols, B., Quince, C., & Mahé, F. (2016). VSEARCH: a versatile open source tool for metagenomics. PeerJ, 4, e2584. doi:10.7717/peerj.2584  

St. John, J. (2016, Downloaded). SeqPrep. Retrieved from https://github.com/jstjohn/SeqPrep/releases  

Wang, Q., Garrity, G. M., Tiedje, J. M., & Cole, J. R. (2007). Naive Bayesian Classifier for Rapid Assignment of rRNA Sequences into the New Bacterial Taxonomy. Applied and Environmental Microbiology, 73(16), 5261–5267. doi:10.1128/AEM.00062-07  

Last updated: July 17, 2020
