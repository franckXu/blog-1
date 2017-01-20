Memory Management
=================

Memory management involves handling memory resources allocated for a certain
task, ensuring that the memory is freed once it is no longer needed so it can
be reused for some other task. If the memory is not freed in a timely manner,
the system might run out of resources or incur degraded performance. A memory
resource that is never freed once no longer needed is called a leak - the
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

Automatic memory management attempts to move the responsibility of tracking
when a memory resource is no longer needed (and handling its deallocation) from
the developer to the system. Such a system is called *garbage collected*, as
memory that is no longer needed ("garbage") is reclaimed by the system
automatically. The two most popular methods used to automatically free memory
are *tracing garbage collectors* and *reference counting*.

Tracing Garbage Collector
~~~~~~~~~~~~~~~~~~~~~~~~~

Tracing garbage collectors work by tracing references to objects on the heap
and checking whether a given resource allocated on the heap has at least one
reference path to it from the stack. If such a path exists, it means that from
the stack (an argument to a function, a local variable), there is a way to
perform a set of dereference and access the memory resource. If such a path
doesn’t exist, it means the memory is unreachable, so regardless of how
executing code accesses other objects on the heap, there is no way to access
this resource - which means the memory can be safely deallocated.

For example, a naïve tracing garbage collection algorithm, *mark-and-sweep*,
involves adding an “in-use” bit to each memory resource allocated then, during
collection, following all references starting from the stack and marking each as
"in-use". Once all used resources are marked, the sweep stage involves walking
the whole heap and for each memory resource, if not marked as "in-use", freeing
it.

Tracing garbage collectors are used by many popular runtimes, like JVM and .NET.
In C#:

.. code-block:: C#

    struct Bar { }

    struct Foo
    {
        public Bar bar;
    }

    ...

    {
        Foo foo = new Foo();
        foo.bar = new Bar();

        // there is no stack variable pointing to the Bar object, but it can
        // still be reached through foo (foo.bar), so there exists a path from
        // the stack to it, meaning code can still access it.
    }

    // foo goes out of scope which means neither foo nor its member Bar can be
    // accessed any longer, so they can be safely collected

There are a couple of disadvantages with the tracing GC approach: first, the
system needs to ensure memory resources are not being allocated while a garbage
collection is taking place. This means code execution is paused during
collection, which obviously impacts performance. The second disadvantage of
this approach is that the system is not as lean as other memory management
models: memory resources are kept allocated longer than really needed, for the
time interval between the last reference to them goes out of scope until the
actual collection is performed.

Reference Counting
~~~~~~~~~~~~~~~~~~

An alternative to tracing garbage collectors is reference counting. As the name
implies, a memory resource in such a system has an associated reference count -
the number of references to it. As soon as the last reference goes out of scope,
when the reference count reaches zero, the memory can be safely deallocated.
Unlike tracing, reference counting is performed as code executes: the count of a
given memory resource is automatically increased with each assignment where the
resource is on the right-hand-side, and is automatically decreased whenever a
reference goes out of scope.

Python manages memory using reference counting:

.. code-block:: python

    class Foo: pass

    # allocate Foo, its reference count is 1
    foo1 = Foo()

    # reference count is 2 after assignment
    foo2 = foo1

    ...
    # once foo1 and foo2 go out of scope, reference count becomes 0 and memory
    # is automatically freed

C++ smart pointers work in a similar manner:

.. code-block:: c++

    struct Foo { };

    ...

    // foo1 is a shared_ptr pointing to a Foo stored on the heap. Reference
    // count for the Foo object is 1
    auto foo1 = std::make_shared<Foo>();

    // reference count becomes 2 after assignment
    auto foo2 = foo1;

    ...
    // Once foo1 and foo2 go out of scope, reference count becomes 0 and memory
    // is automatically freed


The main advantages over tracing garbage collection are the fact that execution
doesn’t need to be paused in order to reclaim memory and that resources are
deallocated as soon as they are no longer used (once reference count becomes 0).
There are also several disadvantages with this approach: first, each memory
resource needs to store an additional reference count and updating the reference
count in a multi-threaded environment needs to be performed atomically. Second,
and most important, this memory management model does not handle *reference
cycles*.

Ownership and Lifetimes
~~~~~~~~~~~~~~~~~~~~~~~

Ownership and lifetime concepts. C++ unique_ptr and refs. Pros and cons.

Rust memory model.

.. comments::
