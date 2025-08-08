+++
title = "Btree Iterators"
slug = "BtreeIterators"
url = "/Architecture/BtreeIterators/"
+++

Btree iterators in bcachefs implement more or less the usual operations for
iterating over an ordered key/value store.

Iterators may be unlocked arbitrarily, due to e.g. transaction restarts (due to
lock ordering, failure to get a journal reservation, btree node splits, etc.).
This means they must remember what they were pointing at, so that subsequent
peek/next etc. operations behave correctly. Also, a given btree transaction may
have multiple iterators in use at the same time; updates to via one iterator
must not invalidate other iterators (but this is a relatively easy requirement
compared to getting restartable iterators correct).

Iterators may be used to iterate over keys or slots; for slots we iterate over
every possible position, returning `KEY_TYPE_deleted` keys for empty slots. They
can iterate forwards or backwards, and they can be used to iterate over extents
or non-extents. Extents always have a nonzero size and have start and end
positions; iterating over extents in slots mode (where we synthesize ranges to
represent holes between keys in the btree) is used frequently by the filesystem
IO code in bcachefs.

## Primary iterator state

* `iter.pos`: the current position of the iterator.

   The iterator position may be changed directly with
   `bch2_btree_iter_set_pos()`, and the various peek()/next()/prev() operations
   all update the iterator position to match what they are returning.

* `iter.l[BTREE_MAX_DEPTH]` (struct `btree_iter_level`)

   These contain, one for each level of the btree, a pointer to a btree node, an
   iterator for that btree node (struct `btree_node_iter`, see bset.h), and a
   lock sequence number. Thus, we have a full path from the root of the btree down
   to a specific key in a specific btree leaf node.

   Btree node iterators are much simpler: they can be treated as if they point
   to a single key within a sorted array of keys. Because btree nodes have
   multiple sorted arrays of code btree node iterators are internally
   maintaining multiple pointers and sorting the results, but this is an
   internal implementation detail.

The most important invariant for btree iterators is that `iter.pos` must always
be consistent with the btree node iterators: that is, for every btree node
iterator, the key the node iterator points to must compare >= `iter.pos`, and
the previous key must compare < `iter.pos`.

That is - the node iterator must point to the correct position for inserting a
new key at `iter.pos`.

For the various peek()/next()/prev() operations, `iter.pos` will _usually_ be
updated to point to the start of the key we're returning, and the leaf btree
node iterator will _usually_ point to the key we're returning, unless we're at a
hole - but there are exceptions for extents.

### A discussion on iterator position

`bch2_btree_iter_peek_slot()` and `bch2_btree_iter_peek_slot_extents()` never
change `iter->pos`. `bch2_btree_iter_peek()` will change `iter->pos` to match
the key just returned.

`iter->pos` should always match the key we're currently at; this is so that
on transaction restart we can get back to where we were at, and also so that
when updating the key at the current position the iterator is at the correct
position for that update.

Also, `iter->pos` is used by code that uses btree iterators to keep track of
where it's at in the work it's doing. Importantly, when iterating over extents
by slots (i.e. in `bch2_read()`), as we call `next_slot()` `iter->pos` will be
advanced to where the previously returned extent ended.

If we're iterating forwards, `iter->pos` will be monotonically increasing, and
if we're iterating forwards by slots we guarantee that we'll return keys/extents
that exactly cover the keyspace.

For non extent iterators, we return the first key >= `iter->pos`; for extent
iterators, we return the first key strictly greater than `iter->pos`. Right now
this is handled by `btree_iter_search_key()`. This is because when we're
iterating over extents we want the first extent that covers the range starting
at `iter->pos`.

For iterating backwards, we have `peek_prev()` that returns the first key <=
`iter->pos`. Note: when using `peek_prev()` for extents, it's the same search
condition as for non extents - first key <= `iter->pos`.

Also: we need `iter->pos` to correspond to and be consistent what the rest of
the iterator physically points to.

So we want two different notions of iterator position:

* One

## Extents

When returning an extent, `iter.pos` will typically be updated to point to the
start of the extent we're returning (which is not the key the extent is indexed
by; extents are indexed by where they end, not where they start).

This may mean that when we return an extent, the leaf btree node iterator won't
necessarily point to the key we're returning - due to our #1 invariant, and the
possible existence of whiteouts (that we filter out when iterating and don't
return to user code).

Additionally, iter.pos won't be updated to point to the start of the extent
we're returning when that would move iter.pos backwards (unless we're actually
iterating backwards); this is because user code wants to see a monotonically
increasing iterator position (as user code often uses `iter.pos` to track how
far we've gotten/how much work we have to do).
