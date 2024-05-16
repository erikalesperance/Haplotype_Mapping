# Haplotype_Mapping
Pipeline for mapping RNAseq reads (from leaf tissue, SAM, and the floral dev stages used for genome annotations) to each haplotype of Bidens cv. Compact Yellow. Buit from Emily Yaklich's sunflower dev_RNAseq pipeline: https://github.com/erikalesperance/dev_RNAseq/blob/main/README.md


Programs used: 
FASTQC: https://dnacore.missouri.edu/PDF/FastQC_Manual.pdf
MultiQC: https://multiqc.info/
Trimmomatic: http://www.usadellab.org/cms/uploads/supplementary/Trimmomatic/TrimmomaticManual_V0.32.pdf
STAR: http://chagall.med.cornell.edu/RNASEQcourse/STARmanual.pdf

## Pre-Processing data: 

Begins with raw sequence data (mine wsa in .fq.gz format). Copy data into working scratch directory. 

## Step one: quality assessment 

Run Quality_Assessment on raw FastQ files. 
To run Quality_Assessment, all common and handler-specific variables must be defined within the configuration (`Config`) file. 
Once these variables have been defined, Quality_Assessment can be submitted to a job scheduler with the following command (assuming you are in the directory containing `dev_RNAseq`): 

`./dev_RNAseq.sh Quality_Assessment Config` where `Config` is the full file path to the configuration file 

A directory containing your files to be analyzed MUST be specified within the Config file. The directory can have sub-directories containing the rawdata for each sample. 

After the quality assessment is completed for each sample, the FastQC results will be summarized by MultiQC. The summary statistics will be located in the output directory specified within the Config file. 


## Step two: adapter trimming
