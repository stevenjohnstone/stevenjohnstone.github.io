---
title: "xchg rax,rax"
date: 2019-01-12T17:16:29Z
---

# x86-64 Kata

The book [xchg rax,rax](https://www.amazon.co.uk/xchg-rax-xorpd/dp/1502958082) (content available free [here](https://www.xorpd.net/pages/xchg_rax/snip_00.html)) is
a collection of 0x40 x86-64 snippets. It has no text other
than chunks of assembly language. You read the code and ponder
its meaning. [Kata](https://en.wikipedia.org/wiki/Kata) for the budding reverse engineer.

# Spoiler Alert

Program 0x03 is pretty neat:

<pre>
sub rdx,rax
sbb rcx,rcx
and rcx,rdx
add rax,rcx
</pre>

Let me spoil this for you: at the end of this program, rax will contain
the smaller of the original rax and rdx values. I know this from reading
it. What if I couldn't work out what this did? What tools could I use?

# Assemble It

We can embed the snippet in a C program and run it for some test cases:

{{<gist stevenjohnstone 5d8e6a2c7ab64613bb585856b52ac271>}}

Compile with

<pre>
gcc -masm=intel x3.c
</pre>

and run to get

<pre>
0 (rax: 0000000000000000 rdx 0000000000000001) ->  rax: 0000000000000000
1 (rax: 0000000000000001 rdx 0000000000000001) ->  rax: 0000000000000001
2 (rax: 000000000000000f rdx 000000000000000e) ->  rax: 000000000000000e
3 (rax: 000000000000000e rdx 000000000000000f) ->  rax: 000000000000000e
4 (rax: 00000000ffffffff rdx 0000000000000000) ->  rax: 0000000000000000
</pre>


# Emulation

Suppose that we aren't running on the target system (something without an
x86-64 processor) or we're concerned about the code being malware (when we start
to look at more complex code). Then actually running the code natively isn't an
option.

Emulation allows us to run the code while keeping our tinfoil helmets on. Here's a
program I wrote in Golang which

* assembles [0x03](https://www.xorpd.net/pages/xchg_rax/snip_03.html) into machine code using [keystone](http://www.keystone-engine.org/)
* sets up the [unicorn](https://www.unicorn-engine.org/) emulator to run the machine code
* passes in a few pairs of register values and prints the value of rax at the end

{{<gist stevenjohnstone cbf145560e94b56c0d09ae1aeb24ceb2>}}

Here's the output (just as above):
<pre>
0 (rax: 0000000000000000 rdx 0000000000000001) ->  rax: 0000000000000000
1 (rax: 0000000000000001 rdx 0000000000000001) ->  rax: 0000000000000001
2 (rax: 000000000000000f rdx 000000000000000e) ->  rax: 000000000000000e
3 (rax: 000000000000000e rdx 000000000000000f) ->  rax: 000000000000000e
4 (rax: 00000000ffffffff rdx 0000000000000000) ->  rax: 0000000000000000
</pre>

It appears plausible that 0x03 gives the minimum of rax and rdx. However, this is not a proof. Let's take things
further.

# Z3

[Z3](https://github.com/Z3Prover/z3): read about it for yourself, I wouldn't do it justice. Suffice it to say,
we can use SMT solvers to prove properties of software.

The plan of attack is:

* turn the assembly into [SSA](https://en.wikipedia.org/wiki/Static_single_assignment_form) form
* make _assertions_ which model the effects of the instructions in the program
* show that an assertion which contradicts our _theorem_ leads to the model being unsatisfiable

I've opted to use [SMT-LIB](http://smtlib.cs.uiowa.edu/) to demonstrate.

{{<gist stevenjohnstone da22d90c193c26ab614ce918bb427a77>}}


Running with z3 gives

<pre>
sat
unsat
sat
unsat
</pre>


# Why?

This is surely overkill for such a small problem? Yes. However, as we move to more complex binary executables (e.g. malware)
the ideas touched on here come into their own.









