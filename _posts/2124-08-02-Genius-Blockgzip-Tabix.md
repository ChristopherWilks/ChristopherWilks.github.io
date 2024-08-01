## The genius of block-gzip and Tabix for single dimensional indexing

As a computational biology data scientist (that is a mouthful!), I work extensively with large(ish) genomics and related datasets.
Much of the time, though not always, these datasets will have a one or more columns that have a genomic region specified, typically in chromosomal coordinates,
e.g. `chr2:4500-4600`).  This is quite common in general in genomic data science.  Thus there has been a lot of work around optmizing software to store and
allow for efficient retrieval of genomic entities (e.g. a gene) based on its location on the genome.

One of the more popular (and older, those go together) software tools is the well-attested HTSLib suite of software tools.
Technically this only includes the HTSLib library, but for the sake of completeness I'll include Samtools and bcftools here as well.

While these tools have many uses---much like a multi-tool of bioinformatics---I'm going to focus on one specific but core aspect that is shared but all of these tools.
Specifically, the compression and related indexing approach they all use leveraging the block-gzip "standard" and the binning index used by Tabix on top of that compression.

Before I dig into the details, let me say, that I use these tools almost every day in my work, and extensively used them in my PhD.
This is due not only to the obvious utility of having a region-based, highly flexible (tab-delimited) format that is both compressible and indexed, but 
also due to the fact that this can be easily extended to do single-column non-region indexing.  
Therefore avoiding the need for the higher cost of moving to a relational database system mainly for its indexing capabilities, which I'm usually loath to do.
