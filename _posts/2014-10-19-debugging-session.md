---
layout: single
title: "Smalltalk/X and STX:LIBJAVA Debugging Session"
description: ""
category:
tags: []
date: 2014-10-19
---

[STX:LIBJAVA](https://swing.fit.cvut.cz/projects/stx-libjava/wiki) is
gradualy becoming more and more stable, but there are still few bugs,
which can cause the virtual machine (VM, runtime) to crash.

Some of the crashes are caused by the actual STX:LIBJAVA code. Some are
caused by bugs in the VM, since STX:LIBJAVA stresses the runtime in
a different way than the standard Smalltalk code. And also, some bugs
can be brought in by used libraries and compilers. In past, STX:LIBJAVA
has revealed bugs in the VM that were older than 10 years. And one
particularly peculiar bug was found in [gcc](https://gcc.gnu.org/). Ask
[Jan Vraný](https://swing.fit.cvut.cz/vranyj1) how much fun he had for
those 2 sleepless weeks he's spent debugging.

Maybe some of you would be interested in how we (well, Jan. I, for most of
the time, am involved with introducing the bug) debug these crashes.
We will show you how to find the cause and interpret the problem. In
this article you'll be honored to watch Jan Vraný debugging :)

## Actual Session

Imagine we were playing with STX:LIBJAVA and the VM crashed. This is what we
might see in the terminal:

```sh
Program received signal SIGSEGV, Segmentation fault.
0x5107b104 in ?? ()
```

Because this is not the first time it crashed, we made sure to start it in the
gdb:

```sh
$> projects/smalltalk/smalltalk --gdb
(gdb) run -I --quick
```

Now we can examine the state of the system at the moment of the crash.
Let's look around to see what's the state of the program:

```asm
  (gdb) info registers
  eax            0x5107b104       1359458564
  ecx            0xf1600b83       -245363837
  edx            0xf3a23210       -207474160
  ebx            0xf3a1fff4       -207486988
  esp            0xf068d50c       0xf068d50c
  ebp            0x3b     0x3b
  esi            0x3e     62
  edi            0xf38836e0       -209176864
  eip            0x5107b104       0x5107b104
  eflags         0x210206 [ PF IF RF ID ]
  cs             0x23     35
  ss             0x2b     43
  ds             0x2b     43
  es             0x2b     43
```

Ok so this, at least for me, is not showing anything suspecting.  We
will look at the code are we currently executing:

```asm
(gdb) disas $pc - 10, $pc + 10
Dump of assembler code from 0x5107b0fa to 0x5107b10e:
   0x5107b0fa:  add    %al,(%eax)
   0x5107b0fc:  add    %al,(%eax)
   0x5107b0fe:  add    %al,(%eax)
   0x5107b100:  add    %al,(%eax)
   0x5107b102:  add    %al,(%eax)
=> 0x5107b104:  adc    $0xc7,%al
   0x5107b106:  pop    %es
   0x5107b107:  push   %ecx
   0x5107b108:  mov    %al,(%eax)
   0x5107b10a:  add    %al,(%eax)
   0x5107b10c:  add    %eax,(%eax)
End of assembler dump.
```

This was more helpful.  We see, that we must have jumped to the data
part of the memory, because code like that is too funny to be generated
by any reasonable compiler.  We can verify that by checking memory
layout:

```asm
(gdb) call dumpSpaceInfo()
MEM:  new: f1600000 .. f16efd34 .. f1800000 surv: f1200000 .. f1400000
MEM:  old: 50000000 .. 546007b8 .. 5986a000 cold: 0 .. 0
MEM:  rem: f07a0000 .. f07a04c8 .. f07a8000
MEM: nrem: f0700000 .. f0700000 .. f0708000
MEM: lrem: f2a08000 .. f2a083cc .. f2a0c000
```

So we see we jumped into the oldspace. Let's check stack pointer:

```asm
(gdb) p $sp
$23 = (void *) 0xf068d50c
```
Good, stack pointer is ok, it points somewhere into stack segment of
Smalltalk/X VM. Usually, anything that start with `0xfxxx` is stack or
newspace. Stack usually takes `0xf0xx`, and `0xf100` and above is the
newspace.  This is of course just a guess, you are never 100% sure when
dealing with complex C programs. Now we can look at what stack pointer
points to.

```asm
(gdb) p /x *((int*)$sp)
$26 = 0xf388373e
```

If we could guess, the instruction before the current one was `call`.
This instruction, on x86 architecture, stores current program counter to
`++sp` and jumps. Therefore, stack pointer gives us address from where
we jumped. Let's examine the code so we can be a little bit more sure it
was `call` instruction:

```asm
less  /proc/$(pgrep stx)/maps
  ...
  f375e000-f3761000 r-xp 00000000 fd:02 11014091 /lib32/libdl-2.13.so
  f3761000-f3762000 r--p 00002000 fd:02 11014091 /lib32/libdl-2.13.so
  f3762000-f3763000 rw-p 00003000 fd:02 11014091 /lib32/libdl-2.13.so
  f3763000-f378b000 r-xp 00000000 fd:02 11014092 /lib32/libm-2.13.so
  f378b000-f378c000 r--p 00028000 fd:02 11014092 /lib32/libm-2.13.so
  f378c000-f378d000 rw-p 00029000 fd:02 11014092 /lib32/libm-2.13.so
  f378d000-f37b3000 rw-p 00000000 00:00 0
  f37b3000-f39e3000 r-xp 00000000 fd:03 5643709  .../stx/librun/librun.so
  f39e3000-f3a20000 r--p 0022f000 fd:03 5643709  .../stx/librun/librun.so
  ...
```

So the address from where we jumped is somewhere in the text segment of
`librun.so`. `librun` is one of the libraries of Smalltalk/X.

```asm
(gdb) info symbol 0xf388373e
__jInterpret + 28274 in section .text of .../smalltalk/librun.so
```

Good, now which line is it:

```asm
(gdb) info line *0xf388373e
Line 2940 of "jinterpret.c" starts at address 0xf388370f <__jInterpret+28227>
 2937│   switch (nA) {
 2938│     case 0:
 2939│     SET_LINENO_FOR_SEND(dummy0, bp-3-bp0);
 2940├>    rslt = (*code)(rec, sel,
 2941│                __JavaMethodInstPtr(resolvedMethod)->jm_javaClass,
 2942│                &dummy0);
 2943│     break;
```

And here we go. This is the actual code:

```c
OBJFUNC code = __ExecutableCodeInstPtr(resolvedMethod)->ex_code;

/* INVSTATIC or INVNONVIRT */

if (code) {
switch (nA) {
  case 0:
  SET_LINENO_FOR_SEND(dummy0, bp-3-bp0);
  rslt = (*code)(rec, sel,
       __JavaMethodInstPtr(resolvedMethod)->jm_javaClass,
       &dummy0);
break;
```

Somehow, code contains normal data instead of being a code pointer for
resolvedMethod - which is normal Java method object used internally in
the VM. Code pointer should point at compiled machine code (compiled by
Just-in-Time compiler). It surely cannot point at some data.

The rest of the debugging session is not that interesting. We had to
find who set the incorrect value there. Usually where many bugs go hide
is a C code sequence of variable assignment, message send, and variable
access. Garbage collector can be executed on message sends, and we use
Baker algorithm, which moves objects within heap. When you assign an object
into a variable, and GC moves the object, you will get some garbage in
there instead. The most fun bugs show themselves after many GC cycles,
and in a completely different place in the execution, when you cannot
possibly know where the bug was introduced.

For this the VM contains `PROTECT` and `UNPROTECT` macros, which will
make sure your variable is updated when GC runs. You just have to
remember to use them. And I always forget.

## Conclusion

This was quite meaty post. What you should take from it is that you can
get a ton of information out of crashed C program, if you know where to
look. So next time the STX:LIBJAVA, or Ruby or whichever virtual machine
you use crashes, try to investigate a bit. You will learn something, for
sure!
