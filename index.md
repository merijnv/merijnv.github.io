# Merijns musings

*A blog, of sorts*


## 2021-04-27 Optimizations of the simulator

### Java has an undeserved reputation

Some people still consider Java to be "not so fast". This is arguably not the
case. The optimizers within most implementations are really good. OpenJDK is
already good and the commercially available JVMs are possibly better in quite a
few cases. Oracle develops GraalVM, while Azul and Microsoft, IBM and Red Hat
have their own variants.

Tailoring to a modern AMD or Intel processor is hard. The instruction set of the 
x86 has become really large and compilers and runtimes are the only way you can interact with them.

I've seen cases where Java goes through (calculation) loops so quickly that I'm sure
it uses SIMD instructions out of the box. From Java 16 onwards, if you need to,
you can code type-safe Java in such a way that you really know what



### Memory allocation is fast, as is GC

Memory allocation in Java is generally very fast. Typically the JVM only
increases a pointer (to point to the next place to allocate), writes a few
words of meta data to the heap and returns the original value of the pointer.
This is called 'bump the pointer' allocation.

De-allocation is free from a programmers perspective: when an object becomes
unreachable (it's reference counter becomes zero), is becomes available for
garbage collection.

Short lived objects are really fast to de-allocate. Objects are created in a
memory space called Eden. If Eden gets full a (minor) GC is triggered. Objects
that are still reachable are copied over to the new Eden, the pointer there
becomes the new allocation pointer and on we go.  Objects that get copied a few
rounds in Eden get "promoted" to the next memory space (Survivor).

This may sound like a lot of copying, but if the objects are really
short-lived, the Eden GC has not a lot of work. In current JVMs, the space is
divided cleverly around the address space and collecting garbage has become
fully parallelizable.

Analyzing the behaviour of the simulator I concluded that each second, less
than a handful of milliseconds were spent on GC. And I create heaps of garbage
in the process (pun intended): each next state of the Ben Eater machine gets
stored in its own immutable object. Judging by the GC counter, 16 cores running
this creates over 300Mb of garbage each second!

### Heap memory comes at a cost: address space and cache misses

The heap of a JVM lives in RAM, which is accessed through the CPU caches.  In
the pre-optimized version, on my machine, it is accessing memory for data at a
rate of about 2.5G requests/second. The number of bytes fetched for each
request is 8 according to the AMD Professor Programming Reference (PPR)
documentation.


### Good news and bad news

As can be seen in perf output below, we quite often miss the L1 cache. About
20% of all memory requests miss the L1Data cache. It is 'only' 32k per core on
my Ryzen 1700 CPU.

Overall, it looks like only 0.07% of the memory requests miss all caches and go
to main memory. Apparenly, most of the time we stay in the L3 cache, even
though more than 500Mb/s of garbage is created. Still I do not know for sure
what to make of this CPU counter. The AMD PPR documentation is not too helpful.


```text
 Performance counter stats for 'java -jar target/ben-eater-sim-1.0-SNAPSHOT-jar-with-dependencies.jar':

   804,784,020,712      cache-references:u
       544,127,706      cache-misses:u            #    0.068 % of all cache refs
 3,831,997,271,795      L1-dcache-loads:u
   650,546,928,348      L1-dcache-prefetches:u
   708,160,686,499      L1-dcache-load-misses:u   #   18.48% of all L1-dcache accesses

      98.504823274 seconds time elapsed

    1529.546817000 seconds user
       5.635150000 seconds sys
 ```

This run is obtained by executing all programs that start with a 0 in the first
memory position and then simulating all 2^24 (16.777.216) possible programs.


### Optimization strategy

How to optimize this?

Observations:
* I keep an array of 8192 pointers (4 bytes each) in each thread to hold the CPU states.
	* Flushing this effectively writes 32kb of 0s to memory, probably effectively cleaning the cache.
* Each state has a record associated with it which gets created on the heap
    * These records are immutable.
	* Typically the are short-lived
    * They need to be allocated and GC'd
    * In all the 16M programs, most of the programs will run for only 4-16 steps.
	* If the average is, say, 16, then we allocate 256M of these records.


What to change:
* Make this core allocation-free
    * Make the state trace mutable `Objects` (rather than immutable `records`)
    * Allocate all 8192 objects for the trace on each thread once
    * Clean-up only the records that were used on each iteration

Expected results:
* Fewer allocations (by a lot)
* Fewer cache misses
* Faster execution

TO BE CONTINUED...

------

## 2021-04-27 Gathering statistics

See previous post for a description of the "machine" we're working with.

The machine to analyze in this blog post is the 4 byte memory (2 bits of
address space) variant of the Ben Easter breadboard computer. My program can
simulate all 4 billion programs in six hours of hard work.

### Statistics - number of steps before either halting or repeating

My susipcion is that most of the programs are very very short lived. Many of
the programs change nothing in the memory.  Many of the programs do not change
the register content. The only thing changing in the machine is the program
counter, which by design goes from 0 to 15. As soon as it reaches 0 again, the
state gets repeated.

Halting is another matter: if the program contains a `HLT` instruction and no
jumps, the program will die really quickly.

These statistics can help deduce where we are likely to find optimizations.
Also they just serve curiosity: how many steps did we actually simulate in total?


### Implementing stats gathering

My simulator is written in Java. Why? Because I'm always trying to prove that
Java is really fast and for me it is the language I can develop in most
quickly.

Because this problem is *embarissingly parallelizable* it spins up 16 threads
that each run a `SimulationRunner`. These threads fetch a task, which is just
the first `n-1` bytes of memory. It then runs all 256 programs where it changes
the last byte of memory, before requesting the next task.

Each simulation run results in a `SimulationResult` object that currently
contains the best looping programs and best halting program.

There is a static class collector `BestProgram` which keeps track of the best program so far.


------

## 2021-04-20 Submitted a result, more may follow!

See previous post.

I changed a few things around and discovered a couple of interesting programs.
Now my hard-working program will take random programs and run them. If they run for too long, they are logged as interesting...

Submitted is this result, which runs for over 18000 steps before it detects the loop of length 8482.

For my simulation, illegal instructions are treated as NOPs. The code is the same as data in this system, and every interesting program (a program that runs in a long loop) is self-modifying.


```
bestLoop = Loop size: 8482 Program: 0xd5: ILL 5 | 0xe8: OUT 8 | 0x28: ADD 8 | 0xc4: ILL 4 | 0x77: JC 7 | 0x29: ADD 9 | 0x49: STA 9 | 0x2d: ADD d |  Repeated state: Cpu[a=0x96, d=0x0, pc=0x7, zero=false, carry=true, m=[0xd5, 0x96, 0x28, 0xc4, 0x77, 0x29, 0x49, 0x2d]]
```

----

## 2021-04-19 Ben Eater 8 bit machine explorations

A few days ago I found [this youtube video](https://www.youtube.com/watch?v=ym8x8B930tM) on the channel of [Rob Simmons aka simrob](https://www.youtube.com/channel/UCjbRsZ-KFZ0-oFWZeyfrB8A).

It presents the fact that because the state space of a computer is limited,
halting is fundamentally decidable. TL;DR (but please read [the
paper](https://sisyphean.glitch.me/paper.pdf)): If a computer has n bits of
memory (including registers), then it has two to the power of n states.  On
each step of the computer, it must either reach a halting state (execute the
HLT instruction) or it must change the state.  After two to the power of n plus
one states, due to the pigeonhole principle, a state must be repeating.

Due to the deterministic nature of computers, from then on, a loop is reached.

So fundamentally, there are two possibilities, either a computer halts or it
enters a loop.


### The challenge

> Given a very small computer, can we calculate the longest running program on that computer?

Rob Simmons chose the 8-bit breadboard computer as presented by [Ben Eater on
youtube](https://www.youtube.com/watch?v=HyznrdDSSGM&list=PLowKtXNTBypGqImE405J2565dvjafglHU) to try this experimentally.

It has a very limited state space: it has one general register, a program
counter, a few bytes of memory, an 8-bit display and a few CPU state bits.  The
total bit count is about 142 bits and the space is still two to the power of 142.
That is still quite a big space. The instruction set is very limited, with
less than sixteen instructions.

By limiting the memory to only four addresses of one byte, we can just try out every
possible program and let it run! That is 2^32 or 4.2 billion programs.

The challenge now is executing all those program and detecting the program that
can run for the longest before it either halts or loops.

Doing that on the 8-bit bread board computer itself is unfeasible: it typically
runs a a couple of instructions per second. Maybe it can reach a thousand per
second but that's not enough to run 4 billion programs on to completion. Also
there is no way to detect that a state has been reached earlier.

To execute this challenge, I built a simulator that has the same memory and cpu
state behaviour as the Ben Eater 8-bit breadboard computer. It keeps track of
all states while executing, checking after each step if a state repeats.

It took some debugging and finally executing a full run with all 4.2 billion
possible programs.

### Mistakes I made...

The internal state behaviour of the CPU is non-intuitive when considering the
`carry` flag in combination with subtraction. This is due to the clever way
that Ben Eater executes subtraction: it actually is addition with the twos
complement of the value one wants to subtract. So subtraction toggles the carry
flag in the same way that the add instruction does. Which is not quite the same
as borrowing.

But at the end of the weekend, I had a final result.

### Reproducing the results by @simrob

Running on 16 cores of a Ryzen 1700 processor, the execution of a full run with
all programs took six hours.  Finally it completed with:


```
======================== FINAL RESULTS =======================
Steps: 859 Program: 0x31: SUB 1 | 0x8f: JZ f | 0x60: JMP 0 | 0x42: STA 2 |  Final state: Cpu[a=0x62, d=0x5f, pc=0xa, zero=false, carry=true, m=[0x31, 0x8f, 0xf1, 0x42]] - HALTED
Loop size: 1079 Program: 0x62: JMP 2 | 0x21: ADD 1 | 0x71: JC 1 | 0x33: SUB 3 |  Repeated state: Cpu[a=0xcd, d=0x0, pc=0x2, zero=false, carry=false, m=[0x62, 0x21, 0x71, 0x33]]
running time: 21540444ms
```

This confirms the result by Rob Simmons that the longest running program executes for 859 steps before halting and that the longest loop is 1079 steps long.

Bonus fact: some loops were reached after over 2048 steps. I discovered that by accident, that is, running out of space for detecting loops :).


### Extending the results

Now comes the fun part, let's extend this with longer programs in an attempt to find longer runs.
Stay tuned...
