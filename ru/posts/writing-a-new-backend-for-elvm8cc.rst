.. title: Writing a new backend for ELVM/8cc
.. slug: writing-a-new-backend-for-elvm8cc
.. date: 2026-01-31 22:58:35 UTC+01:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text

Every person will, at some point of their lives, decide
to design an ISA, emulate it, and eventually write a
toolchain for building software for it. This is one of
those immutable laws of the universe, kind of like
programmer thermodynamics. Surprisingly, the hardest
part of this journey thus far has, for me, been making
the emulator, not the surrounding pomp. Turns out it's
surprisingly easy to port a C compiler to your device,
assuming you're willing to put up with awful code.

===============
Lay of the Land
===============

I won't bore the audience with an elabourate description
of the chip. There are eight general-purpose registers,
a shadow registry for quick context switching in kernel,
a hardware stack pointer, and a 24-bit address space. There
is an MMU which pages out the memory space into 12-bit
pages and tracks read-write permissions for each. None
of that is especially important.

I decided I wanted to port a C compiler to target my
chip so I could feel the transient emotional satisfaction
of actually *compiling* a Thing of code for my chip.
It's a form of validation: look at it! There it goes,
there it works!

But porting a C compiler, or writing a new backend for
one is, to put it lightly, a bit of a bother. Some
projects are said to be easy to port, needing only 
a *month or two of focused work* to emit working code.

I'll show you how to do it in a day.

============
Our compiler
============

There are many C compilers out there, certainly dozens
and probably reaching into the hundreds. Unlike writing
a C++ compiler, which is a sisyphean task that even
giants like Microsoft end up having trouble with, you
can just sit down and crank out a working (albeit not
good) C compiler and be the same age you were when you
started it.

Most of them are not meant to be retargetable and so
will produce code for only one platform, or will have
a very small set of alternatives that are difficult
to extend. The one C compiler I've found isn't that
hard to work with is the excellent 8cc/ELVM_ pair. The
``8cc`` compiler serves as the frontend and
accepts normal-looking C code (including, surprisingly, 
itself), and emits ELIR assembly, while the ``elc``
backend consumes that and emits various other languages.

.. _ELVM: https://github.com/shinh/elvm/tree/master

The code quality is atrocious. The ELIR architecture has
four general-purpose registers and two special ones,
only addition and subtraction for number manipulation
(so not even bitwise operations), a few conditionals, and
char input and output. The compiler is laughably stupid,
producing code that would make ``gcc -O0`` feel performant.
That's fine, because we're looking for something that
is easy to get working.

In the ``/target`` folder, you should create a file that
will contain your entire backend. I went with ``enn.c``,
as the target is the enncpu. At the top you'll want
to include the following:

.. code-block:: c
	
	...
	#include <ir/ir.h>
	#include <target/util.h>
	...

These contain some of the basic definitions and things
the compiler internals use.

The main datatypes you'll be dealing with are:

.. code-block:: c

	typedef enum {
	  A, B, C, D, BP, SP
	} Reg;

	typedef enum {
	  REG, IMM
	} ValueType;

	typedef enum {
	  OP_UNSET = -2, OP_ERR = -1,
	  MOV = 0, ADD, SUB, LOAD, STORE, PUTC, GETC, EXIT,
	  JEQ = 8, JNE, JLT, JGT, JLE, JGE, JMP,
	  // Optional operations follow.
	  EQ = 16, NE, LT, GT, LE, GE, DUMP,
	  LAST_OP
	} Op;

	typedef struct {
	  ValueType type;
	  union {
	    Reg reg;
	    int imm;
	    void* tmp;
	  };
	} Value;

	typedef struct Inst_ {
	  Op op;
	  Value dst;
	  Value src;
	  Value jmp;
	  int pc;
	  int lineno;
	  char* magic_comment;
	  struct Inst_* next;
	} Inst;

	typedef struct Data_ {
	  int v;
	  struct Data_* next;
	} Data;

	typedef struct {
	  Inst* text;
	  Data* data;
	} Module;

The backend driver calls your target through one
function, which should return ``void`` and take 
one pointer of type ``Module*``. I named mine
``void target_enn(Module* module)``. The single 
argument is what the driver passes to your target
and it contains the entirety of the information extracted
from the code. This target is going to be
pasted verbatim into the backend driver code and
compiled into it, so it has to include a trailing
newline (to avoid messing up includes that come
after it; yes, I got burnt on this myself).

