# System hacking basic

Reminding system hacking basic things. 

## Linux Memory Layout
Linux: Divide process's memory into largely 5 segments -> (code, data, bss, heap, stack)

### Code
Segment for 'exeuctable' machine code. Also known as Text Segment. <br>
Doesn't have execute capability in most cases. 

### Data 
Segment for global variables and constants whose values are already fixed during compile. <br>
Usually have read capability. Data segment is again divided into two segments: writable, nonwritable

Writable: data segment
Non writable: rodata segment 

### Bss
Segment for global variables and constants whose values are not fixed yet during compile.<br>
When program starts, this segment is initialized with zero.  <br>
Usually have 'read', 'write'.

### Stack
Segment for temporary variables such as parameter for functions and local variables.  <br>
Usually used in units called 'stack frames'. Grows toward 'low address'.

### Heap 
Segment for heap data. Dynamically allocated during runtime. <br>
Usually has 'read', 'write'. 

## Shell code
If attacker can controll rip, attacker can execute arbitrary assembly code. 

### ORW shell code

Bellow C code is pseudo code that shows what shell code does. 

```
char buf[0x30];

int fd = open("/tmp/flag", RD_ONLY, NULL);
read(fd, buf, 0x30); 
write(1, buf, 0x30);
```

syscall rax arg0(rdi)         arg1(rsi) arg2(rdx) <br>
read    0x0 unsigned int fd   char *buf size_t count <br>
write   0x1 unsigned int fd   char *buf size_t count <br>
open    0x2 char *filename    int flag  umode_t mode <br>

#### 1. int fd = open("/tmp/flag", RD_ONLY, NULL)
First, string "/tmp/flag" should be on the memory. -> push <br>
But, push operation works in units of 8 bytes. So, push 0x67, then push 0x616c662f706d742f. <br>

"/tmp/flag" in little endian: 0x67616c662f706d742f <br>

O_RDONLY = 0 -> set rsi 0 <br>
mode -> meaningless, set rdx 0 <br>
rax -> 0x2(open) <br>

```
push 0x67
mov rax, 0x616c662f706d742f 
push rax
mov rdi, rsp    ; rdi = "/tmp/flag"
xor rsi, rsi    ; rsi = 0 ; RD_ONLY
xor rdx, rdx    ; rdx = 0
mov rax, 2      ; rax = 2 ; syscall_open
syscall         ; open("/tmp/flag", RD_ONLY, NULL)
```

#### 2. read(fd, buf, 0x30)
File descriptor number obtained by 'open' goes to rax. So, 'mov rdi, rax' <br>
```
mov rdi, rax      ; rdi = fd
mov rsi, rsp
sub rsi, 0x30     ; rsi = rsp-0x30 ; buf
mov rdx, 0x30     ; rdx = 0x30     ; len
mov rax, 0x0      ; rax = 0        ; syscall_read
syscall           ; read(fd, buf, 0x30)
```

#### 3. write(1, buf, 0x30)
standard output(stdout) -> set rdi 0x1 <br>
rsi, rdx stays same <br>

```
mov rdi, 1        ; rdi = 1 ; fd = stdout
mov rax, 0x1      ; rax = 1 ; syscall_write
syscall           ; write(fd, buf, 0x30)
```

How to compile into ELF 
```
// Compile: gcc -o orw orw.c -masm=intel

__asm__(
    ".global run_sh\n"
    "run_sh:\n"

    "push 0x67\n"
    "mov rax, 0x616c662f706d742f \n"
    "push rax\n"
    "mov rdi, rsp    # rdi = '/tmp/flag'\n"
    "xor rsi, rsi    # rsi = 0 ; RD_ONLY\n"
    "xor rdx, rdx    # rdx = 0\n"
    "mov rax, 2      # rax = 2 ; syscall_open\n"
    "syscall         # open('/tmp/flag', RD_ONLY, NULL)\n"
    "\n"
    "mov rdi, rax      # rdi = fd\n"
    "mov rsi, rsp\n"
    "sub rsi, 0x30     # rsi = rsp-0x30 ; buf\n"
    "mov rdx, 0x30     # rdx = 0x30     ; len\n"
    "mov rax, 0x0      # rax = 0        ; syscall_read\n"
    "syscall           # read(fd, buf, 0x30)\n"
    "\n"
    "mov rdi, 1        # rdi = 1 ; fd = stdout\n"
    "mov rax, 0x1      # rax = 1 ; syscall_write\n"
    "syscall           # write(fd, buf, 0x30)\n"
    "\n"
    "xor rdi, rdi      # rdi = 0\n"
    "mov rax, 0x3c	   # rax = sys_exit\n"
    "syscall		   # exit(0)");

void run_sh();

int main() { run_sh(); }
```

