# Arnaud Bouchez

**mORMot entry to The One Billion Row Challenge in Object Pascal.**

## mORMot 2 is Required

This entry requires the **mORMot 2** package to compile.

Download it from https://github.com/synopse/mORMot2

It is better to fork the current state of the *mORMot 2* repository, or get the latest release.

## Licence Terms

This code is licenced by its sole author (A. Bouchez) as MIT terms, to be used for pedagogical reasons.

I am very happy to share decades of server-side performance coding techniques using FPC on x86_64. ;)

## Presentation

Here are the main ideas behind this implementation proposal:

- **mORMot** makes cross-platform and cross-compiler support simple - e.g. `TMemMap`, `TDynArray.Sort`,`TTextWriter`, `SetThreadCpuAffinity`, `crc32c`, `ConsoleWrite` or command-line parsing;
- The entire 16GB file is `memmap`ed at once into memory - it won't work on 32-bit OS, but avoid any `read` syscall or memory copy;
- Process file in parallel using several threads - configurable via the `-t=` switch, default being the total number of CPUs reported by the OS;
- Input is fed into each thread as 64MB chunks: because thread scheduling is unbalanced, it is inefficient to pre-divide the size of the whole input file into the number of threads;
- Each thread manages its own `Station[]` data, so there is no lock until the thread is finished and data is consolidated;
- Each `Station[]` information is packed into a record of exactly 16 bytes, with no external pointer/string, to leverage the CPU L1 cache size (64 bytes) for efficiency;
- Maintain a `StationHash[]` hash table for the name lookup, with crc32c perfect hash function - no name comparison nor storage is needed with a perfect hash (see below);
- On Intel/AMD/AARCH64 CPUs, *mORMot* uses hardware SSE4.2 opcodes for this crc32c computation;
- Store values as 16-bit or 32-bit integers, as temperature multiplied by 10;
- Parse temperatures with a dedicated code (expects single decimal input values);
- The station names are stored as UTF-8 pointers to the memmap location where they appear first, in `StationName[]`, to be emitted eventually for the final output, not during temperature parsing;
- No memory allocation (e.g. no transient `string` or `TBytes`) nor any syscall is done during the parsing process to reduce contention and ensure the process is only CPU-bound and RAM-bound (we checked this with `strace` on Linux);
- Pascal code was tuned to generate the best possible asm output on FPC x86_64 (which is our target);
- Can optionally output timing statistics and resultset hash value on the console to debug and refine settings (with the `-v` command line switch);
- Can optionally set each thread affinity to a single core (with the `-a` command line switch).

If you are not convinced by the "perfect hash" trick, you can define the `NOPERFECTHASH` conditional, which forces full name comparison, but is noticeably slower. Our algorithm is safe with the official dataset, and gives the expected final result - which was the goal of this challenge: compute the right data reduction with as little time as possible, with all possible hacks and tricks. A "perfect hash" is a well known hacking pattern, when the dataset is validated in advance. And since our CPUs offers `crc32c` which is perfect for our dataset... let's use it! https://en.wikipedia.org/wiki/Perfect_hash_function ;)

## Why L1 Cache Matters

Take great care of the "64 bytes cache line" is quite unique among all implementations of the "1brc" I have seen in any language - and it does make a noticeable difference in performance.

The L1 cache is well known in the performance hacking litterature to be the main bottleneck for any efficient in-memory process. If you want things to go fast, you should flatter your CPU L1 cache.

Min/max values will be reduced as 16-bit smallint - resulting in temperature range of -3276.7..+3276.8 which seems fair on our planet according to the IPCC. ;)

In our first attempt (see "Old Version" below), we stored the name into the `Station[]` array, so that each entry is 64 bytes long exactly. But since `crc32c` is a perfect hash function for our dataset, it is enough to just store the 32-bit hash instead, and not the actual name.

Note that if we reduce the number of stations from 41343 to 400, the performance is much higher, also with a 16GB file as input. The reason is that since 400x16 = 6400, each dataset could fit entirely in each core L1 cache. No slower L2/L3 cache is involved, therefore performance is better. The cache memory seems to be the bottleneck of our code. Which is a good sign.

## Usage

If you execute the `abouchez` executable without any parameter, it will give you some hints about its usage (using *mORMot* `TCommandLine` abilities):

```
ab@dev:~/dev/github/1brc-ObjectPascal/bin$ ./abouchez 
The mORMot One Billion Row Challenge

Usage: abouchez  <filename> [options] [params]

   <filename>         the data source filename

Options:
  -v, --verbose       generate verbose output with timing
  -a, --affinity      force thread affinity to a single CPU core
  -h, --help          display this help

Params:
  -t, --threads <number> (default 16)
                      number of threads to run
```
We will use these command-line switches for local (dev PC), and benchmark (challenge HW) analysis.

