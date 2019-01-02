This is a security-focused general purpose memory allocator providing the
malloc API along with various extensions. It provides substantial hardening
against heap corruption vulnerabilities. The security-focused design also leads
to much less metadata overhead and memory waste from fragmentation than a more
traditional allocator design. It aims to provide decent overall performance
with a focus on long-term performance and memory usage rather than allocator
micro-benchmarks. It has relatively fine-grained locking and will offer good
scalability once arenas are implemented.

This project currently aims to support Android, musl and glibc. It may support
other non-Linux operating systems in the future. For Android and musl, there
will be custom integration and other hardening features. The glibc support will
be limited to replacing the malloc implementation because musl is a much more
robust and cleaner base to build on and can cover the same use cases.

This allocator is intended as a successor to a previous implementation based on
extending OpenBSD malloc with various additional security features. It's still
heavily based on the OpenBSD malloc design, albeit not on the existing code
other than reusing the hash table implementation for the time being. The main
differences in the design are that it is solely focused on hardening rather
than finding bugs, uses finer-grained size classes along with slab sizes going
beyond 4k to reduce internal fragmentation, doesn't rely on the kernel having
fine-grained mmap randomization and only targets 64-bit to make aggressive use
of the large address space.  There are lots of smaller differences in the
implementation approach. It incorporates the previous extensions made to
OpenBSD malloc including adding padding to allocations for canaries (distinct
from the current OpenBSD malloc canaries), write-after-free detection tied to
the existing clearing on free, queues alongside the existing randomized arrays
for quarantining allocations and proper double-free detection for quarantined
allocations. The per-size-class memory regions with their own random bases were
loosely inspired by the size and type-based partitioning in PartitionAlloc. The
planned changes to OpenBSD malloc ended up being too extensive and invasive so
this project was started as a fresh implementation better able to accomplish
the goals. For 32-bit, a port of OpenBSD malloc with small extensions can be
used instead as this allocator fundamentally doesn't support that environment.

# Dependencies

Debian stable determines the most ancient set of supported dependencies:

* glibc 2.24
* Linux 4.9
* Clang 3.8 or GCC 6.3

However, using more recent releases is highly recommended. Older versions of
the dependencies may be compatible at the moment but are not tested and will
explicitly not be supported.

For external malloc replacement with musl, musl 1.1.20 is required. However,
there will be custom integration offering better performance in the future
along with other hardening for the C standard library implementation.

Major releases of Android will be supported until tags stop being pushed to
the Android Open Source Project (AOSP). Google supports each major release
with security patches for 3 years, but tagged releases of the Android Open
Source Project are more than just security patches and are no longer pushed
once no officially supported devices are using them anymore. For example, at
the time of writing (September 2018), AOSP only has tagged releases for 8.1
(Nexus 5X, Nexus 6P, Pixel C) and 9.0 (Pixel, Pixel XL, Pixel 2, Pixel 2 XL).
There are ongoing security patches for 6.0, 6.0.1, 7.0, 7.1.1, 7.1.2, 8.0, 8.1
and 9.0 but only the active AOSP branches (8.1 and 9.0) are supported by this
project and it doesn't make much sense to use much older releases with far
less privacy and security hardening.

# Testing

The `preload.sh` script can be used for testing with dynamically linked
executables using glibc or musl:

    ./preload.sh krita --new-image RGBA,U8,500,500

It can be necessary to substantially increase the `vm.max_map_count` sysctl to
accomodate the large number of mappings caused by guard slabs and large
allocation guard regions. The number of mappings can also be drastically
reduced via a significant increase to `CONFIG_GUARD_SLABS_INTERVAL` but the
feature has a low performance and memory usage cost so that isn't recommended.

It can offer slightly better performance when integrated into the C standard
library and there are other opportunities for similar hardening within C
standard library and dynamic linker implementations. For example, a library
region can be implemented to offer similar isolation for dynamic libraries as
this allocator offers across different size classes. The intention is that this
will be offered as part of hardened variants of the Bionic and musl C standard
libraries.

# Configuration

You can set some configuration options at compile-time via arguments to the
make command as follows:

    make CONFIG_EXAMPLE=false

Configuration options are provided when there are significant compromises
between portability, performance, memory usage or security. The core design
choices are not configurable and the allocator remains very security-focused
even with all the optional features disabled.

The following boolean configuration options are available:

