---
title: Simple TCP server in assembly
metaDescription: 
date: 2026-02-09T02:00:00.000Z
summary: Talks about how a simple TCP server can be built in assembly.
tags:
  - assembly
  - TCP
  - server
  - registers
---

- Do you need to know assembly? No
- Will you need it in your day 2 day life as software engineer? No
- Then, why learn assembly?
    - To write very high performant and low memory footprint code. Sometimes   situation is too critical.
    - During your system boot-up, no C, Java, C++ gets loaded, but only assembly which directly runs on CPU and load BIOS and other critical things from disk.
    - Read more on this reddit thread [here](https://www.reddit.com/r/asm/comments/p5d9tf/why_should_i_learn_assembly/).

So today w'll be talking about web server in assembly. So, how do we interact with the outside world? Like emitting radio waves, receiving waves etc etc…
- There is no single instruction in x86_64 assembly which allows to do so otherwise it will be a big security mess.
- Instead we can ask the kernel, hey I want to connect to the outside world. Check my permissions and other things, I am a legit user not a bad one.
- So to ask kernel, we need to use **syscalls** which will help us to connect to the outside world.
- Now these syscalls each have a specific number associated with them which is loaded into the register (rax) and then instruction **syscall**. 

Now first let's understand the assembly registers in details.

- Assembly instructions operate on registers, small pieces of very fast memory inside the processor. To process data stored in memory, the processor first needs to load it into registers; and once it has completed working on the data in a register, it needs to store it back to memory.
- First we will look what are the available registers and common instructions before getting into details of TCP server.

![Registers](/src/assets/img/regs.png "registers")

**Caller (the function which is calling another function) saved registers (volatile registers):**
- The caller must save these before calling a function - if it needs them preserved.
- These registers may be destroyed by the function you call.
- **rdi, rsi, rdx, rcx, rax, r8, r9, r10, r11.**
- These are also used for syscall and function arguments.
- If you call a function, assume these registers will change.

**Callee (function which is being called) saved registers (non-volatile registers):**
- The callee must restore these to their original value before returning.
- **rbx, rbp, r12, r13, r14, r15**
- If your function uses any of these registers, you MUST:
- The caller expects these registers to stay unchanged.
```nasm
push r12
…. 
pop r12
ret
```

Common instruction set in x86-64 assembly. These instructions are self explanatory and for more details on them, google them once.

![Instructions](/src/assets/img/instr.jpeg "Instructions")


#### TCP server
- Let's break down TCP server functionality and features.
- So when we develop the TCP server in high level language, we do:
    - Open the socket
    - Bind
    - Listen
    - Now in the for loop:
    - Accept the incoming connection ( parent process will accept the connection )
    - Fork the process.
    - Now child will do all heavy lifting to process the request and parent will keep accepting the request.
    - Child will check whether it's GET or POST and then will act accordingly.
    - rax register is responsible for holding return values and syscall numbers.

1. So opening the socket, binding the address and listening.
```nasm

.intel_syntax noprefix
.globl _start

.section .text

_start:
        # open the socket 
        mov rdi, 2
        mov rsi, 1
        mov rdx, 0 
        mov rax, 41 # syscall number for socket in rax
        syscall

        mov r12, rax

        # bind
        mov rax, 49
        mov rdi, r12
        mov rsi, offset sockaddr_in
        mov rdx, 16
        syscall

        # listen 
        mov rax, 50
        mov rdi, r12
        mov rsi, 0
        syscall 

        mov r13, r12

        jmp iterative_server
```

2. Now iterative server - accept the connection from the client, fork the process and jump to parent or child process depending on return value in rax register.
```nasm
iterative_server:

        # accept 
        mov rax, 43
        mov rdi, r12
        mov rsi, 0
        mov rdx, 0
        syscall

        mov r12, rax

        # forking
        mov rax, 57
        syscall

        cmp rax, 0
        je child_process
        jmp parent_process
```

3. Now the child process will process the request and based on GET or POST. In this exercise we will see only GET request.
```nasm
child_process:

        # close the listening socket connection
        mov rdi, r13
        mov rax, 3
        syscall

        # read request 
        mov rax, 0
        mov rdi, r12
        mov rsi, offset buffer
        mov rdx, 1024
        syscall
        mov r10, rax

        cmp byte ptr [rsi+0], 'G'
        je process_get_request
```

4. Now the code for processing GET request. For now we are sending the static content, but it can be modified to send dynamic content as well.
```nasm
process_get_request:

        # write static response
        mov rdi, r12
        mov rsi, offset static_response
        mov rdx, 19
        mov rax, 1
        syscall 

     
        # close the fd 
        mov rdi, r12
        mov rax, 3
        syscall

        jmp exit
```

5. Parent process 
```nasm
parent_process:

        # close the request processing fd 
        mov rdi, r12
        mov rax, 3
        syscall

        mov r12, r13
        jmp iterative_server

exit:
        mov rdi, 0
        mov rax, 60
        syscall
```

6. Finally how we are storing our data like sock address on which we are listening and static response.
```nasm
.section .data
sockaddr_in:
    .2byte 2
    .2byte 0x5000
    .4byte 0x00000000
    .8byte 0x0

static_response:
    .string "HTTP/1.0 200 OK\r\n\r\n"


.section .bss
buffer: 
    .skip 1024


request_method:
    .skip 5
```

Difference between .data and .bss??
- .data holds initialized global/static data, bytes are stored inside the executable, takes space in the binary on disk, Read/write at runtime.
- .bss holds uninitialized (or zero-initialized) data, not stored in the executable, loader allocates memory and fills it with zeros, exists only in RAM. 
- .rodata - constants (read-only), any write segmentation fault, shared between processes.

#### FORK role in TCP server:

- Let's say we had sock fd as 3, it means we were listening on 3 sock fd and accepting new client connections and getting new client fds.
- So somehow if we could keep accepting connections in parent and then delegate them to child for processing, it will increase the throughput.
- It means we should keep sock fd 3 open in parent, and client specific fd open in child. But when we fork we get the same fds in both parent and child and closing the fd in one does not affect the other.
- So now lets' say on sock fd 3, the new client has connected with sock fd 4 and we fork now, we will get 3 and 4 in the child process. So we can close 3 ( accepting connection fd ) in child and 4 ( client specific fd ) in parent, otherwise there would be chaos of output sent to client. This way we will keep accepting connection requests in the parent process and process the requests in the child process.

So here is the complete code for the simple tcp server:
```nasm
.intel_syntax noprefix
.globl _start

.section .text

_start:
        # open the socket 
        mov rdi, 2
        mov rsi, 1
        mov rdx, 0 
        mov rax, 41
        syscall

        mov r12, rax

        # bind
        mov rax, 49
        mov rdi, r12
        mov rsi, offset sockaddr_in
        mov rdx, 16
        syscall

        # listen 
        mov rax, 50
        mov rdi, r12
        mov rsi, 0
        syscall 

        mov r13, r12

        jmp iterative_server


iterative_server:

        # accept 
        mov rax, 43
        mov rdi, r12
        mov rsi, 0
        mov rdx, 0
        syscall

        mov r12, rax

        # forking
        mov rax, 57
        syscall

        cmp rax, 0
        je child_process
        jmp parent_process

child_process:

        # close the listening socket connection
        mov rdi, r13
        mov rax, 3
        syscall

        # read request 
        mov rax, 0
        mov rdi, r12
        mov rsi, offset buffer
        mov rdx, 1024
        syscall
        mov r10, rax

        cmp byte ptr [rsi+0], 'G'
        je process_get_request


process_get_request:

        # write static response
        mov rdi, r12
        mov rsi, offset static_response
        mov rdx, 34
        mov rax, 1
        syscall 

        # close the fd 
        mov rdi, r12
        mov rax, 3
        syscall

        jmp exit


parent_process:

        # close the request processing fd 
        mov rdi, r12
        mov rax, 3
        syscall

        mov r12, r13
        jmp iterative_server

exit:
        mov rdi, 0
        mov rax, 60
        syscall

.section .data
sockaddr_in:
    .2byte 2
    .2byte 0x901F
    .4byte 0x00000000
    .8byte 0x0

static_response:
    .string "HTTP/1.0 200 OK FROM TCP SERVER\r\n\r\n"


.section .bss
buffer: 
    .skip 1024

request_method:
    .skip 5
```

- Compile the code like shown below:
```bash

as -o server.o  server.s && ld -o server server.o

# run it
./server
```

- You will see something like below:
![Output](/src/assets/img/output.jpeg "Output")

- Check the complete code at [github](https://github.com/ayush-garg341/c_codes/blob/master/assembly/simple_tcp_server.s). For more complex code, there is another file tcp_server.s which also serves POST request. Try to read that code and understand.

I keep sharing stuff like this, if you have any doubt or question, DM me. Follow me for more such content. Happy learning!!
