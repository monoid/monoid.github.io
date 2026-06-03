---
title: "False Sharing Alignment"
date: 2026-04-27T15:02:20+02:00
draft: true
ShowToc: true
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

Moreover, this behavior is controlled by a per-core MSR setting called either
"Adjacent Cache Line Prefetcher Disable" or "L2 Adjacent Cache Line Prefetcher
Disable". Check it before benchmarking!

# Benchmark

I tried to reproduce the 128-byte alignment improvement on a real benchmark. It
wasn't easy! Just starting 2 threads updating an atomic each is not enough: once
the atomics are in the respective CPU caches, no MESI interaction is done. Let's
try something more involved: a vector of atomics should do the trick.

Let imitate a classical two-pointer atomic queue: a atomic for head being
incremented by reader, and an atomic for tail incremented by writer. Using
either `#[repr(align(64))]` or `#[repr(align(128))]` in the Rust code below for
a wrapper type similar to `crossbeam-utils` will make the atomics being either 64
or 128 bytes away.

You can get the runnable code at
<https://github.com/monoid/junk/tree/master/false_sharing>, but for your
convenience, simplified code is below.
{{< details "Source code" >}}

```rust
use std::ops::Deref;
use std::sync::atomic::{AtomicU64, Ordering};

const N: usize = 1_000_000_000;
const ORDERING: Ordering = Ordering::AcqRel;
const SIZE: usize = 8;

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
    let data = Vec::from_iter(std::iter::repeat_with(|| AddSub::default()).take(SIZE));
    let addsub: Box<[AddSub]> = data.into();
    let addsub = &*Box::leak(addsub);

    let t1 = std::thread::spawn(move || {
        #[cfg(target_arch = "x86_64")]
        affinity::set_thread_affinity(&[0]).unwrap();

        for _ in 0..(N / SIZE) {
            for elt in addsub {
                elt.add.fetch_add(1u64, ORDERING);
            }
        }
    });
    let t2 = std::thread::spawn(move || {
        #[cfg(target_arch = "x86_64")]
        // Not 1! Core 1 is usually a virtual core of the same physical one as core 0.
        affinity::set_thread_affinity(&[2]).unwrap();

        for _ in 0..(N / SIZE) {
            for elt in addsub {
                elt.sub.fetch_add(1u64, ORDERING);
            }
        }
    });
    t1.join().unwrap();
    t2.join().unwrap();

    for item in &*addsub {
        eprintln!("{}", item.get_value());
    }
}
```
{{< /details >}}

1. TODO pinning
2. TODO looping

## Digital Ocean

I started benchmarking from a Digital Ocean VPS instance.  Unfortunately, I
wasn't able to reproduce anything reliably.  I suspect that that Adjacent Prefetcher
is simply disabled on Digital Ocean.

## Amazon Web Services

I was able to reliably reproduce the difference at AWS's `c5d.4xlarge` instance
(Intel(R) Xeon(R) Platinum 8124M CPU @ 3.00GHz).  And execution time was not the
only difference between 64 and 128 byte alignment!

Unfortunately, I was not able to analyze programs' execution on AWS as most of
hardware counters are restricted here.


{{< details "CPU info" >}}
```
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 85
model name      : Intel(R) Xeon(R) Platinum 8124M CPU @ 3.00GHz
stepping        : 4
microcode       : 0x2007006
cpu MHz         : 3354.253
cache size      : 25344 KB
physical id     : 0
siblings        : 16
core id         : 0
cpu cores       : 8
apicid          : 0
initial apicid  : 0
fpu             : yes
fpu_exception   : yes
cpuid level     : 13
wp              : yes
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology nonstop_tsc cpuid aperfmperf tsc_known_freq pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault pti fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid mpx avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves ida arat pku ospke
bugs            : cpu_meltdown spectre_v1 spectre_v2 spec_store_bypass l1tf mds swapgs itlb_multihit mmio_stale_data retbleed gds bhi spectre_v2_user its
bogomips        : 6000.01
clflush size    : 64
cache_alignment : 64
address sizes   : 46 bits physical, 48 bits virtual
power management:
```
{{< /details >}}

I've compiled a bunch of binaries with different parameters, and run them with
```shell
hyperfine --warmup 3  --export-json results.json ./false_sharing_*
```

