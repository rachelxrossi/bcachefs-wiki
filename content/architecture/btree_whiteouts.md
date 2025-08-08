+++
title = "Btree Whiteouts"
slug = "BtreeWhiteouts"
url = "/Architecture/BtreeWhiteouts/"
+++

## Whiteout optimizations

bcachefs btree nodes are log structured, and thus are also effectively a hybrid
compacting data structure. Most btree node keys are in large sorted lists of
keys that are too big to be efficiently inserted to or deleted from - but since
we're not doing random updates on them, we can build special data structures to
highly accelerate lookups.

### Tracking when whiteouts need to be retained/written out

Since we usually can't delete or update in place an existing key, this means we
need whiteouts, but whiteouts add their own complications. When we generate a
whiteout because we overwrite or delete an existing key, we may or may not need
to keep that whiteout around and write it out to disk: if the key we overwrote
had been written out to disk we need to ensure that we write something out to
disk that overwrites it. If we're updating that key - i.e. inserting a new
version of that key - then writing out the new key is sufficient, but if we were
deleting we need to ensure that that whiteout is kept around and written out to
disk.

To track this we have the flag `bkey.needs_whiteout`: keys have this flag set
when they've been written out to disk, or when they overwrote something that was
written out to disk.

### Storing whiteouts separately from other keys

Additionally: as mentioned, some whiteouts need to be retained until the next
btree node write, but we don't want to keep them mixed in with the rest of the
keys in a btree node where they'd have to be skipped over when we're iterating
over live keys. Therefore, when a given bset in a btree node has too many
whiteouts (as a fraction of the total amount of data in that bset), we do a
compact operation that drops whiteouts, saving the ones that need to be written
where the next btree node write will find it.

### Deleting keys without emitting a new whiteout

We can delete a key, even one that's been written out to disk, without emitting
a new whiteout (because that would require inserting into the last bset and an
expensive memmove).

This works by changing the key type to `KEY_TYPE_deleted` - as usual whenever we
overwrite an existing key - and leaving `needs_whiteout` set for that key, and
additionally calling `reserve_whiteout()` to reserve space in the next btree
write. The btree write code will scan the already-written bsets for whiteouts
that need to be written and pick them up.

## Extents

Extents add their own complications. `KEY_TYPE_deleted` is used for whiteouts
for normal keys, and it's also used for extents that have been trimmed to 0
size. But a 0 size extent is meaningless - these can always be dropped.
`KEY_TYPE_discard` is used for nonzero size extent whiteouts - i.e. whiteouts
that actually do overwrite other extents.
