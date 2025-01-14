# Parallelism

## Parallel Organisation

- Flynn's Taxonomy:
  - SISD
    - Standard uniprocessor stuff
  - SIMD
    - Vector/Array Processors
    - Single machine instruction executes on a number of processing elements in lockstep
  - MISD
    - Not really used
  - MIMD
    - Distributed memory systems (cluster-based)
      - Communicate via message passing, very scalable
    - Shared memory systems
      - Communicate via memory and are easy to program but memory contention can happen
      - Symmetric multiprocessors
      - NUMA
- Vector computers employ lots of arithmetic pipelines for SIMD processing
  - Instructions operate on vectors of numbers (one or two dimensional)
  - One operation specified for all elements of the vector
  - 2 main types of architecture:
    - memory-to-memory
    - register-to-register (specific vector registers)
  - Chaining often used - chain pipelines together for operations such as FMA
    - Connect inputs/outputs via crossbar switches
  - SIMD array computers had good performance for specific applications, but they're old and no-one makes them anymore
    - Special set of instructions broadcast to processing elements for execution
  - Array computer are dead but MMX, SSE, AVX are big in x86
  - ARM has NEON coprocessor, a 10-stage SIMD pipeline
- Interconnection structure are important in allowing data or memory to be shared
  - In distributed memory systems, communication is in software via ethernet or infiniband
  - More efficient interconnects are needed to share memory
    - A shared bus allows processor and memory to share a communication network
      - Need to resolve bus contention issues
      - Poor reliability
      - Only good for small systems
    - A cross-bar switch matrix uses a matrix of interconnects
      - Functional units require minimal logic
      - Switch is complex, large and costly
      - Potentially high bandwith, but still struggles with contention
    - Static links between each processor enable dedicated communication
      - More links -> better communication rate
      - Different patterns have different performance properties
      - Chosen architecture of links usually is a tradeoff between cost and performance
        - Hypercube is a good balance
        - Number of connections and links per node are a good indication of cost
        - Maximum inter-node distance is an indicator of worst-case communication delay
      - Can have a dedicated link for each pair but that's expensive and rarely necessary
  - Multistage switching networks can be either cross-bar or cell-based
    - Requirement is to connector each processor to any other processor
      - Known as the full access property
    - Another useful property is that connections are non-blocking
    - CLOS networks (multi-stage cross-bar switches) showed that a network with 3 or more stages can be non-blocking
    - A CLOS network with 2x2 cross-bar elements is known as a Benes Network, classified as cell-based
      - Most cell-based networks are highly blocking but require few switches

## Cache Coherence

- Shared memory MIMD systems are easy to program, and can overcome memory contention via cache
- Copies of the same data may now be in different places
  - Cache coherence must be maintained
  - A write-through policy is not sufficient as that only updates main memory
  - It is necessary to update other caches too
- Possible solutions include:
  - Shared caches
    - Poor performance for more than a few processors
  - Non-cacheable items
    - Can only write to main memory, causes problems
  - Broadcast write
    - Every cache write request is broadcast to all other caches
    - Copies either updated or invalidated, preferably the latter as it is faster
    - Increases memory transactions and wastes bus bandwidth
  - Snoop bus
    - Suitable for single-bus architectures
    - Cache write-through is used
    - A bus watcher (cache controller) is used and snoops on the system bus
      - Detects memory write operations, and invalidates local cached copies if main memory updated
  - Directory methods
    - A directory is a list of entries identifying cached copies
      - Used when a processor writes to a cached location to invalidate or update other copies
    - Various methods exist
    - Suitably for shared memory systems with multistage or hierarchical interconnects where broadcast systems are hard to implement
    - Full directory has a directory in main memory
      - A set of pointers per cache and a dirty bit is used with each shared data item
      - Bit set high if cache has a copy
      - Each word/block/line in cache has two state bits:
        - Valid bit, set if cache data is valid
        - Private bit, set if processor is allowed to write to the block
    - Limited directories only stored pointer for the number of caches that have the data
      - Saves memory storing pointers for caches that don't have data
      - Only $n$ pointers required, but each pointer must uniquely identify one of the $N$ caches
        - $\log_2 N$ pointers required for each pointer instead of 1 bit
      - Requires $n \log_2 N$ bits instead of $N$ bits
      - Scales much better as entries grow less than linearly
    - Chained directories also attempt to reduce the size of the directory
      - Use a linked list to hold directory items
      - Shared memory directory entry points to one copy in a cache, from there a pointer points to next copy, so on..
      - $N$ copies may be maintained
      - Whenever a new copy called for, list broken and pointers altered
