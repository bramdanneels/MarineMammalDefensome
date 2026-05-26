# Marine mammal chemical defensome analysis pipeline

This repository contains the code and pipeline used for analysing the inventory of chemical defensome genes in different marine mammals lineages and their relatives.
The analysis pipeline is implemented in Snakemake, and all necessary software can be installed through Conda.

## Overview of the pipeline

![Defensome pipeline overview](https://github.com/bramdanneels/MarineMammalDefensome/blob/main/images/DefensomeWorkflow3.png)

The pipeline identifies and compares the gene inventory of the chemical defensome of the input genomes. It goes through the following steps:

Input: annotated genomes, list of defensome genes, reference proteome

1. Filter input annotations to only keep one (longest) isoform per locus.
2. Functional annotation of the input proteomes using the [eggNOG mapper](https://github.com/eggnogdb/eggnog-mapper)
3. Ortholog prediction between input proteomes and the reference proteome using [Orthofinder](https://github.com/OrthoFinder/OrthoFinder)
4. Filter orthologous groups based on the functional annotatation of their component genes (retaining only orthogroups containing chemical defensome genes)
5. Identify pseudogenes by alignment of reference proteins to the input genomes (using Diamond, Genewise2, Exonerate, and Miniprot)
6. Summarize orthogroup information, including functional annotation and pseudogene information
7. Calculate comparative statistics of chemical defensome gene presence between different input lineages

A more detailed description of the analysis pipeline can be found in the [preprint](https://www.biorxiv.org/content/10.64898/2026.05.21.726804v1).

## Installation and setup

All necessary software can be installed in one go using conda or mamba:

```shell
conda create -n defensome -c bioconda -c conda-forge snakemake diamond agat wise2 exonerate miniprot orthofinder eggnog-mapper seqtk pandas
```

### Input data

The pipeline requires the following inputs:

- Genome sequences of query genomes (in fasta format)
- Genome annotations of query genomes (in gff, gff3, or gtf format)
- Reference proteome(s) (protein fasta file(s))
- Correspondence table linking every genome to a lineage 
- Patterns of gene names for filtering (in this case, chemical defensome genes)

The file prefix of the query genome sequences and annotation files should be the same (e.g. OrcinusOrca.fasta and OrcinusOrca.gff).
The file prefix will also be used to link the genome to a lineage in the correspondence table.
The correspondence table is a tab-separated file with the genome names in the first column, and their corresponding lineage in the second column (e.g. [here](profile/SpeciesOverview.tsv))
The patterns used for the chemical defensome genes are found in the [profile](profile) folder (`pattern.any`, `pattern.letter`, and `patter.number`).
If you want to use a set of custom gene names, see the ["Adapting the pipeline for other genes" section](##Adapting-the-pipeline-for-other-genes).

### Parameter file

The parameter file needs to be set up before launching the pipeline. A template can be found [here](profile/params.yaml).
The parameters that can be adjusted are as follows:

Input data:

- `genome_folder`: path to the folder containing the query genome files
- `genome_suffix`: suffix of the query genomes (e.g. `.fa`, `.fasta`, `_genome.fasta`)[^1]
- `annot_folder`: path to the folder containing the query genome annotations
- `annot_suffix`: suffix of the query genome annotations (e.g. `.gff`, `_annotation.gtf`)[^2]
- `ref_folder`: path to the folder containing the reference proteomes
- `ref_suffix`: suffix of the reference proteomes (e.g. `.faa`, `_proteins.fasta`)
- `genes_any`: file containing gene name patterns that can by followed by any character
- `genes_letter`: file containing gene name patterns that can only be followed by letters
- `genes_number`: file containing gene name patterns that can only be followed by numbers
- `genomeMap`: file containing mapping between each genome and their lineage

Pseudogene prediction:
- `lengthCutoff`: Cutoff on the ratio of protein lengths between the query protein and the best hit in the reference proteome(s). Genes with a length ration below this threshold will be marked as pseudogenes.
- `percID_cutoff`: Cutoff on %similarity (of dimaond blastp alignments) between the query protein and the best hit in the reference proteome(s). Genes with alignments below this %ID will be marked as pseudogenes. 
- `genePadding`: The number of basepairs to add to both sides of genomic region of the query gene for pseudogene alignment.

Software parameters
- `eggnog_evalue`: E-value for eggnog emapper for functional annotation
- `diamond_evalue`: E-value for diamond blastp

Computational paramters:
- `max_threads`: maximal number of threads to use by a single rule in the pipeline.
- `diamond_threads`: number of threads to use for diamond blastp analysis (can often be lower than the maximal number of threads, to allow multiple diamond blastp analyses to run simultaniously)

Other (optional):
- `miniprot_bin`: path to the miniprot binary, if different than the default (default = "miniprot").
 
[^1]: After removal of the suffix, only the identifying prefix should remain, which identifies the genome throughout the analysis. Genome sequences can be compressed (in `.gz` format). However, during the analysis the genomes will be temporary decompressed.
[^2]: After removal of the suffix, only the identifying prefix should remain. This prefix should match a prefix of a genome sequence. Annotation files cannot be compressed.

The settings for Snakemake itself, are defined in the [config.yaml](profile/config.yaml) file.

## Running the pipeline

Once the input data and parameter files are set up, the pipeline can be run as follows:

```shell
snakemake -c {number_of_threads} --profile {folder_containing_snakemake_config} all --configfile {parameter_file}
```

It is advised to run the pipeline using the `--dry-run` mode first, to check if everything is working.


## Output files

All output and intermediate files will be stored in the "results" folder, which will be created in the folder from which the Snakemake command was run. The "results" folder contains the following subfolders:

```
results/
├── annotation/
│   ├── {genome}.longestIso.gff #GFF-file containing only the longest isoform per locus
│   ├── {genome}.longestIso.prot.fasta #protein fasta file of only the longest isoforms per locus
│   ├── {genome}.agat.gff.log & {genome}.agat.prot.log #AGAT log files (selecting longest isoform, and extracting protein sequences)
│   └── {genome}.genome.fasta.index.dir & {genome}.genome.fasta.index.dir #Genome indexes for extracting protein sequences; can be removed)
├── blastp/
│   └── {genome}_ref.blastp #diamond blastp output of the genome agains the reference proteins
├── eggNOG/
│   ├── allGenomes.annots #eggNOG annotation for every gene
│   ├── allGenomes.emapper.annotations #eggNOG mapper annotation output
│   ├── allGenomes.emapper.hits #eggNOG mapper blastp hits
│   └── allGenomes.emapper.seed_orthologs #eggNOG mapper orthologs used for infering function
├── filtered/
│   ├── gene_aln.combo #alignment information of each chemical defensome gene with the reference gene in the same orthogroup (if applicable)[^1]
│   ├── ortho.annot #functional annotation of every chemical defensome orthogroup, and all the genes in the orthogroups
│   ├── ortho.groups #orthologous groups part of the chemical defensome, and the genes in these groups
│   └── pseudostates.tbl #overview of pseudogene status of every gene in the chemical defensome [^2]
├── orthology/
│   ├── all.ortho.annot #functional annotation of every orthogroup, and every gene in these groups
│   ├── all.ortho.groups #orthologous groups and the genes in those groups
│   ├── all.ortho.txt #orthologous groups and the genes in those groups (orthofinder output)
│   ├── proteins/ #folder containing input proteins for Orthofinder
│   └── orthofinder/ #Orthofinder output files
├── overview/
│   ├── genecounts.tbl #number of functional (= not pseudogene) genes of each species/genome in each orthogroup
│   └── stats.tsv #overview statistics (see below)
└── pseudoAlignments/
    ├── allRef.fasta #fasta file containig all reference proteins
    ├── {genome}.genelocations #genomic location of every gene in the genome
    └── {genome}/ #folder containing pseudogene aligments
         └── {query_gene}_{reference_gene}.{method} #protein-to-genome aligment of reference genes to the genomic locus of the query gene, using a certain method (genewise2 (gw2), exonerate (exo), miniprot (mnp))

[^1]Columns: query gene - orthogroup - orthologous reference gene - length ratio between query and reference protein - % identity between query and reference protein
[^2]Columns: query gene - orthogroup - Functional/Pseudogene - orthologous reference gene - length ratio between query and reference protein - % identity between query and reference protein - detected frameshifts/internal stop codons from Genewise2, Exonerate, and Miniprot alignments
```

The "filtered" folder contains an overview of the chemical defensome genes, which orthogroup they belong to, and their pseudogene assessment.
The "overview" folder contains the main overview of gene absence/presence in certain orthogroups (~ genes/gene families) per species or per lineage.
The `stats.tsv` file is the main output file. It contains an overview of how many functional genes are present per lineage in every orthologous group. 
Two statistical test are used to determine differences in gene inventory bewteen lineages in each orthogroup. 
First, the proportions of genomes with at least one functional gene in each lineage were compared using Fisher exact tests on the corresponding contingency tables.
Second, the average number of functional gene copies in each genome per lineage was compared using non-parametric Mann-Whitney U tests.
The table reports both raw p-values and adjusted p-values (Benjamini-Hochberg, controlling FDR at 0.05).

## Adapting the pipeline for other genes

The pattern files provided in this github filter out genes and orthogroups related to the chemical defensome.
However, custom patterns can be specified to search for a specific set of gene names. 
The patterns for genes of interest are stored in 3 separate files (one pattern per line):

- Gene names followed by any character
- Gene names only followed by a letter
- Gene names only followed by a number

The patterns will be used to search for relevant genes in the functional annotation.
For example, adding CYP2 to the first file, will detect all genes beginning with CYP2, so both members of the CYP2 family (e.g. CYP2A1) and members of CYP2X familes (e.g. CYP27A1).
Adding CYP2 to the second file will only yield members of the CYP2 family, and adding it to the third file will only yield members of CYP2X families.
Note: gene patterns will only be matched to the start of the string.
For example, if the search pattern is GST, the MGST gene name will not be matched.
The pattern search is case-insensitive.

Currently there is no option to disable the gene-name filtering. 
If you want to consider all genes in the analysis, you could include all letters (A->Z), and "-" (each on a separate line) in the genes_any file.

## References and citation

This pipeline was developed for the study on the marine mammal chemical defensome, described [here](https://www.biorxiv.org/content/10.64898/2026.05.21.726804v1).

If using this pipeline in your own work, please cite both this github page and the reference manscript linked above.
Please also consider citing the software on which the pipeline is build on:

- [AGAT](https://agat.readthedocs.io/en/latest/)
- [Orthofinder](https://orthofinder.github.io/OrthoFinder/)
- [Diamond](https://github.com/bbuchfink/diamond)
- [eggNOG database](http://eggnog5.embl.de/) and the [eggNOG mapper](http://eggnog-mapper.embl.de/)
- [Genewise2](https://www.ebi.ac.uk/Tools/psa/genewise/)
- [Exonerate](http://www.biomedcentral.com/1471-2105/6/31)
- [Miniprot](https://github.com/lh3/miniprot)
- [Seqtk](https://github.com/lh3/seqtk)
