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
# Intel(R) Xeon(R) Platinum 8280 CPU @ 2.70GHz at DigitalOcean droplet.
$ hyperfine ./false_sharing_* --runs 20

Benchmark 1: ./false_sharing_8_noalign
  Time (mean ± σ):     22.548 s ±  1.376 s    [User: 44.942 s, System: 0.002 s]
  Range (min … max):   21.032 s … 24.943 s    10 runs

Benchmark 1: ./false_sharing_8_64
  Time (mean ± σ):      5.840 s ±  0.014 s    [User: 11.372 s, System: 0.001 s]
  Range (min … max):    5.815 s …  5.865 s    10 runs

Benchmark 1: ./false_sharing_8_128
  Time (mean ± σ):      5.544 s ±  0.021 s    [User: 11.075 s, System: 0.002 s]
  Range (min … max):    5.514 s …  5.578 s    10 runs
--------------------------------------------------------------------------------
$ hyperfine ./false_sharing_* --runs 20
Benchmark 1: ./false_sharing_1_128
  Time (mean ± σ):      4.929 s ±  0.015 s    [User: 9.851 s, System: 0.001 s]
  Range (min … max):    4.911 s …  4.967 s    20 runs
 
Benchmark 2: ./false_sharing_1_64
  Time (mean ± σ):      4.932 s ±  0.015 s    [User: 9.857 s, System: 0.001 s]
  Range (min … max):    4.918 s …  4.984 s    20 runs
 
Benchmark 3: ./false_sharing_2_128
  Time (mean ± σ):      4.928 s ±  0.009 s    [User: 9.851 s, System: 0.001 s]
  Range (min … max):    4.917 s …  4.956 s    20 runs
 
Benchmark 4: ./false_sharing_2_64
  Time (mean ± σ):      4.923 s ±  0.010 s    [User: 9.841 s, System: 0.001 s]
  Range (min … max):    4.910 s …  4.943 s    20 runs
 
Benchmark 5: ./false_sharing_3_128
  Time (mean ± σ):      5.472 s ±  0.015 s    [User: 10.940 s, System: 0.001 s]
  Range (min … max):    5.455 s …  5.524 s    20 runs
 
Benchmark 6: ./false_sharing_3_64
  Time (mean ± σ):      5.746 s ±  0.010 s    [User: 11.212 s, System: 0.001 s]
  Range (min … max):    5.730 s …  5.775 s    20 runs
 
Benchmark 7: ./false_sharing_4_128
  Time (mean ± σ):      5.196 s ±  0.012 s    [User: 10.114 s, System: 0.001 s]
  Range (min … max):    5.181 s …  5.236 s    20 runs
 
Benchmark 8: ./false_sharing_4_64
  Time (mean ± σ):      5.207 s ±  0.017 s    [User: 10.407 s, System: 0.001 s]
  Range (min … max):    5.184 s …  5.260 s    20 runs
 
Benchmark 9: ./false_sharing_5_128
  Time (mean ± σ):      5.475 s ±  0.012 s    [User: 10.944 s, System: 0.001 s]
  Range (min … max):    5.459 s …  5.504 s    20 runs
 
Benchmark 10: ./false_sharing_5_64
  Time (mean ± σ):      5.476 s ±  0.012 s    [User: 10.946 s, System: 0.001 s]
  Range (min … max):    5.460 s …  5.510 s    20 runs
 
Benchmark 11: ./false_sharing_6_128
  Time (mean ± σ):      5.620 s ±  0.034 s    [User: 11.233 s, System: 0.001 s]
  Range (min … max):    5.596 s …  5.756 s    20 runs
 
Benchmark 12: ./false_sharing_6_64
  Time (mean ± σ):      5.630 s ±  0.023 s    [User: 11.254 s, System: 0.002 s]
  Range (min … max):    5.594 s …  5.676 s    20 runs
 
Benchmark 13: ./false_sharing_7_128
  Time (mean ± σ):      5.892 s ±  0.025 s    [User: 11.777 s, System: 0.001 s]
  Range (min … max):    5.868 s …  5.977 s    20 runs
 