* `CONFIG_NATIVE`: `true` (default) or `false` to control whether the code is
  optimized for the detected CPU on the host. If this is disabled, setting up a
  custom `-march` higher than the baseline architecture is highly recommended
  due to substantial performance benefits for this code.
* `CONFIG_CXX_ALLOCATOR`: `true` (default) or `false` to control whether the
  C++ allocator is replaced for slightly improved performance and detection of
  mismatched sizes for sized deallocation (often type confusion bugs). This
  will result in linking against the C++ standard library.
* `CONFIG_ZERO_ON_FREE`: `true` (default) or `false` to control whether small
  allocations are zeroed on free, to mitigate use-after-free and uninitialized
  use vulnerabilities along with purging lots of potentially sensitive data
  from the process as soon as possible. This has a performance cost scaling to
  the size of the allocation, which is usually acceptable.
* `CONFIG_WRITE_AFTER_FREE_CHECK`: `true` (default) or `false` to control
  sanity checking that new allocations contain zeroed memory. This can detect
  writes caused by a write-after-free vulnerability and mixes well with the
  features for making memory reuse randomized / delayed. This has a performance
  cost scaling to the size of the allocation, which is usually acceptable.
* `CONFIG_SLOT_RANDOMIZE`: `true` (default) or `false` to randomize selection
  of free slots within slabs. This has a measurable performance cost and isn't
  one of the important security features, but the cost has been deemed more
  than acceptable to be enabled by default.
* `CONFIG_SLAB_CANARY`: `true` (default) or `false` to enable support for
  adding 8 byte canaries to the end of memory allocations. The primary purpose
  of the canaries is to render small fixed size buffer overflows harmless by
  absorbing them. The first byte of the canary is always zero, containing
  overflows caused by a missing C string NUL terminator. The other 7 bytes are
  a per-slab random value. On free, integrity of the canary is checked to
  detect attacks like linear overflows or other forms of heap corruption caused
  by imprecise exploit primitives. However, checking on free will often be too
  late to prevent exploitation so it's not the main purpose of the canaries.
* `CONFIG_SEAL_METADATA`: `true` or `false` (default) to control whether Memory
  Protection Keys are used to disable access to all writable allocator state
  outside of the memory allocator code. It's currently disabled by default due
  to being extremely experimental and a significant performance cost for this
  use case on current generation hardware, which may become drastically lower
  in the future. Whether or not this feature is enabled, the metadata is all
  contained within an isolated memory region with high entropy random guard
  regions around it.

The following integer configuration options are available. Proper sanity checks
for the chosen values are not written yet, so use them at your own peril:

* `CONFIG_SLAB_QUARANTINE_RANDOM_LENGTH`: `0` (default) to control the number
  of slots in the random array used to randomize reuse for small memory
  allocations. This sets the length for the largest size class (currently
  16384) and the quarantine length for smaller size classes is scaled to match
  the total memory of the quarantined allocations.
* `CONFIG_SLAB_QUARANTINE_QUEUE_LENGTH`: `0` (default) to control the number of
  slots in the queue used to delay reuse for small memory allocations. This
  sets the length for the largest size class (currently 16384) and the
  quarantine length for smaller size classes is scaled to match the total
  memory of the quarantined allocations.
* `CONFIG_GUARD_SLABS_INTERVAL`: `1` (default) to control the number of slabs
  before a slab is skipped and left as an unused memory protected guard slab
* `CONFIG_GUARD_SIZE_DIVISOR`: `2` (default) to control the maximum size of the
  guard regions placed on both sides of large memory allocations, relative to
  the usable size of the memory allocation
* `CONFIG_REGION_QUARANTINE_RANDOM_LENGTH`: `128` (default) to control the
  number of slots in the random array used to randomize region reuse for large
  memory allocations
* `CONFIG_REGION_QUARANTINE_QUEUE_LENGTH`: `1024` (default) to control the
  number of slots in the queue used to delay region reuse for large memory
  allocations
* `CONFIG_REGION_QUARANTINE_SKIP_THRESHOLD`: `33554432` (default) to control
  the size threshold where large allocations will not be quarantined
* `CONFIG_FREE_SLABS_QUARANTINE_RANDOM_LENGTH`: `32` (default) to control the
  number of slots in the random array used to randomize free slab reuse
* `CONFIG_CLASS_REGION_SIZE`: `34359738368` (default) to control the size of
  the size class regions

