---
layout: post
title: "Reduce Build Time"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Great ROI

Consolidate repos - one DAG make parallel make
- more modular your code is the more you parallise your compile

Things inspired from Matt Hargett track:

Agenda: 
 - Vocabulary
 - If only it were simple.. 
 - Accelerating what gets built
 - Reducing what gets built 
 - Culture shifts that result


Vocabulary

- Link Time Optimization(LTO)
  - aka Link-Time Code Generation(LTCG)
  - source -> IR/IL non-naitve object files
  - all IR/IL object files -> native code at link time

  In this source files get compile into non-native object files (like
  bytecode) and then at link time they get converted to native
  code. This is awesome because now we can inline across link
  boundaries ( I am not sure what this means?)

- Profile Guided Optimization(PGO)
  - sources -> profile-generative native code
  - source + profile -> profile-optimized native code

 PGO compiles sources + some instrumentation and generated run time
 profile files; Then compile it again it uses the run time profile and
 optimizes the code paths. For instance, cold code path are optimized
 for size instead of speed.

- -march
  - generate code for a specific CPU
  - optimizes for L1/L2/L3 cachse size, pipeline, SIMD, ...

It's not just about instruction sets. (Learn form Scott Mayer's talk
why cache lines are crucial)


- ccache

 - open source project that creates on-disk cache of object files,
   indexed by pre-processed source hash
 - work simply by preprending the 'cache' commdand in front ($ ccache
   gcc ... )
 
