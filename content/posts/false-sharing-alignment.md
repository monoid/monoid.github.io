---
title: "False Sharing Alignment"
date: 2026-04-27T15:02:20+02:00
draft: true
tags: ["atomic"]
---

# What is false sharing

When CPUs and their cores read and update atomic variables, special hardware
protocols make it correct and efficient, while keeping each core's caches consistent.
The coordination doesn't happen per-address: memory always works
with cache-line-sized chunks.

What happens when two atomic variables happen to reside on the same cache line?
For example:

- an array of atomics where each atomic is used by a separate thread (e.g. per-thread counters);
- a queue with two atomic pointers where a writer updates the tail and a
  reader updates the head. Of course, you define both pointers in the same
  struct and they have adjacent addresses!

Each update operation makes the cache line dirty and requires coordination
between multiple CPUs.

# What to do

The solution is simple: separate the atomics by making them reside
in different cache lines. It is done with some language-dependent alignment
directive, though it is not the alignment, but the spacing matters.

Both _Rust Atomics and Locks_ by Mara Bos and _Performance Analysis and Tuning on
Modern CPUs_ by Denis Bakhvalov recommend using cache line size, which is
64 bytes for `x86_64`.
However, libraries ([`crossbeam-utils`](https://github.com/crossbeam-rs/crossbeam/blob/822eb3abf00094fab5d817538aab42477c62c42a/crossbeam-utils/src/cache_padded.rs#L82-L90), Facebook's
[`folly`](https://github.com/facebook/folly/blob/4d6b3c7999940ddf2e9c1eb9ed2c9cf3d8c2280f/folly/lang/Align.h#L186-L187))
use 128 bytes. Why?

The reason is that since the Sandy Bridge architecture, the spatial prefetcher[^prefetcher] in modern Intel and AMD CPUs' may load cache lines _in pairs_. It doesn't make the effective cache line size equal to 128 bytes, but still has its side-effects.

[^prefetcher]: A spatial prefetcher tries to predict random memory access patterns, and a streaming prefetcher handles sequential ones.

Moreover, this behavior is controlled by a per-core MSR setting called either "Adjacent Cache Line Prefetcher Disable" or "L2 Adjacent Cache Line Prefetcher Disable". Check before benchmarking!

# Benchmark

I tried to reproduce the 128-byte aligment improvement on a real benchmark. It wasn't easy! Just starting 2 threads updating an atomic each is not enough: once the atomics are in the respective CPU caches, no MESI interaction is done. Let's try something more involved: a vector of atomics should do the trick.

```rust
use std::ops::Deref;
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};

// Uncomment any of thses lines.
// #[repr(align(64))]
// #[repr(align(128))]
#[derive(Default, Debug)]
struct CachePadded<T> {
    value: T,
}

impl<T> Deref for CachePadded<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.value
    }
}

// Two atomics side-by-side separated by cache padding configured above.
#[derive(Default, Debug)]
struct AddSub {
    // Updated by the first thread.
    add: CachePadded<AtomicU64>,
    // Updated by the second thread.
    sub: CachePadded<AtomicU64>,
}

impl AddSub {
    fn get_value(&self) -> u64 {
        let add = self.add.load(Ordering::Relaxed);
        let sub = self.sub.load(Ordering::Relaxed);
        add.wrapping_sub(sub)
    }
}

fn main() {
    const N: usize = 1_000_000_000;
    const ORDERING: Ordering = Ordering::AcqRel;
    const SIZE: usize = 8;

    let data = Vec::from_iter(std::iter::repeat_with(|| AddSub::default()).take(SIZE));
    let addsub: Arc<[AddSub]> = data.into();
    let addsub1 = addsub.clone();
    let addsub2 = addsub.clone();

    let t1 = std::thread::spawn(move || {
        affinity::set_thread_affinity(&[0]).unwrap();
        for i in 0..N {
            addsub1[i % SIZE].add.fetch_add(i as u64, ORDERING);
        }
    });
    let t2 = std::thread::spawn(move || {
        // Not 1! Core 1 is a virtual core of the same physical one.
        affinity::set_thread_affinity(&[2]).unwrap();
        for i in 0..N {
            addsub2[i % SIZE].sub.fetch_add(i as u64, ORDERING);
        }
    });
    t1.join().unwrap();
    t2.join().unwrap();

    for item in &*addsub {
        eprintln!("{}", item.get_value());
    }
}
```

```
Benchmark 1: ./false_sharing_8_noalign
  Time (mean ± σ):     22.548 s ±  1.376 s    [User: 44.942 s, System: 0.002 s]
  Range (min … max):   21.032 s … 24.943 s    10 runs

Benchmark 1: ./false_sharing_8_64
  Time (mean ± σ):      5.840 s ±  0.014 s    [User: 11.372 s, System: 0.001 s]
  Range (min … max):    5.815 s …  5.865 s    10 runs

Benchmark 1: ./false_sharing_8_128
  Time (mean ± σ):      5.544 s ±  0.021 s    [User: 11.075 s, System: 0.002 s]
  Range (min … max):    5.514 s …  5.578 s    10 runs

Intel(R) Xeon(R) Platinum 8280 at DigitalOcean droplet.
```

# Conclusion

Modern processors are complex but unpredictable beasts. They make our lives harder trying to make our lifes easier. Benchmarking them is a minefield.

# Reading list

1. Mara Bos, _Rust Atomics and Locks_. O'Reilly Media, 2023. ISBN 978-1098119447.
2. Denis Bakhvalov, _Performance Analysis and Tuning on Modern CPUs_. 2024, ISBN 979-8869584229.
3. [Intel® 64 and IA-32 Architectures Software Developer’s Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html).
