Clean Code - Part 2
===================

Write Stateless Code
--------------------

    A **pure function** is a **function** where the return value is only
    determined by its input values, without observable side effects. This is
    how **functions** in math work: ``Math.cos(x)`` will, for the same value
    of ``x``, always return the same result.

    *Source:* http://www.sitepoint.com/functional-programming-pure-functions/

Things break when a program gets into a "bad state". There are a couple of ways
to make this less likely to happen: making data immutable and writing functions
that don't have side-effects (*pure* functions).

Immutable data
~~~~~~~~~~~~~~

If something shouldn't change, mark it as immutable and let the compiler enforce
that. A good rule of thumb is to mark things as ``const`` (``const&``,
``const*`` etc.) and/or ``readonly`` by default, and make them mutable only when
truly needed [*]_.

A simple example:

.. code-block:: c++

    struct StringUtil
    {
        static std::string Concat1(std::string& str1, std::string& str2)
        {
            return str1 + str2;
        }

        static std::string Concat2(std::string& str1, std::string& str2)
        {
            return str1.append(str2);
        }
    };

Both ``StringUtil::Concat1`` and ``StringUtil::Concat2`` return the same thing
for the same input, the difference being that ``Concat2``, as opposed to
``Concat1``, modifies its first argument. In a bigger function, such a change
might be introduced accidentally and have unexpected consequences down the line.

A simple way to address this is by explicitly marking the arguments as
``const``:

.. code-block:: c++

    struct StringUtil
    {
        static std::string Concat1(const std::string& str1, std::string& str2)
        {
             return str1 + str2;
        }

        static std::string Concat2(const std::string& str1, std::string& str2)
        {
             return str1.append(str2); // Won't compile - can't call append on str1
        }
    };

In this case, ``Concat2`` won't compile, so we can rely on the compiler to
eliminate this type of unintended behavior.

Another example: a simple ``UpCase`` function which calls ``toupper`` on each
character of the given string, upcasing it in place:

.. code-block:: c++

    void UpCase(char *string)
    {
        while (*string)
        {
            *string = toupper(*string);
            string++;
        }
    };

Calling it with

.. code-block:: c++

    char* myString = "Foo";
    ...
    UpCase(myString);

will lead to a crash at runtime - the function will try to call ``toupper`` on
the characters of ``myString``. The problem is that ``myString`` is a ``char*``
to a string literal which gets compiled into the read-only data segment of the
binary. This cannot be modified.

To catch this type of errors at compile-time, we again only need to mark the
immutable data as such:

.. code-block:: c++

    const char* myString = "Foo";
    ...
    UpCase(myString); // Won't compile - can't call UpCase on myString

In contrast with the previous example, the argument to ``UpCase`` is mutable by
design (the API is modifying the string in-place), but marking ``myString`` as
``const`` tells the complier this is non-mutable data, so it can't be used with
this API.

Pure functions
~~~~~~~~~~~~~~

Another way to reduce states is to use pure functions. Unfortunately there isn't
a lot of syntax-level support for this in C++ and C# (C++ supports ``const``
member functions, which guarantee at compile time that calling the member
function on an instance of the type won't change the attributes of that
instance) [*]_

This goes back to the recommendation from Part 1 of using generic algorithms and
predicates rather than implementing raw loops. In many cases, traversal state is
encapsulated in the library algorithm or in an iterator, and predicates ideally
don't have side-effects.

.. code-block:: c#

    var squares = numbers.
                    Where(number => number % 2 != 0).
                    Select(number => number * number);

Above code (also from Part 1) doesn't hold any state: traversal is handled by
the Linq methods, the predicates are pure.

In general, try to encapsulate state in parts of the code built to manage state,
and keep the rest stateless. Functional languages are great at keeping functions
pure and data immutable, the tradeoff there being that transformations incur
data copy, which comes with a performance penalty.

Note that immutable data and pure functions are also an advantage in concurrent
applications, since they can't generate race conditions.

Key takeaways:

- Prefer pure functions to stateful functions and, if state is needed, keep it
  contained
- By default mark everything as ``const`` (or ``readonly``), and only remove
  the constraint when mutability is explicitly needed

Write Readable Code
-------------------

    In computer science, the **expressive power** (also called
    **expressiveness** or expressivity) of a language is the breadth of ideas
    that can be represented and communicated in that language [...]

    - regardless of ease (theoretical expressivity)
    - **concisely and readily** (practical expressivity)

    *Source:* https://en.wikipedia.org/wiki/Expressive_power_(computer_science)

----

.. [*] At the time of this writing, there is an `active proposals <https://github.com/dotnet/roslyn/issues/7626>`_
   to extend the C# language with an ``immutable`` keyword.

.. [*] C# has a ``PureAttribue`` in the ``System.Diagnostics.Contracts``
   namespace (purity not compiler-enforced) and there is an `active proposal <https://github.com/dotnet/roslyn/issues/7561>`_
   to add a keyword for it too.

.. comments::
