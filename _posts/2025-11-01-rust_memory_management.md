# Freeing memory with C that was initialized by Rust

## Background

This past weeks I've been working on a Rust PAM client to authenticate a password. Although there are [some crates out there](https://crates.io/crates/pam) that do this for you, I thought I could do one myself and interact with PAM directly through FFI. This means creating the Rust bindings with [bindgen](https://docs.rs/bindgen/latest/bindgen/) using `pam_appl.h` (found in `/usr/include/security/pam_appl.h` in Debian), and then reading [PAM's documentation](https://man7.org/linux/man-pages/man3/pam.3.html) to understand how to do the auth dance.

One of those dance steps involves understanding how PAM expects [`pam_conv`](https://man7.org/linux/man-pages/man3/pam_conv.3.html). This is a function called by PAM modules when your client calls (for example) `pam_authenticate`. So you create this conv function and send it to `pam_start`, it returns a blind struct `pam_handle_t` which the caller needs to pass over through all the PAM functions it calls (this struct contains the function). Later when the client calls `pam_authenticate`, PAM will retrieve the conversation function from `pam_handle_t` and call it with an array of `pam_message`. It'll also initialize a pointer to an array of `pam_response` and send it over for the client to fill in and reply back. PAM will later free the memory in this response array.

So an FFI between a Rust conversation function and C PAM module means there's a `malloc` and `free` dance that might end in disaster. Rust has a _robust_ way of allocating/deallocating memory while C has a more _simpler_ way.

## C and memory management

This is a (very quick) summary of how C manages memory on any given program.

Consider the following program:

```C
#include <stdio.h>

int main() {
    int a = 1;

    printf("The value of a=%d\n", a);
    return 0;
}
```

This program creates a variable `a` with the value `1` and prints it using `printf`. Because the value of this variable is known at compile time there's no need for runtime allocations, the compiler places it in the stack:

```bash
$ gdb stack_test
...
(gdb) b main
Breakpoint 1 at 0x1155: file stack_test.c, line 5.
(gdb) r
Starting program: /home/nicolas/stack_test 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, main () at stack_test.c:5
5           int a = 1;
(gdb) disassemble /s main
Dump of assembler code for function main:
stack_test.c:
4       int main() {
   0x0000555555555149 <+0>:     endbr64
   0x000055555555514d <+4>:     push   %rbp
   0x000055555555514e <+5>:     mov    %rsp,%rbp
   0x0000555555555151 <+8>:     sub    $0x10,%rsp

5           int a = 1;
=> 0x0000555555555155 <+12>:    movl   $0x1,-0x4(%rbp)

6
7           printf("The value of a=%d\n", a);
   0x000055555555515c <+19>:    mov    -0x4(%rbp),%eax
   0x000055555555515f <+22>:    mov    %eax,%esi
   0x0000555555555161 <+24>:    lea    0xe9c(%rip),%rax        # 0x555555556004
   0x0000555555555168 <+31>:    mov    %rax,%rdi
   0x000055555555516b <+34>:    mov    $0x0,%eax
   0x0000555555555170 <+39>:    call   0x555555555050 <printf@plt>

8           return 0;
   0x0000555555555175 <+44>:    mov    $0x0,%eax

9       }
   0x000055555555517a <+49>:    leave
   0x000055555555517b <+50>:    ret
End of assembler dump.
```

The instruction `movl   $0x1,-0x4(%rbp)` means "move the value `1` into the contents of `%rbp` (the stack base pointer)". Because we're dealing with `int` in my machine that's a 32-bit value (4 bytes), so it shifts 4 bytes to store the contents (and uses `movl` instead of `mov`).

```bash
(gdb) s
6           printf("The value of a=%d\n", a);
(gdb) info registers rsp rbp
rsp            0x7fffffffe280      0x7fffffffe280
rbp            0x7fffffffe290      0x7fffffffe290
(gdb) x/2gx $rsp
0x7fffffffe280: 0x00007fffffffe370      0x00000001ffffe3b8
```

And there you go, `0x00000001ffffe3b8` has `0x00000001` in its upper half, the value of `a`.

Let's see how C manages the **heap**, consider the following program:

```C
#include <stdio.h>
#include <stdlib.h>

int main() {
    int *array;
    int len = 5;

    array = (int *) malloc(len * sizeof(int));

    for (int i = 0; i < len; i++) {
        array[i] = i;
        printf("array[%d] = %d\n", i, array[i]);
    }

    free(array);
    return 0;
}
```

The disassembly of the binary is as follows:

```bash
$ gdb free_test 
...
(gdb) b main
Breakpoint 1 at 0x1195: file free_test.c, line 6.
(gdb) r
Starting program: /home/nicolas/free_test 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, main () at free_test.c:6
6           int len = 5;
(gdb) disassemble /s main
Dump of assembler code for function main:
free_test.c:
4       int main() {
   0x0000555555555189 <+0>:     endbr64
   0x000055555555518d <+4>:     push   %rbp
   0x000055555555518e <+5>:     mov    %rsp,%rbp
   0x0000555555555191 <+8>:     sub    $0x10,%rsp

5           int *array;
6           int len = 5;
=> 0x0000555555555195 <+12>:    movl   $0x5,-0xc(%rbp)

7
8           array = (int *) malloc(len * sizeof(int));
   0x000055555555519c <+19>:    mov    -0xc(%rbp),%eax
   0x000055555555519f <+22>:    cltq
   0x00005555555551a1 <+24>:    shl    $0x2,%rax
   0x00005555555551a5 <+28>:    mov    %rax,%rdi
   0x00005555555551a8 <+31>:    call   0x555555555090 <malloc@plt>
   0x00005555555551ad <+36>:    mov    %rax,-0x8(%rbp)

9
10          for (int i = 0; i < len; i++) {
   0x00005555555551b1 <+40>:    movl   $0x0,-0x10(%rbp)
   0x00005555555551b8 <+47>:    jmp    0x555555555206 <main+125>

11              array[i] = i;
   0x00005555555551ba <+49>:    mov    -0x10(%rbp),%eax
   0x00005555555551bd <+52>:    cltq
   0x00005555555551bf <+54>:    lea    0x0(,%rax,4),%rdx
   0x00005555555551c7 <+62>:    mov    -0x8(%rbp),%rax
   0x00005555555551cb <+66>:    add    %rax,%rdx
   0x00005555555551ce <+69>:    mov    -0x10(%rbp),%eax
   0x00005555555551d1 <+72>:    mov    %eax,(%rdx)

12              printf("array[%d] = %d\n", i, array[i]);
   0x00005555555551d3 <+74>:    mov    -0x10(%rbp),%eax
   0x00005555555551d6 <+77>:    cltq
   0x00005555555551d8 <+79>:    lea    0x0(,%rax,4),%rdx
   0x00005555555551e0 <+87>:    mov    -0x8(%rbp),%rax
   0x00005555555551e4 <+91>:    add    %rdx,%rax
   0x00005555555551e7 <+94>:    mov    (%rax),%edx
   0x00005555555551e9 <+96>:    mov    -0x10(%rbp),%eax
   0x00005555555551ec <+99>:    mov    %eax,%esi
   0x00005555555551ee <+101>:   lea    0xe0f(%rip),%rax        # 0x555555556004
   0x00005555555551f5 <+108>:   mov    %rax,%rdi
   0x00005555555551f8 <+111>:   mov    $0x0,%eax
   0x00005555555551fd <+116>:   call   0x555555555080 <printf@plt>

10          for (int i = 0; i < len; i++) {
   0x0000555555555202 <+121>:   addl   $0x1,-0x10(%rbp)
   0x0000555555555206 <+125>:   mov    -0x10(%rbp),%eax
   0x0000555555555209 <+128>:   cmp    -0xc(%rbp),%eax
   0x000055555555520c <+131>:   jl     0x5555555551ba <main+49>

13          }
14
15          free(array);
   0x000055555555520e <+133>:   mov    -0x8(%rbp),%rax
   0x0000555555555212 <+137>:   mov    %rax,%rdi
   0x0000555555555215 <+140>:   call   0x555555555070 <free@plt>

16          return 0;
   0x000055555555521a <+145>:   mov    $0x0,%eax

17      }
   0x000055555555521f <+150>:   leave
   0x0000555555555220 <+151>:   ret
End of assembler dump.
```

Similarly as before it used the stack to save the contents of `len` and `array` with `movl` and `mov` respectively. Let's see how that looks like in the stack:

```bash
(gdb) stepi
8           array = (int *) malloc(len * sizeof(int));
(gdb) info registers rsp rbp
rsp            0x7fffffffe280      0x7fffffffe280
rbp            0x7fffffffe290      0x7fffffffe290
(gdb) x/2gx $rsp
0x7fffffffe280: 0x00000005ffffe370      0x00007fffffffe3b8
(gdb) 
```

It executed `movl   $0x5,-0xc(%rbp)`: "Move the 32 bit data `0x5` into `%rbp` shifted `0xc` bytes". We can see the value at the upper half of the 64 bit data: `0x00000005ffffe370`. Different than before where it only shifted `0x4`, because it now made space for the 64 bit address that `array` is going to have. The following instructions will assign this memory using malloc:

```bash
0x000055555555519c <+19>:    mov    -0xc(%rbp),%eax
0x000055555555519f <+22>:    cltq
0x00005555555551a1 <+24>:    shl    $0x2,%rax
0x00005555555551a5 <+28>:    mov    %rax,%rdi
0x00005555555551a8 <+31>:    call   0x555555555090 <malloc@plt>
0x00005555555551ad <+36>:    mov    %rax,-0x8(%rbp)
```

The instructions before the call to `malloc` are meaningful for `malloc` only, it's preparing all the input the function needs. We should only focus on the return that `malloc` gave us in `%rax`, this is the value of `array` and will be placed to the stack accordingly: `mov    %rax,-0x8(%rbp)`. Let's run this code and inspect the stack:

```bash
(gdb) advance 10
main () at free_test.c:10
10          for (int i = 0; i < len; i++) {
(gdb) x/2gx $rsp
0x7fffffffe280: 0x00000005ffffe370      0x00005555555592a0
(gdb) x/1gx 0x00005555555592a0
0x5555555592a0: 0x0000000000000000
(gdb) 
```

`advance 10` just means to move to line 10 in the code, handy `gdb` function when debugging symbols ar available. This effectively triggers the `malloc` code. Now the stack contains a new value: `0x00005555555592a0`, this is the result of `mov    %rax,-0x8(%rbp)` (store the result of `malloc` to `%rbp` shifted 8 bytes). The value `0x00005555555592a0` is the address of `array`, and the OS promised that at that address + 5 bytes belong to this program. We can explore the contents of the memory (now the **heap**):

```bash
(gdb) x/3gx 0x00005555555592a0
0x5555555592a0: 0x0000000000000000      0x0000000000000000
0x5555555592b0: 0x0000000000000000
```

Because `int` default to 32 bit in my machine, `malloc` allocated 5 32 bit spaces, represented here by 64 bit chunks. Let's keep running the code to see how they fill up.

```bash
(gdb) advance 15
array[0] = 0
array[1] = 1
array[2] = 2
array[3] = 3
array[4] = 4
main () at free_test.c:15
15          free(array);
(gdb) info registers rsp rbp
rsp            0x7fffffffe280      0x7fffffffe280
rbp            0x7fffffffe290      0x7fffffffe290
(gdb) x/2gx $rsp
0x7fffffffe280: 0x0000000500000005      0x00005555555592a0
(gdb) x/3gx 0x00005555555592a0
0x5555555592a0: 0x0000000100000000      0x0000000300000002
0x5555555592b0: 0x0000000000000004
(gdb) 
```

Don't worry about the code in the `for` loop, although we can dive into it and explain bit by bit, it's not the focus of this post. The important thing is that the resulting numbers were placed in the **heap** at `0x00005555555592a0` (the address that `malloc` gave us) and 5 32 bit spaces after.

Then we call `free()` to tell the OS that we don't want that memory anymore.

```bash
(gdb) advance 16
main () at free_test.c:16
16          return 0;
(gdb) info registers rsp rbp
rsp            0x7fffffffe280      0x7fffffffe280
rbp            0x7fffffffe290      0x7fffffffe290
(gdb) x/2gx $rsp
0x7fffffffe280: 0x0000000500000005      0x00005555555592a0
(gdb) x/3gx 0x00005555555592a0
0x5555555592a0: 0x0000000555555559      0x86c25f8360a4c502
0x5555555592b0: 0x0000000000000004
(gdb) 
```

`free()` will remove the assignment it made for the memory we requested, but will not alter the data that exists there. Although we can see a `0x00000004` in there from our code, it's not guaranteed to exist after `free()` (for example, the other numbers were replaced by other things).

In C you need to make sure to alloc and dealloc memory as you use it, this is why many modern langauges like Go or Rust have memory management included for you by default (meaning memory assigned to variables in the **heap** will be freed automatically when they are no longer used).

## Rust and memory management

The C examples above did not use a garbace collector. Modern langagues like Java or Go have a runtime program called "Gabage collector" (abbreviated GC), this takes care of memory management for us so we don't have to worry about pairing `malloc` to `free` in our code.

[Rust doesn't have a GC](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#memory-and-allocation), it automatically returns the memory after leaving out of scope. It does this by calling the method `drop` in the `Drop` trait for the object.

Let's see this with an example, let's start with a simple example:

```rust
fn main() {
    let a = 1;

    println!("The value of a={}", a);
}
```

Because `rustc` (Rust's compiler) knows the size and value of `a` at compile time, it places the variable in the stack:

```bash
$ gdb rust_stack_memory 
...
(gdb) start
Temporary breakpoint 1 at 0x15354: file rust_stack_memory.rs, line 2.
Starting program: /home/nicolas/rust_stack_memory 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Temporary breakpoint 1, rust_stack_memory::main () at rust_stack_memory.rs:2
2           let a = 1;
(gdb) disassemble /s
Dump of assembler code for function _ZN17rust_stack_memory4main17hd3a3de1a38df406aE:
rust_stack_memory.rs:
1       fn main() {
   0x0000555555569350 <+0>:     sub    $0x58,%rsp

2           let a = 1;
=> 0x0000555555569354 <+4>:     movl   $0x1,0x4(%rsp)

3
4           println!("The value of a={}", a);
   0x000055555556935c <+12>:    lea    0x48(%rsp),%rdi
   0x0000555555569361 <+17>:    lea    0x4(%rsp),%rsi
   0x0000555555569366 <+22>:    call   0x555555569480 <_ZN4core3fmt2rt8Argument11new_display17h7e24be30af2fb36bE>
   0x000055555556936b <+27>:    mov    0x48(%rsp),%rax
   0x0000555555569370 <+32>:    mov    %rax,0x38(%rsp)
   0x0000555555569375 <+37>:    mov    0x50(%rsp),%rax
   0x000055555556937a <+42>:    mov    %rax,0x40(%rsp)
   0x000055555556937f <+47>:    lea    0x8(%rsp),%rdi
   0x0000555555569384 <+52>:    lea    0x413cd(%rip),%rsi        # 0x5555555aa758
   0x000055555556938b <+59>:    lea    0x38(%rsp),%rdx
   0x0000555555569390 <+64>:    call   0x555555569440 <_ZN4core3fmt2rt38_$LT$impl$u20$core..fmt..Arguments$GT$6new_v117hcff756169ed10857E>
   0x0000555555569395 <+69>:    lea    0x8(%rsp),%rdi
   0x000055555556939a <+74>:    call   *0x43f20(%rip)        # 0x5555555ad2c0

5       }
   0x00005555555693a0 <+80>:    add    $0x58,%rsp
   0x00005555555693a4 <+84>:    ret
End of assembler dump.
(gdb) 
```

Inspecting the stack:

```bash
(gdb) stepi
4           println!("The value of a={}", a);
(gdb) info registers rsp rbp
rsp            0x7fffffffe020      0x7fffffffe020
rbp            0x7fffffffe210      0x7fffffffe210
(gdb) x/1gx $rsp
0x7fffffffe020: 0x0000000100000000
(gdb) 
```

We find the value was pushed to the stack in `%rsp`, so far this is the same behaviour as C. Let's see an example with the **heap**:

```rust
fn main() {
    let a = String::from("Hello World!");
    println!("a={}", a);
}
```

The [`String::from`](https://doc.rust-lang.org/std/convert/trait.From.html#tymethod.from) method creates a [`String`](https://doc.rust-lang.org/std/string/struct.String.html) object that can grow, so this must be done in the **heap**. We can see what it does by disassembling the binary:

```bash
$ gdb rust_heap 
...
(gdb) start
Temporary breakpoint 1 at 0x16a77: file rust_heap.rs, line 2.
Starting program: /home/nicolas/rust_heap 

[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Temporary breakpoint 1, rust_heap::main () at rust_heap.rs:2
2           let a = String::from("Hello World!");
(gdb) disassemble /s
Dump of assembler code for function _ZN9rust_heap4main17h9468d83c908b7b2bE:
rust_heap.rs:
1       fn main() {
   0x000055555556aa70 <+0>:     sub    $0x88,%rsp

2           let a = String::from("Hello World!");
=> 0x000055555556aa77 <+7>:     lea    -0xca43(%rip),%rsi        # 0x55555555e03b
   0x000055555556aa7e <+14>:    lea    0x8(%rsp),%rdi
   0x000055555556aa83 <+19>:    mov    %rdi,(%rsp)
   0x000055555556aa87 <+23>:    mov    $0xc,%edx
   0x000055555556aa8c <+28>:    call   0x55555556a8b0 <_ZN76_$LT$alloc..string..String$u20$as$u20$core..convert..From$LT$$RF$str$GT$$GT$4from17hd60d7e630ec9d734E>
   0x000055555556aa91 <+33>:    mov    (%rsp),%rsi
   0x000055555556aa95 <+37>:    lea    0x68(%rsp),%rdi

3           println!("a={}", a);
   0x000055555556aa9a <+42>:    call   0x555555569c30 <_ZN4core3fmt2rt8Argument11new_display17he8b03bd6e4c77b4cE>
   0x000055555556aa9f <+47>:    jmp    0x55555556aac0 <_ZN9rust_heap4main17h9468d83c908b7b2bE+80>
   0x000055555556aaa1 <+49>:    lea    0x8(%rsp),%rdi

4       }
   0x000055555556aaa6 <+54>:    call   0x55555556a1a0 <_ZN4core3ptr42drop_in_place$LT$alloc..string..String$GT$17h2767dddabd238b94E>
   0x000055555556aaab <+59>:    jmp    0x55555556ab0a <_ZN9rust_heap4main17h9468d83c908b7b2bE+154>
   0x000055555556aaad <+61>:    mov    %rax,%rcx
   0x000055555556aab0 <+64>:    mov    %edx,%eax
   0x000055555556aab2 <+66>:    mov    %rcx,0x78(%rsp)
   0x000055555556aab7 <+71>:    mov    %eax,0x80(%rsp)
   0x000055555556aabe <+78>:    jmp    0x55555556aaa1 <_ZN9rust_heap4main17h9468d83c908b7b2bE+49>

3           println!("a={}", a);
   0x000055555556aac0 <+80>:    movups 0x68(%rsp),%xmm0
   0x000055555556aac5 <+85>:    movaps %xmm0,0x50(%rsp)
   0x000055555556aaca <+90>:    lea    0x41357(%rip),%rsi        # 0x5555555abe28
   0x000055555556aad1 <+97>:    lea    0x20(%rsp),%rdi
   0x000055555556aad6 <+102>:   lea    0x50(%rsp),%rdx
   0x000055555556aadb <+107>:   call   0x555555569bf0 <_ZN4core3fmt2rt38_$LT$impl$u20$core..fmt..Arguments$GT$6new_v117hb30cbabe5203282dE>
   0x000055555556aae0 <+112>:   jmp    0x55555556aae2 <_ZN9rust_heap4main17h9468d83c908b7b2bE+114>
   0x000055555556aae2 <+114>:   mov    0x43edf(%rip),%rax        # 0x5555555ae9c8
   0x000055555556aae9 <+121>:   lea    0x20(%rsp),%rdi
   0x000055555556aaee <+126>:   call   *%rax
   0x000055555556aaf0 <+128>:   jmp    0x55555556aaf2 <_ZN9rust_heap4main17h9468d83c908b7b2bE+130>

4       }
   0x000055555556aaf2 <+130>:   lea    0x8(%rsp),%rdi
   0x000055555556aaf7 <+135>:   call   0x55555556a1a0 <_ZN4core3ptr42drop_in_place$LT$alloc..string..String$GT$17h2767dddabd238b94E>
   0x000055555556aafc <+140>:   add    $0x88,%rsp
   0x000055555556ab03 <+147>:   ret

1       fn main() {
   0x000055555556ab04 <+148>:   call   *0x43e7e(%rip)        # 0x5555555ae988
   0x000055555556ab0a <+154>:   mov    0x78(%rsp),%rdi
   0x000055555556ab0f <+159>:   call   0x5555555aad00 <_Unwind_Resume@plt>
End of assembler dump.
(gdb) 
```

Well this is way more than what was happening in the C version of this binary. Because `String` is an object it's calling its constructor: `call   0x55555556a8b0 <_ZN76_$LT$alloc..string..String$u20$as$u20$core..convert..From$LT$$RF$str$GT$$GT$4from17hd60d7e630ec9d734E>`, this is a mangled function name but essentially allocates the String in the **heap** and returns information about the object. The function itself is not important (although there's a lot happening inside), what's important is how Rust prepares the arguments for the function and how it retrieves them back:

```asm
0x000055555556aa77 <+7>:     lea    -0xca43(%rip),%rsi        # 0x55555555e03b
0x000055555556aa7e <+14>:    lea    0x8(%rsp),%rdi
0x000055555556aa83 <+19>:    mov    %rdi,(%rsp)
0x000055555556aa87 <+23>:    mov    $0xc,%edx
0x000055555556aa8c <+28>:    call   0x55555556a8b0 <_ZN76_$LT$alloc..string..String$u20$as$u20$core..convert..From$LT$$RF$str$GT$$GT$4from17hd60d7e630ec9d734E>
0x000055555556aa91 <+33>:    mov    (%rsp),%rsi
0x000055555556aa95 <+37>:    lea    0x68(%rsp),%rdi
```

TODO: Continue.
