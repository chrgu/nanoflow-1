# Nanoflow: a NANOpore sequencing data bioinformatics workFLOW

Nanoflow is a pipeline written in snakemake to automate many of the steps of quality control, de novo assemblies and genome annotation in whole genome sequencing analysis, using Oxford Nanopore sequencing data.

[![CircleCI](https://circleci.com/gh/zhaoc1/nanoflow.svg?style=shield)](https://circleci.com/gh/zhaoc1/nanoflow)

20190104: add github page

## Installation

1. Install [Conda](https://conda.io/miniconda.html) environment manager, and make sure the `~/.condarc` is your home directory.
  ```bash
  nano ~/.condarc
  ```
  
  Copy the following to the file.
   ```bash
   channels:
  - bioconda
  - conda-forge
  - defaults
  ```

2. Install GCC5, by cloning [Jesse](https://github.com/ressy)'s conda-gcc5 repository and create an new conda environment `nanoflow`.
  
  ```bash
  cd ~
  git clone https://github.com/ressy/conda-gcc5.git
  cd conda-gcc5
  bash setup.sh nanoflow
  ```
  
3. Clone this repository into a local directory and install the packages into `nanoflow` environment.

  ```bash
  git clone https://github.com/zhaoc1/nanoflow.git nanoflow
  cd nanoflow
  source activate nanoflow
  
  conda install -n nanoflow -c bioconda snakemake=4.8.1
  conda env update --name=nanoflow --file env.yml
  ```
 
4. Clone Ryan Wick's Basecalling-comparison repository
  ```bash
  mkdir local
  cd local
  git clone https://github.com/rrwick/Basecalling-comparison.git
  ```

5. Download other packages into local directory
  ```bash
  ## Canu 1.8
  wget https://github.com/marbl/canu/archive/v1.8.tar.gz
  tar -xvf v1.8.tar.gz
  cd canu-1.8/src
  make -j 4

  ## Nanopolish v0.9.0
  git clone --recursive https://github.com/jts/nanopolish.git
  cd nanopolish
  make
  
  ## Unicycler
  git clone https://github.com/rrwick/Unicycler.git
  cd Unicycler
  python3 setup.py install
  
  ## set up for Quast
  git clone https://github.com/lucian-ilie/E-MEM.git
  cd E-MEM
  make
  ```

## Usage
1. Basecalling: the raw fast5 signal data files were basecalled using ONT’s Albacore command line tool (v.2.2.7), with barcode demultiplexing and fastq output. You can perform the basecalling step either by snakemake or run the `run_albacore.sh` bash script, with proper directory info.

  ```bash
  snakemake --configfile all_basecalling
  ```

2. Preprocess: quality filter, confidently-binned, and subsampled subsample long reads

  ```bash
  snakemake --configfile config.yml --cores 8 all_qc
  ```
 
3. Hybrid assembly option 1: [ Canu](http://canu.readthedocs.io/en/latest/quick-start.html) + [ Nanopolish](http://nanopolish.readthedocs.io/en/latest/installation.html#installing-a-particular-release) (+ [ Circlator](https://github.com/sanger-pathogens/circlator/wiki/Brief-instructions) + [ Pilon](https://github.com/broadinstitute/pilon/wiki))

    * long reads only product: long reads only assembly polished by signal data, can be used by hybrid assembly option 3.
    
  ```bash
  snakemake --configfile config.yaml --cores 8 all_draft1
  ```
  
4. Hybrid assembly option 2: [ Unicycler](https://github.com/rrwick/Unicycler#method-hybrid-assembly) (default mode)

   * `depth=X` in the FASTA header: to preserve the relative depths. This is mainly used for plasmid sequences, which should be more represented in the reads than the chromosomal sequence.
 
  ```bash
  snakemake --configfile config.yaml --cores 8 all_draft2
  ```

5. Hybrid assembly option 3: [ Unicycler](https://github.com/rrwick/Unicycler#method-hybrid-assembly) (existing long reads assembly option)

 
  ```bash
  snakemake --configfile config.yaml --cores 8 all_draft3
  ```  

6. For the final draft genome, a common practice is to choose two of the assemblies results you are happy with, assess them with the provided reference genome, compare one to the other, and map reads back to the draft genomes to calcualate the coverage. All of these tasks are implemented in the `assembly.rules`.

    * We sequenced C diff isoaltes at PCMP, and therefore in the `run_prokka` rules, I used the `genus` level prokka database. If you have a different organisms to study, please build the prokka genus database by yourself and change the corresponding lines in the `run_prokka` rule. 

  ```bash
  snakemake --configfile config.yaml --cores 8 all_final
  ```

6. Assembly assess and comparison

  * Metrics description
    
    * `Misjoins`: locations where two adjacent sequences in the assembly should be split apart and placed at distinct locations in order to match the reference.

    * `Relocation`: a misjoin where a segments needs to be moved elsewhere on the chromosome.
    
     * `Misassemblies`: QUAST categories misassemblies as either local (less than 1kbp discrepancy) or extensive (more than 1 kbp discrepancy)
    
  * A good reference guide for interpretting the dot plot is available [ here](http://mummer.sourceforge.net/manual/AlignmentTypes.pdf).
    
  * Some good tutorials 😳
    - Align two draft sequences using [ MUMmer](http://mummer.sourceforge.net/manual/#aligningdraft).
    - Evaluate the assembly using [ MUMmer](http://nanopolish.readthedocs.io/en/latest/quickstart_consensus.html).
    - Assembly evaluation with [ QUAST](http://denbi-nanopore-training-course.readthedocs.io/en/latest/assembly_qc/quast.html).
    - Multiple assemblies comparison using [ QUAST](http://quast.bioinf.spbau.ru/manual.html#faq_q16).
    - Highly similar sequences with rearrangments using [ run-mummer3](http://mummer.sourceforge.net/manual/#mummer3) [TODO].
    - Assembly to assembly comparisons using [ Minimap2](https://github.com/lh3/minimap2/issues/109) [TODO].
    - [Microbial genomics tutorials](http://sepsis-omics.github.io/tutorials/modules/cmdline_assembly_v2/) using PacBio long reads from ABRPI-Training.
    - de.NBI [Nanopore Training Course](https://denbi-nanopore-training-course.readthedocs.io/en/latest/index.html).
    
   * Wish you knew sooner 😔
      - Minimap2 and the future of BWA, by Heng Li's [blog](https://lh3.github.io/2018/04/02/minimap2-and-the-future-of-bwa).
      - Long reads assembly: indels cause interrupted genes, by Mick Watson's [blog](http://www.opiniomics.org/a-simple-test-for-uncorrected-insertions-and-deletions-indels-in-bacterial-genomes/). I also have an example for this issue [ demo_interrupted_genes](https://github.com/zhaoc1/nanoflow/blob/master/demo_interruptted_genes.pdf)
      - This [paper](https://academic.oup.com/bioinformatics/advance-article-abstract/doi/10.1093/bioinformatics/bty833/5106166) talks about the commonly incorrect use of the *max_target_seqs* of BLAST.
      
   * Two optional features provided by Nanoflow:
      1. assess draft genomes using QUAST 
      ```bash  
      snakemake --configfile config.yaml _all_quast --use-conda
      ```
      
      2. IGV: short/long reads mapped to draft assembly
        * Refer to the subworkflow of [ sunbeam](http://sunbeam.readthedocs.io/en/latest/): [ sbx_igv](https://github.com/sunbeam-labs/sbx_igv)
   
     ```bash
     snakemake --configfile config.yaml _all_igv
     ```
 
 7. Generate bioinformatics report refer to `bioinfo_report.Rmd`. An example output is shown in [**bioinfo_report.pdf**](https://github.com/zhaoc1/nanoflow/blob/master/bioinfo_report.pdf).
