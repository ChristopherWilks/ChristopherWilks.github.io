## The genius of block-gzip and tabix for single dimensional indexing

As a computational biology data scientist (that is a mouthful!), I work extensively with large(ish) genomics and related datasets.
Much of the time, though not always, these datasets will have a one or more columns that have a genomic region specified, typically in chromosomal coordinates,
e.g. `chr2:4500-4600`).  This is quite common in general in genomic data science.  Thus there has been a lot of work around optmizing software to store and
allow for efficient retrieval of genomic entities (e.g. a gene) based on its location on the genome.

One of the more popular (and older, those go together) software tools is the well-attested [HTSLib](https://github.com/samtools/htslib) suite of software tools.
Technically this only includes the HTSLib library, but for the sake of completeness I'll mention [Samtools](https://github.com/samtools/samtools) and [bcftools](https://github.com/samtools/bcftools) here as well.

While these tools have many uses---much like a multi-tool of bioinformatics---I'm going to focus on one specific but core aspect that is shared but all of these tools.
Specifically, the compression and related indexing approach they all use leveraging the block-gzip "standard" and the binning index used by tabix on top of that compression.

Before I dig into the details, let me say, that I use these tools almost every day in my work, and extensively used them in my PhD.
This is due not only to the obvious utility of having a region-based, highly flexible (tab-delimited) format that is both compressible and indexed, but 
also due to the fact that this can be easily extended to do single-column non-region indexing.  
Therefore avoiding the need for the higher cost of moving to a relational database system mainly for its indexing capabilities, which I'm usually loath to do.

### Block gzip

Leveraging the almost ubiquitous gzip program, Bob Handsaker and Heng Li designed the block GZip extension, which allowed for [indexing](http://www.htslib.org/doc/#file-formats).

`gzip` itself was a front end on the [zlib](https://www.zlib.net/) library which in turn was/is an implementation of the [DEFLATE](https://www.zlib.net/feldspar.html) compression algorithm as [implemented](https://www.gzip.org/) by Jean-loup Gailly and Mark Adler, who also went on to code the highly useful [pigz](https://zlib.net/pigz/) parallel compression version of gzip.

If you want to keep going, `DEFLATE` is really the clever combination of *two* compression algorithms, run length encoding (RLE) via [LZ77](https://en.wikipedia.org/wiki/LZ77_and_LZ78) and [Huffman coding](https://en.wikipedia.org/wiki/Huffman_coding), which uses a dictionary.  I may create another post where I dive into the details of this further, but suffice it to say `DEFLATE` is very popular, though somewhat dated now.

Original gzip assumed a continuous stream of compressed content, disallowing for any breaks and thus not able to support random access on the compressed stream (no indexing).  However, block-gzip changed this to instead encode the original stream as a series of smaller gzip files or blocks (with both gzip headers and footers) where each gzip file or block would be a set size (64KB or smaller).  This could slightly increase the size of the final gzipped file, but the marginal extra size is offset (no pun intended) by the ability to index and therefore random access different parts of the gzipped file.

### Indexing via Tabix

To index the block-gzipped file, the file *must* be sorted by one or more columns that you're going to index on *and* must be delimited in some consistent way (usually via tabs, hence the name "tabix").  The original intention was to sort by chromosome (e.g. first tab-delimited column in the file) and start coordinate, (e.g. 2nd tab-delimited column in the file), e.g. `LC_ALL=C sort -k1,1 -k2,2n file.tsv | bgzip > file.tsv.gz`, then index on those columns for a region-based lookup, e.g. `tabix -s1 -b2 -e2 file.tsv.gz`.

I'll save the details of the indexing for another post, but it's important to note that the size of the index is almost always tiny in comparison with the size of the compressed file (much less the uncompressed file).  I've seen the index grow to only a few megabytes for even massive block-gzipped files (2.8TBs).  This is mainly due to the fact that the index is not tracking every row/record in the original file but only certain ones similar to the key-frame idea in video encoding [need to double check this].

Retrieval of records from a Tabix-indexed, block-gzipped file is greatly enhanced by the fact that *only* the compressed blocks (remember 64KB or less in size) that match a query need to be read and decompressed, which in a random access scenario of a few point queries is usually much smaller than decompressing the whole file.  This means substantially faster performance for individual record lookups than trying to scan the full file.

That said, if one's use case is to pull large numbers of records from most of the file anyway, you're probably better off decompressing the full file and linearly scanning it for the records you want.  This is the "streaming" mode of [tabix](http://www.htslib.org/doc/tabix.html) rather than the "index-jumping" I'm outlining above (`-T` vs. `-R` tabix options).

### Cloud Object Store Support

Initially, `bgzip` and `tabix` only supported [POSIX filesystems](https://pubs.opengroup.org/onlinepubs/9699919799.2018edition/), e.g. `EXT4` or equivalent.  However, as the "cloud" (AWS, Google, MS) grew in popularity, the hardworking maintainers of HTSlib added support for [object stores](https://en.wikipedia.org/wiki/Object_storage) beyond just POSIX filesystems.  This is a *huge* enhancement for those that use the cloud regularly to store and retrieve information (e.g. putting most of your files on S3).  Instead of needing to maintain large, expensive POSIX compliant filesystems via EBS, EFS, or FSx (in AWS), we can now store our block-gzipped, tabix indexed files in object stores for potentially subantial cost savings while getting the benefit of random access querying!

The additional technology that makes this work is HTTP 1.1 byte-range access support (or the S3/GS equivalent).  The HTTP 1.1 [specification](https://httpwg.org/specs/rfc7233.html#rfc.section.2.1) allows for parts of a file hosted on a HTTP(S) web server to be requested via a byte-range parameter in the request, rather than having to download the full file and then access just the bytes you wanted.  This of course saves on unncessary downloading of bytes in general, but also plays very well with the existing block-gzip/tabix approach to indexing.

The index is first queried to get tha actual byte-range(s) needed from the block-gzipped compressed file stored on the object store.
Just these bytes are then downloaded and decompressed on the client side, saving both time and bandwidth.

### Extending tabix indexing to non-genomic-coordinate use cases

So now, with the basic technology in place, we can talk about extending the approach to use with non-genomic-intervals as the index column.
First, it may be obvious already, but block-gzip will compress anything, it doesn't require a strict format adherence.
Second, similarly, tabix while requiring tab-delimited files, only requires 2 columns for its indexing, a column assumed to be typed as a string (the chromosome identifier) and a column assumed to be typed as an integer (the coordinate column).  These columns also must be sorted appropriately as mentioned above.  Beyond that, tabix doesn't check what's in those columns.

We can leverage this to our benefit by simply adding an extra integer column (say of all "1"s) to any file we want to index an existing single column of values in, e.g. gene names and then sort by the gene names.  Then we just compress via block-gzip and index using those two columns as in:

```
tabix -s<string_column_we_want_to_index_by_originally> -b<added_stub_integer_column_of_all_1s> -e<added_stub_integer_column_of_all_1s> file.tsv.gz
```

Boom, we have a single-column index on an arbitrary column type for a compressed file that can be queried from S3/GS!

What this allows us to do is sidestep moving to a more expensive option (e.g. a relational database system) mainly to index on a single column.

Now to be clear, there are still cases where having a relational database is the better approach, but in most of the data scenarios I work with day-to-day, this approach is sufficient.
