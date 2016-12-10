Deconstructing Sort
===================

Sort in Haskell and C
---------------------

The `Introduction on the Haskell wiki <https://wiki.haskell.org/Introduction>`_
shows how quicksort can be implemented in Haskell compared to C. The Haskell
implementation looks like this:

.. code-block:: haskell

    quicksort :: Ord a => [a] -> [a]
    quicksort []     = []
    quicksort (p:xs) = (quicksort lesser) ++ [p] ++ (quicksort greater)
        where
            lesser  = filter (< p) xs
            greater = filter (>= p) xs

The proposed C implementation looks like this:

.. code-block:: c

    // To sort array a[] of size n: qsort(a,0,n-1)
    void qsort(int a[], int lo, int hi)
    {
      int h, l, p, t;

      if (lo < hi) {
        l = lo;
        h = hi;
        p = a[hi];

        do {
          while ((l < h) && (a[l] <= p))
              l = l+1;
          while ((h > l) && (a[h] >= p))
              h = h-1;
          if (l < h) {
              t = a[l];
              a[l] = a[h];
              a[h] = t;
          }
        } while (l < h);

        a[hi] = a[l];
        a[l] = p;

        qsort( a, lo, l-1 );
        qsort( a, l+1, hi );
      }
    }

Ugly. Hard to read. Not very expressive. To quote the wiki page:

    Whereas the C program describes the particular steps the machine must make
    to perform a sort -- with most code dealing with the low-level details of
    data manipulation -- the Haskell program encodes the sorting algorithm at a
    much higher level, with improved brevity and clarity as a result (at the
    cost of efficiency unless compiled by a very smart compiler).

So the above C implementation doesn't really make the algorithm obvious, rather
it outlines the steps the machine needs to take to implement it. Can we do
better in C++? The main disadvantage of Haskell, as outlined on the wiki, is
that it doesn't perform the sort in-place, so lots of data gets allocated and
freed while running the sort. It is also not quite a quicksort implementation,
because it is stable.

How would an in-place stable sort implementation look like in C++? Could we
write it so that the algorithm stands out?

Deconstructing Sort
-------------------

So how does sort work? Given a list of elements, we pick a pivot element,
partitions the list so that all elements smaller than the pivot go to its left
and the elements larger than the pivot go to its right, then the algorithm is
recursively applied to the left and right lists. An empty list is already
sorted, so that's where the recursion stops.

Let's review the Haskell implementation. We stop when the list is empty:

.. code-block:: haskell

    quicksort []     = []

For a non-empty list, pick the head as the pivot and recursively apply the
function to elements lesser and greater than the pivot (concatenating back the
results):

.. code-block:: haskell

    quicksort (p:xs) = (quicksort lesser) ++ [p] ++ (quicksort greater)

``lesser`` and ``greater`` are obtained by filtering the list using ``< p`` and
``>= p`` as predicates:

.. code-block:: haskell

        where
            lesser  = filter (< p) xs
            greater = filter (>= p) xs

