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
The handler, Adapter_Trimming, utilizes Trimmomatic to trim adapter sequences from FastQ files. 
Trimmomatic takes paired-end information into account when doing so (if applicable). 

The handler can accept an input EITHER as a directory (which can be to multiple sub-directories for each sample) or a text-file list of forward samples (it will find the reverse samples based on the naming suffix specifie din the Config file) 

To run Adapter_Triming, all common and handler-specific variables must be defined within the Configuration file. Once the variables have been defined, Adapter_Trimming can be submitted to a job scheduler with the following command (assuming you are in the directory containing `dev_RNAseq`) 
`./dev_RNAseq.sh Adapter_Trimming Config` 
where `Config` is the full file path to the configuration file 

It is recommended that you re-run Quality_Assessment after adapter trimming to ensure that any adapter contamination was eliminated. 

## Step three: genome index 
This handler will generate a genome index using FASTA and GFF3 or GTF formatted annotations. This step only needs to be performed once for each genome/annotation combination. 

If using a GFF3 file for the genome indexing rather than the default GTF file, the option `--sjdbGTFtagExonParentTranscript Parent` is added to the script 

To run Genome_Index, all common and handler-specific variables must be defined within the configuration file. Once those variables have been defined, then the Genome_Index can be submitted to a job scheduler with the following command (assuming you are in the directory containing `dev_RNAseq`) 
`./dev_RNAseq.sh Genome_Index Config` 
where `Config` is the full file path to the configuration file. 

You will use the contents of the output (directory specified in Config file) for the next step! 

## Step four: read mapping 
The Read_Mapping handler uses STAR to map reads to the genome index in the previous step (step 3) 
This handler can accept an input as EITHER a directory or a text-file list of forward samples (it will find the reverse samples based on the naming suffix specified in the config file) 

### We are using -GeneCounts flag of STAR (specified in Config file) which will also quantify our transcripts and the output will be in a `.tab` file (one for each sample) with the rest of the STAR outputs. These are the files that we will use to analyze our expression data, so this is the most important output from this step. 

#### Option for 2-pass mapping 
STAR can perform a 2-pass mapping strategy to increase mapping senstivity around novel splice junctions. This works by running a 1st mapping pass for all samples with the "usual" parameters to identify un-annotated splice junctions. Then mapping is re-run, using the junctions detected in the first pass (in addition to the annotated junctions specified in the genome annotation file). This 2-pass mapping strategy is recommended by GATK and ENCODE best-practices for better alignments around novel splice junctions. 

While STAR can perform 2-pass mapping on a per-sample basic, in a study with multiple samples, it is recommended to collect 1st pass junctions from all samples for the second pass. Therefore, the recommended 2-pass procedure is described below: 

##### Step 4a: Collect_Junctions
This step identifies novel junctions during a first read-mapping pass and outputs them as "SJ.out.tab" files for each sample. In the handler used here, only junctions supported by at least one uniquely mapped read will be output. This step shares variables for Read_Mapping in the Config file, so ensure that these are filled out. 

Once the variables have been defined, Collect_Junctions can be submitted to a job scheduler with the following command (again, assuming that you are in the directory containing `dev_RNAseq`)
`./dev_RNAseq.sh Collect_Junctions Config` 
where `Config` is the full file pathway to the configuration file. 
