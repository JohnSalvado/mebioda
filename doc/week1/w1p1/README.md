Week 1 Data analysis of mycorrhizal diversity
---------------------------------------------

- **Step-by-step analysis of micorrhizal diversity**

1. Create a directory where you will store the files to use for the analyses, you can call it e.g. w1p1.
   Place the reads and the reference in the directory. Extract them: `gunzip *.gz`

2. **Cutadapt**

- Python package to trim the bad quality ends of the reads and the primers.

 [cutadapt.read.docs](https://cutadapt.readthedocs.io/en/v1.10/installation.html)     
   
- **Installation**     
        
        pip install --user --upgrade cutadapt

3. **Trimming the bad quality ends**

```bash
mkdir out_trimmed
for fq in *.fastq; do
  cutadapt -q 20,20 -o out_trimmed/"${fq%.fastq}_trimmed_ends.fastq" ${fq}
done
```

4. **Trimming the 3' primer**

```bash
for fq in out_trimmed/*.fastq; do 
  cutadapt -a TCCTCCGCTTATTGATAGC -o "${fq/_ends/_primers}" ${fq} 
done
```

5. **Trimming the 5' primer**

```bash
for fq in out_trimmed/*_trimmed_primers.fastq; do 
  cutadapt -g GTGARTCATCGAATCTTTG -o "${fq/_primers/_primers2}" ${fq}
done
```

 Vsearch
 ------------
 Rognes et al., 2016. 
 
 [Vsearch github repository](https://github.com/torognes/vsearch)
 
 We have now the trimmed fastq file and we can proceed with the next steps:
 
1. **Quality filtering**

```bash   
for fq in out_trimmed/*_trimmed_primers2.fastq; do
  parameters="--fastq_qmax 46 --fastq_maxee_rate 1 --fastq_trunclen 200"
  vsearch $parameters --fastq_filter ${fq} --fastaout "${fq%.fastq}.fa"
done
```

- `--fastq_qmax`: specify the maximum quality score accepted when reading FASTQ files. 
- `--fastq_maxee_rate 1`: we specify the expected error per base: 1 
- `--fastq_trunclen 200`, all the sequences will be truncated at 200 bp; 

2. **Dereplicating at sample level and relabel with sample_n**

```bash
cd out_trimmed
for fq in *_trimmed_primers2.fa; do
  vsearch --derep_fulllength $fq --output ${fq/trimmed_primers2/uniques} --relabel ${fq/trimmed_primers2/seq} --sizeout --minsize 2
done 
```

- `--derep_fulllength`: Merge strictly identical sequences contained in filename.   
- `--sizeout`: the number of occurrences (i.e. abundance) of each sequence is indicated 
  at the end of their fasta header.
- `--relabel`: sequence renaming is done to add sample prefixes to the sequence identifiers, 
  so that we can later merge the sequences
        

3.  **Merging all the samples into one file**

```bash
rm *_ends.fastq *_primers.fastq *_primers2.fastq *_primers2.fa
cat *uniques.fa* > CPuniques.fasta
```

<!--
4. **Dereplicating across samples and remove singletons**

```bash
vsearch \
--derep_fulllength CPuniques.fasta \
--minuniquesize 2 \
--sizein \
--sizeout \
--fasta_width 0 \
--uc all.derep.uc \
--output CPuniq_no_sing.fasta
```
-->

4. **Clustering at 97% before chimera detection**

```bash
vsearch \
--cluster_size CPuniques.fasta \
--id 0.97 \
--strand plus \
--sizein \
--sizeout \
--fasta_width 0 \
--uc CP_otus.uc \
--relabel OTU_ \
--centroids CP.otus.fasta \
--otutabout CP.otutab.txt
```

- `--cluster_size`: sorts sequences by decreasing abundance before
    clustering.
- `--id`: reject the sequence match if the pairwise identity is lower than 0.97
- `--fasta_width`: Fasta files produced by vsearch are wrapped (sequences are written on lines of integer
    nucleotides, 80 by default). To eliminate the wrapping, the value is set to zero.
- `--centroids`: The centroid is the sequence that seeded the cluster (i.e. the first sequence of the cluster).     
- `--otutabout`: Output an OTU table in the classic tab-separated plain text format as a matrix containing
the abundances of the OTUs in the different samples.
 

5. **De novo chimera detection**

```bash
vsearch \
--uchime_denovo CP.otus.fasta \
--sizein \
--sizeout \
--fasta_width 0 \
--nonchimeras CP.otus.nonchim.fasta 
```

--uchime_denovo: Detect chimeras present in the fasta-formatted filename, without external references (i.e. de novo)       

6. **Comparing target sequences**

```bash
vsearch \
--usearch_global CP.otus.nonchim.fasta \
-db ../utax_reference_dataset_10.10.2017.fasta \
-id 0.7 \
-blast6out CPotus.m8 \
-strand both \
-maxaccepts 1 \
-maxrejects 256
```

- `--usearch_global`: Compare target sequences (--db) to the fasta-formatted query sequences contained in
filename, using global pairwise alignment.

Part 2
------
We now have the all the output files that we need. However, the `CPotus.m8` needs to be
processed in the following way:

- The fourth column contains the sequence length. We only want sequences longer than
  150 bp
- The first column contains both the OTU id and the cluster size, like this: 
  `OTU_1;size=9159;`. We just want to keep the `OTU_1` part, so before the first 
  semicolon.
- For each distinct OTU id, we want to keep the match with the highest percentage (there
  might be multiple matches for the same OTU).

This [script](filter.pl) performs these operations. It is executed like this:

    perl ../filter.pl CPotus.m8 > CPotus.uniq.tsv

We need now to _join_ the OTU-to-taxonomy match table with the OTU-by-sample table 
("_in the classic tab-separated plain text format as a matrix containing the abundances 
of the OTUs in the different samples_")

    join CPotus.uniq.tsv CP.otutab.txt > CPotus.samples.tsv
 
**Open the file we just generated in LibreOffice Calc.**

After the OTUs' abundance values you can calculate the sum, e.g `=SUM(a1:g1)`

Next to the sum we are going to write an IF condition that will return the value 1 if the 
abundance value is greater than 0, otherwise it will return 0. `=IF(e1>0,1,0)`

Finally, we want the sum of each absence-presence column, so at the end of each column 
calculate the sum.