## Results
Size | align 64, time (mean ± σ) | align 128, time (mean ± σ)
-----|-------------------|----------------
1    |7.426 s ±  0.001 s |7.427 s ±  0.001 s
2    |5.464 s ±  0.002 s |5.349 s ±  0.001 s
3    |5.475 s ±  0.093 s |5.249 s ±  0.002 s
4    |5.541 s ±  0.092 s |5.420 s ±  0.001 s
5    |5.696 s ±  0.247 s |5.363 s ±  0.002 s
6    |5.609 s ±  0.239 s |5.396 s ±  0.001 s
7    |5.684 s ±  0.201 s |5.137 s ±  0.001 s
8    |6.800 s ±  0.241 s |6.679 s ±  0.000 s
12   |6.258 s ±  0.064 s |6.189 s ±  0.001 s
16   |6.943 s ±  0.281 s |6.646 s ±  0.001 s

128-byte alignment is not just a few percent faster, it also demonstrates **more
consistent timings**, which may be crucial for real-time and latency-sensitive
applications like HFT. The reason is clear: prefetcher-induced cache coherence
in 64-byte alignment happens in unpredictable order and takes unpredictable
time, while 128-byte alignment eliminates this contention.

{{< details "Raw results" >}}
```
$ hyperfine --warmup 3  --export-json results.json ./false_sharing_* ./false_sharing_12_128
Benchmark 1: ./false_sharing_12_128
  Time (mean ± σ):      6.189 s ±  0.001 s    [User: 12.373 s, System: 0.001 s]
  Range (min … max):    6.188 s …  6.190 s    10 runs
 
Benchmark 2: ./false_sharing_12_64
  Time (mean ± σ):      6.258 s ±  0.064 s    [User: 12.497 s, System: 0.002 s]
  Range (min … max):    6.206 s …  6.415 s    10 runs
 
Benchmark 3: ./false_sharing_16_128
  Time (mean ± σ):      6.646 s ±  0.001 s    [User: 13.285 s, System: 0.002 s]
  Range (min … max):    6.645 s …  6.647 s    10 runs
 
Benchmark 4: ./false_sharing_16_64
  Time (mean ± σ):      6.943 s ±  0.281 s    [User: 13.878 s, System: 0.001 s]
  Range (min … max):    6.649 s …  7.378 s    10 runs
 
Benchmark 5: ./false_sharing_1_128
  Time (mean ± σ):      7.427 s ±  0.001 s    [User: 14.845 s, System: 0.001 s]
  Range (min … max):    7.426 s …  7.428 s    10 runs
 
Benchmark 6: ./false_sharing_1_64
  Time (mean ± σ):      7.426 s ±  0.001 s    [User: 14.843 s, System: 0.002 s]
  Range (min … max):    7.424 s …  7.427 s    10 runs
 
Benchmark 7: ./false_sharing_2_128
  Time (mean ± σ):      5.349 s ±  0.001 s    [User: 10.690 s, System: 0.001 s]
  Range (min … max):    5.348 s …  5.349 s    10 runs
 
Benchmark 8: ./false_sharing_2_64
  Time (mean ± σ):      5.464 s ±  0.002 s    [User: 10.809 s, System: 0.001 s]
  Range (min … max):    5.463 s …  5.468 s    10 runs
 
Benchmark 9: ./false_sharing_3_128
  Time (mean ± σ):      5.249 s ±  0.002 s    [User: 10.492 s, System: 0.001 s]
  Range (min … max):    5.247 s …  5.251 s    10 runs
 
Benchmark 10: ./false_sharing_3_64
  Time (mean ± σ):      5.475 s ±  0.093 s    [User: 10.923 s, System: 0.001 s]
  Range (min … max):    5.414 s …  5.726 s    10 runs
 
Benchmark 11: ./false_sharing_4_128
  Time (mean ± σ):      5.420 s ±  0.001 s    [User: 10.832 s, System: 0.001 s]
  Range (min … max):    5.419 s …  5.421 s    10 runs
 
Benchmark 12: ./false_sharing_4_64
  Time (mean ± σ):      5.541 s ±  0.092 s    [User: 11.073 s, System: 0.001 s]
  Range (min … max):    5.439 s …  5.757 s    10 runs
 
Benchmark 13: ./false_sharing_5_128
  Time (mean ± σ):      5.363 s ±  0.002 s    [User: 10.717 s, System: 0.002 s]
  Range (min … max):    5.359 s …  5.367 s    10 runs
 
Benchmark 14: ./false_sharing_5_64
  Time (mean ± σ):      5.696 s ±  0.247 s    [User: 11.387 s, System: 0.001 s]
  Range (min … max):    5.361 s …  6.235 s    10 runs
 
Benchmark 15: ./false_sharing_6_128
  Time (mean ± σ):      5.396 s ±  0.001 s    [User: 10.785 s, System: 0.001 s]
  Range (min … max):    5.395 s …  5.397 s    10 runs
 
Benchmark 16: ./false_sharing_6_64
  Time (mean ± σ):      5.609 s ±  0.239 s    [User: 11.210 s, System: 0.001 s]
  Range (min … max):    5.371 s …  6.012 s    10 runs
 
Benchmark 17: ./false_sharing_7_128
  Time (mean ± σ):      5.137 s ±  0.001 s    [User: 10.265 s, System: 0.001 s]
  Range (min … max):    5.135 s …  5.138 s    10 runs
 
Benchmark 18: ./false_sharing_7_64
  Time (mean ± σ):      5.684 s ±  0.201 s    [User: 11.362 s, System: 0.001 s]
  Range (min … max):    5.343 s …  6.057 s    10 runs
 
Benchmark 19: ./false_sharing_8_128
  Time (mean ± σ):      6.679 s ±  0.000 s    [User: 13.136 s, System: 0.001 s]
  Range (min … max):    6.678 s …  6.679 s    10 runs
 
Benchmark 20: ./false_sharing_8_64
  Time (mean ± σ):      6.800 s ±  0.241 s    [User: 13.590 s, System: 0.002 s]
  Range (min … max):    6.684 s …  7.473 s    10 runs
 
Benchmark 21: ./false_sharing_12_128
  Time (mean ± σ):      6.189 s ±  0.000 s    [User: 12.375 s, System: 0.001 s]
  Range (min … max):    6.189 s …  6.190 s    10 runs
 
Summary
  ./false_sharing_7_128 ran
    1.02 ± 0.00 times faster than ./false_sharing_3_128
    1.04 ± 0.00 times faster than ./false_sharing_2_128
    1.04 ± 0.00 times faster than ./false_sharing_5_128
    1.05 ± 0.00 times faster than ./false_sharing_6_128
    1.06 ± 0.00 times faster than ./false_sharing_4_128
    1.06 ± 0.00 times faster than ./false_sharing_2_64
    1.07 ± 0.02 times faster than ./false_sharing_3_64
    1.08 ± 0.02 times faster than ./false_sharing_4_64
    1.09 ± 0.05 times faster than ./false_sharing_6_64
    1.11 ± 0.04 times faster than ./false_sharing_7_64
    1.11 ± 0.05 times faster than ./false_sharing_5_64
    1.20 ± 0.00 times faster than ./false_sharing_12_128
    1.20 ± 0.00 times faster than ./false_sharing_12_128
    1.22 ± 0.01 times faster than ./false_sharing_12_64
    1.29 ± 0.00 times faster than ./false_sharing_16_128
    1.30 ± 0.00 times faster than ./false_sharing_8_128
    1.32 ± 0.05 times faster than ./false_sharing_8_64
    1.35 ± 0.05 times faster than ./false_sharing_16_64
    1.45 ± 0.00 times faster than ./false_sharing_1_64
    1.45 ± 0.00 times faster than ./false_sharing_1_128
```
{{< /details >}}

# Conclusion

Modern processors are complex beasts. They make our lives harder trying to make our lives easier. Benchmarking them is a minefield.

# Reading list

1. Mara Bos, _Rust Atomics and Locks_. O'Reilly Media, 2023. ISBN 978-1098119447.
2. Denis Bakhvalov, _Performance Analysis and Tuning on Modern CPUs_. 2024, ISBN 979-8869584229.
3. [Intel® 64 and IA-32 Architectures Software Developer’s Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html).


# Disclaimer

This text was written by Ivan Boldyrev. I used AI only for proofreading, but its
help was invaluable.
