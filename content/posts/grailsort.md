---
title: "Rediscovering Sorting's Holy Grail"
date: 2022-03-23T14:03:03-07:00
math: true
summary: "Implementing a stable, linearithmic sort in constant space."
---

###### Dedication

What follows is dedicated to Andrey Astrelin. After a life dedicated to the
pursuit of knowledge—recounted by his family [here][obit]—Andrey passed away in
2017.

## Introduction

When I learned about sorting algorithms, I was given the impression
that one could have at most two out of the following three:

* stability
* $O(n \log{} n)$ runtime
* $O(1)$ space

Sorting is one of the first topics covered in any introductory computer science
course, so you can imagine my surprise when I learned that this was false!
Worse, I was not alone in this misconception; here is a slide from Robert
Sedgewick's algorithms course at Princeton (circa 2010):

![A slide from Sedgewick's Algorithms course at Princeton, depicting sorting algorithms with various properties as well as yet-to-be-discovered "holy sorting grail"](/images/grailsort/grail-sort-slide.png)

I first saw this slide in a [blog post][grailsort] by Andrey Astrelin (AKA
Mrrl). It goes on to describe the implementation of a sorting algorithm
fulfilling all three criteria. Andrey called his implementation [grailsort][], a
reference to the final row in the table above. As it turns out, a "holy sorting
grail" was described as early as 1974 by Luis Trabb Pardo.[^first-blocksort]
Andrey implemented a more recent version from Bing-Chao Huang and Michael A.
Langston.[^practical-in-place] Both papers use similar techniques, so
I'll refer to this family of algorithms as [block sort][block-sort-wiki].

In this post, I'm going to implement Huang and Langston's block sort much like
Andrey did. My original goal was to create a block sort implementation suitable
for inclusion in Rust's standard library.[^std-blocksort] I'm not the first to
try this; the [Rewritten Grailsort project][rewritten-grailsort] has many
translations of Andrey's implementation in various languages (including
Rust). However, this is the first one I know of that uses the block tagging
scheme described by Huang and Langston, which halves the requisite length
of the internal buffer compared to other implementations.  I'm going to
describe this method, as well as the rest of the algorithm, in detail.
Hopefully by expanding on the parts I had difficulty with, others will be
able to learn about an interesting corner of computer science.

[grailsort]: https://github.com/Mrrl/GrailSort

[^std-blocksort]: I'm not convinced that block sort is an appropriate choice
    for general purpose programming languages, but it's nice to have the
    option!

    I first became interested in the subject when I learned of the
    effort to include Rust inside the Linux kernel. One obstacle to running
    Rust in kernel-space is that the standard library assumes that allocation
    failure is unrecoverable and results in process termination. This
    assumption is invalid inside a kernel, which makes developing them in Rust
    unpleasant. Many functions in the standard library must be
    rewritten to propagate allocator errors to the caller.

    Amusingly, one such function is [`sort`][rust-sort]! `sort` is guaranteed
    to be stable in Rust and implemented via an optimized merge sort, which
    needs to allocate. I suspect most users expect `sort` to succeed
    unconditionally, and would not know what to do if it returned an error.
    [Zig], which forces programmers to consider the possibility of allocation
    failure and also guarantees stability for [`sort`][zig-sort], uses block
    sort for its implementation.

    Empirically, most users do not need a stable sorting algorithm, so standard
    library `sort` should probably default to a fast, constant-space, unstable
    one, avoiding the overhead of block sort. If needed, a stable algorithm can
    be requested specifically, a lá C++'s [`stable_sort`][].

[`stable_sort`]: https://en.cppreference.com/w/cpp/algorithm/stable_sort

## Merging In-Place

A block sort is fundamentally a merge sort. It repeatedly merges two sorted
subsequences (runs) until the entire input is sorted. Unlike a typical merge
sort, however, we're not allowed to use an external buffer, so we're going to
need a merge subroutine that works in-place. Unfortunately without *any*
auxiliary storage, merging takes quadratic time. We need to find some sort of
workaround.