### execve shell code

Only uses execve systemcall: execve("/bin/sh",null,null) <br>

syscall rax   arg0(rdi)         arg1(rsi)    arg2(rdx) <br>
execve  0x3b  char *filename    char *argv   char *const *envp

```
mov rax, 0x68732f6e69622f
push rax
mov rdi, rsp  ; rdi = "/bin/sh\x00"
xor rsi, rsi  ; rsi = NULL
xor rdx, rdx  ; rdx = NULL
mov rax, 0x3b ; rax = sys_execve
syscall       ; execve("/bin/sh", null, null)
```

### How to extract shell code

How to extract shellcode in the form of byte code(opcode) 

```
$ objdump -d shellcode.o
shellcode.o:     file format elf32-i386
Disassembly of section .text:
00000000 <_start>:
   0:	31 c0                	xor    %eax,%eax
   2:	50                   	push   %eax
   3:	68 2f 2f 73 68       	push   $0x68732f2f
   8:	68 2f 62 69 6e       	push   $0x6e69622f
   d:	89 e3                	mov    %esp,%ebx
   f:	31 c9                	xor    %ecx,%ecx
  11:	31 d2                	xor    %edx,%edx
  13:	b0 0b                	mov    $0xb,%al
  15:	cd 80                	int    $0x80
```

We can use xxd command 

```
$ objcopy --dump-section .text=shellcode.bin shellcode.o
$ xxd shellcode.bin
00000000: 31c0 5068 2f2f 7368 682f 6269 6e89 e331  1.Ph//shh/bin..1
00000010: c931 d2b0 0bcd 80                        .1.....
```

## Stack buffer overflow

buffer: temporary data storage <br>

Buffer overflow: happens when bigger data than the size of buffer goes into it.

### 1. modification of data

```
int check_auth(char *password) {
    int auth = 0;
    char temp[16];
    
    strncpy(temp, password, strlen(password));
    
    if(!strcmp(temp, "SECRET_PASSWORD"))
        auth = 1;
    
    return auth;
}
```

If 'password' is longer than 16 byte -> buffer overflow <br>

'auth' variable is located behind 'temp' buffer, so if an overflow occurs <br>
value of auth can be tampered.

### 2. Data leak

In C language, normal string terminates with null byte. <br>
If we can overwrite null byte using buffer overflow, it can lead to data leakage.

```
char secret[16] = "secret message";
char barrier[4] = {};
char name[8] = {};
memset(barrier, 0, 4);
printf("Your name: ");
read(0, name, 12);
printf("Your name is %s.", name);
```

### 3. Control flow manipulation

When caller calls callee, it pushes return address. When callee returns, it pops the return address and jumps. <br>
With buffer overflow, return address can be modified.

```
void win() {
    printf("You won!\n");
}

int main(void) {
    char buf[8];
    printf("Overwrite return address with %p:\n", &win);
    read(0, buf, 32);
    return 0;
}
```

buf + saved_rbp(8byte) + ret(8byte) -> can overwrite return address

## Stack Canary

Stack canary: Inserts random value between stack buffer and return address. In the function's epilogue, <br>
it checks for any modification of this value. If altered, process is terminated.

```
+  mov    rax,QWORD PTR fs:0x28
+  mov    QWORD PTR [rbp-0x8],rax
+  xor    eax,eax
+  lea    rax,[rbp-0x10]
-  lea    rax,[rbp-0x8]
   mov    edx,0x20
   mov    rsi,rax
   mov    edi,0x0
   call   read@plt
   mov    eax,0x0
+  mov    rcx,QWORD PTR [rbp-0x8]
+  xor    rcx,QWORD PTR fs:0x28
+  je     0x6f0 <main+70>
+  call   __stack_chk_fail@plt
```

### Canary dynamic analysis

#### Canary insertion

```
pwndbg> ni
   0x5555555546b2 <main+8>     mov    rax, qword ptr fs:[0x28] <0x5555555546aa>
   0x5555555546bb <main+17>    mov    qword ptr [rbp - 8], rax
 ► 0x5555555546bf <main+21>    xor    eax, eax
pwndbg> x/gx $rbp-0x8
0x7fffffffe238:	0xf80f605895da3c00
```

Fetch data from **fs:0x28** and save it to **rbp-0x8** <br>
 
fs: Linux uses fs(segment register) as a pointer to TLS(Thread Local Storage). <br>
TLS: Saves various data that are required when executing process(including canary)

#### Canary check

