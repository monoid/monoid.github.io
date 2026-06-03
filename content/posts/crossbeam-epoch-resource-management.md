---
title: "Managing resources with crossbeam-epoch"
date: 2026-06-06T09:25:35+02:00
draft: true
---

`crossbeam-epoch` is a crate for "epoch-based memory reclamation". But the
problem of concurrent reads and removals arises with many kind of resources --
memory is just one of those. Can `crossbeam-epoch` help us?

# A sample problem

Imaging a flat piece of memory mmap'ed to a file and divided into cells. Some
cells are free, and writers may (re-)use them. Some writers are just deleting
cells, making them free. And readers gonna read the allocated cells, randomly or
sequentially. A bitmask defines if the cell is allocated.

## Mutexes and locks

We may just put the memory under RW-lock. Reader hold read-locks, writers hold a
write-lock. Update is just overwriting the cell.

However, if readers are slow and many, writers will wait for them, and with
fairness implementations like `parking_lot`, subsequent readers will form a
convoy waiting for the waiting writer.

## Making it atomic
<!-- TODO use allocating/deallocating instead of using/freeing. -->

Alternative architecture might be something like this.

1. Validity bitmask is an array of atomics.  Allocating or deallocating the cell
   is just setting atomically a bit flag.
2. Writers should coordinate with each other (at most one writer should do its
   operation), but not with readers.
3. Updating a cell is copy on write: remove the cell and insert updated one.
4. Deallocating a cell is setting the bitmask as free and adding its index
   to a free list.
5. Allocaing a cell is getting its index from a free list, initializing it and
   making it as allocated in the bitmask.

And we have a problem with deletion similar to what `crossbeam-epoch` is
solving: what if the reader reads a cell being deleted, how can we prevent a
next writer to write to that cell? The epoch machinery can solve this problem:
adding the cell to free list (and runnign `Drop` if any) must be deferred until
readers' epoch is complete, while the cell is immediately marked deallocated in
the bitmask. Thus subsequent readers (with later epochs) will not be able to use
it and interfere with writers, until someone re-initializes it and marks as
allocated.

## First approach


