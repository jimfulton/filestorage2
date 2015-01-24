==================================
ZODB FileStorage second generation
==================================

Background
==========

The original ZODB FileStorage design was extremely simple. A
FileStorage is essentially a log with an index.  Almost all
operations are file appends.  This design was meant to be very simple
and, hopefully, fool proof.  It has worked out extremely well, and far
better than I expected in most respects. File storages are extremely
compact and do relatively little I/O.

The primary disadvantages of file storages are:

- in-memory indexes

- high cost of performing packs.

Overview
========

Filestorage2 is designed to address these issues.

In filestorage2, a file storage consists of a collection of segments,
consisting of an active segment and previous segments.  The active
segment has an in-memory index.  Each previous segment has an index
that can be mapped (mmap) into memory. Each segment has a reference to
the previous segment (if any).  A previous segment is only modified if
its previous segment changes.  Segments may span file-system
partitions.

A storage object in memory has a sequence of indexes, an in memory
index for the active segment, and 0 or more memory-mapped indexes for
previous segments.  When searching for the location of an object record,
it searches the active index and then the previous-segment indexes in
reverse order.  When previous segments are merged, an API call is used to
notify the storage that it has to reassemble it's indexes.

Indexes are stored in separate files alongside each segment. If an
index is lost, it can be reconstructed from segment data.

Packing is performed by merging 2 or more past segments.  After
merging past segments, old past segments are suitable for archival.

The most recent past segment is created by creating a new active
segment and converting the old active segment to a past segment.  This
creating a new in-memory index for the new segment. In the background,
the old in-memory index is written to disk.  The disk index is then
mapped into memory and the memory-mapped index replaces the old
in-memory index.

Filestorage2 doesn't provide garbage collection. Rather, it relies on
external garbage collectors to determine which objects are garbage and
to issue delete requests.

Performance improvements
========================

Reduced memory footprint for index
----------------------------------

Most index data are in memory mapped files.  Only the active index is
in normal memory.  When packing, we don't have to create temporary
in-memory indexes. We can simply merge existing memory-mapped indexes.

Reduced packing load
--------------------

In the original FileStorage, packing was performed internally. CPU
competed with normal storage requests for CPU resources.
zc.FileStorage moved most of the packing work to an external process,
which removed load from the main pack process for most of the packing
operation. (It made other improvements to the pack operation as well.)
The last phase of packing with both the original packer and with
zc.FileStorage involved copying the most recent records to the new
file as application were writing them.  This typically resulted in
temporary a loss of availability that could be quite noticeable for
large databases.

With filestorage2, creating a new active segment is very cheap.  We
don't have to copy recent records to it.  We can merge past segments
at our leisure.

Disk I/O
--------

In this design, merging segments still involved a lot of disk I/O to
copy data.  We can mitigate this in a number of ways, such as merging
gradually and continuously, to avoid contention with normal storage
requests.  We can use fadvise, as zc.FileStorage does, to avoid
wrecking the disk cache while copying.

Another option is to merge segments less frequently.  Without other
changes, this would have the downside of causing the storage to search
more indexes, but this might be acceptable.  We could mitigate this by
merging indexes in some fashion.  We could, for example, create a
redundant merged index that mapped oids to both a segment identifier
and a in-segment file offset.

A variation on the above idea would be to have each previous segment's
index contain information about all prior previous segments. Then the
storage object would only need to keep the active index and a small
number of previous indexes.