- MESI is the good protocol
  - Snoop bus arrangement used with a write-back policy
  - Two status bits per cache line tag so it can be in one of four states
    - Modified: entry valid, main memory invalid, no copies exist
    - Exclusive: no other cache holds line, memory up to date
    - Shared: multiple caches hold line, memory is up to date
    - Invalid: cache entry is garbage
  - When machine booted, all entries are invalid
  - First time memory is read, block referenced is fetched by CPU 1 and marked exclusive
    - Subsequent reads by same processor use cache
  - CPU 2 fetches same block
    - CPU 1 sees by snooping it is no longer alone and announces it has a copy
    - Both copies marked shared
  - CPU 2 wants to write to the block
    - Puts invalidate signal on bus
    - Cached copy goes into modified state
    - If block was exclusive, no need to signal on bus
  - CPU 3 wants to read block from memory
    - CPU 2 has the modified block, so tells 3 to wait while it writes it back
  - CPU 1 wants to write a word in the block (cache)
    - Assuming fetch on write, block must be read before writing
    - CPU 1 generates a Read With Intend To Modify (RWITM) sequence
      - CPU 2 has a modified copy so interrupts the sequence and write to memory, invaliding it's own copy
      - CPU 1 reads block from memory, updates it and marks it modified
  - All read hits do not alter block state
  - All read misses cause a change to shared state
- Intel and AMD took different approaches to extending MESI
  - Intel uses MESIF
    - Forward state is a specialised shared state
    - Serving multiple caches in shared state is inefficient, so only the cache with the special forward state responds to requests
      - Allows cache-to-cache speeds
  - AMD uses MOESI
    - Owned state is when a cache has exclusive write rights, but other caches may read from it
      - Changes to line are broadcast to other caches
    - Avoids writing dirty line back to main memory
      - Modified line provided from the owning cache

## Data Level Parallelism

- The utilisation of SIMD depends on applications having a degree of data-level parallelism
  - Matrix oriented computation
  - Image and sound processing
- Sequential thinking but parallel processing makes it easy to reason about
- Vector-specific architecures make SIMD easy but practicality is limited
  - Reduced fetch/decode bandwith as fewer instructions
  - Programmers view is:
    - Transfer data elements to register files
      - Essentially compiler-managed buffers for data
      - Fixed length buffer to store a single vector
        - Eg, each register holds 64 words
        - Needs enough ports to service all functional units
        - Ports connect to functional units over crossbar switch
    - Operate on register files
      - Functional units heavily pipleined
      - Integrated control units detect structural or data hazards
      - Also provide scalar units to compute addresses
        - Can be chained with vector units
    - Place results back in memory
  - Loads and stores are pipleined
    - Program pays memory latency cost just once, instead of once per data element
  - Three contributing performance factors are:
    - Length of vector ops
    - Structural hazards
    - Data dependencies
  - Performance can be considered in terms of vector length or initiation rate
  - Modern vector computers employ parallel pipelines known as lanes
    - Superscalar architecture
  - Convoys are sets of vector instructions that can execute together
    - Performance of code sections can be estimated by counting number of convoys
    - Need to ensure no structural hazards exist
    - A chime refers to the unit of time to execute a single convoy
      - A vector sequence of $n$ convoys executes in $n$ chimes
      - Approximation ignores processor specific overhead and allows to readon about inherent data-level parallelism
  - Chaining can be used to acheive performance, as it allows operations to be initiated as soon as individual elements of the vector source are available
    - Earliest implementations work in a similar way to forwarding in scalar pipelines
    - Flexible chaining allows a vector instruction to chain to almost any other active vector instruction
      - Have to take care not to introduce hazards
      - Supported by modern architectures
  - A number of techniques can be applied to optimise vector architectures
    - Can have multiple lanes, a single vector instruction can be split up to execute accross the lanes
      - Doubling lanes but halving clock rate does not change speed
      - Increases size and energy consumption
    - Vector length registers vary the size of the vector operations
      - Value cannot be greater than the max vector length, the physical register size
      - Strip mining is a technique that generates code such that each vector operation is done for a size less than or equal to the max vector length
    - Vector mask registers allow for conditional execution of each element operation, when usually conditionals would be needed that hinder performance
    - Memory banking spreads memory accesses across multiple memory banks to improve the start up time for a vector load
- MMX/SSE/AVX provide SIMD in x86
  - Many media applications operate on a narrower range of data types than 32-bit processors are designed for
    - 8-bit colour components
    - 16-bit audio samples
  - A 256-bit adder can operate on 32 8-bit values at once
  - MMX was introduced by intel in 1996
    - Used 64-bit FP registers to provide 8 and 16-bit operations
  - SSE was introduced as the successor, adding 128-but wide registers
  - AVX introduced in 2010 adds 256 bit registers with a focus on double precision FP
    - AVX-512 introduced doubles register size again
  - Focus of SIMD extensions is to accelerate carefully implemented code
    - Low cost to use
    - Require little extra state compared to vector architectures
    - No virtual memory problems
