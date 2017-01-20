Memory Management
=================

Memory management involves handling memory resources allocated for a certain
task, ensuring that the memory is freed once it is no longer needed so it can
be reused for some other task. If the memory is not freed in a timely manner,
the system might run out of resources or incur degraded performance. A memory
resource that is never freed once no longer needed is called a leak – the
resource becomes unusable, usually for the duration of the process. Another
issue is *use after free*, in which a memory resource that was already freed
is used as if it wasn’t. This usually causes unexpected behavior as the code
is trying to read, modify or wrongly interpret data at a memory location.
Memory management can be *manual* - with code explicitly handling
deallocation, or *automatic*, in which memory gets freed once no longer needed
by an automated process.

Manual Memory Management
------------------------

Manual memory management is efficient, since allocations and deallocations
don’t incur any overhead. In C:

.. code-block:: c

    typedef struct _Foo {
        ...
    } Foo;

    ...

    Foo* foo = (Foo*)malloc(sizeof(Foo));

    ...

    free(foo);

The disadvantage of this approach, and the main reason automatic memory
management models were invented, is that this puts the developer in charge of
making sure memory doesn’t leak and that it is not used after it is freed. As
the complexity of the code increases, this becomes increasingly difficult. As
pointers are passed around the system and get stored in various data structures,
it becomes difficult to know given some pointer that is no longer needed
whether: a) this was the very last piece of code that actually needed to access
the location pointed to by this pointer, in which case the memory should be
freed, and b) whether the memory this pointer is pointing to is still valid and
hasn’t been freed previously.

Automatic Memory Management
---------------------------

Types of automatic memory management: tracing GC, reference counting.

Tracing Garbage Collector
~~~~~~~~~~~~~~~~~~~~~~~~~

Describe tracing GC with pros and cons.

Reference Counting
~~~~~~~~~~~~~~~~~~

Describe reference counting with pros and cons. Reference cycles.

**Python** approach: reference counting supplemented by GC to avoid cycles.

**C++** approach: reference counting with additional mechanisms: non-owning
pointers, weak_ptr.

Ownership and Lifetimes
~~~~~~~~~~~~~~~~~~~~~~~

Ownership and lifetime concepts. C++ unique_ptr and refs. Pros and cons.

Rust memory model.

.. comments::
