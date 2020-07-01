# RNA-seq processing Tutorial

Let's start playing with RNA-seq data!

In labshare, open `DATASETS/2012_RNAseq_SAN`and you'll find a directory filled with RNA-seq fastq files generated from PC and ACM cells. To reduce computing power, we are not going to analyze all of them. Instead, we are just going to look at a couple of these files. Specifically, lets look at the pairs `Embryonic_D11.5_RightAtrial_Mar7_2012_306_TGTGAA_UNPROC_L7_pair1.fq.gz` and `Embryonic_D11.5_RightAtrial_Mar7_2012_306_TGTGAA_UNPROC_L7_pair2.fq.gz` and `Embryonic_D11.5_SinoatrialNode_Mar7_2012_305_GTGTTA_UNPROC_L5_pair1.fq.gz` and `Embryonic_D11.5_SinoatrialNode_Mar7_2012_305_GTGTTA_UNPROC_L5_pair2.fq.gz`. This data is from this [paper](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4344860/). Read up on the methods if you're curious how they processed their data. Specifically:

* what adapter sequences were used?
* what aligner was used?
* how did they call reads?

## Quality Check

Open these files up with [fastqc](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/). 

We can see that there are a lot of different figures/data which fastqc reports on. Note that not all of them need to be green! For example, since we are conducting RNA-seq, we might expect some kmers in our data which would not show up by chance alone (start/stop codons, splice junctions...). Pay special attention to the length of reads and the quality scores.

Are there any adapters? Should we trim?

### Trimming

There are a lot of different tools to use for trimming, with one popular one being [trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic). If you think you should trim the data, use your software of choice to trim it! But be careful not to [trim too much](https://www.rna-seqblog.com/you-might-be-ruining-your-rna-seq-data/).

Trimmomatic is written in java, so make sure you have the latest version of java installed if you want to use it! It takes in two files, the forward and reverse of the paired-end reads, and it outputs four files (two corresponding to paired forward and reverse sequences after trimming and two corresponding to unpaired or the sequences that result when the corresponding sequence in a pair is deleted because of trimming). The commands can be unnecessarily long, and have many arguments which are needed for proper trimming. You can check all the arguments supported [here](http://www.usadellab.org/cms/uploads/supplementary/Trimmomatic/TrimmomaticManual_V0.32.pdf). Note that the order of arguments is important (adapter trimming must be first). Important arguments to consider is ILLUMINACLIP (requires a fasta file of adapter sequences), SLIDINGWINDOW (useful for triming) and MINLEN (sequences which are too small can hurt alignment and RNA-seq).

If you did trim, use fastqc to check your results. Do they look better this time?

## Alignment

Now we can start aligning. Though there are many types of alignment algorithms online, I recommend using [HISAT2](https://daehwankimlab.github.io/hisat2), because it specifically helps map alternative spliced transcripts to the correct gene. I already installed the reference genome for mm10 in labshare (`genomes/HISAT2`). 

An example usage would be...

`$HISAT2_HOME/hisat2 -x $HISAT2_HOME/example/index/22_20-21M_snp -1 $HISAT2_HOME/example/reads/reads_1.fq -2 $HISAT2_HOME/example/reads/reads_2.fq -S eg2.sam`

Where

* -x indicates the basename of the referene genome (name of the files not including .1.ht2/etc)
* -1 indicates the first of the pair reads
* -2 indicates the second of the pair reads
* -S indicates the output

Note that by default HISAT2 will output a SAM file, which will unnecessarily take up space. So we'll need to convert it to a BAM file using samtools. We can do this with a command after running HISAT2, or we can combine them into a single line command using [pipes](https://stackoverflow.com/questions/9834086/what-is-a-simple-explanation-for-how-pipes-work-in-bash). 

For example...

`hisat2 -x mm10 -1 reads_1.fq.gz -2 reads_2.fq.gz | samtools view -bS -@ 8 | samtools sort -@ 8 -m 4 -o align.bam`

Here we run hisat2 on our reads, convert it to a bam, then sort the bam file before finally outputting it to `align.bam`. There are some extra arguments here to specify how many processors to use and how much memory to allocate, which will increase the speed of the command but also depends on how powerful of a computer or server you are using.

Alright, now lets try running HISAT2 on our fastq files!

## Read Calling

Now that we have our BAM file, lets count the number of reads in the exons! To do this, lets use [featureCounts](http://bioinf.wehi.edu.au/featureCounts/), which will correct for an increase in read counts due to paired-ends. I downloaded exons for mm10 from [https://uswest.ensembl.org/Mus_musculus/Info/Index](https://uswest.ensembl.org/Mus_musculus/Info/Index) and uploaded them to labshare `genomes/exons`. Run featureCounts on all of our BAM files, then compile the results into a single file which we could run DE analysis on. 

For help, check out the examples on the site, as well as try running `featureCounts --help` to see a detailed list of every argument and flag the command takes in. Which of the examples would we be following from the website?