Benchmark 14: ./false_sharing_7_64
  Time (mean ± σ):      5.893 s ±  0.015 s    [User: 11.778 s, System: 0.001 s]
  Range (min … max):    5.866 s …  5.920 s    20 runs
 
Benchmark 15: ./false_sharing_8_128
  Time (mean ± σ):      5.225 s ±  0.018 s    [User: 10.169 s, System: 0.001 s]
  Range (min … max):    5.199 s …  5.264 s    20 runs
 
Benchmark 16: ./false_sharing_8_64
  Time (mean ± σ):      5.227 s ±  0.019 s    [User: 10.446 s, System: 0.001 s]
  Range (min … max):    5.200 s …  5.282 s    20 runs
 
Summary
  ./false_sharing_2_64 ran
    1.00 ± 0.00 times faster than ./false_sharing_2_128
    1.00 ± 0.00 times faster than ./false_sharing_1_128
    1.00 ± 0.00 times faster than ./false_sharing_1_64
    1.06 ± 0.00 times faster than ./false_sharing_4_128
    1.06 ± 0.00 times faster than ./false_sharing_4_64
    1.06 ± 0.00 times faster than ./false_sharing_8_128
    1.06 ± 0.00 times faster than ./false_sharing_8_64
    1.11 ± 0.00 times faster than ./false_sharing_3_128
    1.11 ± 0.00 times faster than ./false_sharing_5_128
    1.11 ± 0.00 times faster than ./false_sharing_5_64
    1.14 ± 0.01 times faster than ./false_sharing_6_128
    1.14 ± 0.01 times faster than ./false_sharing_6_64
    1.17 ± 0.00 times faster than ./false_sharing_3_64
    1.20 ± 0.01 times faster than ./false_sharing_7_128
    1.20 ± 0.00 times faster than ./false_sharing_7_64

--------------------------------------------------------------------------------
# hyperfine ./false_sharing_?_* ./false_sharing_16_{64,128} --runs 20
Benchmark 1: ./false_sharing_1_128
  Time (mean ± σ):      4.942 s ±  0.015 s    [User: 9.877 s, System: 0.001 s]
  Range (min … max):    4.917 s …  4.967 s    20 runs
 
Benchmark 2: ./false_sharing_1_64
  Time (mean ± σ):      4.934 s ±  0.015 s    [User: 9.860 s, System: 0.002 s]
  Range (min … max):    4.912 s …  4.965 s    20 runs
 
Benchmark 3: ./false_sharing_2_128
  Time (mean ± σ):      4.927 s ±  0.012 s    [User: 9.845 s, System: 0.001 s]
  Range (min … max):    4.913 s …  4.947 s    20 runs
 
Benchmark 4: ./false_sharing_2_64
  Time (mean ± σ):      4.934 s ±  0.017 s    [User: 9.860 s, System: 0.002 s]
  Range (min … max):    4.915 s …  4.981 s    20 runs
 
Benchmark 5: ./false_sharing_3_128
  Time (mean ± σ):      5.475 s ±  0.015 s    [User: 10.941 s, System: 0.002 s]
  Range (min … max):    5.458 s …  5.510 s    20 runs
 
Benchmark 6: ./false_sharing_3_64
  Time (mean ± σ):      5.749 s ±  0.026 s    [User: 11.219 s, System: 0.001 s]
  Range (min … max):    5.728 s …  5.847 s    20 runs
 
Benchmark 7: ./false_sharing_4_128
  Time (mean ± σ):      5.206 s ±  0.016 s    [User: 10.133 s, System: 0.002 s]
  Range (min … max):    5.182 s …  5.249 s    20 runs
 
Benchmark 8: ./false_sharing_4_64
  Time (mean ± σ):      5.199 s ±  0.015 s    [User: 10.392 s, System: 0.001 s]
  Range (min … max):    5.184 s …  5.238 s    20 runs
 