- GPUs are powerful vector units that are similar to vector architectures
  - Hardware designed for graphics but usually supplemented to improve the performance of a wider range of applications
  - Heterogeneous execution model
    - CPU is host, GPU is device
  - NVIDIA have CUDA for programming, OpenCL is vendor-independent
  - GPUs provide high levels of every form of parallelism, but it is hard to achieve performance as must also manage
    - Scheduling of computation
    - Transfer of data to GPU memory
  - CUDA threads are the lowest form of parallelism, one associated with each data element
    - Can group thousands of threads to yield other forms of parallelism
    - Threads organised into blocks, multithreaded SIMD processor executed a whole thread block
    - Blocks organised into grids, executed independently and in any order
    - GPU hardware handles thread management

## Multicore Systems

- Can consider the performance of a processor in terms of the rate at which it executes instructions
  - MIPS = freq \* IPC
  - Leads to an focus on increasing clock frequency and processor efficiency
    - We've kinda hit a ceiling with this
- Alternative approach is multithreading
  - Divide instruction stream into smaller streams to execute threads in parallel
  - Various designs and implementations
    - Threads may or may not be the same as software threads in multiprogrammed OS
- A process is an instance of a running program
  - Processes own resources in their virtual address space
  - Processes are scheduled by the OS
  - Process switch is an operation that switches the processor form one process to another
- A thread is a unit of work within a process
  - Thread switch switches processor control from one to another within the same process
  - Far less costly than processes & process switches
- Implicit multithreading is the concurrent execution of multiple threads from a single sequential program
  - Statically defined by compiler or dynamically in hardware
  - Rarely done as it hard
- Most processors have adopted explicit multithreading, which concurrently execute instructions form different threads by either:
  - Uses separate program counter for each thread
  - Instruction fetching happens per thread
  - Each thread treated and optimised separately
  - Multiple approaches:
    - Interleaved, where processor deals with more than one at a time, switching at each clock cycle
      - Thread skipped when blocking
    - Blocking or coarse grained, where threads execute successively until an event occurs that may cause a delay
      - Delay prompts a switch to another thread
    - SMT, where instructions are issues from multiple threads to the execution units of a superscalar processor
      - Performance comes from superscalar capability combined with multiple thread contexts
    - Chip multiprocessing replicates entire processor on same chip
      - Multicore
  - Interleaved and blocked do not provied true concurrency, whereas SMT and multicore are actual simultaneous execution
  - Multicore systems combine multiple cores on a single die
    - Each core has its own components (ALU, registers, PC) and caches
    - Pollack's rule: performance increase is roughly proportional to square root of increase in complexity
      - If we double the logic, will deliver 40% perf boost
      - Multicore has potential for near-linear improvement but is hard to acheive
    - Main variables are number of cores, and levels and amount of shared cache
      - Can have dedicated L1/L2
      - Can share L2 or have dedicated L2 and share L3
      - Shared L2 cache has advantages over reliance on dedicated cache
        - Constructive interference can reduce miss rates
        - Data shared is not replicated in shared cache
        - Amount of shared cache for each core is dynamic
        - Interprocessor communication can happen through cache
        - Confines cache coherence problem to L1 cache
- Clusters
  - A group of interconnected whole computers working together as a unified computing resource, that creates the illusion of a single machine
  - Alternative to multiprocessing for high performance and availability
  - Attractive for servers
  - Absolute and incremental scalability, high reliability, superior price/performance ratio
  - High-speed interconnects needed
- With uniform memory access, all processors have access to all the memory in uniform time
  - NUMA, Non Uniform Memory Access, gives different access times to different processors for different regions of memory
    - All processors can still access all memory, just slower
    - Cache Coherent NUMA (CC-NUMA) extends NUMA with cache coherence between the processors
  - Used because SMP approaches don't scale, and allows for transparent-system wide memory
  - Could motivate clusters, but clusters are hard to program effectively

## Thread Level Parallelism

- Synchronisation primitives exist in hardware that allow high-level synchronisation constructs to be built
  - Establish building blocks to build actual constructs used by programmers
- Most important hardware provision is the atomic instruction
  - Uninterruptible and capable of incurring value change
  - May actually be an atomic instruction sequence
- In high-contention sequence, synchronisation can become a performance bottleneck
- Atomic exchange is a primitive that swaps a value in a register for a value in memory
  - Can be used to build locks for synchronisation
    - Assume a value of 0 indicates the lock is free, 1 indicates it is unavailable
  - Simplest possible situation where two processors both wish to perform an atomic exchange
    - One processor will enter the exchange first
    - This processor will ensure that a value of 1 is returned to any other processor that next attempts an exchange
    - The two simultaneous exchange operations will be ordered by write serialisation mechanisms
