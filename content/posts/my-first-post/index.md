+++
date = '2024-12-14T17:36:13+10:00'
title = 'Introduction to Metagenomics'
+++

I am studying bioinformatics, and through my work at [Microba](https://microba.com/), I became interested in metagenomics.

This posts serves as a basic introduction to metagenomcs for software engineers. It also introduces basic biology and genomics!

Get some data: https://www.ebi.ac.uk/ena/browser/view/SAMEA115625108?show=reads

This is brent goose faecal sample. There should be plenty of plasmids here. I want to use spades to assemble the plasmids.

Get the data using `get_data.sh`:

```sh
wget -nc ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR131/023/ERR13148223/ERR13148223_2.fastq.gz
wget -nc ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR131/023/ERR13148223/ERR13148223_1.fastq.gz
```

## Take a look

The datasets are both 2GB or so even with gzip. How many reads?

```sh
gzcat data/11299_45_S149_R1_001.fastq.gz | wc -l #=> 147402040
```

147m reads.

They are 150bp long - illumina data. We can confirm:

```sh
gzcat data/11299_45_S149_R1_001.fastq.gz | head -n 4 | awk 'NR % 4 == 2 {print length($0)}' #=> 150
```

Just look at the second entry and see the length. As a reminder, fastq is 4 lines: header, read, +, quality.

```fastq
@A00940:346:H7FY3DSXC:2:1101:2067:1000:ATCTCGTATTC 1:N:0:ATGCTCCC+CCCTGCTG
CATACAAATTGTTCTCATCATACCCAACGAAAGCAACCTTGAGAGACTTGCTCCACAAGAGGACCACAGATCGGAAGAGCACACGTCTGAACTCCAGTCACATGCTCCCATCTCGTATGCCGTCTTCTGCTTGAAAAGGTGGGGGGGGGG
+
FFFFFFFFFFFFFFFFFFFFFFFFF:FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF:FF,:F,::,,:,,,:,,,,,,:,:,,,::FFFFFFFF
```

To get some working quickly, we can make a subset of the data using `seqtk`:

```sh
seqtk sample -s100 11299_45_S149_R1_001.fastq.gz 100000 > subset/subsampled_R1.fastq
seqtk sample -s100 11299_45_S149_R2_001.fastq.gz 100000 > subset/subsampled_R2.fastq
```

Some basic QC, remove adapters, etc, with `fastp`.

```sh
./fastp -i subset/subsampled_R1.fastq \
  -I subset/subsampled_R2.fastq \
  -o R1.fastq.gz \
  -O R2.fastq.gz  \
  --disable_length_filtering \
  --cut_mean_quality 20 \
  --qualified_quality_phred 20 \
  --low_complexity_filter \
  --cut_front \
  --cut_tail
```

Finally, we run `spades.py` with the `--plasmid` flag.

```sh
spades.py --pe1-1 R1.fastq.gz --pe1-2 R2.fastq.gz -o spades_output --meta
```

The output is something like this:

```
spades_output/
|-- K21
|-- K33
|-- K55
|-- assembly_graph.fastg
|-- assembly_graph_with_scaffolds.gfa
|-- before_rr.fasta
|-- contigs.fasta
|-- contigs.paths
|-- corrected
|-- dataset.info
|-- first_pe_contigs.fasta
|-- input_dataset.yaml
|-- misc
|-- params.txt
|-- scaffolds.fasta
|-- scaffolds.paths
|-- spades.log
|-- tmp
`-- warnings.log
```

The interesting files are

- contigs.fasta: resulting contig sequences in FASTA format;
- scaffolds.fasta: resulting scaffold sequences in FASTA format;
- assembly_graph.gfa: assembly graph and scaffolds paths in GFA 1.0 format;
- assembly_graph.fastg: assembly graph in FASTG format;
- contigs.paths: paths in the assembly graph corresponding to contigs.fasta;
- scaffolds.paths: paths in the assembly graph corresponding to scaffolds.fasta;
- spades.log: file with all log messages.

I grabbed this list from the paper, [Using SPAdes De Novo Assembler](https://currentprotocols.onlinelibrary.wiley.com/doi/full/10.1002/cpbi.102).

`contigs.fasta` has 8177 sequences: `cat spades_output/contigs.fasta | grep "^>" | wc -l`. Remember, this is using the subset of our data - the full contigs list would be much longer if running against a larger dataset.

We don't know the organisms, or how many are present - there could be just 1, to whose genome all 8177 contigs belong. Or, there could be a much larger number. In all likelihood, there are many different organisms in the sample (bacteria, various types, archaea, etc).

Remember, the goal is to get a full set of sequences that represents every organism in the sample. Ideally, a full taxanomic classification. 

As a reminder, the taxanomical classifications are:

![](https://upload.wikimedia.org/wikipedia/commons/thumb/7/71/Taxonomic_Rank_Graph.svg/1280px-Taxonomic_Rank_Graph.svg.png)

This analysis is about gut bacteria, so the useful levels of taxanomic classification are generally:

- Phylum. Example:
  - Bacteroidetes
  - Firmicutes
- Genus: *Bacteroides*, *Lactobacillus*, *Clostridium*. Often combined with phylum, for example *Firmicutes Clostridium*.
- Species: Another subcategory. In *Clostridium* there is *difficile*, so you could say *Clostridium difficile*. This particular one is known to cause diarrhea.

Back to the spades output: metaSPAdes, which is the tool used when running `--meta`, is a **consensus assembly**. From the metaSPAdes paper:

> There is no universal outcome for metagenome assembly. The **ideal scenario is to obtain the complete genome sequence of each and every species** presented in the dataset. However, usually this is not possible ... to deal with this problem, metaSPAdes aims to achieve a so-called **consensus assembly by detecting and masking variations between seemingly related strains**.

So, even with a lot of data and the contigs, we may loose some granularity because within each species, there can be different strains (where a single species. For example: several different **Clostridium difficile** variations - a single base can make a difference!

Either way, now we have a lot of contigs, we can attempt to sort them out. This is called **binning**, where we group the contigs based on ones that look like they belong together. The perfect outcome would be species level bins, but the goal here is **putting the ones that are closely related together**.