In C++, one way to implement an equivalent stable sort is this [#]_:

.. code-block:: c++

    template <typename I> void sort(I f, I l)
    {
        if (f == l) return;

        auto p = stable_partition(f, l,
            [pivot = *f](const auto& elem) { return elem < pivot; });

        sort(f, p);
        sort(++p, l);
    }

Pretty concise. We use iterators, so ``I`` here would be an iterator over the
container we want to sort. ``f`` and ``l`` are the iterators to the start and
end of the range. Note iterators denote half-open ranges ``[f, l)``, so ``l``
does not have to point to a valid element (it points to "after the last element
in the range").

We introduced ``stable_partition``, an algorithm which, given a range and a
predicate, partitions the range such that the elements which satisfy the
predicate appear before the elements which don't satisfy the predicate. It is
stable because it won't change the relative order of elements in either of the
two groups. For example, applying ``stable_partition`` to ``5 4 3 2 1`` with
the predicate ``< 3`` would always yield ``2 1 5 4 3`` - first group is ``2 1``
(``< 3``), second group is ``5 4 3`` (``>= 3``) and the relative order of
elements in each group is the same as in the initial range (``2`` before ``1``,
``5`` before ``4`` before ``3``). ``stable_partition`` returns an iterator to
the first element not satisfying the predicate (in this case the iterator would
point to ``5``).

Line by line, if the range is empty, stop:

.. code-block:: c++

    if (f == l) return;

Get an iterator to the position of the pivot element, using ``stable_partition``
and the predicate "less than the value of the pivot":

.. code-block:: c++

    auto p = stable_partition(f, l,
        [pivot = *f](const auto& elem) { return elem < pivot; });

The lambda captures the value of ``*f`` and uses it in the predicate
``elem < pivot``. After this step, our range is partitioned and we have an
iterator to the new position of the pivot. We need to recursively sort the
range from the first element up to the pivot:

.. code-block:: c++

    sort(f, p);

Then we also need to sort the range from the element next to the pivot up to the
last element:

.. code-block:: c++

    sort(++p, l);

Done.

Stable Partition
----------------

This might arguably sound like cheating a bit, since we introduced
``stable_partition``. On the other hand, ``stable_partition`` is a standard
algorithm and the Haskell implementation uses ``filter`` itself. I will stop
the Haskell comparison here [#]_ and attempt to decompose further, so in the end
we have a sort built out of nice functional building blocks.

So far we have a recursive sort implementation which uses ``stable_partition``.
How would we go about implementing ``stable_partition``?
Here is one way to do it:

.. code-block:: c++

    template <typename I, typename P> I stable_partition(I f, I l, P pred)
    {
        if (f == l) return l;
        if (l - f == 1) return f + pred(*f) ? 1 : 0;

        auto m = f + (l - f) / 2;

        return rotate(stable_partition(f, m, pred),
                      m,
                      stable_partition(m, l, pred));
    }

This is a recursive implementation with the following basic idea: we stop if we
have an empty range or a range with a single element. Otherwise we divide the
range in two by finding the middle point. We then recursively partition the
ranges left and right of the midpoint. After this is done, we have two consecutive
ranges which are both partitioned, it's just that the elements satisfying the
predicate in the right range should come before the elements not satisfying the
predicate in the left range. We need to rotate them around the midpoint.

Let's take as an example the range over ``10 1 9 2 8 3 7 4 6 5`` and the
predicate ``< 7``. We would split this in two ranges around the midpoint:
``10 1 9 2 8`` and ``3 7 4 6 5``. Assuming the sub-ranges get stable-partitioned,
we end up with ``1 2 | 10 9 8`` and ``3 4 6 5 | 7`` (I'm using ``|`` to mark the
partition point). Now sticking them back together, we have ``1 2 | 10 9 8 | 3 4 6 5 | 7``
but we would like to have ``1 2 3 4 6 5 | 10 9 8 7`` (note this is a stable
partition, the elements aren't supposed to get sorted, rather relative order is
preserved). In other words, the first partition of the left range, ``1 2``, is
OK, the last partition of the right range, ``7`` is OK, we only need to swap
``10 9 8`` with ``3 4 6 5``. This is done with a rotation.

``rotate`` takes as arguments an iterator to the beginning of a range, an
iterator to the element around which we want to rotate, and an iterator to the
end of the range. It helps that ``stable_partition`` returns just the iterators
we need to feed into ``rotate`` (the partition points of the left and right
sub-ranges). Rotating them around the midpoint yields the stable partition of
the whole range.

Line by line, ``stable_partition`` of an empty range returns the iterator to
the last element:

.. code-block:: c++

    if (f == l) return l;

``stable_partition`` of a single element returns either ``f`` or ``f + 1``,
depending on whether ``*f`` satisfies the predicate:

.. code-block:: c++

    if (l - f == 1) return f + pred(f*) ? 1 : 0;

Next, we determine the midpoint, which is ``f`` + half the distance between
``f`` and ``l``:

.. code-block:: c++

    auto m = f + (l - f) / 2;

Then we recursively apply ``stable_partition`` to ``[f, m)``, ``[m, l)``, and
rotate the partition points returned around the midpoint ``m``:

.. code-block:: c++

    return rotate(stable_partition(f, m, pred),
                  m,
                  stable_partition(m, l, pred));

Rotate
------

``rotate`` is also a standard algorithm, but let's look at a possible
implementation. Given the beginning and end iterators over a range, and an
iterator within the range around which we want to rotate, we move the elements
so that the first group appears after the second group and return an iterator
to where the initial first element ends up in the final sequence.

There are more efficient ways of rotating by swapping elements while making
sure we don't overlap ranges, but a neat way of doing a rotation is using
reverse:

.. code-block:: c++

    template <typename I> I rotate(I f, I p, I l)
    {
        reverse(f, p);
        reverse(p, l);
        reverse(f, l);

        return l - p + f;
    }

Given a pivot point ``p`` around which we want to rotate, we can perform the
rotation by reversing first the range ``[f, p)``, then the range ``[p, l)``,
then the whole range ``[f, l)``.

For example, given ``E F G H I J A B C D``, we would like to rotate it so
``A B C D`` appears before ``E F G H I J``. So marking the pivot point with ``|``:
``E F G H I J | A B C D``. First, we reverse the first group and end up with
``J I H G F E | A B C D``. Next, we reverse the second group, and end up with
``J I H G F E | D C B A``. Last, we reverse the whole range: ``A B C D | E F G H I J``,
which ends up with exactly what we wanted.

I won't go line by line over the above as the calls to reverse are trivial, the
only thing worth noting is the returned iterator, determined by adding the
distance between the pivot point and the end of the range to the iterator
pointing to the beginning of the range. This is, in fact, the new position the
initial first element takes (refer to the example above where first element is
``E`` with initial position ``f`` and final position ``f + 4``, 4 being the
distance between the pivot and end of the range, namely the length of the range
``A B C D``).

Reverse
-------

For completeness, let's also provide a tail-recursive implementation of reverse:

.. code-block:: c++

    template <typename I> void reverse(I f, I l)
    {
        if (f == l) return;
        if (f == --l) return;

        swap(*f, *l);

        reverse(++f, l);
    }

We swap the first and last elements, then recurse, stopping when we either have
an empty range, or a range consisting of a single element, as it is meaningless
to swap an element with itself. Note the check for swapping an element with
itself could be pushed down to the ``swap`` function, but by design it isn't
because ``swap`` is used in many algorithms and performing this check on every
call quickly becomes expensive.

Line by line, we stop if we have an empty range:

.. code-block:: c++

    if (f == l) return;

We then decrement ``l`` so we have a closed range ``[f, l]`` to work with as
opposed to the half open ``[f, l)``. We check again that the range is not empty:

.. code-block:: c++

    if (f == --l) return;

It might be tempting to skip the first check and just perform this one, the
problem is that if ``f == l`` and we call ``--l`` we are trying to move ``l`` to
before the beginning of the range, which is undefined behavior.

Once we have both ``f`` and ``l`` pointing to the beginning and end elements in
the range, we swap their values:

.. code-block:: c++

    swap(*f, *l);

Finally, we advance ``f`` and recurse. No need to decrement ``l``, as by
convention we expect a half open range ``[f, l)`` so we take care of decrementing
``l`` inside the function as explained above.

.. code-block:: c++

    reverse(++f, l);

Summary
-------

We deconstructed sort in C++ and covered a few standard algorithms with
naïve implementations (the standard library provides highly optimized
implementations of these algorithms):

* ``sort`` can be implemented recursively by relying on ``stable_partition``
* ``stable_partition`` can be implemented recursively by relying on ``rotate``
* ``rotate`` can be implemented as three calls to ``reverse``
* ``reverse`` can be implemented recursively by relying on ``swap``

The algorithms implemented above are all 4 lines long and are implemented in a
functional style which avoids the complex loops of the C implementation
proposed on the Haskell wiki.

----

.. [#] We can also use ``bind(less<>{}, placeholders::_1, *f)`` and have a
       curried ``less`` instead of a lambda predicate.

.. [#] Haskell is an awesome language and I'm not trying to bash on it here, or
       write a C++ vs Haskell post. I was simply inspired by the Haskell wiki
       page about sort and started thinking about a modern C++ way of
       approaching the same problem.

.. comments::