- distcc

 - open source project that distributes compilation by trasferring
   pre-preocessed src to toher nodes for compilation
 - distccd runs on other hosts (you have it configured with 0conf so
   that you don't have to some bla bla.. listen again at ~9:10)


If only it were simple...

- ccache and distcc tools rely on pre-processing src
- Gains are poor if pre-processing is slow
- pre-processing is the first bottleneck, many whitepapers on it
- What makes pre-processing slow
  - giant transitive closures of header includes (templates :( )
  - slow network I/O 
  - slow local disk I/O (8 cores but 200 rpm disk)
  - unnecessary include/library directory search (giant command line
    include search paths due to make file decleration)
  - unwittingly invalidating the ccache cache
- distcc needs fast local network (no WIFI)

Accelerate the work

- make the source checkout/update FAST
  - do a simple transfer rate test (different geographic regions)
  - do a simple manual load tests with concurrent clients (there are
    open source tools for this)
  - montior your source servers (CPU, RAM, disk saturation, any use of
    swap)
  - Real world example - 10 MB hub or server under Alice's desk

- use RAM disks
  - speeding up compilation itself
  - mount $TEMP as a RAM disk 
  - link is mainly io heavy so if you put BUILD output in RAM disk
  - use tempfs (on Linux) to further streamline
  - ram disk are best bang for the buck, even the best SSDs 
     - use direct PCI-E EFDs as backing store
  - often, other bottleneck come to the fore (because ramdisk IO
    becomes awesomely fast)
  - Best part? No source code changes means language/technology
    agnostic

- optimizing file servers
  - disk io's are big bottleneck especially with linkers
  - monitor and tune the SAN/NAS for real-work  traffic
    - fast switches/disks < server being CPU-bound
  - optimize (NFS) mount options on clients
    - large rsize, wsize values
    - fail faast with low timeo and retrans values
  - mount read-only when possible (eliminate atime, mtime, acl, attr
    writes)
  - use jumbo frame MTU size to lower fragmentation
  - no source code change

- use gold linker (any linker that is threaded and can partition;
  - use gold linker in threaded mode (not default; --threads
    --preread-archive-symbols (could save 20% at link time)
  - enable plug-ins!
    - still a compile time switch, not runtime
    - binutils/configure --enable-gold --enable-plugins
    - using linker with plugins requires rebuilding toolchain with
      *that* linker..
 
- optimize the compiler for your environment (Big one; requires
  bootstraping the compiler)
  - use profilebootstrap target and bootstrap-lto config
  - copy bootstrap-lto.mk and add -march native
  - build GCC as C++ (./configure --enable-build-with-cxx; already
    default in 4.8; it matters for profile bootstrap)
  - add a non-functional file to compile with your headers
  - using static LTO libraries? link them in via Makefile.in
  - do the same for linker build and static analysis tools (-Ofast
    -march=nativ -flto)

- using ccache
  - put the cache on a RAM disk if you can, not network
  - monitor stats for unexpected cache misses
  - increase max cache size based on misses (CCACHE_DIR=/tmp/build
    ccache -M 4GB)
  - "direct moce" versus "pre-processor" mode

- use distcc
   - still a huge issue using distcc with certain libraries (boost -
     transitive closure of header files)
   - can be a win if
     - super fast network between nodes
     - proifle-optimized compiler/pre-processor
     - header files/trasitive closures are quite clean
   - almost requires back-end build-as-a-service
   - could use ccache with distcc (use CCACHE_PREFIX); haven't seen
     huge win

- parallelizing the work
  - what make -jN works best? (Typically n-1 or n-2 works best)
    - just because it looks like your have n cores in your machine but
      you don't because the way BIOS present those data is as if they
      are distinct core but they are not
    - make sure hyperthreading is enabled in BIOS ( on some DELL
      machien it was disabled; enable it, it's a WIN)
    - bring out weird dependency issues; recursive make nightmare
    - note command-lines when number of compiler processes less than
      N-1 for more than a few seconds. This means make DAG can be
      optimized (watch -n 1 'ps auxww | grep g++')
   - when using LTO, make sure to try -flto=N (which is number of partition)
   - Honza'a blog on LTO in firefox (really code c++ codebase)
     http://hubicka.blogspot.com/2014/04/linktime-optimization-in-gcc-1-brief.html

Reducing what gets built

- sources that are compiled but not used/linked
  - makefile crufts
  - find *.o[bj] comapare with linker command lines
  - objects linked in but maybe eliminated in linker GC?
  - -Wl --as-needed, still needs to do work so better to 
- source that get re-compiled multiple times
  - ifdef ps3 ifdef win32 in one file screws you out of ccache; have
    abstract header and platform specific implementation file
  - shift preprocessor defines to runtime composition of specialization
  - favor version.cpp instead of -DVERSION=0.0.1 or __DATE__ (good
    example scummvm on github)
    https://github.com/scummvm/scummvm/blob/master/base/version.cpp
  - extract function definition into source files
  - if you really care about inlining use LTO
  - capture strace log and look for repeated fopen()
- headers that get parsed unnecessarity
  - resuce transitive includes
  - apply interface Segregation principle (no module should be force
    to depend on methods it does not use)
  - use forward decleration whenever possible! (C++11 allows forwarddecl of enums)
- debug symbols, fat LTO, object etc. 

- reducing disk IO
  - using LTO? disable at LTO objects by using enabling slim LTO
    objects in modern gcc, if you toolchain allows
  - using debug symbols? how many kinds? can your debugger handle more
    optimal format? could just generate dwarf
  - don't add unnecessary include/link paths
    - killer for network file systems
    - only -l and -L that are relevant for source/modules
    - try to optimize the ordering, per module

Shift in the culture
  - Slow builds == waiting & context switches
  - Fast builds mean less ping pong
  - Most people love getting more done per day 
  - The build is just the begining
    - profile/arch/LTO-optimize code analysis tools
    - make unit test faster
    - parallelize system/acceptance tests
 - Perfecting, not Perfection 

Other Tips
- reduce and accelerate in lock-step.
- Reduction work takes time, but has biggest ROI
- Don't virtualize C++ builds. LCX, docker etc cool
- get a toolchain support contract (SLA for fixes to compile-time
  regression)

Conclusion
- builds can be made much faster
  - with no change to source code
  - minimal capital investment
- fantastic ROI for medium sized orgs
  - $100/hour * 5 hours waiting for build/week * 20 FTEs * 40 weeks/year
  - Even in an org with 20 people that builds once a day, one hour
    build, that half a million dollar a year. Quite a money left on table.
  - same match applies for time spend resolving merges
  - doesn't even include opportunity cost. People doing more
    productive things in that time
