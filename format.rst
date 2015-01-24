==============
Segment format
==============

Units below are either bytes or 4-byte words, or 4-word super words.

A file consists of a sequence of records, starting with a header,
followed by 0 or more transaction or padding records.

Each record

- starts with a 1-word record marker

- followed by a 1-word record length, in super-words.

- ends with a 1-word record length, in super-words.

- align on super-word boundaries (and thus may be null padded).

In the definitions below, the trailing record lengths and padding for
word alignment are implicit.

A file has an alignment size, specified in the header. Records cannot
span alignment boundaries. Padding records are added, as needed to
avoid this.

File header
-----------

'fs2 '
  header marker/File magic number.

1-word header length, in super words
  This is always 256 and can be inspected to determine endianness.
  (Generally, numbers are little endian, but we may support either,
  with conversion, in the future.)

1-byte base-2 log of the alignment size (super words)
  The alignment size allows quick scanning of a segment for
  transaction boundaries.  At offsets that are alignment-size multiples
  into the file, we can find either header, transaction, or padding
  beginnings.  For example, a value of 28 provides an 4GB alignment
  size.

  Note that because transactions are not allowed to span alignment
  boundaries, a value less than 32 reduces the maximum transaction size.

2-byte length of previous segment path.
  If 0, then no previous path.

4081 bytes reserved for previous-segment path name.
  This space is reserved because data may be overwritten if previous
  segments are merged.

Padding
-------

Note that padding records have 2 uses:

1. While a transaction is being written to a file, it's marked as a
   padding record.  When the file write is finshed, the marker is
   changed from 'PPPP' to 'TTTT'.  This is similar to the use of the
   status flag in the original FileStorage.

2. To pad a segment to an alignment boundary when a transaction wouldn't fit.

Format:

1-word padding marker, 'PPPP'

1-word padding length, in words

padding

Transactions
------------

1-word transaction marker , 'TTTT' (normal) or 'tttt' (packed)

1-word transaction length, in words

8-byte tid

2-byte length of user name

2-byte length of description

1-word length of extension data

1-word number of data records

user name

description

extension data

data records

Data sub-record
---------------

8-byte oid

8-byte previous-record file position.
  0 means this is first. 1 means not first, but previous is in previous segment.

4-byte offset to begining of transaction

4-byte data length.
   If 0, then this is a deletion record.

crc32 data checksum

data

============
Index format
============

Every segment has an index file.  If the index file is lost, it can be
recomputed from the segment file.

The index file consists of a sequence of one or more oid/position
pairs followed by a footer.

Each oid/position pair consists of a 8-byte oid and an 8-byte segment
(file) offset, in words.

The footer consists of:

8-byte segment size in words
8-byte starting transaction
8-byte ending transaction
