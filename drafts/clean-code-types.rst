Clean Code: Types
=================

.. highlight:: c++

I recently revived my Clean Code tech talk which I put together a couple of
years ago and with which I started this blog: `Clean Code - Part 1 <https://vladris.com/blog/2016/01/04/clean-code-part-1.html>`_ 
and `Clean Code - Part 2 <https://vladris.com/blog/2016/01/07/clean-code-part-2.html>`_.
I took the opportunity to completely revamp the talk and ended up with 3 parts:
*Algorithms*, *Types*, and *State*. The *Algorithms* is mostly covered by the
`Fibonacci <https://vladris.com/blog/2018/02/11/fibonacci.html>`_ post, so I
will start with *Types*.

Mars Climate Orbiter
--------------------

The Mars Climate Orbiter crashed and disintegrated in the Mars atmosphere
because a component developed by Lockheed provided momentum measured in
pound-force seconds, while another component developed by NASA expected momentum
as Newton seconds.

We can image the component developed by NASA being something like this::

    // Will not disintegrate as long as momentum >= 2 N s
    void trajectory_correction(double momentum)
    {
        if (momentum < 2 /* N s */)
        {
            disintegrate();
        }
        /* ... */
    }

We can also imagine the Lockheed component calling into the above with::

    void main()
    {
        trajectory_correction(1.5 /* lbf s */);
    }

A pound-force second (lbfs) is about 4.448222 Newton seconds (Ns). So from
Lockheed's perspective, passing in 1.5 lbfs to ``trajectory_correction`` should
be just fine: 1.5 lbfs is about 6.672333 Ns, way above the 2 Ns threshold.

The problem is the interpretation of the data. The NASA component ends up
comparing lbfs to Ns without conversion, misinterpreting the lbfs input as Ns.
1.5 is less than 2, thus the orbiter disintegrates. This is a known anti-pattern
called "Primitive Obsession"

Primitive Obsession
-------------------

Primitive obsession happens when we use a primitive data type to represent a
value in the problem's domain and causes situations like the above. Representing
zip codes as numbers, telephone numbers as strings, Ns and lbfs as ``double``
are all examples of this.

A more type safe solution would have defined a simple ``Ns`` type::

    struct Ns
    {
        double value;
    };

    bool operator<(const Ns& a, const Ns& b)
    {
        return a.value < b.value;
    }

We can simlarly define a simple ``lbfs`` type::

    struct lbfs
    {
        double value;
    };

    bool operator<(const lbfs& a, const lbfs& b)
    {
        return a.value < b.value;
    }

Now we can implement a type safe ``trajectory_correction``::

    // Will not disintegrate as long as momentum >= 2 N s
    void trajectory_correction(Ns momentum)
    {
        if (momentum < Ns{ 2 })
        {
            disintegrate();
        }
        /* ... */
    }

Calling this with ``lbfs`` as below fails to compile as the types are
incompatible::

    void main()
    {
        trajectory_correction(lbfs{ 1.5 });
    }

Note how the meaning of the values, which used to be specified in comments
(``2 /* Ns */``, ``/* lbfs */``) gets pulled into the type system and expressed
in code (``Ns{ 2 }``, ``lbfs{ 1.5 }``).

We can, of course, provide casting from ``lbfs`` to ``Ns`` as an explicit
operator::

    struct lbfs
    {
        double value;

        explicit operator Ns()
        {
            return value * 4.448222;
        }
    };

Equipped with this, we can call ``trajectory_correction`` via a static cast::

    void main()
    {
        trajectory_correction(static_cast<Ns>(lbfs{ 1.5 }));
    }

This does the right thing of multiplying by the ratio. The cast can also be
made implicit (by using the ``implicit`` keyword instead), in which case it is
applied automatically. As a rule of thumb, it's best to follow the Zen of
Python:

    Explicit is better than implicit 

.. comments::
