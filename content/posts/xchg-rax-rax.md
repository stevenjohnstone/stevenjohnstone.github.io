---
title: "xchg rax,rax"
categories:
    - reverse engineering
tags:
    - xchg rax,rax
    - assembly language
    - unicorn engine
    - Z3
    - SAT
date: 2019-01-12T17:16:29Z
---

# x86-64 Kata

The book [xchg rax,rax](https://www.amazon.co.uk/xchg-rax-xorpd/dp/1502958082) ([content available free here](https://www.xorpd.net/pages/xchg_rax/snip_00.html)) is
a collection of 0x40 x86-64 snippets. It has no text other
than chunks of assembly language. You read the code and ponder
its meaning. [Kata](https://en.wikipedia.org/wiki/Kata) for the budding reverse engineer.

# Spoiler Alert

Program 0x03 is pretty neat:

```asm
sub rdx,rax
sbb rcx,rcx
and rcx,rdx
add rax,rcx
```

Let me spoil this for you: at the end of this program, rax will contain
the smaller of the original rax and rdx values. I know this from reading
it. What if I couldn't work out what this did? What tools could I use?

# Assemble It

We can embed the snippet in a C program and run it for some test cases:

```C
#include <stdio.h>

static int x3(int a, int b) {

    int out;
    asm(
        // it's not stated in xchg rax,rax but it's
        // intel syntax. Note the noprefix means we
        // don't need % before registers
        ".intel_syntax noprefix;"
        // a bit of preamble to get registers
        // set up correctly. We're okay with just working
        // with int rather than long int
        "mov eax, %1;"
        "mov edx, %2;"

        // 0x03 code below
        "sub rdx, rax;"
        "sbb rcx,rcx;"
        "and rcx,rdx;"
        "add rax,rcx;"

        // put the result into out
        "mov %0, eax;"
        // tell the compiler that out is in memory
        : "=m" (out)
        // tell the compiler that a and b are inputs
        : "m" (a) , "m" (b)
        // this is the list of registers which are clobbered
        : "rax","rdx","rcx");
    return out;
}

struct input {
    int a;
    int b;
};

int main(int argc, const char **argv) {


    struct input inputs[] = {
        {0x0, 0x1},
        {0x1, 0x1},
        {0xf, 0xe},
        {0xe, 0xf},
        {~0, 0x00},
    };

    for (int i = 0; i < sizeof(inputs)/sizeof(inputs[0]); i++) {
        printf("%d (rax: %016x rdx %016x) ->  rax: %016x\n",
                i, inputs[i].a, inputs[i].b, x3(inputs[i].a, inputs[i].b));
    }
   return 0;
}
```

Compile with

```
gcc -masm=intel 0x3.c
```

and run to get

```
0 (rax: 0000000000000000 rdx 0000000000000001) ->  rax: 0000000000000000
1 (rax: 0000000000000001 rdx 0000000000000001) ->  rax: 0000000000000001
2 (rax: 000000000000000f rdx 000000000000000e) ->  rax: 000000000000000e
3 (rax: 000000000000000e rdx 000000000000000f) ->  rax: 000000000000000e
4 (rax: 00000000ffffffff rdx 0000000000000000) ->  rax: 0000000000000000
```


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

```golang
package main

import (
	"fmt"

	// a branch of keystone golang bindings which builds on linux
	"github.com/stevenjohnstone/keystone/bindings/go/keystone"
	uc "github.com/unicorn-engine/unicorn/bindings/go/unicorn"
)

// This program explores a few inputs to
// problem 0x03 of xchg rax,rax:
// https://www.xorpd.net/pages/xchg_rax/snip_03.html
//
// This is a good excuse to give keystone and unicorn
// a spin in golang
func main() {

	// first, convert the assembly language into numerical opcodes
	// (machine language)
	assembly := "sub rdx,rax; sbb rcx,rcx; and rcx,rdx; add rax,rcx"

	ks, err := keystone.New(keystone.ARCH_X86, keystone.MODE_64)
	if err != nil {
		panic(err)
	}
	defer ks.Close()

	err = ks.Option(keystone.OPT_SYNTAX, keystone.OPT_SYNTAX_INTEL)
	if err != nil {
		panic(err)
	}

	insn, _, ok := ks.Assemble(assembly, 0)
	if !ok {
		panic("failed to assemble")
	}

	// choose some initial values for rax and rdx
	// to get a feel for what the algorithm does
	regPairs := []struct {
		rax, rdx uint64
	}{
		{0x0, 0x1},
		{0x1, 0x1},
		{0xf, 0xe},
		{0xe, 0xf},
		{0xffffffff, 0x00},
	}

	// try each pair of initial register values with the emulator
	for i, regPair := range regPairs {
		mu, _ := uc.NewUnicorn(uc.ARCH_X86, uc.MODE_64)
		mu.MemMap(0x1000, 0x1000)
		mu.MemWrite(0x1000, insn)

		mu.RegWrite(uc.X86_REG_RAX, regPair.rax)
		mu.RegWrite(uc.X86_REG_RDX, regPair.rdx)

		if err := mu.Start(0x1000, 0x1000+uint64(len(insn))); err != nil {
			panic(err)
		}

		endRax, _ := mu.RegRead(uc.X86_REG_RAX)
		fmt.Printf("%d (rax: %016x rdx %016x) ->  rax: %016x\n", i, regPair.rax, regPair.rdx, endRax)
	}
}
```



Here's the output (just as above):
```
0 (rax: 0000000000000000 rdx 0000000000000001) ->  rax: 0000000000000000
1 (rax: 0000000000000001 rdx 0000000000000001) ->  rax: 0000000000000001
2 (rax: 000000000000000f rdx 000000000000000e) ->  rax: 000000000000000e
3 (rax: 000000000000000e rdx 000000000000000f) ->  rax: 000000000000000e
4 (rax: 00000000ffffffff rdx 0000000000000000) ->  rax: 0000000000000000
```

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

```lisp
(set-logic QF_BV)

; Convention here is add a label to the end of the register
; to mark a step in the program for which the value applies.
; e.g.
; rdx0 is the first value of rdx, rdx1 is the value at the
; next step of the program, rdxN is the value at the Nth
; step.
;
; Essentially, we're turning an assembly program into SSA form
(declare-fun rdx0 () ( _ BitVec 64))
(declare-fun rdx1 () ( _ BitVec 64))

(declare-fun rcx0 () ( _ BitVec 64))
(declare-fun rcx1 () ( _ BitVec 64))
(declare-fun rcx2 () ( _ BitVec 64))

(declare-fun rax0 () ( _ BitVec 64))
(declare-fun rax3 () ( _ BitVec 64))

; The carry flag
(declare-fun cf () (_ Bool))

; sub sets the carry flag if unsigned subtraction will result in
; an overflow i.e. rax0 > rdx0
(assert ( = cf (bvult rdx0 rax0)))
(assert ( = rdx1 (bvsub rdx0 rax0)))

; sbb is 'subtract with carry'. If the carry flag is set, we take
; away one from the result of sub
(assert ( = rcx1
(ite (= cf true)
 (bvsub (bvsub rcx0 rcx0) #x0000000000000001)
 (bvsub rcx0 rcx0))
))

; and rxc1,rdx1
(assert ( = rcx2 (bvand rcx1 rdx1)))
; add rax0,rcv2
(assert ( = rax3 (bvadd rax0 rcx2)))

; assert conditions which should fail if this is indeed a minimum
; function. In the case below, we assert that rax0 <= rdx0 and
; rax3 != rax0 (which we suspect is false) and so if we get 'unsat'
; as the result, then we've proved rax0 <= rdx0 => rax3 == rax0

; note the use of push here. Allows us to reuse the work above by
; popping the asserts below from the stack
(push 1)
    (assert ( or (bvult rax0 rdx0) (not (distinct rax0 rdx0))))
        (check-sat)
        (push 1)
            (assert ( distinct rax3 rax0))
            (check-sat)
(pop 2)

; We show here that rax0 > rdx0 (along with our previous assertions) can
; can be satisfied. We then show that adding rax3 != rdx0 leads to
; unsat, which means that rax3 == rdx0
(assert (bvugt rax0 rdx0))
(check-sat)
(assert ( distinct rax3 rdx0))

(check-sat)
```


Running with z3 gives

```
sat
unsat
sat
unsat
```


# Why?

This is surely overkill for such a small problem? Yes. However, as we move to more complex binary executables (e.g. malware)
the ideas touched on here come into their own.









