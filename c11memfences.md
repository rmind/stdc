Memory fences, atomics and C11 memory model
===========================================

27 September 2019 (upstream at: https://github.com/rmind/stdc)

Mindaugas Rasiukevicius (rmind at noxt eu)


Introduction
------------

This document is a brief summary describing the key points of the C11
memory model and key aspects of the memory fence semantics as well as
atomic operations in different systems/environments.  It is neither a
formal specification nor a comprehensive introduction into this field.

- Prerequisites: the reader is expected to be familiar with the general
concepts of lock-free programming in C, including the atomic operations,
as well as CPU cache coherence, the fact that CPU and/or compiler can
reorder memory operations, the purpose of memory fences, etc.  Basic
awareness of what has been introduced in the C11 standard is expected.

- Purpose: this document can be considered a cheat sheet for the
practitioners who work in the field of lock-free programming.  It can
serve as a brief guide on how to migrate from generally _assumed_ pre-C11
memory model (with respect to concurrency) and "classic" memory barriers
to the post-C11 world.  The dry C11/C17 standard text may leave the gaps
in understand various aspects; this document attempts to fill those gaps.

Just to recap some key points:

- Memory fences (the term "memory barrier" is used interchangeably)
prevent reordering of accesses to the main memory.  They do not provide
any guarantees of ordering when used with device memory.

- Memory fences on their own do not make sense.  They must be paired,
e.g. an acquire fence must be paired with the release fence or another
appropriate fence (e.g. full fence).  The same applies to load (read)
and store (write) fences.

- This does not cover cache coherency with the respect to devices
performing direct memory access (DMA) or memory-mapped I/O (MMIO).


Read/write memory fences
------------------------

These are the "classic" memory fences which first appeared in UNIX-like
systems and were supported by early SMP hardware.  Key points:

- The FULL FENCE orders prior loads/stores against later loads/stores.

- The READ (LOAD) FENCE imposes load ordering.
  It orders prior loads against later loads.

- The WRITE (STORE) FENCE imposes store ordering.
  It orders prior stores against later stores.

- These fences can be considered symmetric.

- There are no equivalents of READ/WRITE fences in C11.
  However, they can be safely emulated, see below.

Table of different APIs:

    +---------+----------------------+---------------------+-------------------+
    | FENCE   | C11/C17              | Linux               | Solaris/NetBSD    |
    +---------+----------------------+---------------------+-------------------+
    | READ    | not available        | smp_rmb()           | membar_consumer() |
    | WRITE   | not available        | smp_wmb()           | membar_producer() |
    | FULL    | memory_order_seq_cst | smp_mb()            | membar_sync()     |
    +---------+----------------------+---------------------+-------------------+


Acquire/release fences
----------------------

Key points:

- The ACQUIRE FENCE orders prior loads against later loads/stores.

- The RELEASE FENCE orders prior loads/stores against later stores.

- These fences can be considered asymmetric.

Table:

    +---------+----------------------+---------------------+-------------------+
    | FENCE   | C11/C17              | Linux               | Solaris/NetBSD    |
    +---------+----------------------+---------------------+-------------------+
    | ACQUIRE | memory_order_acquire | smp_load_acquire()  | not available     |
    | RELEASE | memory_order_release | smp_store_release() | membar_exit()     |
    +---------+----------------------+---------------------+-------------------+

Note: Solaris defined `membar_enter()` as `membar #StoreLoad|#StoreStore`,
while `membar_exit()` as `membar #LoadStore|#StoreStore`.

WARNING: In C11, memory fence must be paired with an atomic load/store!
See the section below.


Emulation in C11
----------------

In practice, Linux/Solaris memory READ/WRITE memory barriers can be
safely emulated with the following C11 memory fences:

    smp_rmb(), membar_consumer()  => atomic_thread_fence(memory_order_acquire)
    smp_wmb(), membar_producer()  => atomic_thread_fence(memory_order_release)
     smp_mb(), membar_sync()      => atomic_thread_fence(memory_order_seq_cst)



C11 memory_order_acq_rel
------------------------

In addition to the above, C11 has the ACQUIRE-RELEASE (`memory_order_acq_rel`)
fence, which should not be confused with a full barrier:

- It orders 1) prior loads/stores against later stores
  2) and prior loads against later loads.

- In other words: it does *not* order prior stores against later loads.

Purpose: this fence does not defeat the hardware store-buffer optimisation.
It is essentially the PowerPC `lwsync` instruction.


Using memory fences in C11/C17
------------------------------

