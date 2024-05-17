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

##### Step 4b: Filter_Junctions 
This step will concatenate the junction files discovered in Step 4a across samples, then filter them based on user-defined parameters in the config file. This step is not required, as STAR will automatically concatenate the "SJ.out.tab" files before mapping. 
If you want to skip this step, you can instead just pass a list of all the "SJ.out.tab" files for all samples from step 4a directly into the `FILTERED_JUNC_LIST` variable for Read_Mapping. 
However, filtering junctions is recommended when you have large numbers of samples, and there are several reasons we have implemented this intermediate filtering step: 
1) Spurious junctions may increase the number of reads that map to multiple places in the genome
2) Filtering and concatenating junction files before mapping speeds up the mapping step in 4c -- both because of the smaller number of junctions and because STAR then doesn't need to perform the concatenation across large numbers of samples for each sample separately.
3) This list or lists can be more easily saved in case one wants to redo the mapping!

Once the variables in the configuration files have been defined, Filter_Junctions can be submitted to a job scheduler with the following command (assuming in the directory containing `dev_RNAseq `) 
`./dev_RNAseq.sh Filter_Junctions Config` 
Where `Config` is the full file path to the configuration file 

##### Step 4c: Read_Mapping 
This step will take all of the novel junctions discovered in the first pass and use them to re-map reads for each sample (in addition to the already-annotated junctions from your annotation file). 

The `FILTERED_JUNC_LIST` variable can be one of two things: 
1) the filepath to the filtered junctions file output from the "Filter_Junction" handler.

Alternatively, if you have multiple outputs from "Filter_Junctions" (if you process samples in batches like we do); this variable can be:

2) a .txt file with a list of the full filepaths to all filtered junction files to be included. STAR can handle multiple junction files as input (an will concatenate before mapping).
As previously mentioned, this list could also just be a list of filepaths to the un-filtered, un-concatenated "SJ.out.tab" files from all samples (if skipping the "Filter_Junctions" step).

To run Read_Mapping, all common and handler-specific variables must be defined within the config file. Once those have been defined, Read_Mapping can be submitted to a job scheduler with the following command (assuming you are in the directory containing `dev_RNAseq`) 
`./dev_RNAseq.sh Read_Mapping Config`
where `Config` is the full file path to the configuration file. 

### Option for 1-pass mapping 
If you want to map your reads without the addition of novel junctions discovered in a first mapping step, you can skip the "Collect_Junctions" and "Filter_Junctions" steps and leave the `FILTERED_JUNC_LIST` variable blank. All other variables need to be specified for Read_Mapping in the configuration file. Once the variables have been defined, Read_Mapping can be submitted to a job scheduler with the same command as listed above. 

### Final notes: 
#### Read groups
If you have sequence data from the same sample across multiple lanes/runs, the best practice is to map these separately (in order to test for batch effects). 

#### Alignment file output 
Two alignment files are output from STAR with this handler - one alignment file in genomic coordinates and one translated into transcrip coordinates (e.g., `*Aligned.toTranscriptome.out.bam`). The default format of the former is an unsorted SAM file. If you plan on using the genomic coordinate alignments for SNP calling, you have the option of getting these output as coordinate-sorted BAM files (similar to the `samtools sort` command) by putting a "yes" for the `GENOMIC_COORDINATE_BAMSORTED` variable in the config. Note that this will add significant computation time and memory. 
