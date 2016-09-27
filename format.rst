==============
Segment format
==============

A file consists of a sequence of records, starting with a header,
followed by 0 or more transaction or padding records.

Numeric values are little endian.

Each record

- starts with a 4-byte record marker

- followed by a 8-byte record length.

- ends with a 8-byte record length.

The record length includes the header and trailing length.

In the definitions below, the record lengths are implicit.

A file has an alignment size, specified in the header. Records cannot
span alignment boundaries. Padding records are added, as needed to
avoid this.

File header
-----------

Record marker: 'fs2 '

The record length is always 4096.

8-byte alignment size
  The alignment size allows quick scanning of a segment for
  transaction boundaries.  At offsets that are alignment-size multiples
  into the file, we can find either header, transaction, or padding
  beginnings.

2-byte length of previous segment path.
  If 0, then no previous path.

2048 bytes reserved for previous-segment path name.
  This space is reserved because data may be overwritten if previous
  segments are merged.

Padding
-------

Record marker: 'PPPP'

Note that padding records have 2 uses:

1. While a transaction is being written to a file, it's marked as a
   padding record.  When the file write is finshed, the marker is
   changed from 'PPPP' to 'TTTT'.  This is similar to the use of the
   status flag in the original FileStorage.

2. To pad a segment to an alignment boundary when a transaction wouldn't fit.

   Note that in this case, we don't write any padding data (e.g. null bytes).

Transactions
------------

Record marker: 'TTTT'

8-byte tid

2-byte length of user name, in bytes

2-byte length of description, in bytes

4-byte length of extension data, in bytes

4-byte number of data records

user name

description

extension data

data records

Data sub-record
---------------

Note that the data sub-records don't follow the higher-level record
format. They don't have record markers or lengths.

8-byte tid

4-byte data length.
   If 0, then this is a deletion record.

8-byte oid

8-byte previous-record file position.
  0 means this is first.
  1 means not first, but previous is in a previous segment.

4-byte offset to begining of transaction

crc32 data checksum

data

============
Index format
============

Every segment has an index file.  If the index file is lost, it can be
recomputed from the segment file.

The index file consists of a header followed by a sequence of one or
more oid/position pairs.  Each oid/position pair consists of a 8-byte
oid and an 8-byte segment (file) offset.

The header consists of:

4-byte magic string: b'fs2i'
8-byte index length
8-byte segment size in bytes
8-byte starting transaction
8-byte ending transaction