## Local Analysis

On my PC, it takes less than 3 seconds to process the 16GB file with 8/10 threads.

Let's compare `abouchez` with a solid multi-threaded entry using file buffer reads and no memory map (like `sbalazs`), using the `time` command on Linux:

```
ab@dev:~/dev/github/1brc-ObjectPascal/bin$ time ./abouchez measurements.txt -t=20 >resmormot.txt

real 0m2,350s
user 0m40,165s
sys  0m0,888s

ab@dev:~/dev/github/1brc-ObjectPascal/bin$ time ./sbalazs measurements.txt 20 >ressb.txt

real 0m25,330s
user 6m44,853s
sys  0m31,167s
```
We defined 20 threads for both executables, because our PC CPU has 20 threads in total, and using them all seems to achieve the best resutls.

Apart from the obvious global "wall" time reduction (`real` numbers), the raw parsing and data gathering in the threads match the number of threads and the running time (`user` numbers), and no syscall is involved by `abouchez` thanks to the memory mapping of the whole file (`sys` numbers, which contain only memory page faults, is much lower).

The `memmap()` feature makes the initial/cold `abouchez` call slower, because it needs to cache all measurements data from file into RAM (I have 32GB of RAM, so the whole data file will remain in memory, as on the benchmark hardware):
```
ab@dev:~/dev/github/1brc-ObjectPascal/bin$ time ./abouchez measurements.txt -t=20 >resmormot.txt

real 0m6,042s
user 0m53,699s
sys  0m2,941s
```
This is the expected behavior, and will be fine with the benchmark challenge, which ignores the min and max values during its 10 times run. So the first run will just warm up the file into memory.

On my Intel 13h gen processor with E-cores and P-cores, forcing thread to core affinity does not make any huge difference (we are within the error margin):
```
ab@dev:~/dev/github/1brc-ObjectPascal/bin$ ./abouchez measurements.txt -t=10 -v
Processing measurements.txt with 20 threads and affinity=false
result hash=8A6B746A,, result length=1139418, stations count=41343, valid utf8=1
done in 2.36s 6.6 GB/s

ab@dev:~/dev/github/1brc-ObjectPascal/bin$ ./abouchez measurements.txt -t=10 -v -a
Processing measurements.txt with 20 threads and affinity=true
result hash=8A6B746A, result length=1139418, stations count=41343, valid utf8=1
done in 2.44s 6.4 GB/s
```
Affinity may help on Ryzen 9, because its Zen 3 architecture is made of identical 16 cores with 32 threads, not this Intel E/P cores mess. But we will validate that on real hardware - no premature guess!

The `-v` verbose mode makes such testing easy. The `hash` value can quickly check that the generated output is correct, and that it is valid `utf8` content (as expected).

## Benchmark Integration

Every system is quite unique, especially about its CPU multi-thread abilities. For instance, my Intel Core i5 has both P-cores and E-cores so its threading model is pretty unbalanced. The Zen architecture should be more balanced.

So we first need to find out which options leverage at best the hardware it runs on.

On the https://github.com/gcarreno/1brc-ObjectPascal challenge hardware, which is a Ryzen 9 5950x with 16 cores / 32 threads and 64MB of L3 cache, each thread using around 2.5MB of its own data, we should try several options with 16-24-32 threads, for instance:

```
time ./abouchez measurements.txt -v -t=8
time ./abouchez measurements.txt -v -t=16
time ./abouchez measurements.txt -v -t=24
time ./abouchez measurements.txt -v -t=32
time ./abouchez measurements.txt -v -t=16 -a
time ./abouchez measurements.txt -v -t=24 -a
time ./abouchez measurements.txt -v -t=32 -a
```
Please run those command lines, to guess which parameters are to be run for the benchmark, and would give the best results on the actual benchmark PC with its Ryzen 9 CPU. We will see if core affinity makes a difference here.

Then we could run:
```
time ./abouchez measurements.txt -v -t=1
```
This `-t=1` run is for fun: it will run the process in a single thread. It will help to guess how optimized (and lockfree) our parsing code is, and to validate the CPU multi-core abilities. In a perfect world, other `-t=##` runs should stand for a perfect division of `real` time per the number of working threads, and the `user` value reported by `time` should remain almost the same when we add threads up to the number of CPU cores.

## Back To Reality

Our proposal has been run on the benchmark hardware, using the full automation. 