```
0x5555555546dc <main+50>    mov    rcx, qword ptr [rbp - 8] <0x7ffff7af4191>
0x5555555546e0 <main+54>    xor    rcx, qword ptr fs:[0x28]
0x5555555546e9 <main+63>    je     main+70 <main+70>
0x5555555546eb <main+65>    call   __stack_chk_fail@plt <__stack_chk_fail@plt>
```

xor two values: data from **rbp-0x8** and data from **fs:0x28** <br>
If the two values are not same: **__stack_chk_fail** and terminate 

### Canary bypass

#### TLS read, write

Address of TLS changes everytime. But, if approach to TLS is possible during runtime, reading canary value or <br>
overwriting canary is possible. 

#### Canary leak

```
int main() {
  char memo[8];
  char name[8];
  
  printf("name : ");
  read(0, name, 64);
  printf("hello %s\n", name);
  
  printf("memo : ");
  read(0, memo, 64);
  printf("memo %s\n", memo);
  return 0;
} 
```

**name** buffer overflow -> overwrite 1 byte(null byte) of canary -> canary leak <br>

With this canary value, overwriting name(8byte) + canary(8byte) + rbp(8byte) + ret(8byte) is possible

## NX & ASLR

### NX 

NX: No-eXecutable -> Seperates memory space for execution from memory space for write <br>

When NX is enabled..

```
         Start                End Perm     Size Offset File
      0x400000           0x401000 r--p     1000      0 /home/dreamhack/nx
      0x401000           0x402000 r-xp     1000   1000 /home/dreamhack/nx
      0x402000           0x403000 r--p     1000   2000 /home/dreamhack/nx
      0x403000           0x404000 r--p     1000   2000 /home/dreamhack/nx
      0x404000           0x405000 rw-p     1000   3000 /home/dreamhack/nx
0x7ffff7d7f000     0x7ffff7d82000 rw-p     3000      0 [anon_7ffff7d7f]
0x7ffff7d82000     0x7ffff7daa000 r--p    28000      0 /usr/lib/x86_64-linux-gnu/libc.so.6
0x7ffff7daa000     0x7ffff7f3f000 r-xp   195000  28000 /usr/lib/x86_64-linux-gnu/libc.so.6
0x7ffff7f3f000     0x7ffff7f97000 r--p    58000 1bd000 /usr/lib/x86_64-linux-gnu/libc.so.6
0x7ffff7f97000     0x7ffff7f9b000 r--p     4000 214000 /usr/lib/x86_64-linux-gnu/libc.so.6
0x7ffff7f9b000     0x7ffff7f9d000 rw-p     2000 218000 /usr/lib/x86_64-linux-gnu/libc.so.6
```

No execution capability except code section

### ASLR

ASLR: Allocates stack, heap, shared library, etc into arbitrary memory address whenever binary is executed <br>

```
$ ./addr
buf_stack addr: 0x7ffcd3fcffc0
buf_heap addr: 0xb97260
libc_base addr: 0x7fd7504cd000
printf addr: 0x7fd750531f00
main addr: 0x400667
$ ./addr
buf_stack addr: 0x7ffe4c661f90
buf_heap addr: 0x176d260
libc_base addr: 0x7ffad9e1b000
printf addr: 0x7ffad9e7ff00
main addr: 0x400667
$ ./addr
buf_stack addr: 0x7ffcf2386d80
buf_heap addr: 0x840260
libc_base addr: 0x7fed2664b000
printf addr: 0x7fed266aff00
main addr: 0x400667
```

Address of every function except **main**(in code section) always changes. <br>
Lower 12 bits of the **libc_base** and **printf** address is not changed(due to paging of linux) <br>
Distance between **libc_base** and **printf** is always same. 

### Library

Library: enables sharing functions that are used often in common for efficiency <br>

### Link

Linking -> usually last step of compiling. If the program uses functions from library, those functions are actually linked to libary <br>

In linux, C code => preprocess, compilation, assembly => translated into ELF object file. 

```
gcc -c hello-world.c -o hello-world.o
```

object file has executable format, but doesn't have information about location of functions. <br>
In linkage procedure, linker matches symbols used in the program with actual defenitions. <br>

Dynamic linking: When binary is executed, dynamic library is mapped into process's memory. <br>
Static linking: Static linked binary includes all the functions of library -> doesn't need outer library 

### PLT & GOT

PLT(Procedure Linkage Table), GOT(Global Offset Table) -> Used to find dynamically linked symbol <br>
GOT -> table for address of **resolved** functions <br>

At first, GOT is empty -> PLT calls dynamic linker, write GOT with actual address <br>

### Return To Library



