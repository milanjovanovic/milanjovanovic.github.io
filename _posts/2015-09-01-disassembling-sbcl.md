---
layout: post
title: Disassembling SBCL
---

If you want to play with dynamic programming language and see how all fancy stuff are implemented at the assembly level then [SBCL](http://www.sbcl.org) (Common Lisp implementation) is right tool for that task.
It's Common Lisp implementation, so it has REPL and [disassemble](http://clhs.lisp.se/Body/f_disass.htm) function.
SBCL especially because you have all the sources at your hand and beside standard [disassemble](http://clhs.lisp.se/Body/f_disass.htm) function you have many non-standard ones (like <em>sb-disassem:disassemble-memory</em>).
SBCL disassemble prints Intel assembly syntax (on X86).

Let's do some simple analyzing of disassemble. 
We will define two simple functions and see how is runtime resolving function calls.

{% highlight text %}
CL-USER> (defun foo () 'test)
FOO
CL-USER> (defun bar () (foo))
BAR
CL-USER> (disassemble 'foo)
; disassembly for FOO
; Size: 25 bytes. Origin: #x1008221B44
; 44:       498B4C2460       MOV RCX, [R12+96]                ; thread.binding-stack-pointer
                                                              ; no-arg-parsing entry point
; 49:       48894DF8         MOV [RBP-8], RCX
; 4D:       488B159CFFFFFF   MOV RDX, [RIP-100]               ; 'TEST
; 54:       488BE5           MOV RSP, RBP
; 57:       F8               CLC
; 58:       5D               POP RBP
; 59:       C3               RET
; 5A:       0F0B10           BREAK 16                         ; Invalid argument count trap
NIL
CL-USER> (disassemble 'bar)
; disassembly for BAR
; Size: 27 bytes. Origin: #x1008221A24
; 24:       498B4C2460       MOV RCX, [R12+96]                ; thread.binding-stack-pointer
                                                              ; no-arg-parsing entry point
; 29:       48894DF8         MOV [RBP-8], RCX
; 2D:       488B059CFFFFFF   MOV RAX, [RIP-100]               ; #<FDEFINITION for FOO>
; 34:       31C9             XOR ECX, ECX
; 36:       FF7508           PUSH QWORD PTR [RBP+8]
; 39:       FF6009           JMP QWORD PTR [RAX+9]
; 3C:       0F0B10           BREAK 16                         ; Invalid argument count trap
NIL
CL-USER>
{% endhighlight %}

Lets analyse how is BAR calling function FOO.
Instructions important for calling function FOO are:

{% highlight text %}
; 2D:       488B059CFFFFFF   MOV RAX, [RIP-100]               ; #<FDEFINITION for FOO>
; 39:       FF6009           JMP QWORD PTR [RAX+9]
{% endhighlight %}
(JMP is used here because call to FOO is at tail position and compiler optimize it).

How this instruction sequence works ?

When generating function code compiler generates header table just before function code so it has fixed offset from RIP but relative to address that SBCL gets from OS. This is something like GOT (Global Offset table) in Position-Independent-Code in shared libraries but here it's local to function.
That's why <code>MOV RAX,[RIP-100]</code>.

We need another instruction to resolve our function address because now in <code>RAX</code> we have address of cell that holds FOO address not a actual FOO.
So we need another instruction to resolve it - <code>JMP QWORD PTR [RAX+9]</code>.<br>
With this design we can redefine function at REPL and runtime will only need to update address in cell and not in all callee headers.
Of course, when GC runs it needs to update references in function header table.

Lets trace all of this and we should get FOO address <code>#x1008221B44</code>.
We'll use sb-disassem:disassemble-memory for this task.
 
Since RIP is always holding address of next instruction we know that when <code>MOV RAX, [RIP-100]</code> is executed RIP will be <code>#x1008221A34</code>.

{% highlight text %}
CL-USER> (sb-sys:without-gcing (sb-disassem:disassemble-memory (- #x1008221A34 100) 8))
; Size: 8 bytes. Origin: #x10082219D0
; 0:       8F               BYTE #X8F
; 1:       5A               POP RDX
; 2:       1008             ADC [RAX], CL
; 4:       1000             ADC [RAX], AL
; 6:       0000             ADD [RAX], AL
NIL
CL-USER>
{% endhighlight %}

X86(-64) is little endian architecture so address is <code>#x1008105A8F</code>.
Now lets get address on <code>[RAX+9]</code>, our function FOO should be there.

{% highlight text %}
CL-USER> (sb-sys:without-gcing (sb-disassem:disassemble-memory (+ #x1008105A8F 9) 8))
; Size: 8 bytes. Origin: #x1008105A98
; 98:       381B             CMP [RBX], BL
; 9A:       2208             AND CL, [RAX]
; 9C:       1000             ADC [RAX], AL
; 9E:       0000             ADD [RAX], AL
NIL
CL-USER>
{% endhighlight %}

Again, our address is <code>#x1008221B38</code> and our disassemble of function FOO
is showing <code>#x1008221B44</code> as start address.

{% highlight text %}
CL-USER> (- #x1008221B44 #x1008221B38)
12
CL-USER>
{% endhighlight %}

Difference is only 12 bytes, let's see what's there.

{% highlight text %}
CL-USER> (sb-sys:without-gcing (sb-disassem:disassemble-memory #x1008221B38 12))
; Size: 12 bytes. Origin: #x1008221B38
; 38:       8F4508           POP QWORD PTR [RBP+8]
; 3B:       4885C9           TEST RCX, RCX
; 3E:       751A             JNE #x1008221B5A
; 40:       488D65F8         LEA RSP, [RBP-8]
NIL
CL-USER>
{% endhighlight %}

So, this missing code is just setting frame right, testing that we got
right number of arguments (RCX holds number of arguments obviously),
none in our case (TEST, RCX, RCX and JNE ...).
For some reason SBCL disassemble doesn't show this code though.

It's really cool and fun that you can do this kind of stuff directly from
REPL without need of endless recompiling and loading. 

P.S. thanks @stassats for explaining me sbcl design regarding function calls 
