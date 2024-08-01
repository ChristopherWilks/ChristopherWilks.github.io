## GNU Zip (gzip) Technical Deep Dive

A little bit ago I took a deep dive into the internals of gzip and by extension the BGZF (block-gzip) and Tabix indexing approaches.

I've already covered the why's of using BGZF and Tabix together in a previous post, but I also hinted at doing a deeper dive into gzip compression, which is 
what this post is about.

### Run Length Encoding (LZ77)

### Huffman Coding

### Combining RLE and Huffman Coding to make DEFLATE

### Block-gzip Compressed Files (BGZF)

### Tabix Indexing of BGZF