### MᴇʀɢᴇBᴜғ

To achieve a linear-time merge, we first extract a subset of elements from
the initial input that can be arbitrarily permuted without compromising the
overall stability of the sort. Assuming we have collected a contiguous sequence
of such elements—call this the "internal buffer"— we can merge two contiguous
sequences $A$ and $B$ by repeatedly comparing the leftmost element from each
and swapping the lesser one with the leftmost element in the internal buffer.

{{<figure src="/images/grailsort/merge-with-buffer.png" caption="Two steps of an in-place merge using an internal buffer.">}}

Dotted lines indicate searches or comparisons, while solid lines indicate
movement—in this case swaps.

This requires only that the internal buffer is at least as long as $B$. There
are no requirements on the length of $A$. To see why, observe that this routine
fails only when the merged elements collide with $A$. When we swap an element
from $B$, the distance between the merged elements and $A$ decreases by one,
but when we swap an element from $A$, the distance stays the same.

Furthermore, if the size of the internal buffer is equal to $|B|$, then once
one of the inputs is exhausted, the internal buffer will be contiguous in
memory with any remaining elements of $B$ to its right. These elements can be
appended to the merged ones via a single rotation, moving the internal buffer
to the right of the merged sequence.[^detail]

[^detail]: This detail will be important later.

##### Complexity

It should be obvious that this takes linear time. Each step merges one element
and requires exactly one comparison and one swap. Also, note that we can
construct a mirrored version of this subroutine, one where the internal buffer
starts on the right and moves to the left. In the mirrored version, $A$ and $B$
are processed from right to left (greatest to least). I'll refer to the two
versions as MᴇʀɢᴇBᴜғLᴇғᴛ and MᴇʀɢᴇBᴜғRɪɢʜᴛ, where the direction indicates the
side on which the internal buffer starts.

### MᴇʀɢᴇNᴏBᴜғ

MᴇʀɢᴇBᴜғ is not enough on its own, however. At some point, we will need to
merge the internal buffer back into the rest of the input.  This will require
an in-place merge with *zero* auxiliary storage. Let's call it MᴇʀɢᴇNᴏBᴜғ. As
discussed earlier, this routine cannot run in linear time, but by using it only
in limited circumstances, we can still guarantee an overall runtime of $O(n
\log{} n)$.

MᴇʀɢᴇNᴏBᴜғ executes the following steps repeatedly.

1. Do a binary search in $B$ to find the index that the leftmost element in
   $A$ would occupy.
2. Search forward in $A$ linearly to find all elements equal to the leftmost one.[^2]
3. Rotate all elements between the locations found in steps 1 and 2 to the
   right by the index found in step one.

Each step puts at least one element from $A$ in its final, merged position. It
continues until one of the sequences is exhausted.

{{<figure src="/images/grailsort/merge-no-buffer.png" caption="A single step of an in-place merge with no auxiliary buffer.">}}

First, we find the appropriate position in $B$ for the ⚁ at the start of $A$.
Next we find the number of duplicates of that element (2). Finally, we rotate
all elements in $B$ less than ⚁—in this case, a single ⚀—into their final
position at the start of the input. At the end of this iteration, three
elements have been merged in total.

[^2]: This version of MᴇʀɢᴇNᴏBᴜғ is not fully optimized. For example, Step 2
    could use an [exponential search][exp-search] instead of a linear one, or
    it could find the rightmost element in $B$ whose final position is the same
    as the one found in step 1 instead of stopping at duplicates. However,
    these optimizations are not necessary to achieve the desired runtime and
    they complicate analysis, so I won't discuss them further.

##### Complexity

It should be clear that the number of iterations is bounded by the number of
distinct elements in $A$. Call these $A\_{\ne}$. Step 1 requires a binary
search in $B$, and $B$ may or may not shrink at each iteration, therefore the
number of comparisons is $O(|A\_{\ne}| \log{} |B|)$. Step 2 will only
visit each element in $A$ exactly once across all iterations, so its complexity
is $O(|A|)$.