Benchmark 9: ./false_sharing_5_128
  Time (mean ± σ):      5.471 s ±  0.010 s    [User: 10.935 s, System: 0.001 s]
  Range (min … max):    5.458 s …  5.491 s    20 runs
 
Benchmark 10: ./false_sharing_5_64
  Time (mean ± σ):      5.491 s ±  0.020 s    [User: 10.975 s, System: 0.001 s]
  Range (min … max):    5.463 s …  5.537 s    20 runs
 
Benchmark 11: ./false_sharing_6_128
  Time (mean ± σ):      5.628 s ±  0.013 s    [User: 11.249 s, System: 0.002 s]
  Range (min … max):    5.602 s …  5.649 s    20 runs
 
Benchmark 12: ./false_sharing_6_64
  Time (mean ± σ):      5.635 s ±  0.016 s    [User: 11.264 s, System: 0.001 s]
  Range (min … max):    5.610 s …  5.672 s    20 runs
 
Benchmark 13: ./false_sharing_7_128
  Time (mean ± σ):      5.881 s ±  0.012 s    [User: 11.755 s, System: 0.001 s]
  Range (min … max):    5.864 s …  5.910 s    20 runs
 
Benchmark 14: ./false_sharing_7_64
  Time (mean ± σ):      5.880 s ±  0.012 s    [User: 11.754 s, System: 0.001 s]
  Range (min … max):    5.866 s …  5.917 s    20 runs
 
Benchmark 15: ./false_sharing_8_128
  Time (mean ± σ):      5.223 s ±  0.038 s    [User: 10.166 s, System: 0.002 s]
  Range (min … max):    5.184 s …  5.344 s    20 runs
 
Benchmark 16: ./false_sharing_8_64
  Time (mean ± σ):      5.229 s ±  0.024 s    [User: 10.450 s, System: 0.002 s]
  Range (min … max):    5.196 s …  5.292 s    20 runs
 
Benchmark 17: ./false_sharing_16_64
  Time (mean ± σ):      5.442 s ±  0.508 s    [User: 10.877 s, System: 0.002 s]
  Range (min … max):    4.932 s …  6.462 s    20 runs
 
Benchmark 18: ./false_sharing_16_128
  Time (mean ± σ):      5.447 s ±  0.061 s    [User: 10.433 s, System: 0.002 s]
  Range (min … max):    5.377 s …  5.618 s    20 runs
 
Summary
  ./false_sharing_2_128 ran
    1.00 ± 0.00 times faster than ./false_sharing_2_64
    1.00 ± 0.00 times faster than ./false_sharing_1_64
    1.00 ± 0.00 times faster than ./false_sharing_1_128
    1.06 ± 0.00 times faster than ./false_sharing_4_64
    1.06 ± 0.00 times faster than ./false_sharing_4_128
    1.06 ± 0.01 times faster than ./false_sharing_8_128
    1.06 ± 0.01 times faster than ./false_sharing_8_64
    1.10 ± 0.10 times faster than ./false_sharing_16_64
    1.11 ± 0.01 times faster than ./false_sharing_16_128
    1.11 ± 0.00 times faster than ./false_sharing_5_128
    1.11 ± 0.00 times faster than ./false_sharing_3_128
    1.11 ± 0.00 times faster than ./false_sharing_5_64
    1.14 ± 0.00 times faster than ./false_sharing_6_128
    1.14 ± 0.00 times faster than ./false_sharing_6_64
    1.17 ± 0.01 times faster than ./false_sharing_3_64
    1.19 ± 0.00 times faster than ./false_sharing_7_64
    1.19 ± 0.00 times faster than ./false_sharing_7_128
```

# Conclusion

Modern processors are complex but unpredictable beasts. They make our lives harder trying to make our lifes easier. Benchmarking them is a minefield.

# Reading list

1. Mara Bos, _Rust Atomics and Locks_. O'Reilly Media, 2023. ISBN 978-1098119447.
2. Denis Bakhvalov, _Performance Analysis and Tuning on Modern CPUs_. 2024, ISBN 979-8869584229.
3. [Intel® 64 and IA-32 Architectures Software Developer’s Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html).
