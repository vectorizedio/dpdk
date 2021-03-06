..  SPDX-License-Identifier: BSD-3-Clause
    Copyright(c) 2019 Arm Limited.

.. _RCU_Library:

RCU Library
============

Lockless data structures provide scalability and determinism.
They enable use cases where locking may not be allowed
(for example real-time applications).

In the following sections, the term "memory" refers to memory allocated
by typical APIs like malloc() or anything that is representative of
memory, for example an index of a free element array.

Since these data structures are lockless, the writers and readers
are accessing the data structures concurrently. Hence, while removing
an element from a data structure, the writers cannot return the memory
to the allocator, without knowing that the readers are not
referencing that element/memory anymore. Hence, it is required to
separate the operation of removing an element into two steps:

#. Delete: in this step, the writer removes the reference to the element from
   the data structure but does not return the associated memory to the
   allocator. This will ensure that new readers will not get a reference to
   the removed element. Removing the reference is an atomic operation.

#. Free (Reclaim): in this step, the writer returns the memory to the
   memory allocator only after knowing that all the readers have stopped
   referencing the deleted element.

This library helps the writer determine when it is safe to free the
memory by making use of thread Quiescent State (QS).

What is Quiescent State
-----------------------

Quiescent State can be defined as "any point in the thread execution where the
thread does not hold a reference to shared memory". It is up to the application
to determine its quiescent state.

Let us consider the following diagram:

.. _figure_quiescent_state:

.. figure:: img/rcu_general_info.*

   Phases in the Quiescent State model.


As shown in :numref:`figure_quiescent_state`, reader thread 1 accesses data
structures D1 and D2. When it is accessing D1, if the writer has to remove an
element from D1, the writer cannot free the memory associated with that
element immediately. The writer can return the memory to the allocator only
after the reader stops referencing D1. In other words, reader thread RT1 has
to enter a quiescent state.

Similarly, since reader thread 2 is also accessing D1, the writer has to
wait till thread 2 enters quiescent state as well.

However, the writer does not need to wait for reader thread 3 to enter
quiescent state. Reader thread 3 was not accessing D1 when the delete
operation happened. So, reader thread 1 will not have a reference to the
deleted entry.

It can be noted that, the critical sections for D2 is a quiescent state
for D1. i.e. for a given data structure Dx, any point in the thread execution
that does not reference Dx is a quiescent state.

Since memory is not freed immediately, there might be a need for
provisioning of additional memory, depending on the application requirements.

Factors affecting the RCU mechanism
-----------------------------------

It is important to make sure that this library keeps the overhead of
identifying the end of grace period and subsequent freeing of memory,
to a minimum. The following explains how grace period and critical
section affect this overhead.

The writer has to poll the readers to identify the end of grace period.
Polling introduces memory accesses and wastes CPU cycles. The memory
is not available for reuse during the grace period. Longer grace periods
exasperate these conditions.

The length of the critical section and the number of reader threads
is proportional to the duration of the grace period. Keeping the critical
sections smaller will keep the grace period smaller. However, keeping the
critical sections smaller requires additional CPU cycles (due to additional
reporting) in the readers.

Hence, we need the characteristics of a small grace period and large critical
section. This library addresses this by allowing the writer to do
other work without having to block until the readers report their quiescent
state.

RCU in DPDK
-----------

For DPDK applications, the start and end of a ``while(1)`` loop (where no
references to shared data structures are kept) act as perfect quiescent
states. This will combine all the shared data structure accesses into a
single, large critical section which helps keep the overhead on the
reader side to a minimum.

DPDK supports a pipeline model of packet processing and service cores.
In these use cases, a given data structure may not be used by all the
workers in the application. The writer does not have to wait for all
the workers to report their quiescent state. To provide the required
flexibility, this library has a concept of a QS variable. The application
can create one QS variable per data structure to help it track the
end of grace period for each data structure. This helps keep the grace
period to a minimum.

How to use this library
-----------------------

The application must allocate memory and initialize a QS variable.

Applications can call ``rte_rcu_qsbr_get_memsize()`` to calculate the size
of memory to allocate. This API takes a maximum number of reader threads,
using this variable, as a parameter. Currently, a maximum of 1024 threads
are supported.

Further, the application can initialize a QS variable using the API
``rte_rcu_qsbr_init()``.

Each reader thread is assumed to have a unique thread ID. Currently, the
management of the thread ID (for example allocation/free) is left to the
application. The thread ID should be in the range of 0 to
maximum number of threads provided while creating the QS variable.
The application could also use ``lcore_id`` as the thread ID where applicable.

The ``rte_rcu_qsbr_thread_register()`` API will register a reader thread
to report its quiescent state. This can be called from a reader thread.
A control plane thread can also call this on behalf of a reader thread.
The reader thread must call ``rte_rcu_qsbr_thread_online()`` API to start
reporting its quiescent state.

Some of the use cases might require the reader threads to make blocking API
calls (for example while using eventdev APIs). The writer thread should not
wait for such reader threads to enter quiescent state.  The reader thread must
call ``rte_rcu_qsbr_thread_offline()`` API, before calling blocking APIs. It
can call ``rte_rcu_qsbr_thread_online()`` API once the blocking API call
returns.

The writer thread can trigger the reader threads to report their quiescent
state by calling the API ``rte_rcu_qsbr_start()``. It is possible for multiple
writer threads to query the quiescent state status simultaneously. Hence,
``rte_rcu_qsbr_start()`` returns a token to each caller.

The writer thread must call ``rte_rcu_qsbr_check()`` API with the token to
get the current quiescent state status. Option to block till all the reader
threads enter the quiescent state is provided. If this API indicates that
all the reader threads have entered the quiescent state, the application
can free the deleted entry.

The APIs ``rte_rcu_qsbr_start()`` and ``rte_rcu_qsbr_check()`` are lock free.
Hence, they can be called concurrently from multiple writers even while
running as worker threads.

The separation of triggering the reporting from querying the status provides
the writer threads flexibility to do useful work instead of blocking for the
reader threads to enter the quiescent state or go offline. This reduces the
memory accesses due to continuous polling for the status.

The ``rte_rcu_qsbr_synchronize()`` API combines the functionality of
``rte_rcu_qsbr_start()`` and blocking ``rte_rcu_qsbr_check()`` into a single
API. This API triggers the reader threads to report their quiescent state and
polls till all the readers enter the quiescent state or go offline. This API
does not allow the writer to do useful work while waiting and introduces
additional memory accesses due to continuous polling.

The reader thread must call ``rte_rcu_qsbr_thread_offline()`` and
``rte_rcu_qsbr_thread_unregister()`` APIs to remove itself from reporting its
quiescent state. The ``rte_rcu_qsbr_check()`` API will not wait for this reader
thread to report the quiescent state status anymore.

The reader threads should call ``rte_rcu_qsbr_quiescent()`` API to indicate that
they entered a quiescent state. This API checks if a writer has triggered a
quiescent state query and update the state accordingly.

The ``rte_rcu_qsbr_lock()`` and ``rte_rcu_qsbr_unlock()`` are empty functions.
However, when ``CONFIG_RTE_LIBRTE_RCU_DEBUG`` is enabled, these APIs aid
in debugging issues. One can mark the access to shared data structures on the
reader side using these APIs. The ``rte_rcu_qsbr_quiescent()`` will check if
all the locks are unlocked.