Step 3 is more complex. Each iteration involves a [linear-time][lin-rot]
rotation of some elements of $A$. Much like in step 2, each element of $B$ is
part of no more than one rotation, so the complexity of this step across
all iterations is $O(|A\_{\ne}||A| + |B|)$.[^rot-single]

[^rot-single]: If you find this too hand-wavy, here's a more formal
    description. The $k$-th rotation has complexity $O(|A\_k| +
    |B\_k|)$, where $A\_k$ are the elements in that rotation originating from
    $A$, etc. Add this up over all rotations to get $\sum_{k=0}^{|A\_{\ne}|}
    O(|A\_k| + |B\_k|)$ and sum the two terms separately.

    For the first sum, replace $|A\_k|$ by a conservative upper bound $|A|$ to
    obtain $O(|A\_{\ne}||A|)$.

    For the second, as we discussed above, no element in $B$ is rotated more
    than once, so all the $B\_k$s are mutually disjoint and the sum can be
    simplified to $O(|B|)$.

Combine these and simplify to get a total complexity of
$O(|A\_{\ne}||A| + |A\_{\ne}| \log{} |B| + |B|)$. Assuming that $A$ has
many distinct elements ($|A\_{\ne}| \approx |A|$), this operation is quadratic in
$|A|$ but linear in $|B|$.

As with MᴇʀɢᴇBᴜғ, a mirrored version exists that is quadratic in $|B|$ instead.
I'll refer to the version illustrated above as MᴇʀɢᴇNᴏBᴜғLᴇғᴛ and the mirrored
version as MᴇʀɢᴇNᴏBᴜғRɪɢʜᴛ. The direction indicates which input is
quadratic. MᴇʀɢᴇNᴏBᴜғRɪɢʜᴛ can be implemented by rotating all duplicates of the
*rightmost* element of $B$ into the correct position in $A$ at each iteration.

## Grailsort

Finally, we're ready to implement our in-place, stable sort.

Our algorithm proceeds like a bottom-up merge sort and divides the input (of
length $n$) into runs of length $m$, starting with $m=1$.[^small-base] At each
step $k$ of the sort, $m=2^k$, and pairs of runs are merged into a single run
of length $2m$. Each merge must be stable, otherwise the stability of the sort
would be compromised. This continues iteratively until the entire input is
sorted ($n \le m$). As long as each step is done in $O(n)$ time—requiring that
each merge be done in $O(m)$ time—the overall runtime of the algorithm is $O(n
\log{} n)$.

### Phase 0: Setup

As discussed above, MᴇʀɢᴇBᴜғ requires an internal buffer comprised of elements
that can be shuffled arbitrarily. The first step is to extract $x ≈ \sqrt{n}$
such elements. To make indexing easier, we round $x$ up to the nearest power of
two. If the input does not have enough distinct elements, we are forced to
round down instead.

#### BᴜғExᴛʀᴀᴄᴛ

How do we find such elements? We are implementing a
stable sort, after all, so we cannot permute elements at will. However,
relative order only matters for elements that are equal. That means, if we
extract a subset of elements such that

- No two elements in the subset are equal.
- Each element in the subset appeared *before* any of its duplicates in the
  original input.

We can shuffle that subset arbitrarily without compromising the stability of
our sort. When merging it back into the input, we must remember to prefer
elements from the internal buffer since they appeared first in the original
input.

The first element in the input always fulfills the aforementioned criteria, so
we begin with an internal buffer of length 1. The buffer must remain sorted
throughout the process. BᴜғExᴛʀᴀᴄᴛ proceeds as follows until the buffer
reaches the desired size:

1. Search forward in the remaining elements of the input until you find an
   element that is not yet in the buffer. For this, do a binary search within
   the internal buffer.
2. When a satisfactory element is found, rotate the internal buffer before that element.
3. Rotate that element into sorted order within the internal buffer.

{{<figure src="/images/grailsort/extract-buffer.png" caption="Extracting the internal buffer">}}

