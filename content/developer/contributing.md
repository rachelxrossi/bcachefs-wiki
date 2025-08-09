+++
title = "Contributing"
slug = "Contributing"
url = "/Developer/Contributing/"
weight = -100
+++

bcachefs contributor's guide:

The first thing to get out of the way: I aim to hold bcachefs code to a very high standard of quality, which is going to be considerably higher than most people are used to, even those with previous filesystems experience.

I want to be very up front about this because I know from prior experience that this can be hard on people and a source of considerable friction, but my goal isn't to make people's lives more difficult - quite the opposite. My purpose is that I hate debugging, and I don't know anyone else who enjoys debugging - particularly when you're responsible for chasing down bugs that are responsible for corrupting customer data and you can't reproduce yourself. That's no fun and it can be incredibly stressful.

If, instead, we are successful in keeping the bcachefs codebase as stable and reliable as possible - that means getting to do development at a much more leisurely pace, uninterrupted by fires that need putting out and with much more time for lazy afternoons going hiking or drinking mojitos.

How do we actually get to this promised land of filesystem development?

* Be selective about new features and new code. If you've got a feature
   request, and you can't figure out a way of doing it that doesn't make the
   codebase worse - don't do it. I don't care what the business case is, it's
   not worth it and I won't merge it.

   This usually won't mean saying no to people, but it often will mean having to
   think hard before coming up with a way of doing something that lets you say
   yes.

* Take as much time as you need. Don't rush.

   We developers should always remember that bcachefs - indeed most of our
   filesystems - already works for the vast majority of users, and already does
   everything we really need it to do. Any more improvements are just gravy.

   So there's really no rush to do even more stuff, and it makes no sense to
   rush if it comes at the cost of bugs that affect the people for whom bcachefs
   does already work for.

   Also, with any new feature the cost in developer time after it gets merged -
   debugging, ongoing maintenance - is usually far higher than the cost of
   initial development up until when it's first merged.

   To a large degree this is always going to be unavoidable, but keeping this in
   mind when writing code should help you go the extra mile and make it as bug
   free as you possibly can. Remember that it's far easier to debug something
   when you find it yourself and you've got the test case right in front of
   you.

* Don't overengineer. Leave anything out if you aren't convinced there's value
   in it. KISS, and be utterly ruthless in this.

   Customers will ask for all kinds of features they want, but don't actually
   need. But even worse are developers - developers will come up with all kinds
   of ideas that sound wonderful and cool, and get attached to them, often
   regardless of if they're actually necessary - and the trouble with developers
   is that they have the skills to turn their bad ideas into code.

   Even after you've written something, you have to be able to honestly ask
   yourself if it still seems like a good idea and if it turned out well enough.
   I've written and not merged almost as much of my own code as I've written and
   merged.

* Prioritize refactoring and improving existing code over adding new code.

   Most of the code in bcachefs is now in a state when I'm reasonably happy with
   it, and I can modify it without being scared of breaking things - but most of
   the code didn't get to that point without rewriting or heavily refactoring it
   four or five or six times. Not all the code is at that point - in particular,
   some of the btree iterator code is still too tricky and subtle. And the code
   relating to asynchronous interior btree updates is - I believe - sound, but
   definitely needs to be clearer.

   This is the normal order of things; it almost always takes quite a few
   attempts before we find clear, understandable, maintainable ways of
   expressing things, but we need to in order to keep the codebase in a state
   where it's sound enough to build new things on top of.

* Incremental development is only way anything comes out halfway decent.

   There are some fiendishly complicated and sophisticated things going on in
   bcachefs, most of them within the btree code. The locking, the iterators, the
   asynchronous interior btree node updates, the auxiliary lookup tables in
   bset.c...

   Without exception, those features all started out from very humble beginnings
   and were added to bit by bit - and every step of the way the codebase was
   kept as stable as possible, fixing bugs and making sure all the tests passed.

   When developing new features - things like erasure coding and snapshots -
   write the minimal amount of code to get something that can be tested. Then,
   add the minimal amount so you can ship it. Then ship it.

* Go the extra mile.

   Before you send something out, try to put yourself in the shoes of someone
   you know who writes really good code or who does really thorough code
   reviews, and ask yourself what they'd do or ask for. Doing so before you send
   out your patches is the quickest way to build trust and respect, and will
   also lead to your patches moving to the top of the list of patches to review
   and merge.

Lastly, I've made every effort to make bcachefs a solid base for others to build
off of, by making it as bug free as possible. Try to keep it that way, and pay
it forward to the people who came after you. At the same time, I'm only one
person, and I'm only human, so there's still lots of room for improvement. If
you see a way to refactor something that makes the code clearer or less fragile,
do it. The existing documentation isn't great - as you're learning the codebase,
that's a great opportunity to think of things that documentation should say, so
take those ideas and either write a bit of documentation yourself, or prod me
and get me to write it.