- Older microprocessors feature a test-and-set atomic instruction in hardware
  - Allowed to define a test against which a value can be tested
  - Value modified if defined test succeeded
- Some current gen microprocessors have fetch-and-increment atomic
  - Return the value at a pointer and increment it
- Atomic instructions usually consist of some read and write
- Requiring an uninterruptible read-write fucks with a good number of things
  - Cache coherence
  - Instruction pipelining
  - Cache performance
- Possible to have a pair of atomic instructions where the second instruction returns a value that indicates if the pair executed atomically
  - Pair includes a special load known as load linked, followed by a special write, store conditional
    - If they memory location specified by load linked is accessed prior to the store conditional then the store fails
    - Also fails if there is a context switch
  - Can implement atomic exchange using this
    - If the store conditional returns a value indicating failure, then a branch jumps back and retries
  - Can also implement fetch-and-increment
    - Maintain a record of the address specified by linked load in a link register
    - If an interrupt occurs or cache block containing address is invalidated, register is cleared
    - Conditional store checks register for address matching to determine success
    - To avoid deadlock, only register to register operations are permitted between linked-store instructions
- Spin locks are locks that processors repeatedly attempt to required
  - Effective when low latency required and lock held for short periods
  - Processors with cache coherence provide a convenient mechanism for spin locks
    - Testing the status of a lock requires local cache access rather than main memory access
    - Temporal locality decreases lock acquisition times
  - Linked-store can avoid needless bus access when multiple processors attempt to acquire a lock
- Cache coherence ensures multiple processors have a consistent view of memory, so allows communication through shared memory
  - Shared memory communications means we only need consider the rules enforced on reads and writes of different processors
    - Don't need to sync _everything_
- Different models of memory consistency exist
  - Simplest is sequential consistency
    - Requires the results of execution be the same if memory accesses of processors were kept in order and interleaved
    - Ensured all processors delay memory accesses until all cache invalidations are complete
    - Simple but slow
  - Synchronised consistency orders all accesses to shared data using synchronisation operations
    - A data reference is ordered by a synchronisation operation if, in every possible execution, a write by one processor and an access by another are separated by a pair of synchronisation operations
    - Whenever a variable might be updated without ordering by synchronisation is a data rate
  - There are relaxed consistency models that allow reads and writes to complete out-of-order but use synchronisation to enforce ordering
    - Three general models
    - A -> B denotes that A must complete before B
    - Total store ordering relaxes W -> R
      - Retrains ordering among writes
    - Partial order store model relaxes W -> W
      - Impractical for most programs
    - Relaxing R -> R and R -> W happens in a variety of models, including weak ordering and release consistency

## High Performance Systems

- Symmetric Multiprocessors (SMP) is an organisation of two or more processors sharing memory
  - Processors connected by bus
  - Uniform memory access
  - All processors are the same and share I/O
  - System controlled by integrated OS
  - Performant for parallel problems
  - All processors are the same so if one processor goes down another is still available
  - Can scale incrementally
  - Most PCs use a time-shared bus but can also use multi-port memory in more complex organisations
- Clusters are an alternative to SMP
  - A cluster computer is defined as a group of interconnected computers (nodes) working together as a unified resources
  - High performance and availability
  - Attractive for server applications
  - Absolute and incremental scalability
  - Superior price/performance
  - High speed message links required to coordinate activity
  - Machines in a cluster may or may not share disks
  - Cluster middleware provides a unified system image to the user
    - Responsible for load balancing, fault tolerance, etc
    - Desireable to have:
    - A single entry and control point/workstation
    - Single file hierarchy
    - Single virtual networking
    - Single memory space
    - Single job-management system
    - Single U/I
    - Single I/O space
    - Single Process space
    - Check pointing, to save the process state and intermediate results
    - Process migration, to enable load balancing
- Both clusters and SMP provide multiple processors for high-demand applications
  - SMP easier to manage and configure, take up less space and power
    - Bus architecture limits processors to around 16~64
  - Clusters dominate high-performance server market
    - Scalable to 1000s of nodes
- Uniform memory access used in SMP organisations
- Memory access time varies in NUMA systems
  - NUMA with no cache coherence is more or less a cluster system
- CC-NUMA is NUMA with cache coherence
  - Objective is to maintain a transparent system memory while permitting multiple nodes
  - Nodes each have own SMP organisations and internal busses/interconnects
  - Each processor sees a single addressable memory
  - Cache coherence usually done via a directory method
  - Can deliver effective performance at higher levels of parallelism than SMP
  - Bus traffic on any individual node is limited by bus capacity
  - If many memory accesses are to remote performance degrades
  - Software changes required to go form SMP to CC-NUMA systems