In the diagram, the elements ⚀, ⚃ and ⚀ are already part of the internal
buffer, so they are ignored. However, ⚁ is not yet present, so the buffer is
rotated immediately before it, and then ⚁ is rotated into sorted order.

##### Complexity

Call the size of the internal buffer $x$ (for au**x**iliary). In the worst
case, we need to search through the entire input to find the desired number of
distinct elements, and each element causes a binary search in the internal
buffer, resulting in $O(n \log{} x)$ comparisons. As for rotations,
each element of the input that is *not* included in the internal buffer is part
of at most one. However, the elements in the internal buffer can be part of
$2x$ rotations in the worst case, making the runtime complexity due to
rotations $O(n + x^2)$.

Luckily, we only need a buffer of size $\sqrt{n}$, so the setup phase takes
$O(n \log{} n)$ time.

### Phase 1: $m < x$

With our setup complete, it's time to start merging runs.

In this phase, $m$ is less than or equal to the size of our auxiliary buffer,
so we can use MᴇʀɢᴇBᴜғ to merge runs directly. Since the auxiliary buffer
begins on the left side of the input, we will use repeated invocations of
MᴇʀɢᴇBᴜғLᴇғᴛ.  Recall that, in addition to merging the two runs, each
invocation of MᴇʀɢᴇBᴜғLᴇғᴛ moves the auxiliary buffer to the right side of the
newly merged elements, conveniently positioning it immediately before the next
pair of runs.

Each level of the merge requires $m=2^k$ elements of the buffer, so the $N$th
level requires $\sum_{k=0}^{N} 2^k = 2^{N+1}-1$ buffer elements in total. In
the last step of this phase, when $m=\frac{x}{2}$, $x-1$ buffer elements will
have been moved to the right side of the input.  If we are clever, we can move
a single buffer element to the right side of the input during Phase 0, then
execute the step using MᴇʀɢᴇBᴜғRɪɢʜᴛ instead of MᴇʀɢᴇBᴜғLᴇғᴛ. This causes the
buffer to move back to the left of the input, putting it in position for the
next phase without any extra rotations.[^lazy-left]

[^lazy-left]: Alternatively, you can rotate the buffer elements back to the
    start of the input, complete the level with MᴇʀɢᴇBᴜғLᴇғᴛ as before, then
    rotate them back a second time to prepare for the next phase. This is what
    the current implementation does.


##### Partial Blocks

Note that, if the length of the input is not itself a power of two, we may have
a run whose length is strictly less than $x$. Similarly, we may have only a
single run at the end of the input instead of a pair. It is not conceptually
difficult to handle these unusually sized runs, but it does require careful
bookkeeping. In particular, MᴇʀɢᴇBᴜғ must handle the case where the size of
the auxiliary buffer is not exactly equal to $|B|$.

### Phase 2: $x \le m < x^2$

At this point, runs are too large to be merged in a single step. Instead, we
split each run into a series of "blocks" of size $x$. We have $\frac{m}{x}$
such blocks.  By carefully arranging the blocks we can use a series of MᴇʀɢᴇBᴜғ
operations to merge the runs in linear time. This phase is the crux of any
block sorting algorithm.

#### BʟᴏᴄᴋRᴏʟʟ

The first step is to sort all blocks in both runs by their rightmost element,
or tail. We call this "rolling" the blocks, a term from the 2008 paper
on which the Wikipedia article is based. This sort must be stable, and we must
also remember which blocks are from the left run and which are from the right
once they are interleaved.  We'll call these $A$ and $B$ blocks respectively.
To preserve the stability of the merge, $A$ block elements must be preferred to
$B$ block elements if equal.

To remember which blocks are which, we treat our auxiliary buffer as a
"movement imitation buffer". Before sorting blocks, we select as many elements
from the auxiliary buffer as there are blocks and sort them. Each element in
the buffer corresponds to one of the blocks, and we keep a pointer to the
element representing the first $B$ block. Every time we swap a pair of blocks
as part of our sort, we swap the corresponding pair of buffer elements, and, if
necessary, update the pointer. Since the buffer began in sorted order, all
blocks whose corresponding buffer element compares less than the pointed-to one
are $A$ blocks. The rest are $B$ blocks.

