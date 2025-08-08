+++
title = "Fsck in bcachefs"
slug = "Fsck"
url = "/Using/Fsck/"
+++


This document is an overview of fsck in bcachefs, as well as what will need to
change for snapshots.

In bcachefs, the fsck code is only responsible for checking higher level
filesystem structure. Btree topology and allocation information are checked by
`btree_gc.c`, which doesn't need to be aware of snapshots. This reduces the
scope of fsck considerably.

What it does have to check is:

* For each extent, dirent and xattr, that a corresponding inode exists and is of
  the appropriate type.

* Dirents and xattrs indexed by hash, with linear probing. The fsck code is
  responsible for checking that they're stored at the correct index (in case
  there was a hash collision, every index between the hash value and the actual
  index must be occupied by keys or hash whiteouts) and that there are no
  duplicates.

* Dirents: check that target inode exists and matches dirent's `d_type`, and
  check the target inode's backpointer fields.

* Extents:
  * Check that extents do not overlap
  * Check that extents do not exist past inode's `i_size`
  * Count extents for each inode and check inode's `i_sectors`.

* Check that root directory exists

* Check that lost+found exists

* Check directory stucture: currently, this is done with a depth-first traversal
  from the root directory. We check for directories with multiple links, and
  then scan the inodes btree for directories that weren't found by the DFS.

* Check each inode's link count: At this point we know that every dirent exists
  in a directory (has a corresponding inode of the correct type), has a valid
  target, and is reachable (because we've checked that all directories are
  reachable). All that's left is to scan the dirents btree, counting the number
  of dirents that point to each inode number, and verifying that against each
  inode's link count.

## Changes for snapshots

### References between keys

Filesystem code operates within a subvolume: at the start of a btree transaction
it looks up the subvolume's snapshot ID, and passes that to the btree
iterator code so that lookups always return the newest key visible to that
snapshot ID.

The fsck code does not operate within a specific subvolume or snapshot ID - we
have to iterate over and process keys in natural btree order, for performance.
From fsck's point of view, any key that points to another key may now point to
multiple versions of that key in different snapshots.

For example, given a dirent that points to an inode, we search for the first
inode with a snapshot ID equal to or an ancestor of the snapshot ID of the
dirent - a normal `BTREE_ITER_FILTER_WHITEOUTS` lookup, but using the snapshot
ID of the dirent, not the subvolume - this key should exist, if the dirent
points to a valid inode. Additionally, the dirent also references inodes at that
position in snapshot IDs that are descendends of the dirent's snapshot ID -
provided a newer dirent in a snapshot ID equal to or ancestor of the inode's
snapshot ID doesn't exist.

### Repair

If we repair an inconsistency by deleting a key, we need to ensure that key is
not referenced by child snapshots. For example, if we find an extent that does
not have an `S_ISREG` inode, currently we delete it. But we might find an extent
that doesn't have a corresponding inode in with a snapshot ID equal to or
ancestor to the extents snapshot ID, but does have an inode in a child snapshot.
If that were to occur, we should probably copy that inode to the extent's
snapshot ID.

We would like it to be as hard a rule as possible that fsck never deletes or
modifies keys that belong to interior node snapshot IDs, because when people are
using snapshots they'll generally want the data in that snapshot to always stay
intact and it would be rather bad form if data was torched due to a bug in fsck.

It may not be possible to adhere to this rule while fixing all filesystem
inconsistencies - e.g. if extents exist above an inode's `i_size`, then
generally speaking the right thing to do is to delete that extent. But if we
find inconsistencies in at interior node snapshot IDs, then perhaps the best
thing to do would be to just leave it, or only apply changes in leaf node
snapshot IDs.

### Algorithmic considerations

We want bcachefs to scale well into the millions of snapshots. This means that
there can't generally be operations that have to run for each subvolume or
snapshot - the fsck algorithms have to work one key at a time, processing them
in natural btree order.

This means that the extents, dirents and xattrs passes need to use
`BTREE_ITER_ALL_SNAPSHOTS`, meaning that as they walk keys for a given inode,
they'll be seeing keys from every snapshot version mixed together - and there
will be multiple versions of the inode, in different snapshots, that they need
to be checked against.

There's a helper in the fsck code that these passes use, `walk_inode()`, that
fetches the inode for the current inode number when it changes. This will
probably change to load every copy of that inode, in each snapshot ID. Also,
where we have to maintain state we'll have to duplicate that state for every
snapshot ID that exists at that inode number.

We'll also need to know, for every key that we process, which snapshot IDs it is
visible in: so that e.g. the extents pass can correctly count `i_sectors`.

Note that "millions of snapshots" will not _typically_ mean "millions of snaphot
IDs at the same time" - when we allocate new inodes we pick inode numbers that
aren't in use in any other snapshot or subvolume, and most files in a filesytem
are created, written to once and then later atomically replaced with a rename
call. When we're dealing with a file that keys with many different snapshot IDs
we need to make sure we don't have any hidden O(n^2) algorithms, but if fsck
requires in the hundreds of megabytes of state to check these files with worst
case numbers of overwrites, that's not the end of the world.