There will be more control over enabled features in the future along with
control over fairly arbitrarily chosen values like the size of empty slab
caches (making them smaller improves security and reduces memory usage while
larger caches can substantially improves performance).

# Basic design

The current design is very simple and will become a bit more sophisticated as
the basic features are completed and the implementation is hardened and
optimized. The allocator is exclusive to 64-bit platforms in order to take full
advantage of the abundant address space without being constrained by needing to
keep the design compatible with 32-bit.

Small allocations are always located in a large memory region reserved for slab
allocations. It can be determined that an allocation is one of the small size
classes from the address range. Each small size class has a separate reserved
region within the larger region, and the size of a small allocation can simply
be determined from the range. Each small size class has a separate out-of-line
metadata array outside of the overall allocation region, with the index of the
metadata struct within the array mapping to the index of the slab within the
dedicated size class region. Slabs are a multiple of the page size and are
page aligned. The entire small size class region starts out memory protected
and becomes readable / writable as it gets allocated, with idle slabs beyond
the cache limit having their pages dropped and the memory protected again.

Large allocations are tracked via a global hash table mapping their address to
their size and guard size. They're simply memory mappings and get mapped on
allocation and then unmapped on free.

This allocator is aimed at production usage, not aiding with finding and fixing
memory corruption bugs for software development. It does find many latent bugs
but won't include features like the option of generating and storing stack
traces for each allocation to include the allocation site in related error
messages. The design choices are based around minimizing overhead and
maximizing security which often leads to different decisions than a tool
attempting to find bugs. For example, it uses zero-based sanitization on free
and doesn't minimize slack space from size class rounding between the end of an
allocation and the canary / guard region. Zero-based filling has the least
chance of uncovering latent bugs, but also the best chance of mitigating
vulnerabilities. The canary feature is primarily meant to act as padding
absorbing small overflows to render them harmless, so slack space is helpful
rather than harmful despite not detecting the corruption on free. The canary
needs detection on free in order to have any hope of stopping other kinds of
issues like a sequential overflow, which is why it's included.  It's assumed
that an attacker can figure out the allocator is in use so the focus is
explicitly not on detecting bugs that are impossible to exploit with it in use
like an 8 byte overflow. The design choices would be different if performance
was a bit less important and if a core goal was finding latent bugs.

# Security properties

* Fully out-of-line metadata
* Deterministic detection of any invalid free (unallocated, unaligned, etc.)
    * Validation of the size passed for C++14 sized deallocation by `delete`
      even for code compiled with earlier standards (detects type confusion if
      the size is different) and by various containers using the allocator API
      directly
* Isolated memory region for slab allocations
    * Divided up into isolated inner regions for each size class
        * High entropy random base for each size class region
        * No deterministic / low entropy offsets between allocations with
          different size classes
    * Metadata is completely outside the slab allocation region
        * No references to metadata within the slab allocation region
        * No deterministic / low entropy offsets to metadata
    * Entire slab region starts out non-readable and non-writable
    * Slabs beyond the cache limit are purged and become non-readable and
      non-writable memory again
        * Placed into a queue for reuse in FIFO order to maximize the time
          spent memory protected
        * Randomized array is used to add a random delay for reuse
* Fine-grained randomization within memory regions
    * Randomly sized guard regions for large allocations
    * Random slot selection within slabs
    * Randomized delayed free for slab allocations
    * [in-progress] Randomized choice of slabs
    * [in-progress] Randomized allocation of slabs
* Slab allocations are zeroed on free
* Detection of write-after-free for slab allocations by verifying zero filling
  is intact at allocation time
* Large allocations are purged and memory protected on free with the memory
  mapping kept reserved in a quarantine to detect use-after-free
    * The quarantine is primarily based on a FIFO ring buffer, with the oldest
      mapping in the quarantine being unmapped to make room for the most
      recently freed mapping
    * Another layer of the quarantine swaps with a random slot in an array to
      randomize the number of large deallocations required to push mappings out
      of the quarantine
* Memory in fresh allocations is consistently zeroed due to it either being
  fresh pages or zeroed on free after previous usage
* Delayed free via a combination of FIFO and randomization for slab allocations
* Random canaries placed after each slab allocation to *absorb*
  and then later detect overflows/underflows
    * High entropy per-slab random values
    * Leading byte is zeroed to contain C string overflows
* Possible slab locations are skipped and remain memory protected, leaving slab
  size class regions interspersed with guard pages
