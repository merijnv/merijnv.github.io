# Merijns musings

*A blog, of sorts*

------


## 2021-04-19 Ben Eater 8 bit machine explorations

A few days ago I found [this youtube video](https://www.youtube.com/watch?v=ym8x8B930tM) on the channel of [Rob Simmons aka simrob](https://www.youtube.com/channel/UCjbRsZ-KFZ0-oFWZeyfrB8A).

It presents the fact that because the state space of a computer is limited,
halting is fundamentally decidable. TL;DR (but please read [the paper]()):
If a computer has n bits of memory (including registers), then it has two to the power of n states.
On each step of the computer, it must either reach a halting state (execute the HLT instruction) or it must change the state.
After two to the power of n plus one states, due to the pigeonhole principle, a state must be repeating.

Due to the deterministic nature of computers, from then on, a loop is reached.

So fundamentally, there are two possibilities, either a computer halts or we enter a loop.


### The challenge

Given a very small computer, can we calculate the longest running program on that computer.

Rob Simmons chose the 8-bit breadboard computer as presented by [Ben Eater on youtube]() to try this experimentally.

It has a very limited state space: it has one general register, a program counter, a few bytes of memory and an 8-bit display.
The total bit count is about 86 bits, so the space is still two to the power of 86, which is quite huge.
The instruction set is very limited, with less than sixteen instructions.

By limiting the memory to only four addresses, we can just try out every possible program and let it run!

The challenge now is executing all those program and detecting the program that can run for the longest before it either halts or loops.

Doing that on the 8-bit bread board computer itself is unfeasible: it typically runs a a couple of hertz, maybe a tens of kilohertz if one pushes it.

To execute this challenge, I built a simulator that has the same memory and cpu internal state behaviour as the Ben Eater 8-bit breadboard computer.
Then it was a matter of a lot of debugging and finally executing a full run with all 4.2 billion possible programs.

### Mistakes I made... assumptions

The internal state behaviour of the CPU is non-intuitive when considering the
`carry` flag in combination with subtraction. This is due to the clever way
that Ben Eater executes subtraction: it actually is addition with the twos
complement of the value one wants to subtract.

Another assumption was that in my first implementation, I skipped all `illegal` instructions: ie undefined instructions.
The computer is a Von Neumann architecture and the most interesting programs modify the cpu instructions. 

Then there were a few bugs when parallelizing it...

But at the end of the weekend when I started this, I had a final result.


### Reproducing the results by @simrob

The execution of a full run with all programs took quite precisely six hours. Running on 16 cores of a Ryzen 1700 processor.
Finally my log ended with:

```
======================== FINAL RESULTS =======================
Steps: 859 Program: 0x31: SUB 1 | 0x8f: JZ f | 0x60: JMP 0 | 0x42: STA 2 |  Final state: Cpu[a=0x62, d=0x5f, pc=0xa, zero=false, carry=true, m=[0x31, 0x8f, 0xf1, 0x42]] - HALTED
Loop size: 1079 Program: 0x62: JMP 2 | 0x21: ADD 1 | 0x71: JC 1 | 0x33: SUB 3 |  Repeated state: Cpu[a=0xcd, d=0x0, pc=0x2, zero=false, carry=false, m=[0x62, 0x21, 0x71, 0x33]]
running time: 21540444ms
```

This confirms the result by Rob Simmons that the longest running program executes for 859 steps before halting and that the longest loop is 1079 steps long.

Bonus fact: some loops were reached after over 2048 steps. I discovered that by accident, that is, running out of memory.


### Extending the results

Now comes the fun part, let's extend this with longer programs in an attempt to find longer runs.
Stay tuned...
