ZODB FileStorage second generation
==================================

The original ZODB FileStorage design was extremely simple. A
FileStorage is essentially a log with an index.  Allmost all
operations are esentially file appends.  This design was meant to be
very simpler and, hopefully, fool proof.  It has worked out extremely
well, and far better than I expected inb most respects. File storagess
are extremely compact and to relatively little I/O.

The primary disadvantages of file storages are:

- in-memory indexes

- high cost of performing packs.

Filestorage2 is designed to address these issues.

In filestorage 2, a file stoage consists of a collection segments,
consisting of active segments and previous segments.  These active
segment has an in-memory index.  Each past segment has an index that
can be mapped into memory. Each segment has a reference to the
previous segment (if any).  A past segment is only modified if its
previous segment changes.  Segments may span file-system partitions.

A storage object in memory has a sequence of indexes, an in memory
index for the active segment, and o or more memory-mapped indexes for
past segments.  When searching for the location of an object record,
it searches the active index and then the past-segment indexes in
reverse order.  When past segments are merged, an api call is used to notify the storage that it has to reassemble it's indexes.

Packing is performed by merging 2 or more past segments.  After
merging past segments, old past segments are suitable for archival.

The most recent past segment is created by creating a new active
segment and converting the old active segment to a past segment, by
writing it's in-memory index to the end of the file.

When merging past segments, we can avoid keeping index data in memory
by leveraging sparse files, the fact that we know an upper limit to
the file, and we know what keys will be in the merged in indexes from
analysis of the keys in the old indexes.  We can preallocare the new
index and filling it in as we copy data.

Filestorage2 doesn't provide garbage collection. Rather, it relies on
external garbage colectors to determine which objects are garbage and
to issue delete requests.
