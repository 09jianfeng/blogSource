---
title: iOS Mach-O file type
date: 2016-08-17 17:22:18
tags: [iOS,逆向]
---

<https://www.objc.io/issues/6-build-tools/mach-o-executables/>

<http://turingh.github.io/2016/03/07/mach-o%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E5%88%86%E6%9E%90/>

```
% touch helloworld.c
```

```
#include <stdio.h>
int main(int argc, char *argv[])
{
    printf("Hello World!\n");
    return 0;
}
```

```
% xcrun clang helloworld.c
% ./a.out
```

or

```
% clang -ccc-print-phases hello.c
```

Briefly put, the compiler will process the helloworld.c input file and produce the executable a.out. This processing consist of multiple steps/stages. What we just did is run all of them in succession:

## Compiler Steps/Stages
[more detail about the clang complier](https://www.objc.io/issues/6-build-tools/compiler/)

### Preprocessing
* Macro expansion
* #include expansion

```
clang -E hello.c | less
```

* Tokenization

```
clang -Xclang -dump-tokens hello.c
```

### Parsing and Semanic Analysis
* Translates preprocessor tokens into a parse tree
* Applies semantic analysis to the parse tree 
* Output an Abstract Syntax Tree (AST) [more AST](http://clang.llvm.org/docs/IntroductionToTheClangAST.html)

```
clang -Xclang -ast-dump -fsyntax-only hello.m
```

### Code Generation and Optimization
* Translates an AST into low-level intermediate code (LLVM IR) [more LLVM](http://www.aosabook.org/en/llvm.html)
* Responible for optimizing the generated code
* target-specific code generation
* Outputs assembly

### Assembler
* Translates assembly code a target object file

### Linker
* Merges multiple object files into an executable(or a dynamic library)


## One step after another to analyse helloworld

### preprocessing
```
% xcrun clang -E helloworld.c | open -f
```

It would output a lot of string since a lot of recursive insertion in `#include <stdio.h>`

PS： In Xcode, you can look at the preprocessor output of any file by seleting `Product -> Perform Action -> Preprocess`


### compilation

```
% xcrun clang -S -o - helloworld.c | open -f
```

PS:  Xcode lets you review the assembly output of any file by selecting Product -> Perform Action -> Assemble.

```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 11
	.globl	_main
	.align	4, 0x90
_main:                                  ## @main
	.cfi_startproc
## BB#0:
	pushq	%rbp
Ltmp0:
	.cfi_def_cfa_offset 16
Ltmp1:
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
Ltmp2:
	.cfi_def_cfa_register %rbp
	subq	$32, %rsp
	leaq	L_.str(%rip), %rax
	movl	$0, -4(%rbp)
	movl	%edi, -8(%rbp)
	movq	%rsi, -16(%rbp)
	movq	%rax, %rdi
	movb	$0, %al
	callq	_printf
	xorl	%ecx, %ecx
	movl	%eax, -20(%rbp)         ## 4-byte Spill
	movl	%ecx, %eax
	addq	$32, %rsp
	popq	%rbp
	retq
	.cfi_endproc

	.section	__TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
	.asciz	"Hello World!\n"


.subsections_via_symbols

```

* The `.section` directive specifies into which section the following will go. More about sections in a bit.
* `.globl` directive specifies that _main is an external symbol. This is our main() function. It needs to be visible outside our binary because the system needs to call it to run the executable.
* The `.align` directive specifies the alignment of what follows. In our case, the following code will be 16 (2^4) byte aligned and padded with 0x90 if needed.
* The `L_.str` label allows the actual code to get a pointer to the string literal. 
* The `.asciz` directive tells the assembler to output a 0-terminated string literal.


## sections
```
% xcrun size -x -l -m a.out 
```

The `__TEXT` segment contains our code to be run. It’s mapped as read-only and executable. The process is allowed to execute the code, but not to modify it. The code can not alter itself, and these mapped pages can therefore never become dirty.

The `__DATA` segment is mapped read/write but non-executable. It contains values that need to be updated.

`__stubs and __stub_`helper are used for the dynamic linker (dyld). This allows for lazily linking in dynamically linked code.

`__const`  are constants, and similarly `__cstring `contains the literal string constants of the executable 

`__bss` section contains uninitialized static variables such as static int a; 

`__common` section contains uninitialized external globals, similar to static variables. An example would be int a; outside a function block.

`__dyld` is a placeholder section, used by the dynamic linker.

[more about session types](https://developer.apple.com/library/mac/documentation/DeveloperTools/Reference/Assembler/)


## Section Content
```
% xcrun otool -s __TEXT __text a.out
```

This is the code of our app. Since -s __TEXT __text is very common, otool has a shortcut to it with the -t argument. We can even look at the disassembled code by adding -v:

```
% xcrun otool -v -t a.out
```