For the sorting itself, Huang and Langston use a standard selection sort.  This
is sufficient to achieve our desired algorithmic complexity (see below), but a
practical implementation can and should advantage of the
fact that the $A$ blocks and $B$ blocks are in sorted order. This reduces the
number of comparisons needed to find the block with the next smallest tail. See
the source code for the fully optimized version.

{{<figure src="/images/grailsort/sort-blocks.png" caption="Use a movement imitation buffer to remember a block's origin.">}}

##### Complexity

Each run has at most $\frac{m}{x}$ blocks, and since the last step in this
phase merges $x^2$ elements at a time, we can have no more than $\sqrt{m}$
blocks. That means we can use a sorting algorithm with a quadratic number of
comparisons so long as it uses a linear number of swaps. Selection sort is one
such algorithm.

#### BʟᴏᴄᴋTᴀɢ

By sorting the blocks by their tails, we have moved them close enough to their
final position that MᴇʀɢᴇBᴜғ can finish the job. Unfortunately, MᴇʀɢᴇBᴜғ
requires the use of the auxiliary buffer, which is currently being used to remember
which blocks are $A$ and which are $B$. To free up the buffer, we're going to
temporarily encode this information in the blocks themselves using a novel
method from Huang and Langston.

Whenever we have two elements in a sorted sequence that are distinct, we can
encode a single bit of information by swapping them. For example, if the first
element in a block is less than the last element, we could swap the two if and
only if it is a $B$ block. Sadly, our input (and therefore our blocks) can
contain arbitrarily many duplicates, and duplicates cannot encode information in
this way.

Instead of directly encoding which blocks are which (one bit per block), we
settle for encoding where a *series* of blocks ends (one bit per series). To do
so, we take advantage of the following invariants post-BʟᴏᴄᴋRᴏʟʟ:

- All elements in the $A$ series remain in sorted order. Same for the $B$ series.
- The tails of each block are in sorted order, with $A$ blocks to the left of
  $B$ blocks when their tails are equal.

As a result of the second invariant, we know that whenever an $A$ block
succeeds a $B$ block the tail of the $A$ block is **strictly** greater than
the tail of the $B$ block. Furthermore, we know that the tail of
*all* subsequent blocks are also strictly greater than the tail of that $B$
block, since the tails are in sorted order.

Whenever we have a series of $A$ blocks, swap the leftmost element (head) of
its immediate $B$ block predecessor with the tail of its immediate $B$ block
successor as shown below with a single $A$ block.

{{<figure src="/images/grailsort/tag-block-tag.png" caption="A swap denotes the end of one B block series and the start of the next.">}}

You may wonder why we swap with the head of the preceding block instead of the
tail. This is to avoid collisions when the $B$ block series in question
consists of only a single block. To convince you that everything still works
correctly, I've illustrated the single block case below.

{{<figure src="/images/grailsort/tag-block-single.png" caption="Swapping with the head avoids collisions in a B block series of length one.">}}

#### BʟᴏᴄᴋUɴᴛᴀɢ

To get an even better understanding of the BʟᴏᴄᴋTᴀɢ routine, let's see how
to extract the information it encoded. We'll do this incrementally by
scanning linearly across the blocks. Assuming we start in a $B$ block series:

1. Compare the head of the $B$ block with its immediate successor in that
   block.  If the head is greater than its successor, this is the last $B$
   block in the series. The next block is an $A$ block.
2. Compare the tail of each block with the tail of its preceding block. If the
   tail of the current block is less than that of its predecessor, the current
   block is the first $B$ block in a $B$ block series.
3. Swap the head of the block from step 1 with the tail of the block from step 2.
   This undoes the swap from BʟᴏᴄᴋTᴀɢ and restores the original order.

{{<figure src="/images/grailsort/tag-block-untag.png" caption="Untagging reconstructs the data from the movement imitation buffer and puts the input back in original order.">}}