The ELIR virtual machine is a Harvard-architecture
thing, and instruction and data codepaths do not mix.
Instructions *are not addressable*, and all addressing
is for the data. The compiler works with 24-bit words,
and data is word-addressed. The compiler makes some
convenient but wild assumptions (``sizeof(char) == sizeof(int)``),
and this makes it both a bit easier and a bit more complicated
to keep track of things.

The dataword stream and the instruction stream are
both in the form of a linked list. The module provided
by the caller contains pointers to the first members
of both of those.

Instructions are the more fun part. Since they aren't
actually addressable, the ELVM uses a genuinely
horrendous scheme to keep track of where jumps go.
Every piece of code under a label has the same ``pc``
address (so a sequence of a hundred instructions will
all have ``pc == 34`` if they are under the 34th
label in the instruction stream!), and jumps go to the
first instruction in that stream. It is best to tackle
addressing at the end.

Before you implement the instructions, read up on
their behaviour_ and whatnot; it's a very short read,
and answers basically everything you need to know.

.. _behaviour: https://github.com/shinh/elvm/blob/master/ELVM.md

Emitting an instruction is fairly easy. The ``Inst``
struct (see above for its layout) has everything
you need to build an instruction out of it. The way I
handled it was:

.. code-block:: c

	static void enn_inst(Inst* inst)
	{
		switch(inst->op)
		{
			case MOV:
				if (inst->src.type == REG) {
					emit_enn_mov_reg(inst->dst.reg, inst->src.reg);
				} else {
					emit_enn_mov_imm(inst->dst.reg, inst->src.imm);
				}
				break;
			case ADD:
				if (inst->src.type == REG) {
					emit_enn_add_reg(inst->dst.reg, inst->src.reg);
				} else {
					emit_enn_add_imm(inst->dst.reg, inst->src.imm);
				}
				break;
		// ...

	static void emit_enn_add_reg(Reg dst, Reg src)
	{ printf("\tADD   %c, %c\n", dst + 'A', src + 'A'); }

		// ...

Really, that's about all there is to it to emitting
instructions. Keep in mind that immediates are ∈ [0, 16777215],
and adjust accordingly.

Since labels/pc breaks don't actually *appear* in the stream 
but are signified by a change in the ``inst->pc`` value,
you'll have to do a walk through the linked list to find out
where these breaks lie. To avoid calculating the byte
addresses of where those jumps go (as was done e.g. in the
Armv7 and x86 backends), I exploit the fact that I'm generating
assembly and offload the bookkeeping to the assembler. I do
one walk through the instruction stream and just emit a string
label whenever a break is encountered. Here is the whole
of my target callback code:

.. code-block:: c

	void target_enn(Module* module)
	{
		emit_enn_data_head();
		
		for(Data* data = module->data; data; data = data->next) {
			printf(".INT24 %d\n", data->v);
		}
		emit_enn_data_tail();

	  	emit_enn_head();

	  	int prev_pc = -1;

	  	for (Inst *inst = module->text; inst; inst = inst->next) {
			if (prev_pc != inst->pc) {
				printf("@L%d\n", inst->pc);
	    		}
	    		enn_inst(inst);
	    		prev_pc = inst->pc;
	  	}

		emit_enn_tail();

		return;
	} // must have tailing newline

Accordingly, when a jump is encountered, I load the address
of the label and jump to it. Here is that bit:

.. code-block:: c

	static void emit_enn_jump_reg(Reg jmp)
	{ printf("\tJMR   %c\n", jmp + 'A'); }
	
	static void emit_enn_adrl_lbl(int jmp)
	{
		printf("\tADRL  G, @L%d\n", jmp);
		printf("\tADRM  G, @L%d\n", jmp);
		printf("\tADRH  G, @L%d\n", jmp);
	}

	static void emit_enn_jump_imm(int jmp)
	{
		emit_enn_adrl_lbl(jmp);
		emit_enn_jump_reg(6);
	}


Data loads and stores are a bit more complicated. The ELIR
abstraction starts placing the data at address 0, growing
upwards, and each word is 3 bytes wide but word-addressed.
This means that you need to start placing your data and memory at
a known base address and calculate offsets off that. I did
it brutishly: data is placed starting at 0x10, after the
header, and then all addresses are adjusted based off that.
Here is an example:

.. code-block:: c

	static void emit_enn_load_reg(Reg dst, Reg src)
	{ 
		printf("\tMOV   G, #3\n");
		printf("\tMULA  %c, G\n", src + 'A');
		printf("\tADD   %c, #16\n", src + 'A');
		printf("\tLDRS  %c, %c\n", dst + 'A', src + 'A');
		printf("\tSUB   %c, #16\n", src + 'A');
		printf("\tDIV   %c, G\n", src + 'A'); 
	}

	static void emit_enn_load_imm(Reg dst, int src)
	{
		src += 16;
		src *= 3;
		emit_enn_mov_imm(6, src);
		emit_enn_load_reg(dst, 6);
		emit_enn_sub_reg(6, 6);
	}

In my case, I know that my base is at #16, and since
I set up all the data as ``.INT24``, everything
is offset in multiples of three. While I considered
padding things out between those to allow for
``LSHL G, #2`` instead of the more convoluted 
multiplication, it would've eaten up somewhat more space
in the data segment, so I decided against it (but will
try it like that some time in the future).

===================
Putting it together
===================

To actually have your thing be usable, you have to do
a small amount of further bookkeeping.

Your first stop will be the ``target/elc.c`` file, where
you'll add your target callback. You really can't
miss the huge block of function pointer declarations
in there, trust me.

The second stop will be going into the Makefile and
rummaging around. The ELVM suite is annoying to build
because it runs a bunch of testsand samples on every
invocation of ``make``. You can change that, and I did,
but I won't waste time here on the how since it's immaterial. In the Makefile you'll want to find
the ``ELC_SRCS`` variable and append your backend
target source file to it. After that, you'll want to go
to the lists of targets under the comment line ``# Targets``,
and add yours. Here's mine:

.. code-block:: make

	TARGET := enn
	RUNNER := tools/runenn.sh
	include target.mk

The ``tools/runenn.sh`` file is just an empty bash
file because I don't want it running any of those tests.
You should then of course hit ``make -j 8`` (or more; and
there are a lot of noisy files in there so the console
will start sparking and sizzling), and inevitably
hit an error because the compilation is passed the 
unfortunate flags ``-W -Wall -Wextra -Werror`` so even
an unused variable will break the build and you'll have to
do it all over again and so on and so forth.

After this whole song and dance is finished, you'll be left
with ``out/8cc`` and ``out/elc``. The first is the C compiler
frontend which takes one or more C files (there's a linker option
that I didn't explore; so one in my case) and produces
assembly out of that. You'll invoke it with ``8cc {infile}.c -S``
and it will produce the corresponding ``{infile}.s`` (same name)
in the same directory (I'm not sure yet how to get it to go
*anywhere else*, but that's the least important thing here).
After that, you will invoke the backend driver with
``elc {infile}.s -enn > outfile.txt`` (or wherever else you
do output; I did mine to stdout). This should, in theory,
leave you with a fully working Thing of assembly that
targets your custom CPU (in practice: I had to fiddle with
it a bit to get it going, and this revealed some bugs
in the assembler I didn't know were there). Isn't that neat!


======
Outro
======

Realistically, it should take a decent C programmer about 
a day to actually write a working backend for 8cc using
ELVM. The code is absolute gore, and there is so much room
for optimisation (such as eliminating dead assignments),
but this simplicity and lack of optimisation (though sorely
needed) is probably a direct factor in how portable this thing
really is. It took me about six hours and the end result
was around 400 lines of surprisingly stupid C; some of those
hours were wasted on figuring out some un(der)documented things
using the x86 and Armv7 backends as reference.

Go do it yourself! Have a go at it, have some fun. I really
doubt there's any other C compiler that's this trivial to
port *and* can compile itself (seriously, that's one of
the test cases and why it takes so goddamn long to make
them).

------------------

Under this line you'll find the actual full source of 
my backend. It's dirty, and I slapped it together in
one day. The version here isn't 100% identical to my
own code as I shuffled around some newlines and
whitespace to make it take up just slightly less
space, but the contents are unchanged. Enjoy.

.. code-block:: c

	static void emit_enn_add_imm(Reg dst, int src);
	static void emit_enn_sub_imm(Reg dst, int src);

	static void emit_enn_head()
	{
		printf(".SEC %%main\n");
		printf("\tADRL H, @_sec_main\n\tADRM H, @_sec_main\n\tADRH H, @_sec_main\n");
		printf("\tMOVL G, #0x80\n\tLSHL G, #2\n");
		printf("\tADD  H, G\n\tWSP  H\n\tSUB  G, G\n\n");
	}
	static void emit_enn_tail()
	{ printf("%%main\n"); }

	static void emit_enn_mov_reg(Reg dst, Reg src)
	{ printf("\tMOV   %c, %c\n", dst + 'A', src + 'A'); }

	static void emit_enn_mov_imm(Reg dst, int src)
	{
		bool neg = false;
		if(src < 0) { src = -src; neg = true; }
		printf("\tMOVL  %c, #0x%02x\n",     dst + 'A', (byte)(src >>  0));
		if(src > 0xff)
			printf("\tMOVM  %c, #0x%02x\n", dst + 'A', (byte)(src >>  8));
		if(src > 0xffff)
			printf("\tMOVH  %c, #0x%02x\n", dst + 'A', (byte)(src >> 16));
		if(neg)
			printf("\tSUB   G, G\n\tSUB   G, %c\n\tMOV   %c, G\n", dst + 'A', dst + 'A');
	}

	static void emit_enn_add_reg(Reg dst, Reg src)
	{ printf("\tADD   %c, %c\n", dst + 'A', src + 'A'); }

	static void emit_enn_add_imm(Reg dst, int src)
	{
		if(src < 0) { emit_enn_sub_imm(dst, -1 * src); return; }
		else if (src < 64)
			printf("\tADD   %c, #%d\n", dst + 'A', src);
		else
		{
			emit_enn_mov_imm(6, src);
			printf("\tADD   %c, G\n", dst + 'A');
		}
	}

	static void emit_enn_sub_reg(Reg dst, Reg src)
	{ printf("\tSUB   %c, %c\n", dst + 'A', src + 'A'); }

	static void emit_enn_sub_imm(Reg dst, int src)
	{
		if(src < 0) { emit_enn_add_imm(dst, -1 * src); return; }
		else if (src < 64)
			printf("\tSUB   %c, #%d\n", dst + 'A', src);
		else
		{
			emit_enn_mov_imm(6, src);
			printf("\tSUB   %c, G\n", dst + 'A');
		}
	}

	static void emit_enn_load_reg(Reg dst, Reg src)
	{ 
		printf("\tMOV   G, #3\n");
		printf("\tMULA  %c, G\n", src + 'A');
		printf("\tADD   %c, #16\n", src + 'A');
		printf("\tLDRS  %c, %c\n", dst + 'A', src + 'A');
		printf("\tSUB   %c, #16\n", src + 'A');
		printf("\tDIV   %c, G\n", src + 'A'); 
	}

	static void emit_enn_load_imm(Reg dst, int src)
	{
		src += 16;
		src *= 3;
		emit_enn_mov_imm(6, src);
		emit_enn_load_reg(dst, 6);
		emit_enn_sub_reg(6, 6);
	}

	static void emit_enn_store_reg(Reg dst, Reg src)
	{ 
		printf("\tMOV   G, #3\n");
		printf("\tMULA  %c, G\n", src + 'A');
		printf("\tADD   %c, #16\n", src + 'A');
		printf("\tSTRS  %c, %c\n", dst + 'A', src + 'A');
		printf("\tSUB   %c, #16\n", src + 'A');
		printf("\tDIV   %c, G\n", src + 'A'); 
	}

	static void emit_enn_store_imm(Reg dst, int src)
	{
		src += 16;
		src *= 3;
		emit_enn_mov_imm(6, src);
		emit_enn_store_reg(dst, 6);
		emit_enn_sub_reg(6, 6);
	}

	static void emit_enn_exit()
	{ printf("\tERR ; \n"); }

	typedef enum {
		CEQ, CNE,
		CLT, CGT,
		CLE, CGE,
		NONE
	} COMPARISON;

	static void emit_enn_cmp_reg(COMPARISON type, Reg dst, Reg src)
	{
		switch(type)
		{
			case CEQ: printf("\tCEQ   %c, %c\n", dst + 'A', src + 'A'); break;
			case CNE: printf("\tCNE   %c, %c\n", dst + 'A', src + 'A'); break;
			case CLT: printf("\tCLT   %c, %c\n", dst + 'A', src + 'A'); break;
			case CGT: printf("\tCGT   %c, %c\n", dst + 'A', src + 'A'); break;
			case CLE: printf("\tCGT   %c, %c\n\tCINV\n", dst + 'A', src + 'A'); break;
			case CGE: printf("\tCLT   %c, %c\n\tCINV\n", dst + 'A', src + 'A'); break;
			case NONE: return;
		}
	}
	static void emit_enn_cmp_imm(COMPARISON type, Reg dst, int src)
	{
		if(type == NONE) { return; }
		if (src < 64)
			printf("\tMOVL  G, #%d\n", src);
		else
		{
			emit_enn_mov_imm(6, src);
			printf("\tADD   %c, G\n", dst + 'A');
		}
		switch(type)
		{
			case CEQ: printf("\tCEQ   %c, G\n", dst + 'A'); break;
			case CNE: printf("\tCNE   %c, G\n", dst + 'A'); break;
			case CLT: printf("\tCLT   %c, G\n", dst + 'A'); break;
			case CGT: printf("\tCGT   %c, G\n", dst + 'A'); break;
			case CLE: printf("\tCGT   %c, G\n\tCINV\n", dst + 'A'); break;
			case CGE: printf("\tCLT   %c, G\n\tCINV\n", dst + 'A'); break;
			case NONE: return;
		}
	}

	static void emit_enn_jmcc_reg(Reg jmp)
	{ printf("\tJMR.P %c\n", jmp + 'A'); }
	static void emit_enn_jump_reg(Reg jmp)
	{ printf("\tJMR   %c\n", jmp + 'A'); }
	static void emit_enn_adrl_lbl(int jmp)
	{
		printf("\tADRL  G, @L%d\n", jmp);
		printf("\tADRM  G, @L%d\n", jmp);
		printf("\tADRH  G, @L%d\n", jmp);
	}

	static void emit_enn_jmcc_imm(int jmp)
	{ emit_enn_adrl_lbl(jmp); emit_enn_jmcc_reg(6); }
	static void emit_enn_jump_imm(int jmp)
	{ emit_enn_adrl_lbl(jmp); emit_enn_jump_reg(6); }

	static void emit_enn_putc(Reg reg)
	{ printf("\tDBGC  %c\n", reg + 'A'); }

	static void enn_inst(Inst* inst)
	{
		switch(inst->op)
		{
			case MOV:
				if (inst->src.type == REG) {
					emit_enn_mov_reg(inst->dst.reg, inst->src.reg);
				} else {
					emit_enn_mov_imm(inst->dst.reg, inst->src.imm);
				}
				break;
			case ADD:
				if (inst->src.type == REG) {
					emit_enn_add_reg(inst->dst.reg, inst->src.reg);
				} else {
					emit_enn_add_imm(inst->dst.reg, inst->src.imm);
				}
				break;
			case SUB:
				if (inst->src.type == REG) {
					emit_enn_sub_reg(inst->dst.reg, inst->src.reg);
				} else {
					emit_enn_sub_imm(inst->dst.reg, inst->src.imm);
				}
				break;
			case EXIT:
				emit_enn_exit();
				break;
			case LOAD:
				if (inst->src.type == REG) {
					emit_enn_load_reg(inst->dst.reg, inst->src.reg);
				} else {
					emit_enn_load_imm(inst->dst.reg, inst->src.imm);
				}
				break;
			case STORE:
				if (inst->src.type == REG) {
					emit_enn_store_reg(inst->dst.reg, inst->src.reg);
				} else {
					emit_enn_store_imm(inst->dst.reg, inst->src.imm);
				}
				break;
			case JMP:
				if (inst->jmp.type == REG) {
					emit_enn_jump_reg(inst->jmp.reg);
				} else {
					emit_enn_jump_imm(inst->jmp.imm);
				}
				break;
			case PUTC:
				emit_enn_putc(inst->src.reg);
				break;
			case JEQ:
				if (inst->jmp.type == REG) {
					emit_enn_cmp_reg(CEQ, inst->dst.reg, inst->src.reg);
					emit_enn_jmcc_reg(inst->jmp.reg);
				} else {
					emit_enn_cmp_imm(CEQ, inst->dst.reg, inst->src.imm);
					emit_enn_jmcc_imm(inst->jmp.imm);
				}
				break;
			case EQ:
				if(inst->src.type == REG)
					emit_enn_cmp_reg(CEQ, inst->dst.reg, inst->src.reg);
				else
					emit_enn_cmp_imm(CEQ, inst->dst.reg, inst->src.imm);
				break;
			case JNE:
				if (inst->jmp.type == REG) {
					emit_enn_cmp_reg(CNE, inst->dst.reg, inst->src.reg);
					emit_enn_jmcc_reg(inst->jmp.reg);
				} else {
					emit_enn_cmp_imm(CNE, inst->dst.reg, inst->src.imm);
					emit_enn_jmcc_imm(inst->jmp.imm);
				}
				break;
			case NE:
				if(inst->src.type == REG)
					emit_enn_cmp_reg(CNE, inst->dst.reg, inst->src.reg);
				else
					emit_enn_cmp_imm(CNE, inst->dst.reg, inst->src.imm);
				break;
			case JLT:
				if (inst->jmp.type == REG) {
					emit_enn_cmp_reg(CLT, inst->dst.reg, inst->src.reg);
					emit_enn_jmcc_reg(inst->jmp.reg);
				} else {
					emit_enn_cmp_imm(CLT, inst->dst.reg, inst->src.imm);
					emit_enn_jmcc_imm(inst->jmp.imm);
				}
				break;
			case LT:
				if(inst->src.type == REG)
					emit_enn_cmp_reg(CLT, inst->dst.reg, inst->src.reg);
				else
					emit_enn_cmp_imm(CLT, inst->dst.reg, inst->src.imm);
				break;
			case JGT:
				if (inst->jmp.type == REG) {
					emit_enn_cmp_reg(CGT, inst->dst.reg, inst->src.reg);
					emit_enn_jmcc_reg(inst->jmp.reg);
				} else {
					emit_enn_cmp_imm(CGT, inst->dst.reg, inst->src.imm);
					emit_enn_jmcc_imm(inst->jmp.imm);
				}
				break;
			case GT:
				if(inst->src.type == REG)
					emit_enn_cmp_reg(CGT, inst->dst.reg, inst->src.reg);
				else
					emit_enn_cmp_imm(CGT, inst->dst.reg, inst->src.imm);
				break;
			case JLE:
				if (inst->jmp.type == REG) {
					emit_enn_cmp_reg(CLE, inst->dst.reg, inst->src.reg);
					emit_enn_jmcc_reg(inst->jmp.reg);
				} else {
					emit_enn_cmp_imm(CLE, inst->dst.reg, inst->src.imm);
					emit_enn_jmcc_imm(inst->jmp.imm);
				}
				break;
			case LE:
				if(inst->src.type == REG)
					emit_enn_cmp_reg(CLE, inst->dst.reg, inst->src.reg);
				else
					emit_enn_cmp_imm(CLE, inst->dst.reg, inst->src.imm);
				break;
			case JGE:
				if (inst->jmp.type == REG) {
					emit_enn_cmp_reg(CGE, inst->dst.reg, inst->src.reg);
					emit_enn_jmcc_reg(inst->jmp.reg);
				} else {
					emit_enn_cmp_imm(CGE, inst->dst.reg, inst->src.imm);
					emit_enn_jmcc_imm(inst->jmp.imm);
				}
				break;
			case GE:
				if(inst->src.type == REG)
					emit_enn_cmp_reg(CGE, inst->dst.reg, inst->src.reg);
				else
					emit_enn_cmp_imm(CGE, inst->dst.reg, inst->src.imm);
				break;
			default: printf("something went wrong with insn %d (d:%d, s:%d, j:%d)\n", 
				inst->op, inst->dst.imm, inst->src.imm, inst->jmp.imm);
		}
	}

	static void emit_enn_data_head()
	{
		printf("\n.SEC %%data\n");
		printf("\tADRL  G, @_sec_data\n");
		printf("\tADRM  G, @_sec_data\n");
		printf("\tADRH  G, @_sec_data\n");
		printf("\tJMR   G\n");
	}
	static void emit_enn_data_tail()
	{ printf(".PAD\n%%data\n"); }

	void target_enn(Module* module)
	{
		emit_enn_data_head();
		for(Data* data = module->data; data; data = data->next)
			printf(".INT24 %d\n", data->v);

		emit_enn_data_tail();
	  	emit_enn_head();
	  	int prev_pc = -1;
	  	for (Inst *inst = module->text; inst; inst = inst->next) {
			if (prev_pc != inst->pc) { printf("@L%d\n", inst->pc); }
		    	enn_inst(inst);
		    	prev_pc = inst->pc;
	  	}
		emit_enn_tail();
		return;
	} // must have tailing newline
