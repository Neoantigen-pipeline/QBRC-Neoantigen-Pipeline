# The QBRC neoantigen calling pipeline 
![preview](https://github.com/tianshilu/QBRC-Neoantigen-Pipeline/blob/master/neoantigen_flow.jpg)
## Introduction
The QBRC neoantigen calling pipeline is a comprehensive and user-friendly neoantigen calling pipeline for human genomics samples. It needs the somatic mutation calling results of the QBRC mutation calling pipeline, the tumor/normal exome-seq data for HLA typing, and optionally RNA-seq data for filtering neoantigens called from the exome-seq data. It profiles both MHC I and II-binding neoantigens. The calculation of CSiN (Cauchy-Schwarz index of Neoantigens), which describes neoantigen clonal balance, is embedded in the pipeline. Please refer to https://qbrc.swmed.edu/labs/wanglab/index.php for more information. 
## Citation
* If you use the pipeline, please cite: \
**"Tumor Neoantigenicity Assessment with CSiN Score Incorporates Clonality and Immunogenicity to Predict Immunotherapy Outcomes"**, Science Immunology 
* For liscence information, please refer to: \
https://github.com/tianshilu/QBRC-Neoantigen-Pipeline/blob/master/License.txt

## Running time
If HLA typing information is available, it takes less than 1 hour to get the neoantigen calling done. If HLA alleles needs to be typed from DNA sequencing, it usually takes around 2 hours to finish the neoantigen calling.
## Dependencies
   64 digits Linux operating system  
   iedb (MHC_I, MHC_II)  
   featureCounts (version>=1.6)  
   samtools (version>=1.4)   
   STAR (if providing RNA sequencing fastq files)   
   annovar (>=2017Jul16, humandb in default position)  
   novoalign  
   Athlates (need lib64 of gcc>=5.4.0 in LD_LIBRARY_PATH copy files under data/msa_for_athlates to Athlates_2014_04_26/db/msa and data/ref.nix to Athlates_2014_04_26/db/ref)  
   python (python 2)  
   perl (version 5, Parallel::ForkManager installed)  
   mixcr (>=2.1.5)  
   gzip  
   Rscript   

## Input files
Exome sequencing can be fastq files or bam files. fastq files must be gzipped. You can choose to input expression data. Expression data can be fastq files single-end or paired-end, gzip-end. Expression data can also be bam files.
## Main procedures:
* Extract mutations meeting following criteria:
    * (1) Only frameshift indels, non-frameshift indels, missense and stop-loss mutations that would lead to protein sequence changes 
    * (2) Only variant allele frequencies (VAFs) downloadedwere <0.02 in the normal sample and VAFs>0.05 in the tumor samples will be analyzed
* Neoantigen length:
    * For class I HLA proteins (A, B, C), the putative neoantigens of 8-11 amino acid in length are called, and for class II HLA proteins (DRB1 and DQB1/DQA1), the putative neoantigens of 15 amino acids in length are called.
* HLA typing:
    * Class I and II HLA subtypes were predicted by the ATHLATES tool. Putative neoantigens with amino acid sequences exactly matching known human protein sequences were filtered out. 
* Neoantigen-HLA binding affinity:
    * (1) For class I bindings, the IEDB-recommended mode (http://tools.iedb.org/main/) was used for prediction of binding affinities, while for class II binding, NetMHCIIpan embedded in the IEDB toolkit was used.
    * (2) Neoantigens were kept only if the predicted ranks of binding affinities were ≤2%. Tumor RNA-seq data were aligned to the reference genome using the STAR aligner. 
    * (3) FeatureCounts was used to summarize gene expression levels. 
* Neoantigen expression: 
    * Neoantigens whose corresponding mutations were in genes with expression level <1 RPKM in either the specific exon or the whole transcript were filtered out. 

## Guided Tutorial
## detect_neoantigen.pl
Identification of neoantigens from somatic mutations.

### Usage
```
perl detect_neoantigen.pl  somatic_result expression_somatic_result min_normal_cutoff max_normal_cutoff build output fastq1 fastq2 expression_files gtf mhc_i mhc_ii percentile_cutoff rpkm_cutoff thread max_mutations
```
* somatic_result: somatic mutaion calling file of exome-seq data, generated by somatic.pl, or can be another file that follows the output format of somatic.pl 
* expression_somatic_result: somatic mutation calling file of the corresponding RNA-Seq data, the format is the same as $somatic if this data are not available, use "NA" instead 
* min_tumor_cutoff: minimum VAF in tumor sample 
* max_normal_cutoff: maximum VAF in normal sample 
* build: human genome build, hg19 or hg38. The pipeline will search for other files in that bundle folder automatically.
* output: output folder, safer to make "output" a folder that only holds results of this analysis job
* fastq1,fastq2: 
  * fastq files (must be gzipped) of tumor exome-seq for HLA type. Alternatively
  * (1) if you do not have raw fastq files but have the paired-end exome-seq bam files, use "bam path_to_bam1,path_to_bam2,path_to_bam3". You can give one or multiple bam files. But using bam files is not preferred. Speed is slow when individual bam file is >10GB.
  * (2) if want to use pre-existing HLA typing results, use "pre-existing path-to-file" in place of the fastq files,the typing file (path-to-file) can contain one or more lines. Each line is for one HLA class (A, B, C, DQB1, DRB1) Any class can appear 0 or 1 time in the typing file, but no more than 1 time. Each line (class) follows this format: "class\ttype1\ttype2\t0". An example: example/typing.txt
* expression_files: expression data
  * <1> If expression data do not exist at all, use "NA" instead, and accordingly, $gtf and $rpkm_cutoff will not matter anymore
  * <2> If raw RNA-Seq fastq files (single-end or paired-end, gzip-ed) are available, use "path-to-STAR-index:path-to-fastq1.gz,path-top-fastq2.gz" or "path-to-STAR-index:path-to-fastq.gz".
  * <3> If bam files are available (paired-end),use "path-to-STAR-index:bam,bam_file_path".
        If transcript level and exon level gene expression data are available,compile them into the formats of these two files: example/exon.featureCounts, example/transcript.featureCounts,and specify these two files in the exp_bam input parameter as "counts:path-to-exon-count,path-to-transcript-count" $gtf will not matter anymore. 
 * gtf: gtf file for featureCounts in the genome reference bundle. 
 * mhc_i, mhc_ii: folders to the iedb mhc1 and mhc2 binding prediction algorithms, http://www.iedb.org/ 
 * percentile_cutoff: percentile cutoff for binding affinity (0-100), recommended: 2 
 * rpkm_cutoff: RPKM cutoff for filtering expressed transcripts and exons, recommended: 1 
 * thread: number of threads to use. 
 * max_mutations: if more than this number of mutations are left after all filtering, the program will abort. Otherwise, it will take too much time. recommended: 50000\
Example data for running the pipeline can be found here https://github.com/Neoantigen-pipeline/QBRC-Neoantigen-Pipeline/tree/master/example_data.
### Command Example: 
```
perl ~neoantigen/detect_neoantigen.pl ~/somatic_result/1799-01/somatic_mutations_hg38.txt NA 0.02 0.05 hg38 ~/neoantigen_result/1799-01/ ~/seq/1799-01T.R1.fastq.gz ~/seq/1799-01T.R2.fastq.gz ~/seq/exp/1799-01.bam ~/ref/hg38/hg38_genes.gtf ~/neoantigen/code/mhc_i ~/neoantigen/code/mhc_ii 2 1 32 50000
```
## job_detect_neoanitgen.pl
Slurm wrapper for calling somatic mutations and it is easy to change for other job scheduler system by revising this line of code: "system("sbatch ".$job)" and using proper demo job submission shell script.
### Usage
```
perl job_detect_neoantigen.pl design.txt example min_normal_cutoff max_normal_cutoff build gtf mhc_i mhc_ii  percentile_cutoff, rpkm_cutoff, thread, max_mutations, n
```
### Note:
* design.txt : the batch job design file, it has 6 columns separated by \t, they correspond to the $somatic,$expression_somatic, $output, $fastq1, $fastq2, and $exp_bam input variables of detect_neoantigen.pl 
* example : the demo job submission shell script. A default one is in this folder 
* min_tumor_cutoff : minimum VAF in tumor sample 
* max_normal_cutoff : maximum VAF in normal sample 
* build : human genome build, hg19 or hg38
* gtf : gtf file for featureCounts 
* mhc_i, "mhc_ii : folders to the iedb mhc1 and mhc2 binding prediction algorithms, http://www.iedb.org/ 
* percentile_cutoff : percentile cutoff for binding affinity (0-100), recommended: 2 
* rpkm_cutoff : RPKM cutoff for filtering expressed transcripts and exons, recommended: 1 
* thread : number of threads to use. 
* max_mutations : if more than this number of mutations are left after all filtering, the program will abort. Otherwise, it will take too much time. recommended: 50000 
* n : bundle $n somatic calling jobs into one submission
### design.txt example (6 columns; columns seperated by tab):
```
~/somatic_result/1799-01/somatic_mutations_hg38.txt NA ~/neoantigen_result/1799-01/ ~/seq/1799-01T.R1.fastq.gz ~/seq/1799-01T.R1.fastq.gz ~/ref/hg38/STAR:bam, ~/seq/exp/1799-01.bam 
~/somatic_result/1799-02/somatic_mutations_hg38.txt NA ~/neoantigen_result/1799-02/ ~/seq/1799-02T.R1.fastq.gz ~/seq/1799-02T.R1.fastq.gz ~/ref/hg38/STAR:bam, ~/seq/exp/1799-02.bam 
~/somatic_result/1799-03/somatic_mutations_hg38.txt NA ~/neoantigen_result/1799-03/ ~/seq/1799-03T.R1.fastq.gz ~/seq/1799-03T.R1.fastq.gz ~/ref/hg38/STAR:bam, ~/seq/exp/1799-03.bam
```
### Command example: 
```
perl ~/neoantigen/job_detect_neoantigen.pl design.txt ~neoantigen/example/example.sh 0.02 0.05 hg38 ~/ref/hg38/hg38_genes.gtf ~/neoantigen/code/mhc_i ~/neoantigen/code/mhc_ii 2 1 32 50000 2
```
