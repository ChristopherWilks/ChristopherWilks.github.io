## The genius of BGZF and tabix for single dimensional indexing

As a computational biology data scientist (that is a mouthful!), I work extensively with large(ish) genomics and related datasets.
Much of the time, though not always, these datasets will have a one or more columns that have a genomic region specified, typically in chromosomal coordinates,
e.g. `chr2:4500-4600`).  This is quite common in general in genomic data science.  Thus there has been a lot of work around optmizing software to store and
allow for efficient retrieval of genomic entities (e.g. a gene) based on its location on the genome.

One of the more popular (and older, those go together) software tools is the well-attested [HTSLib](https://github.com/samtools/htslib) suite of software tools.
Technically this only includes the HTSLib library, but for the sake of completeness I'll mention [Samtools](https://github.com/samtools/samtools) and [bcftools](https://github.com/samtools/bcftools) here as well.

While these tools have many uses---much like a multi-tool of bioinformatics---I'm going to focus on one specific but core aspect that is shared but all of these tools.
Specifically, the compression and related indexing approach they all use leveraging the BGZF "standard" and the binning index used by tabix on top of that compression.

Before I dig into the details, let me say, that I use these tools almost every day in my work, and extensively used them in my PhD.
This is due not only to the obvious utility of having a region-based, highly flexible (tab-delimited) format that is both compressible and indexed, but 
also due to the fact that this can be easily extended to do single-column non-region indexing.  
Therefore avoiding the need for the higher cost of moving to a relational database system mainly for its indexing capabilities, which I'm usually loath to do.

BGZF (block gzip) + Tabix:
![image](https://github.com/user-attachments/assets/59c1293b-1a67-4217-8238-ec82f61a3f6a)

source: https://journals.plos.org/ploscompbiol/article/figure?id=10.1371/journal.pcbi.1009524.g001

### Block GNU Zip Format (BGZF)

Leveraging the almost ubiquitous gzip program, Bob Handsaker and Heng Li designed the block gzip extension, which allowed for [indexing](http://www.htslib.org/doc/#file-formats).

`gzip` itself was a front end on the [zlib](https://www.zlib.net/) library which in turn was/is an implementation of the [DEFLATE](https://www.zlib.net/feldspar.html) compression algorithm as [implemented](https://www.gzip.org/) by Jean-loup Gailly and Mark Adler, the latter went on to code the highly useful [pigz](https://zlib.net/pigz/) parallel compression version of gzip.

If you want to keep going, `DEFLATE` is really the clever combination of *two* compression algorithms, run length encoding (RLE) via [LZ77](https://en.wikipedia.org/wiki/LZ77_and_LZ78) and [Huffman coding](https://en.wikipedia.org/wiki/Huffman_coding), which uses a dictionary.  I may create another post where I dive into the details of this further, but suffice it to say `DEFLATE` is very popular, though somewhat dated now.

Original gzip assumed a continuous stream of compressed content, disallowing for any breaks and thus not able to support random access on the compressed stream (no indexing).  However, BGZF changed this to instead encode the original stream as a series of smaller gzip files or blocks (with both gzip headers and footers) where each gzip file or block would be a set size (64 KiB or smaller).  This could slightly increase the size of the final gzipped file, but the marginal extra size is offset (no pun intended) by the ability to index and therefore random access different parts of the gzipped file.

### Indexing via Tabix

To index the BGZFped file, the file *must* be sorted by one or more columns that you're going to index on *and* must be delimited in some consistent way (usually via tabs, hence the name "tabix").  The original intention was to sort by chromosome (e.g. first tab-delimited column in the file) and start coordinate, (e.g. 2nd tab-delimited column in the file), e.g. `LC_ALL=C sort -k1,1 -k2,2n file.tsv | bgzip > file.tsv.gz`, then index on those columns for a region-based lookup, e.g. `tabix -s1 -b2 -e2 file.tsv.gz`.

I'll save the details of the indexing for another post, but it's important to note that the size of the index is almost always tiny in comparison with the size of the compressed file (much less the uncompressed file).  I've seen the index grow to only a few megabytes (3.2 MBs) for even massive BGZF compressed files (2.8 TBs compressed size).  This is mainly due to the fact that the index is not tracking every row/record in the original file but only the start/end of chunks of rows/records in the original file.  See section 5 of the [SAMv1 format specification](http://samtools.github.io/hts-specs/SAMv1.pdf) which has the most detailed description of how this kind of indexing works (though in that case applied to BAM files).

Retrieval of records from a tabix-indexed, BGZFped file is greatly enhanced by the fact that *only* the compressed blocks (remember 64 KiB or less in size) that match a query need to be read and decompressed, which in a random access scenario of a few point queries is usually much smaller than decompressing the whole file.  This means substantially faster performance for individual record lookups than trying to scan the full file.

That said, if one's use case is to pull large numbers of records from most of the file anyway, you're probably better off decompressing the full file and linearly scanning it for the records you want.  This is the "streaming" mode of [tabix](http://www.htslib.org/doc/tabix.html) rather than the "index-jumping" I'm outlining above (`-T` vs. `-R` tabix options).

Example command line of the tabix default, genomic-coordinate based query:
`tabix file.tsv.gz chr3:1-10000`

### Cloud Object Store Support

Initially, `bgzip` and `tabix` only supported [POSIX filesystems](https://pubs.opengroup.org/onlinepubs/9699919799.2018edition/), e.g. `EXT4` or equivalent.  However, as the "cloud" (AWS, Google, MS) grew in popularity, the hardworking maintainers of HTSlib added support for [object stores](https://en.wikipedia.org/wiki/Object_storage) beyond just POSIX filesystems.  This is a *huge* enhancement for those that use the cloud regularly to store and retrieve information (e.g. putting most of your files on S3).  Instead of needing to maintain large, expensive POSIX compliant filesystems via EBS, EFS, or FSx (in AWS), we can now store our BGZFped, tabix indexed files in object stores for potentially subantial cost savings while getting the benefit of random access querying!

The additional technology that makes this work is HTTP 1.1 byte-range access support (or the S3/GS equivalent).  The HTTP 1.1 [specification](https://httpwg.org/specs/rfc7233.html#rfc.section.2.1) allows for parts of a file hosted on a HTTP(S) web server to be requested via a byte-range parameter in the request, rather than having to download the full file and then access just the bytes you wanted.  This of course saves on unncessary downloading of bytes in general, but also plays very well with the existing BGZF/tabix approach to indexing.

The index is first queried to get the actual byte-range(s) needed from the BGZFped compressed file stored on the object store.
Just these bytes are then downloaded and decompressed on the client side, saving both time and bandwidth.

Example of using tabix on a BGZFped file stored in AWS S3 via a `Bash` command line:

```
export AWS_ACCESS_KEY_ID=<....>
export AWS_SECRET_ACCESS_KEY=<...>
export AWS_SESSION_ID=<....> #optional
tabix -D s3://bucketname/file.tsv.gz chr3:1-10000
```

The `-D` disallows tabix from downloading the index file locally (avoid caching), even though small, this can lead to problems if the remote compressed file changes and the local tabix continues to use the locally cached index file.

Also, just to be clear, like any AWS access, the account referenced by the environmental variables in the command block above needs to have at least read access to that S3 location.  You can test this using the AWS CLI:

```
aws s3 cp s3://bucketname/file.tsv.gz - | zcat | head
```

### Extending tabix indexing to non-genomic-coordinate use cases

So now, with the basic technology in place, we can talk about extending the approach to use with non-genomic-intervals as the index column.
First, it may be obvious already, but BGZF will compress anything, it doesn't require a strict format adherence.
Second, similarly, tabix while requiring tab-delimited files, only requires 2 columns for its indexing, a column assumed to be typed as a string (the chromosome identifier) and a column assumed to be typed as an integer (the coordinate column).  These columns also must be sorted appropriately as mentioned above.  Beyond that, tabix doesn't check what's in those columns.

We can leverage this to our benefit by simply adding an extra integer column (say of all "1"s) to any file we want to index an existing single column of values in, e.g. gene names and then sort by the gene names.  Then we just compress via BGZF and index using those two columns as in:

```
tabix -s<string_column_we_want_to_index_by_originally> -b<added_stub_integer_column_of_all_1s> -e<added_stub_integer_column_of_all_1s> file.tsv.gz
```

Boom, we have a single-column index on an arbitrary column type for a compressed file that can be queried from S3/GS!

What this allows us to do is sidestep moving to a more expensive option (e.g. a relational database system) mainly to index on a single column.

Now to be clear, there are still cases where having a relational database is the better approach, but in most of the data scenarios I work with day-to-day, this approach is sufficient.
