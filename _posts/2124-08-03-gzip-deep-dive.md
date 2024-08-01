## GNU Zip (gzip) Technical Deep Dive

A little bit ago I took a deep dive into the internals of gzip and by extension the BGZF (block-gzip) and Tabix indexing approaches.

I've already covered the why's of using BGZF and Tabix together in a previous post, but I also hinted at doing a deeper dive into gzip compression, which is 
what this post is about.

#### Run Length Encoding (LZ77)

find long, repetitive strings and replace with [offset, length] representation over a sliding window (dictionary)

#### Huffman Coding

replace certain blocks of data with much smaller representation based on the inverse frequency of the block (entropy)

### Combining RLE and Huffman Coding to make DEFLATE

Key idea: weaving the 2 algorithms (LZ77 + Huffman coding) together to seamlessly switch between match vectors and literal characters.

Had to fix the following diagram due to inaccurate use of Huffman coding (did *not* produce prefix-free codes) and lowered bit count:

![gzip_diagram](https://github.com/user-attachments/assets/63320ce3-a6dd-4727-8110-f59aabcace55)
`"each of the next length characters is equal to the characters exactly distance characters behind it in the uncompressed stream“ distance == offset`

sources: https://kinsta.com/blog/enable-gzip-compression/

and

https://en.wikipedia.org/wiki/LZ77_and_LZ78

![image](https://github.com/user-attachments/assets/628e601c-e051-490e-8d97-322084d2421c)

source: http://www.codersnotes.com/notes/elegance-of-deflate/

### Block-gzip Compressed Files (BGZF)

* block-gzip splits up stream into compressed blocks (64KB per block) for an increase in total compressed size (each block as a header/footer)
* blocks provide for indexing original uncompressed data via virtual offsets (64 bit integer), a packed combination of:
* offset of compressed block (48 bits)
* offset within uncompressed block of indexed record (16 bits)
* Indexing is a combination of a binning approach (like a 1D R-tree) + linear coordinate position index mapping to virtual positions
* Query over S3/HTTP 1.1 to get byte ranges, avoids downloading and decompressing entire file

#### Limits
* 48 bits => ~241 TBs compressed file size that can be indexed
* 16 bits => ~65 KBs of uncompressed block
* 64 KiB blocks are padded to that size (fixed ahead of time)


### Tabix Indexing of BGZF

![tabix_bgzip_visuallized](https://github.com/user-attachments/assets/193bd15c-7edf-4b83-a9ba-2e3af9a424dc)

source: https://www.chipestimate.com/Unzipping-the-GZIP-compression-protocol/Altior/Technical-Article/2010/03/23 

and 

Li, Heng. "Tabix: fast retrieval of sequence features from generic TAB-delimited files." Bioinformatics 27.5 (2011): 718-719.

#### Binning index for intervals (1D proximity)

![tabix_binning_index_diagram](https://github.com/user-attachments/assets/1fb805f8-5040-4a6d-a1e0-7eb6e7a3dfb2)

Store intervals (lines) in smallest bin that fully encompasses the interval:
*	Intervals covered by A are in bin 1 (and below)
*	Intervals covered by B are in bin 4 (and below)
*	Intervals covered by C are only in bin 20
  
Larger intervals can increase # of bins searched: A (8) > B (6) > C (3)

source: Kent, W. James, et al. "The human genome browser at UCSC." Genome research 12.6 (2002): 996-1006.

#### Index Jumping vs. Streaming

Don’t always use the index (can ruin cache locality):

![tabix_index_jumping_vs_streaming](https://github.com/user-attachments/assets/ebea7fd3-cae2-4ec8-8aff-c57ad4d63746)

