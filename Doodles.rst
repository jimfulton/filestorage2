Maybe use UUIDs for object ids
------------------------------

Storages don't have to assign.

Storages to verify, for new objects, that UUID isn't being used.

Allow semi-seqential records
----------------------------

One can start at beginning to read sequentially, but doing so may then
require jumping to a later part of the file and then jumping back.  This would allow packing to copy old records to a new file.

For example, suppose we have f1 and f2 that we want to combine/pack
into a single file, where f1 has record written before records in f2.
Rather than copying current records from f1 and f2 into a new file
f12, we can copy non-current records from f1 into f2. These records
will be written to the end of f2. So, if we want to scan records in f2
in cronological order, we'll have to immediately jump to where f1's
records are written and then jump back to read f2's records.  This
strategy allows us to avoid copying records from f2, but defers
packing of non-current records in f2, presumably until it's packed
with f3.  Note that we'll still have to copy f2's index records to
create a new index.

There are 2 variations of the above strategy:

- copy f1 records current as of the end of f1.  This allows time
  travel to the end of f1.

- copy f1 records current as of the end of f2. In this case, we can
  only time-travel back to the end of f2.

  Note that if we want to use this approach, we'd need some way to
  indicate pack time other than in transaction records, such as file
  meta data.

Scribble segments
-----------------

When adding records to a file, when packing or committing a
transaction, we first add a scribble segment and then override the
segment marker when we commit.

We use padding segments for this.
