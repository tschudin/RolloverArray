# Replicated Roll-Over Arrays (ROAR)

How can we publish a mutable data structure via an immutable
storage stream such that the stream can always be pruned?

This question emerges in the context of replicated append-only
logs. We wish to use append-only logs as a replication mechanism
because of their simple synchronization protocol (two peers just have
to compare their replica's height in order to find out which replica
contains more recent data). But because the logs are immutable,
storing information updates regarding higher-level objects will
accumulate in an unbounded way.

In this work we show how a specific higher-level data structure, an
array, can be mapped to a unbounded append-only log yet we keep the
relevant data in an optimally bounded section of the log. In other
words, our encoding of the mutable array permits to always prune the
log. The following pictures depicts our approach:

![A log wrapping around a monowheel cycle.](img/monowheel.png)

The current state of an array is stored in the red section of the log.
Information about the modification of the array is added at the front
of the log. In order to bound the red section (and declare some
entries as obsolete, shown in green), it may be necessary to copy old
data to the front. The trick is to store enough ordering information
such that old array elements can be sitting at the front of the log,
if necessary.

## History

This work is an extension of a technique described in the MSc thesis
of Sebastian Philipp at the University of Basel, 2022 with the
title _"Memory-Bounded Replication of Mutable Data Structures over
Immutable Append-Only Logs"_.

Philipp's encoding for an array data structure is based on storing,
per log entry, one to three basic operation types: "create", "link"
and "head". In this way, insertion as well as deletion can be modeled
as is shown in the example of Figure 3.4:

![Copy of Fig 3.4 in Philipp's MSc thesis, 2022, page 20](img/philipps-fig3.4.png)


Based on the same approach we extended Philipp's technique by adding
an aggressive pruning strategy using "roll-overs". Unless the
modification of the array is about adding a new element (in which case
we have to grow the "red section"), all other actions involve pruning
one log entry. In case that pruned entry contained still revelant
data, we copy its content, as well as updated "rewiring of the links",
to the front of the log. This must be done in a careful way. For
example, deleting an element requires updating pointers in the
adjacent elements with the catch that one of these elements may have
now moved from the back to the front of the log (i.e., in this case
the updating has to be updated before writing the relinking
information to the log).

## Log Entry Encoding

For the current implementation in Python we chose an internal
double-linked list approach when storing the higher-level data
structure in memory. For storing changes in the log, it suffices to
store only one link type, the ```prev``` links, as the other direction
(```next```) can be computed based on the in-memory data. Differently
from Philipp, we chose to keep track of a tail pointer instead of
head, assuming that it is more frequent that elements are appended to
the array e.g., in list operations. Overall we also have three
operations:

```
value "some data"      // corresponds to 'create'
link (at, to)          // in element 'at' replace the prev link by 'to' or nil
tail (to)              // define the new tail element value, can be nil
```

where ```at``` and ```to``` are pointers in form of sequence numbers,
identifying what log entry is referenced. As with Philipp's encoding,
the default field values are ```nil```.

A one-element array would be encoded as:
```
#4475  (value "sole element"), (tail 4475)
```

A two-element array could be encoded as:
```
#6523  (value "first element"), tail(6523)
#6524  (value "2nd element"), link(6524, 6523), tail(6524)
```

If some rollover happened, it can occur that the array order is different
from the log storage order. Using the same content, we have:

```
#6524  (value "2nd element"),  link(6524, 6523), tail(6524)
#6525  (value "first element"), link(6524, 6525)
```

As one can see, the value for the first array element is re-added to
the log at the front (implicitely the ```prev``` pointer is
```nil```).  But because the first element has changed its locations,
we need to relink the ```prev``` link of the second element (at #6524)
and let it point to #6525, which also is part of entry #6525. The tail
information stored in entry #6524 is still valid and does not need
changing.


## Replay

The last example shows that the contents of log entries are not
necessarily valid, if taken out of context: Clearly, #6524 contains
wrong pointer data. But the log's content is immutable and we have to
consider all subsequent entries that modify the (in-memory) data
structure.

Our basic requirement is that the array's content can be fully
reconstructed by replaying exactly the log's element in the "red
section".

In our implementation we do exactly this by simply executing the
instructions in the log and updating the in-memory representation of
the array. The first pass works from tail to head: It creates the
nodes (for each ```value``` commend), defines the ```prev``` link
pointers where necessary, and sets the currrent tail value. A second
pass, now over the in-memory single-linked list, is required to create
the ```next``` pointer values of our desired in-memory double-linked
list.


## A Python Library

...

## Demo

...

---


