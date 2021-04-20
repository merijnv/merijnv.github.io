# Merijns musings

*A blog, of sorts*

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
total bit count is about 86 bits and the space is still two to the power of 86.
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
