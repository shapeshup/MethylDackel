PileOMeth (a temporary name derived due to it using a PILEup to extract METHylation metrics) will process a coordinate-sorted and indexed BAM or CRAM file containing some form of BS-seq alignments and extract per-base methylation metrics from them. PileOMeth requires an indexed fasta file containing the reference genome as well.

Prerequisites
=============

PileOMeth requires HTSlib version 1.1 or newer (version 1.0 is currently unsupported). As HTSlib is a submodule of this repository, this prerequisite should be automatically dealt with.

Compilation
===========

Compilation and installation can be performed via:

    git clone https://github.com/dpryan79/PileOMeth.git
    cd PileOMeth
    git submodule init
    git submodule update
    make
    make install prefix=/some/installation/path

As HTSlib is now a submodule of this repository, you no longer need to manually download and compile it.

Usage
=====

The most basic usage of PileOMeth is as follows:

    PileOMeth extract reference_genome.fa alignments.bam

This will calculate per-base CpG metrics and output them to `alignments_CpG.bedGraph`, which is a standard bedGraph file with column 4 being the number of reads/read pairs with evidence for a methylated C at a given position and column 5 the equivalent for an unmethylated C. An alternate output filename prefix can be specified with the `-o some_new_prefix` option.

By default, PileOMeth will only calculate CpG metrics, but CHG and CHH metrics are supported as well (see the --CHH and --CHG options). If you would like to ignore CpG, metrics, simply specify --noCpG. Each type of metric is output to a different file.

PileOMeth can filter reads and bases according to MAPQ and Phred score, respectively. The default minimums are MAPQ >= 10 and Phred >= 5, though these can be changed with the -q and -p options. PileOMeth can also account for methylation bias (described below) with the `--OT`, `--OB`, `--CTOT`, and `--CTOB` options.

Output
======

The `PileOMeth extract` produces a variant of [bedGraph](http://genome.ucsc.edu/goldenpath/help/bedgraph.html) that similar to the "coverage" file produced by [Bismark](http://www.bioinformatics.babraham.ac.uk/projects/bismark/) and [Bison](https://github.com/dpryan79/bison). In short, each line consists of 6 tab separated columns:

1. The chromosome/contig/scaffold name
2. The start coordinate
3. The end coordinate
4. The methylation percentage rounded to an integer
5. The number of alignments/pairs reporting methylated bases
6. The number of alignments/pairs reporting unmethylated bases

All coordinates are 0-based half open, which conforms to the bedGraph definition. When paired-end reads are aligned, it can often occur that their alignments overlap. In such cases, PileOMeth will not count both reads of the pair in its output as doing so would lead to incorrect downstream statistical results.

An example of the output is below:

    track type="bedGraph" description="SRR1182519.sorted CpG methylation levels"
    1	25115	25116	100	3	0
    1	29336	29337	50	1	1

Note the header line, which starts with "track". This is optional for bedGraph files but produced by PileOMeth. The "description" field is used as a label in programs such as [IGV](http://www.broadinstitute.org/igv/). Each of the subsequent lines describe single Cytosines, the 25116th and 29337th base on chromosome 1, respectively. The first position has 3 alignments indicating methylation and 0 unmethylation (100% methylation) and the second position 1 alignment each supporting methylation and unmethylation (50% methylation).

Methylation bias plotting and correction
========================================

In an ideal experiment, we expect that the probability of observing a methylated C is constant across the length of any given read. In practice, however, there are often increases/decreases in observed methylation rate at the ends of reads and/or more global changes. These are termed methylation bias and including such regions in the extracted methylation metrics will result in noisier and less accurate data. For this reason, users are strongly encouraged to make a methylation bias plot. PileOMeth comes with a function for just this purpose:

    PileOMeth mbias reference_genome.fa alignments.sorted.bam output_prefix

That command will create a methylation bias (mbias for short) plot for each of the strands for which there are valid alignments. The command can take almost all of the same options as `PileOMeth extract`, so if you're interested in looking at only a single region or only CHH and CHG metrics then you can do that (run `PileOMeth mbias-h` for the full list of options). The resulting mbias graphs are in SVG format and can be viewed in most modern web browsers:

![An example SVG image](https://rawgit.com/dpryan79/PileOMeth/master/example_OT.svg)

If you have paired-end data, both reads in the pair will be shown separately, as is the case above. The program will suggest regions for inclusion ("--OT 2,0,0,98" above) and mark them on the plot, if applicable. The format of this output is described in `PileOMeth -h`. These suggestions should not be accepted blindly; users are strongly encouraged to have a look for themselves and tweak the actual bounds as appropriate. The lines indicate the average methylation percentage at a given position and the shaded regions the 99.9% confidence interval around it. This is useful in gauging how many methylation calls a given position has relative to its neighbors. Note the spike in methylation at the end of read #2 and the corresponding dip at the beginning of read #1. This is common and these regions can be ignored with the suggested trimming bounds. Note also that the numbers refer to the first and last base that should be included during methylation extraction, not the last and first base to ignore!.

To do list
==========

 - [ ] Is the output format the most convenient (it's what Bison uses, so converters have already been written)? It makes more sense to output a predefined VCF format, which would allow processing multiple samples at once. This would require a spec., which should have pretty broad input.