Here are some numbers, with 16 threads:
```
-- SSD --
Benchmark 1: abouchez
  Time (mean ± σ):      2.095 s ±  0.044 s    [User: 21.486 s, System: 1.752 s]
  Range (min … max):    2.017 s …  2.135 s    10 runs
``` 

With 24 threads:
```
-- SSD --
Benchmark 1: abouchez
  Time (mean ± σ):      1.944 s ±  0.014 s    [User: 28.686 s, System: 1.909 s]
  Range (min … max):    1.924 s …  1.974 s    10 runs
``` 

With 32 threads:
```
-- SSD --
Benchmark 1: abouchez
  Time (mean ± σ):      1.768 s ±  0.012 s    [User: 33.286 s, System: 2.067 s]
  Range (min … max):    1.743 s …  1.782 s    10 runs
```

If we try with 32 threads and thread affinity (`-a` option):
```
  Time (mean ± σ):      1.771 s ±  0.010 s    [User: 33.415 s, System: 2.056 s]
  Range (min … max):    1.760 s …  1.786 s    10 runs
 ```

So it sounds like if we could just run the benchmark with the `-t=32` option, and achieve the best performance. Thread affinity is no silver bullet here, so we better stay away from it, and let the OS decide about thread scheduling.

The Ryzen CPU has 16 cores with 32 threads, and it makes sense that each thread only have to manage a small number of data per item (a 16 bytes `Station[]` item), so we could leverage all cores and threads.


## Notes about the "Old" Version

In the version same `src` sub-folder, you will find our first attempt of this challenge, as `brcmormotold.lpr`. In respect to the "final/new" version, it did store the name as a "shortstring" within its `Station[]` record, to fill exactly the 64-byte cache line size.

It was already very fast, but since `crc32c` is a perfect hash function, we finally decided to just stored the 32-bit hash, and not the name itself.

You could disable our tuned asm in the project source code, and loose about 10% by using general purpose *mORMot* `crc32c()` and `CompareMem()` functions, which already runs SSE2/SSE4.2 tune assembly. No custom asm is needed on the "new" version: we directly use the *mORMot* functions.

There is a "*pure mORMot*" name lookup version available if you undefine the `CUSTOMHASH` conditional, which is around 40% slower, because it needs to copy the name into the stack before using `TDynArrayHashed`, and has a little more overhead.

As reference, here are the numbers of this "old" version, with 30 threads (on a busy Benchmark system): 
```
-- SSD --
Benchmark 1: abouchez
  Time (mean ± σ):      3.634 s ±  0.099 s    [User: 86.580 s, System: 2.012 s]
  Range (min … max):    3.530 s …  3.834 s    10 runs
 
-- HDD --
Benchmark 1: abouchez
  Time (mean ± σ):      3.629 s ±  0.102 s    [User: 86.086 s, System: 2.008 s]
  Range (min … max):    3.497 s …  3.789 s    10 runs
```

In fact, only the SSD values matters. We can see that once the data is loaded from disk into the RAM cache, there is no difference with a `memmap` file on SSD and HDD. Linux is a great Operating System for sure.

With 24 threads: 
```
-- SSD --
Benchmark 1: abouchez
  Time (mean ± σ):      2.977 s ±  0.053 s    [User: 53.790 s, System: 1.881 s]
  Range (min … max):    2.905 s …  3.060 s    10 runs
```

With 16 threads: 
```
-- SSD --
Benchmark 1: abouchez
  Time (mean ± σ):      2.472 s ±  0.061 s    [User: 27.787 s, System: 1.720 s]
  Range (min … max):    2.386 s …  2.588 s    10 runs
```

With 16 threads and thread affinity (`-a` switch on command line): 
```
-- SSD --
Benchmark 1: abouchez
  Time (mean ± σ):      3.227 s ±  0.017 s    [User: 39.731 s, System: 1.875 s]
  Range (min … max):    3.206 s …  3.253 s    10 runs
```
It is a known fact from experiment that forcing thread affinity is not a good idea, and it is always much better to let any modern Operating System do  the threads scheduling to the CPU cores, because it has a much better knowledge of the actual system load and status. Even on a "fair" CPU architecture like AMD Zen. For a "pure CPU" process, affinity may help a very little. But for our "old" process working outside of the L1 cache limits, we better let the OS decide.

So with this "old" version, it was decided to use `-t=16`. The "old" version is using a whole cache line (16 bytes) for its `Station[]` record, so it may be the responsible of using too much CPU cache, so more than 16 threads does not make a difference with it. Whereas our "new" version, with its `Station[]` of only 16 bytes, could use `-t=32` with benefits. The cache memory access is likely to be the bottleneck from now on.

Arnaud :D