* Zero size allocations are a dedicated size class with the entire region
  remaining non-readable and non-writable
* Protected allocator state (including all metadata)
    * Address space for state is entirely reserved during initialization and
      never reused for allocations or anything else
    * State within global variables is entirely read-only after initialization
      with pointers to the isolated allocator state so leaking the address of
      the library doesn't leak the address of writable state
    * Allocator state is located within a dedicated region with high entropy
      randomly sized guard regions around it
    * Protection via Memory Protection Keys (MPK) on x86\_64 (disabled by
      default due to low benefit-cost ratio on top of baseline protections)
    * [future] Protection via MTE on ARMv8.5+
* Extension for retrieving the size of allocations with fallback
  to a sentinel for pointers not managed by the allocator
    * Can also return accurate values for pointers *within* small allocations
    * The same applies to pointers within the first page of large allocations,
      otherwise it currently has to return a sentinel
* No alignment tricks interfering with ASLR like jemalloc, PartitionAlloc, etc.
* No usage of the legacy brk heap
* Aggressive sanity checks
    * Errors other than ENOMEM from mmap, munmap, mprotect and mremap treated
      as fatal, which can help to detect memory management gone wrong elsewhere
      in the process.
* [future] Memory tagging for slab allocations via MTE on ARMv8.5+
    * random memory tags as the baseline, providing probabilistic protection
      against various forms of memory corruption
    * dedicated tag for free slots, set on free, for deterministic protection
      against accessing freed memory
    * store previous random tag within freed slab allocations, and increment it
      to get the next tag for that slot to provide deterministic use-after-free
      detection through multiple cycles of memory reuse
    * guarantee distinct tags for adjacent memory allocations by incrementing
      past matching values for deterministic detection of linear overflows

# Randomness

The current implementation of random number generation for randomization-based
mitigations is based on generating a keystream from a stream cipher (ChaCha8)
in small chunks. A separate CSPRNG is used for each small size class, large
allocations, etc. in order to fit into the existing fine-grained locking model
without needing to waste memory per thread by having the CSPRNG state in Thread
Local Storage. Similarly, it's protected via the same approach taken for the
rest of the metadata. The stream cipher is regularly reseeded from the OS to
provide backtracking and prediction resistance with a negligible cost. The
reseed interval simply needs to be adjusted to the point that it stops
registering as having any significant performance impact. The performance
impact on recent Linux kernels is primarily from the high cost of system calls
and locking since the implementation is quite efficient (ChaCha20), especially
for just generating the key and nonce for another stream cipher (ChaCha8).

ChaCha8 is a great fit because it's extremely fast across platforms without
relying on hardware support or complex platform-specific code. The security
margins of ChaCha20 would be completely overkill for the use case. Using
ChaCha8 avoids needing to resort to a non-cryptographically secure PRNG or
something without a lot of scrunity. The current implementation is simply the
reference implementation of ChaCha8 converted into a pure keystream by ripping
out the XOR of the message into the keystream.

The random range generation functions are a highly optimized implementation
too. Traditional uniform random number generation within a range is very high
overhead and can easily dwarf the cost of an efficient CSPRNG.

# Size classes

The zero byte size class is a special case of the smallest regular size class.
It's allocated in a dedicated region like other size classes but with the slabs
never being made readable and writable so the only memory usage is for the slab
metadata.

The choice of size classes for slab allocation is the same as jemalloc, which
is a careful balance between minimizing internal and external fragmentation. If
there are more size classes, more memory is wasted on free slots available only
to allocation requests of those sizes (external fragmentation). If there are
fewer size classes, the spacing between them is larger and more memory is
wasted due to rounding up to the size classes (internal fragmentation). There
are 4 special size classes for the smallest sizes (16, 32, 48, 64) that are
simply spaced out by the minimum spacing (16). Afterwards, there are four size
classes for every power of two spacing which results in bounding the internal
fragmentation below 20% for each size class. This also means there are 4 size
classes for each doubling in size.

The slot counts tied to the size classes are specific to this allocator rather
than being taken from jemalloc. Slabs are always a span of pages so the slot
count needs to be tuned to minimize waste due to rounding to the page size. For
now, this allocator is set up only for 4096 byte pages as a small page size is
desirable for finer-grained memory protection and randomization. It could be
ported to larger page sizes in the future. The current slot counts are only a
preliminary set of values.