The C11 standard (sections 5.1.2.4 and 7.17.4) essentially specifies
the fence-to-fence and fence-to-atomic synchronisation (as defined by
"synchronizes with").  Using just the memory fences is not sufficient
to synchronise non-atomic and/or relaxed atomic loads/stores, because
they have to be paired with atomic operations ("... if there exist
atomic operations ..." and ".. is sequenced before ..").  Note: the
C11 standard is careful to not define acquire/release fences as
"acquire/release operations"; instead, they have "acquire/release
semantics".

- Memory fence must be used with `atomic_{load,store}_*()`, even if
  such load/store is `memory_order_relaxed`.  Note: `memory_order_relaxed`
  means no ordering constraints for the operation per se, but the
  relevant fence would take an effect.

- In other words, the C11 standard does not allow plain accesses to
be used for shared memory synchronisation.

Example:

    Thread A:
        *val = 1;
        atomic_thread_fence(memory_order_release);
        atomic_store_explicit(published, 1, memory_order_relaxed);

    Thread B:
        if (atomic_load_explicit(published, memory_order_relaxed) == 1) {
                atomic_thread_fence(memory_order_acquire);
                assert(*val == 1); // will never fail
        }

However:

    Thread A:
        *val = 1;
        atomic_thread_fence(memory_order_release);
        *published = 1;

    Thread B:
        if (*published == 1) {
                atomic_thread_fence(memory_order_acquire);
                assert(*val == 1); // may fail
        }

There is a good reason for this -- see the section below.


On atomicity of loads/stores
----------------------------

In the past, in the pre-C11 world, the unwritten assumption in the
parallel programming was that, on the mainstream architectures, the
word-sized and aligned stores as well as loads are atomic.  This is
not true in the C11 memory model.  It is also no longer true in the
reality as compilers are implementing more aggressive optimisations,
which are allowed by the C standard.  Specifically, in addition to
code-level re-ordering, plain accesses (loads/stores) are subject to:

- LOAD/STORE TEARING: load/store transformed into into multiple
operations (e.g. subword stores).

- LOAD/STORE FUSING: using the result of a prior load(s), which can
e.g. also optimise-out loops; merging multiple stores into one.

- INVENTED LOADS/STORES: for example, pre-setting the value to
save a branch.

Hence, in C11, synchronising (atomic) loads/stores should use the
`atomic_load`/`atomic_store` routines.  They prevent some of the
above transformations by using the 'volatile' access (since these
routines are declared to take volatile parameters in C).

Note: Linux kernel has the `READ_ONCE()` and `WRITE_ONCE()` functions
as an alternative to relaxed `atomic_load()` and `atomic_store()`.


C11 "atomic object"
-------------------

C11 introduces "atomic type", which concerns objects declared/defined
using the `_Atomic` type qualifier or specifier.  These atomic types are
incompatible with their primitives.

- In C11, all variables used for synchronisation (as well as used with
the atomics API routines) need to be qualified as `_Atomic`.

For example, the following is an ill-formed program and therefore may
not compile:

    unsigned int published;
    ...
    atomic_store_explicit(&published, 1, memory_order_relaxed);

Note: while LLVM/clang gives compilation error, GCC 8.3 (latest version
as of writing this text) with `-std=c17` options produces a valid code (this
is because GCC inlines C11 atomics as its own GCC built-ins).

Assuming the `<stdatomic.h>` inclusion, the correct code would be:

    atomic_uint published;
    ...
    atomic_store_explicit(&published, 1, memory_order_relaxed);
    ...

Type incompatibility stems from the C11 standard allowing the atomic
types to be implemented within the abstract C machine in non-lock-free
ways (it helps some old or constrained architectures, e.g. DEC VAX or
SPARC which do not have CAS, only test-and-set).  Note that this can be
tested using `atomic_is_lock_free()`.

- Atomic types can be of a different size than their primitives.

In theory, large structures can be declared to be atomic and the compiler
has a right to invent locks around them.  No sane compiler would do that,
but it is allowed by the C11 standard.


Compiler-level barriers
-----------------------

    C11:          void atomic_signal_fence(memory_order order)
    GCC:          asm volatile("": : :"memory")
    NetBSD:       __insn_barrier()
    Linux:        barrier()


Data dependency barriers
------------------------

Currently, applicable only for DEC Alpha.  Note: `memory_order_consume`
specification in C11 has issues; as of writing this text, all mainstream
compilers treat `memory_order_consume` as `memory_order_acquire`.

    C11:          atomic_thread_fence(memory_order_consume)
    Linux:        smp_read_barrier_depends()
    NetBSD:       membar_datadep_consumer()


Illustration: release operation
-------------------------------

          ^   *p1 = 1;  ----------------------+
      ^   |                                   |
      |   |   v1 = *p2; ------------------+   |
      |   |                               |   |
      |   |                               v   v
    --|---|-------------------------------X---X-----------------------------
      |   |   atomic_store_explicit(&publish, true, memory_order_release);
      |   |
      |   |
      |   +-- *p3 = 1;
      |
      +------ v2 = *p4;


Illustration: acquire operation
-------------------------------

      +------ *p1 = 1;
      |
      |   +-- v1 = *p2;
      |   |
      |   |
      |   |   v = atomic_load_explicit(&publish, memory_order_acquire);
    --|---|-------------------------------X---X-----------------------------
      |   |                               ^   ^
      |   |                               |   |
      |   |   *p3 = 1;  ------------------+   |
      |   v                                   |
      v       v2 = *p4; ----------------------+


References
----------

- ISO/IEC 9899:2011, Information technology -- Programming languages -- C
- https://www.kernel.org/doc/Documentation/memory-barriers.txt
- http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0124r6.html
- https://arxiv.org/pdf/1701.00854.pdf
- https://lwn.net/Articles/793253/
- http://man.netbsd.org/cgi-bin/man-cgi?membar_ops+3+NetBSD-current

---
