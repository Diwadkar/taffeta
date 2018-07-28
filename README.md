taffeta
=======

Reproducible analysis and validation of RNA-Seq data

Authors: Maya Shumyatcher, Mengyuan Kan, Blanca Himes.

## Overview

The goal of taffeta is to preform reproducible analysis and validation of RNA-Seq data, as a part of [RAVED pipeline](https://github.com/HimesGroup/raved):
  * Download SRA .fastq data
  * Perform preliminary QC
  * Align reads to a reference genome
  * Perform QC on aligned files
  * Create a report that can be used to verify that sequencing was successful and/or identify sample outliers
  * Perform differential expression of reads aligned to transcripts according to a given reference genome
  * Create a report that summarizes the differential expression results

Generate LSF scripts with download commands that can be processed in parallel. Specify --fastqc option to run FastQC for .fastq files downloaded.

## Dependencies

* For alignment and quantification: STAR, HTSeq, kallisto (older versions used bowtie2, tophat, cufflinks, cummerbund, whose options are no longer available).
* For DE analysis: R packages DESeq2 and sleuth
* For QC: fastqc, trimmomatic, samtools, bamtools, picardtools.
* Annotation files should be available: reference genome fasta, gtf, refFlat, and index files. We use ERCC spike-ins, so our reference files include ERCC mix transcripts. 
* For adapter trimming, we provide Ilumina TruSeq single index and Illumina unique dual (UD) index adpter and primer sequences. Users can tailor this file by adding sequences from other protocols.
* The Python scripts make use of modules that include subprocess, os, argparse, sys.
* To create reports, R and various libraries should be available, such as DT, gplots, ggplot2, reshape2, rmarkdown, RColorBrewer, plyr, dplyr, lattice, ginefilter, biomaRt. Note that pandoc version 1.12.3 or higher is required to generate HTML report from current RMD files.

## Workflow

### Download data from GEO/SRA

Run script rnaseq\_sra\_download.py to download .fastq files from SRA. Users can provide a phenotype file with a SRA\_ID column. Download samples corresponding to the SRA\_ID. If a phenotype file is not provided, use phenotype information from GEO. SRA\_ID is retrieved from the field relation.1. Corresponding ftp addresses are obtained from SRA sqilte database.

> rnaseq\_sra\_download.py --geo\_id <i>GEO_Accession</i> --path\_start <i>output_path</i> --project\_name <i>output_prefix</i> --template\_dir <i>templete_file_directory</i> --fastqc

Output files: 1. <i>project_name</i>\_SRAdownload\_RnaSeqReport.html; 2. <i>project_name</i>\_sraFile.info 3. <i>GEO_Accession</i>\_withoutQC.txt 4. FastQC results


### User-tailored phentoype file

The sample info file used in the following steps should be provided by users.

Required columns: 'Sample' column containing sample ID, 'Status' column containing variables of comparison state. 'R1' and/or 'R2' columns containing full paths of .fastq files.

Other columns: 'Treatment', 'Disease', 'Donor' (donor or cell line ID if in vitro treatment is used), 'Tissue', 'ERCC\_Mix' if ERCC spike-in sample is used, 'protocol' designating sample preparation kit information.

'Index' column contains index sequence for each sample. If provided, trim raw .fastq files based on corresponding adapter sequences

Most GEO phenotype data do not have index information. However, FastQC is able to detect them as "Overrepresented sequences". Users can tailor Index column based on FastQC results. We provide a file with most updated adapter and primer sequences for FastQC detection.

### Alignment, quantification and QC

1) Run rnaseq\_align\_and\_qc.py to perform: 1) adapter trimming, 2) fastqc, 3) alignment, 4) quantification, 5) QC metrics 

> rnaseq\_align\_and\_qc.py --project\_name <i>output_prefix</i> --sample\_in <i>sample_info_file.txt</i> --aligner star --ref\_genome hg38 --library\_type PE --index\_type truseq\_single\_index --strand nonstrand --path\_start <i>output_path</i> --template\_dir <i>templete_file_directory</i>

> for i in *.lsf; do bsub < $i; done