| size class | worst case internal fragmentation | slab slots | slab size | internal fragmentation for slabs |
| - | - | - | - | - |
| 16 | 93.75% | 256 | 4096 | 0.0% |
| 32 | 46.875% | 128 | 4096 | 0.0% |
| 48 | 31.25% | 85 | 4096 | 0.390625% |
| 64 | 23.4375% | 64 | 4096 | 0.0% |
| 80 | 18.75% | 51 | 4096 | 0.390625% |
| 96 | 15.625% | 42 | 4096 | 1.5625% |
| 112 | 13.392857142857139% | 36 | 4096 | 1.5625% |
| 128 | 11.71875% | 64 | 8192 | 0.0% |
| 160 | 19.375% | 51 | 8192 | 0.390625% |
| 192 | 16.145833333333343% | 64 | 12288 | 0.0% |
| 224 | 13.839285714285708% | 54 | 12288 | 1.5625% |
| 256 | 12.109375% | 64 | 16384 | 0.0% |
| 320 | 19.6875% | 64 | 20480 | 0.0% |
| 384 | 16.40625% | 64 | 24576 | 0.0% |
| 448 | 14.0625% | 64 | 28672 | 0.0% |
| 512 | 12.3046875% | 64 | 32768 | 0.0% |
| 640 | 19.84375% | 64 | 40960 | 0.0% |
| 768 | 16.536458333333343% | 64 | 49152 | 0.0% |
| 896 | 14.174107142857139% | 64 | 57344 | 0.0% |
| 1024 | 12.40234375% | 64 | 65536 | 0.0% |
| 1280 | 19.921875% | 16 | 20480 | 0.0% |
| 1536 | 16.6015625% | 16 | 24576 | 0.0% |
| 1792 | 14.229910714285708% | 16 | 28672 | 0.0% |
| 2048 | 12.451171875% | 16 | 32768 | 0.0% |
| 2560 | 19.9609375% | 8 | 20480 | 0.0% |
| 3072 | 16.634114583333343% | 8 | 24576 | 0.0% |
| 3584 | 14.2578125% | 8 | 28672 | 0.0% |
| 4096 | 12.4755859375% | 8 | 32768 | 0.0% |
| 5120 | 19.98046875% | 8 | 40960 | 0.0% |
| 6144 | 16.650390625% | 8 | 49152 | 0.0% |
| 7168 | 14.271763392857139% | 8 | 57344 | 0.0% |
| 8192 | 12.48779296875% | 8 | 65536 | 0.0% |
| 10240 | 19.990234375% | 6 | 61440 | 0.0% |
| 12288 | 16.658528645833343% | 5 | 61440 | 0.0% |
| 14336 | 14.278738839285708% | 4 | 57344 | 0.0% |
| 16384 | 12.493896484375% | 4 | 65536 | 0.0% |

The slab allocation size classes currently end at 16384 since that's the final
size for 2048 byte spacing and the next spacing class matches the page size of
4096 bytes on the target platforms. This is the minimum set of small size
classes required to avoid substantial waste from rounding. Further slab
allocation size classes may be offered as an option in the future.

# API extensions

The `void free_sized(void *ptr, size_t expected_size)` function exposes the
sized deallocation sanity checks for C. A performance-oriented allocator could
use the same API as an optimization to avoid a potential cache miss from
reading the size from metadata.

The `size_t malloc_object_size(void *ptr)` function returns an *upper bound* on
the accessible size of the relevant object (if any) by querying the malloc
implementation. It's similar to the `__builtin_object_size` intrinsic used by
`_FORTIFY_SOURCE` but via dynamically querying the malloc implementation rather
than determining constant sizes at compile-time. The current implementation is
just a naive placeholder returning much looser upper bounds than the intended
implementation. It's a valid implementation of the API already, but it will
become fully accurate once it's finished. This function is **not** currently
safe to call from signal handlers, but another API will be provided to make
that possible with a compile-time configuration option to avoid the necessary
overhead if the functionality isn't being used (in a way that doesn't change
break API compatibility based on the configuration).

The `size_t malloc_object_size_fast(void *ptr)` is comparable, but avoids
expensive operations like locking or even atomics. It provides significantly
less useful results falling back to higher upper bounds, but is very fast. In
this implementation, it retrieves an upper bound on the size for small memory
allocations based on calculating the size class region. This function is safe
to use from signal handlers already.