Note that this scheme is unable to encode how many $A$ blocks are at the start
and end of the block sequence. There's no preceding/succeeding $B$ block, so we
must pass this information out-of-band. Thankfully it is of constant size.

#### BʟᴏᴄᴋMᴇʀɢᴇBᴜғ

Once we have tagged the block sequence, we can repurpose the auxiliary buffer
as an internal buffer for MᴇʀɢᴇBᴜғ. Recall that the left input to MᴇʀɢᴇBᴜғLᴇғᴛ
can be arbitrarily large so long as the right input is no larger than the
internal buffer (a single block).

We proceed by merging each $A$ (or $B$) block series with the first block of
the $B$ (or $A$) block series that succeeds it. Because we sorted blocks by
their tails, we know that the final position of the tail of the right block
is to the right of any element in the left series. Therefore, MᴇʀɢᴇBᴜғLᴇғᴛ will put
all elements from the series in their final position, with at least one element
from the successor block remaining.

Unlike in Phase 1, where we were merging entire runs with a single MᴇʀɢᴇBᴜғ
invocation, we do not rotate the auxiliary buffer [past the remaining
elements](#fnref:4) of the successor block at this point. The remaining
elements are not in their final merged position, since they may be greater than
some elements from the first block in the next block series.

{{<figure src="/images/grailsort/merge-blocks.png" caption="Merge rolled blocks by repeatedly invoking MᴇʀɢᴇBᴜғ.">}}

The leftover elements from the first block of the right series become the first
elements of the left series for the next invocation of MᴇʀɢᴇBᴜғLᴇғᴛ. After we
merge an $AB$ series pair, we use another invocation of BʟᴏᴄᴋUɴᴛᴀɢ to figure out where to
go next. Like before, the internal buffer moves rightwards through the run until it
reaches the end.

##### Partial Blocks

As in Phase 1, we may have only a single run at the end of the input. In this
case there is no merging to be done, and we can continue to the next value of
$m$. We may also have a pair of runs where the length of the one on the right
($B$) is not exactly $m$. In this case, the final $B$ block may have size less
than $x$. We'll call it a partial block.

Trying to sort and tag the partial block would make it difficult to index any
$A$ blocks that are ordered after it. Instead, we ignore it until all other
blocks are merged. It is not subject to sorting or tagging. If the last series
of blocks was an $A$ series, we do one final merge with the partial $B$ block.
If the last series was a $B$ series, the partial block is already in the
correct position.

### Phase 3: $x^2 \le m$

If $\sqrt{n} \le x$, then $n \le m$ at this point and we are done. However, if
we failed to find sufficiently many distinct elements in Phase 0, we have more
work to do. Specifically, we know that the number of distinct elements in the
input $u$ is bounded by $2\sqrt{n}$ and therefore $x ≈ u$.[^phase0] By
combining the second fact with the inequality that defines this phase, we have
an even tighter bound, $u \le \sqrt{m}$.

We cannot continue as we did in Phase 2 because there are more than $\sqrt{m}$
blocks in each run and BʟᴏᴄᴋRᴏʟʟ is no longer guaranteed to run in $m$ time.
To get around this, each step will use the same number of blocks ($x$) for each
pair of runs, allowing the length of the blocks to grow arbitrarily ($m/x$). We
can continue to BʟᴏᴄᴋRᴏʟʟ as we did in phase 2, but we can no longer use
BʟᴏᴄᴋMᴇʀɢᴇBᴜғ, since the blocks will be too large to fit in the auxiliary
buffer.

[^phase0]: That's because [Phase 0](#phase-0-setup) picks as many distinct
    elements as it can up to $\sqrt{n}$.

#### BʟᴏᴄᴋMᴇʀɢᴇNᴏBᴜғ

Surprisingly, the bound on $u$ allows us to simply replace MᴇʀɢᴇBᴜғ in
BʟᴏᴄᴋMᴇʀɢᴇBᴜғ with MᴇʀɢᴇNᴏBᴜғ while still running in linear time!

Because we no longer need an internal buffer for MᴇʀɢᴇBᴜғ, we can use the
auxiliary buffer exclusively as a movement imitation buffer for the entire
phase. There is no need to call BʟᴏᴄᴋTᴀɢ or BʟᴏᴄᴋUɴᴛᴀɢ. While merging, we will
use the movement imitation buffer directly to determine which blocks are $A$
blocks and which are $B$. Everything else is exactly the same as in
BʟᴏᴄᴋMᴇʀɢᴇBᴜғ.

{{<figure src="/images/grailsort/merge-blocks-no-buf.png" caption="Merge rolled blocks by repeatedly invoking MᴇʀɢᴇNᴏBᴜғ.">}}

##### Complexity

To quote Huang and Langston, "although this method is easy to implement, it is
not perhaps obvious that it takes only linear time".  Recall that the
complexity for MᴇʀɢᴇNᴏBᴜғRɪɢʜᴛ was $O(|R\_{\ne}||R| + |R\_{\ne}| \log{} |L| +
|L|)$, where $L$ and $R$ are the sequences on the left and right
respectively.[^lr-confusion]

Each BʟᴏᴄᴋMᴇʀɢᴇNᴏBᴜғ operates on a single pair of runs and may involve multiple
invocations of MᴇʀɢᴇNᴏBᴜғ. A single element will be part of no more than two
MᴇʀɢᴇNᴏBᴜғ invocations (once on the left, once on the right).  The left input
to MᴇʀɢᴇNᴏBᴜғ may arbitrarily long, so $|L|$ becomes $m$ in the worst-case.
However, the right input is always exactly one block long, so $|R|$ becomes
$\frac{m}{x}$.

The factor $|R\_{\ne}|$—the number of distinct elements in the right sequence—
is more interesting. This is what determines the number of iterations through the
inner loop of MᴇʀɢᴇNᴏBᴜғ.
We know that the total number of unique elements in either
run is bounded by $u$. However, distinct elements can be split across several
block series, and so the same value may appear in the right sequence of a
MᴇʀɢᴇNᴏBᴜғ operation multiple times. Thankfully, each distinct value may appear in
**no more than four MᴇʀɢᴇNᴏBᴜғ operations**, so $|R\_{\ne}|$ can be replaced with
$u$ when computing the total runtime for all operations.

Why four? Since runs are sorted, all equal elements appear consecutively within
their run. Therefore, a single value can be present in exactly two settings:

- any number of consecutive blocks whose tail is equal to that value.
- zero or one blocks whose tail is not equal to that value.

BʟᴏᴄᴋRᴏʟʟ, because it is stable, keeps blocks from one run whose tails are
equal together in a single series, so all blocks from the first case remain
contiguous. This gives us a maximum of two block series in each run that may contain a
given element, for a total of four.

I've illustrated the worst-case scenario below for an input with many ⚂s.
You should be able to convince yourself that BʟᴏᴄᴋRᴏʟʟ cannot result in a fifth
block series containing a ⚂.

{{<figure src="/images/grailsort/block-distinct.png" caption="An input with many ⚂s, which can be split across no more than four block series.">}}

It's time to apply these rules to prove that BʟᴏᴄᴋMᴇʀɢᴇNᴏBᴜғ runs in $O(m)$
time. Sum up all MᴇʀɢᴇNᴏBᴜғ invocations using the rules above, then use the
fact that $u ≈ x \le \sqrt{m}$ and $\log{} m ∈ O(\sqrt{m})$.

\begin{align*}
\sum O(|R\_{\ne}||R| + |R\_{\ne}| \log{} |L| + |L|)
&       = O(u\frac{m}{x} + u \log{} m + m) \cr
& \approx O(x\frac{m}{x} + x \log{} m + m) \cr
&     \le O(x\frac{m}{x} + \sqrt{m} (\sqrt{m}) + m) \cr
&       = O(m)
\end{align*}

Voila!

[^lr-confusion]: This was originally expressed in terms of $A$ and $B$, but in
    this phase we're using those letters to denote the origin of a block
    series. In this phase, every alternate MᴇʀɢᴇNᴏBᴜғ invocation has a $B$
    block series on the left and a single $A$ block on the right.

### Phase 4: Cleanup

Merging continues until all of the input is part of a single, sorted run except
for the part in the internal buffer. Because the buffer has length
$O(\sqrt{n})$, we can sort the internal buffer using a quadratic algorithm,
then call MᴇʀɢᴇNᴏBᴜғ to merge it into the input. Remember that elements in
the internal buffer came before any of their duplicates in the original
input, so they should be preferred when merging to preserve stability.

The input is now sorted!

## Conclusion

With that, we've finished our own holy sorting grail. I honestly find it pretty
miraculous that this is even possible. It's a lot of work just to sort a list!

I've called my implementation
[MrrlSort](https://github.com/ecstatic-morse/MrrlSort) to pay tribute to Andrey
(and because it sounds a bit like "grail"). It's tested using
[proptest](https://docs.rs/proptest/latest/proptest/), a property-based testing
framework that found several subtle bugs during development. Now that it passes
those tests, I'm pretty confident in its correctness.

In a simple benchmark, MrrlSort took about twice as long as Rust's `sort`. Not
bad! Given the number of swaps, rotations, and auxiliary buffer operations I
expected worse. I'm very curious to see how this factor is affected by various
parameters (size of elements, cost of comparisons, etc.) and whether it can be
reduced further, but I think I need to take a break for now. To
paraphrase Gankra, whose offhand mention of Grailsort in [a comment from 2015](https://github.com/rust-lang/rust/issues/19221#issuecomment-70445816)
sent me on this unexpected diversion (and who is also known to publish the
occasional [massive blog post](https://gankra.github.io/blah/fix-rust-pointers/)):

Head empty, only block sort.


<style>
.post-content #dedication {
    display: none;
}

.post-content #dedication + p {
    background-color: lightblue;
    border: dotted black 2px;
    padding: 8px;
    margin-bottom: 2em;
}
body[data-theme="dark"] .post-content #dedication + p {
    background-color: black;
    border: dotted grey 2px;
    padding: 8px;
    margin-bottom: 2em;
}

.post-content .heading-link {
    color: var(--color-primary);
}

figure > img {
    display: block;
    margin-left: auto !important;
    margin-right: auto !important;
}

figcaption > p {
    text-align: center;
    color: var(--color-mute);
}

.katex-html {
    white-space: nowrap;
}
</style>

[rotate]: https://doc.rust-lang.org/std/primitive.slice.html#method.rotate_left

[^small-base]: Instead of starting with $m=1$, a practical merge (or block) sort
    will use a quadratic algorithm (usually insertion sort), to sort runs up to
    some small, fixed size.

[exp-search]: https://en.wikipedia.org/wiki/Exponential_search
[obit]: http://superliminal.com/andrey/biography.html
[gankra]: https://github.com/rust-lang/rust/issues/19221#issuecomment-70445816
[Zig]: https://ziglang.org/
[grailsort]: https://habr.com/en/post/205290/
[rust-sort]: https://doc.rust-lang.org/std/primitive.slice.html#method.sort
[`sort_unstable`]: https://doc.rust-lang.org/std/primitive.slice.html#method.sort_unstable
[block-sort-wiki]: https://en.wikipedia.org/wiki/Block_sort
[zig-sort]: https://ziglang.org/documentation/master/std/#std;sort.sort
[lin-rot]: https://doc.rust-lang.org/std/primitive.slice.html#complexity-1
[first-blocksort]:  http://i.stanford.edu/pub/cstr/reports/cs/tr/74/470/CS-TR-74-470.pdf
[rewritten-grailsort]:  https://github.com/HolyGrailSortProject/Rewritten-Grailsort
[^first-blocksort]:  In a paper called ["Stable Sorting and Merging with Optimal Space and Time Bounds"][first-blocksort].
[^practical-in-place]: See ["Practical In-Place Merging"](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.88.1155&rep=rep1&type=pdf).

