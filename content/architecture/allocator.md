+++
title = "Allocator"
slug = "Allocator"
url = "/Architecture/Allocator/"
+++

It's past time for the allocator to be redesigned, or at least heavily
reconsidered. Nothing should be off the table at this point - we could
conceivably even move away from bucket based allocation.

Here's a list of considerations:

* The bucket based allocator does have advantages. It's fast - most of the work
  in the allocator is per bucket, which are typically 512k to 2M, even up to 8M.
  Metadata is one key per bucket, and the alloc btree uses the btree key
  cache, which is very good for performance - it means extents btree updates
  don't have to do a second (relatively expensive) btree update to update alloc
  info, they're only updating the key cache.

* The bucket based allocator also lets us store and cheaply discard cached data,
  by incrementing the generation number for that bucket.

* Disadvantage: the bucket allocator requires us to have copygc, with everything
  that entails.

* It would be really nice to have backpointers at some point - copygc would
  greatly benefit from having backpointers (currently we have to scan the entire
  extents and reflink btrees to find data that needs to be moved), and erasure
  coding could also make good use of backpointers.

  If we added another btree for backpointers, it wouldn't make sense for it to
  use the btree key cache, and thus extents btree updates would get more
  expensive. Perhaps alloc keys could include a list of backpointers? They'd be
  32 bytes each, and bkeys can be a little short of 2k - that's only ~50
  backpointers per bkey, which we'd easily overflow with small writes and large
  buckets. If a secondary backpointers btree was just used as a fallback, it
  would be getting used right when the performance impact matters most - small
  writes.

  We should investigate the actual performance overhead of adding another btree
  update for backpointers - it might not be all that bad. Right now we're
  looking at ~450k random updates per second on a single core, and that includes
  the full `bch2_trans_commit()` path, and the most expensive things in that are
  per transaction commit, not per btree update (e.g. getting the journal
  reservation). Also, we'll hopefully be getting gap buffers soon, which will
  improve btree update performance.

The most pressing consideration though is that we need to eliminate, as much as
possible, the periodic scanning the allocator thread(s) have to do to find
available buckets. It's too easy to end up with bugs where this scanning is
happening repeatedly (we have one now...), and it's a scalability issue and we
shouldn't be doing it at all.

## In memory bucket array - kill

We need to get rid of the in memory array of buckets - it's another scalability
issue, and at this point it's an anachronism since now it mostly just mirrors
what's in the btree, in `KEY_TYPE_alloc` keys. Exceptions:

* `journal_seq`, which is the journal sequence number of the last btree update
  that touched that bucket. It's needed for knowing whether we need to flush the
  journal before allocating and writing to that bucket - we need to create
  another data structure for bucket that can't be used until the journal is
  flushed.

* `owned_by_allocator`, which is true if an `open_bucket` referencing that
  bucket exists - `open_buckets` represest buckets currently being allocated
  from, they're reference counted to prevent buckets from being double allocated
  (with writes taking a reference when they allocate space and releasing that
  ref after they create the extent that points to that space).

Currently, the main obstacle to getting rid of the bucket array is that we need
to start the allocator threads to run journal replay, but we can't use btree
iterators until journal replay has at least finished replaying updates to
interior btree nodes.

We have code in recovery.c for iterators that iterate over the btree with keys
from the journal overlaid; it may be time to move this support into regular
btree iterators in `btree_iter.c`.

## WHAT WE NEED IN THE REWRITE

### Data structures

* buckets containing cached data that can be used if they are invalidated

* buckets that are waiting on journal commit until they can be used

* buckets ready to be discarded

* buckets that are ready to be used now

Buckets that are ready to be used now should be stored in sorted order.

Buckets that are waiting on journal commit should be indexed by what journal
seq needs to be flushed.

Buckets that contain cached data should be stored on a heap.

TASK LIST:

* Move incrementing of bucket gens to happen when a bucket becomes empty (and
  doesn't have an `open_bucket` pointing to it) - also need to add code to
  `open_bucket_put()` to increment bucket gen if necessary

* At the same time, add bucket to list of buckets awaiting journal commit

* Add a flags field to `bch_alloc_v2` and a flag indicating bucket needs to be
  discarded

* Add a flag indicating whether a bucket had cached data in it - if not, we
  don't need to track `oldest_gen`

STATE TRANSITIONS:

Cached data bucket -> empty uncommitted bucket
Dirty  data bucket -> empty uncommitted bucket

In either case, the empty bucket first goes on the list of buckets awaiting
journal commit.

After journal commit:

empty uncommited bucket -> empty undiscarded bucket

After discard