The "--aligner star" option indicates that star should be used as the aligner (default is tophat). The "--ref\_genome hg38" option refers to using hg38 human genome reference. The "--library\_type PE" option refers to PE (paired-end) or SE (single-end) library. The "--index\_type truseq\_single\_index" option refers to index used in sample library preparation. The "--strand nonstrand" option refers to sequencing that captures both strands (nonstrand) or the 1st strand (reverse) or the 2nd strand (forward) of cDNA.


Following execution of LSF scripts, various output files will be written for each sample in directories structured as:

>
 <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_R1_Trimmed.fastqc <br>
 <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_R2_Trimmed.fastqc <br>
 <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_R1_fastqc <br>
 <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_R2_fastqc <br>
 <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_ReadCount <br>
 <i>path_start</i>/<i>sample_name</i>/<i>aligner</i>_out <br>
 <i>path_start</i>/<i>sample_name</i>/<i>quantification_tool</i>_out <br>

2) Run rnaseq_align\_and\_qc\_report.py. Create an HTML report of QC and alignment summary statistics for RNA-seq samples.

> python rnaseq\_align\_and\_qc\_report.py  --project\_name <i>output_prefix</i> --sample\_in <i>sample_info_file.txt</i> --aligner star --ref\_genome hg38 --library\_type PE --path\_start <i>output_path</i> --template\_dir <i>templete_file_directory</i>
> bsub < <i>project_name</i>_qc.lsf
	
This script uses the many output files created in step 1), converts these sample-specific files into matrices that include data for all samples, and then creates an Rmd document (main template is rnaseq_align_and_qc_report_Rmd_template.txt) that is converted into an html report using pandoc and R package rmarkdown.

The report and accompanying files are contained in:

> <i>path_start</i>/<i>project_name</i>_Alignment_QC_Report/

The RMD and corresponding HTML report files:

> <i>path_start</i>/<i>project_name</i>_Alignment_QC/_Report/<i>project_name</i>_QC_RnaSeqReport.Rmd
> <i>path_start</i>/<i>project_name</i>_Alignment_QC/_Report/<i>project_name</i>_QC_RnaSeqReport.html

### Differential expression (DE) analysis - DESeq2

Run rnaseq_de_report.py to perform DE analysis and create an HTML report of differential expression summary statistics.

> rnaseq_de_report.py --project_name <i>output_prefix</i> --sample_in <i>sample_info_file_withQC.txt</i> --comp <i>sample_comp_file.txt</i> --de_package deseq2 --ref_genome hg38 --path_start <i>output_path</i> --template_dir <i>templete_file_directory</i> bsub < <i>project_name</i>_deseq2.lsf

The "--comp <i>sample_comp_file.txt</i>" specifies comparisons of interest in a tab-delimited text file with one comparison per line with three columns (i.e. Condition1, Condition0, Design), designating Condition1 vs. Condition2. The DE analysis accommodates a "paired" or "unpaired" option specified in Design column. For paired design, specify the condition to correct for that should match the column name in the sample info file - e.g. paired:Donor. Note that if there are any samples without a pair in any given comparison, the script will automatically drop these samples from that comparison, which will be noted in the report.

This script creates an RMD document (main template is rnaseq_de_report_Rmd_template.txt) that uses the DESeq2 R package to load and process the HTSeq output file 2). The report and accompanying files are contained in:

> <i>project_name</i>/<i>project_name</i>_deseq2_out/

The RMD and corresponding HTML report file:

> <i>path_start</i>/<i>project_name</i>_deseq2_out/<i>project_name</i>_DESeq2_Report.Rmd
> <i>path_start</i>/<i>project_name</i>_deseq2_out/<i>project_name</i>_DESeq2_Report.html
	
### acknowledgements
This set of scripts was initially developed to analyze RNA-Seq and DGE data at the [Partners Personalized Medicine PPM](http://pcpgm.partners.org/). Barbara Klanderman is the molecular biologist who led the establishment of PPM RNA-seq lab protocols and played an essential role in determining what components of the reports would be most helpful to PPM wet lab staff. 

