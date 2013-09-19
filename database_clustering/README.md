Generate srst2-compatible clustered database from raw sequences
====

srst2 can help you type allele sequences, i.e. where you have a database with multiple sequence variants (alleles) of the same gene (or for many genes) and you want to know which variant (allele) of each gene is present in each sample. Any number of sequence databases, in fasta format, can be passed to srst2 for typing using --gene_db. 

To report the results properly, srst2 needs to know which sequences are actually alleles of the same gene (or sequence "cluster"). 

This information must be provided via the fasta headers in the database in this format: >[clusterID]__[gene]__[allele]__[seqID]. Note the elements are separated by two underscores '__'. 

Additional annotation can be included after this, separated by a space. See the main readme for a more detailed description of the file format required.

----------

If you already know which alleles belong to the same gene family and should be clustered for reporting, then you can use this info to generate the appropriate headers. 

The provided script, csv_to_gene_db.py can help:

csv_to_gene_db.py -h

Usage: csv_to_gene_db.py [options]

Options:
  -h, --help            show this help message and exit
  -t TABLE_FILE, --table=TABLE_FILE
                        table to read (csv)
  -o OUTPUT_FILE, --out=OUTPUT_FILE
                        output file (fasta)
  -s SEQ_COL, --seq_col=SEQ_COL
                        column number containing sequences
  -f FASTA_FILE, --fasta=FASTA_FILE
                        fasta file to read sequences from (must specify which
                        column in the table contains the sequence names that
                        match the fasta file headers)
  -c HEADERS_COL, --headers_col=HEADERS_COL
                        column number that contains the sequence names that
                        match the fasta file headers

The input table should be csv and have these columns:

seqID,clusterid,gene,allele,(DNAseq),other....

which will be used to make headers of the required form [clusterID]__[gene]__[allele]__[seqID] [other stuff]

- If you have the sequences as a column in the table, specify which column they are in using -s:

csv_to_gene_db.py -t genes.csv -o genes.fasta -s 5

- Alternatively, if you have sequences in a separate fasta file, you can provide this file via -f. You will also need to have a column in the table that links the rows to unique sequences, specify which column this is using -c:

csv_to_gene_db.py -t genes.csv -o genes.fasta -f rawseqs.fasta -c 5

-----------

If your sequences are not already assigned to gene clusters, you can do this automatically using CD-HIT (http://weizhong-lab.ucsd.edu/cd-hit/).

1 - Run CD-HIT to cluster the sequences at 90% nucleotide identity:

cdhit-est -i rawseqs.fasta -o rawseqs_cdhit90 -d 0 > rawseqs_cdhit90.stdout

2 - Parse the cluster output and tabulate the results, check for inconsistencies between gene names and the sequence clusters, and generate individual fasta files for each cluster to facilitate further checking:

python cdhit_to_csv.py --cluster_file rawseqs_cdhit90.clstr --infasta raw_sequences.fasta --outfile rawseqs_clustered.csv

For comparing gene names to cluster assignments, this script assumes very basic nomenclature of the form gene-allele, ie a gene symbol followed by '-' followed by some more specific allele designation. E.g. adk-1, blaCTX-M-15. The full name of the gene (adk-1, blaCTX-M-15) will be stored as the allele, and the bit before the '-' will be stored as the name of the gene cluster (adk, blaCTX). This won't always give you exactly what you want, because there really are no standards for gene nomenclature! But it will work for many cases, and you can always modify the script if you need to parse names in a different way. Note though that this only affects how sensible the gene cluster nomenclature is going to be in your srst2 results, and will not affect the behaviour of the clustering (which is purely sequence based using cdhit) or srst2 (which will assign a top scoring allele per cluster, the cluster name may not be perfect but the full allele name will always be reported anyway).

3 - Convert the resulting csv table to a sequence database using:

csv_to_gene_db.py -t rawseqs_clustered.csv -o rawseqs_clustered.fasta -f rawseqs.fasta -c 4

If there are potential inconsistencies detected at step 2 above (e.g. multiple clusters for the same gene, or different gene names within the same cluster), you may like to investigate further and change some of the cluster assignments or cluster names. You may find it useful to generate neighbour joining trees for each cluster that contains >2 genes, using align_plot_tree_min3.py