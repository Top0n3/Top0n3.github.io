---
title: Hackerlab 2025 qualif  writeup 
categories: CTF 
image: null
published: true
---

Hackerlab 2025 qualif  writeup . By Bug Reapers Team
# Challenge Name: Hide and seek
# Author: W1z4rd

![scoreboard](/assets/images/hide_and_seek.png)

Hide ans seek is a pwn chall. Maybe the most difficul pwn challenge of Hackerlab 2025 qualif.
It is a challenge from Hackerlab 2025 final which come back.
In this challenge, we must exploit advanced format string bug to leak the flag stored on the heap.
This is advanced format string exploit because  string buffer is not locate on the stack . It is on the Heap.
So let start


# know your enemmy to win the battle
We download the binary file.
```bash
file chall1
chall1: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=64c6b1ce1a5b8ad6c70e4d4b36faf0ca3272f98f, for GNU/Linux 3.2.0, not stripped
┌─[top0n3@parrot]─[/media/top0n3/7B29B9A756CC225A/ctf/hlb2k25-qualif/pwn/hide2seek]
└──╼ $pwn checksec ./chall1
[*] '/media/top0n3/7B29B9A756CC225A/ctf/hlb2k25-qualif/pwn/hide2seek/chall1'
    Arch:       amd64-64-little
    RELRO:      Full RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        PIE enabled
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No
```
It an ELF little endianne, not striped. So easy to make reverse 
Whene we run the file, it wait for, get thos input and print it to terminal. Simple like that.
Let look at ghidra to deep more

# Decompiled code of main function
```C

void main(void)

{
  size_t sVar1;
  long in_FS_OFFSET;
  char *user_input;
  char **flag;
  FILE *flag_file;
  undefined8 local_10;
  
  local_10 = *(undefined8 *)(in_FS_OFFSET + 0x28);
  flag = (char **)malloc(0x20);
  user_input = (char *)malloc(0x20);
  setbuf(stdin,(char *)0x0);
  setbuf(stdout,(char *)0x0);
  flag_file = fopen("flag.txt","r");
  fgets((char *)flag,0x20,flag_file);
  fgets(user_input,0x20,stdin);
  flag = &user_input;
  sVar1 = strlen((char *)flag);
  if (sVar1 < 0x18) {
                    /* Format string
                        */
    printf(user_input);
  }
                    /* WARNING: Subroutine does not return */
  _exit(0);
}
```
Clean to be understood!!!
- The prog allocate two array using malloc, so those buffer are on Heap.

- Stored the result off malloc into two local variable
- Open flag.txt and read it contain. Store flag.txt contain on one of malloc array(flag is store on heap)
- Ask user input

- When user input some string, it call printf(user_input)

As we can see, the printf is call with user input directly. That create format string vulnerability..
Our purpuse here is to exploit the format string vuln ro read the flag which is on the heap

# Little explaination of format string vulnerability
for those who don't have a lot of knowledge  about format string !
Format string bug arise when C like programming langace  call function of printf family  with user input without using format specification. 
example in our case: printf(user_input).
By using format specificator on it input, user can read stack data, get arbitrary memory write and so one
# Exploit steps
As buffer is not on the stack, we can not exploit this vuln like clasic format string exploit.
I try a lot of thing and after some try and error,  i google multiple time .
Finally i got an article that explaint a little about this exploit thechnique.
[here is the link](https://www.jaybosamiya.com/blog/2017/04/06/adv-format-string/)

The most important part of this article that help me is bellow.
I advice you to read it carefully as it  the explaination of the technique we must use to exploit the format string here
```
If we have our format string buffer not on the stack, then we can still gain a write-what-where primitive, though it is a little more complex. First off, we need to stop using the position specifiers n$, since if this is used, then printf internally copies the stack (which we will be modifying as we go along). Now, we find two pointers that point ahead into the stack itself, and use those to overwrite the lower order bytes of two further ahead pointing pointers on the stack, so that they now point to x+0 and x+2 where x is some location further ahead on the stack. Using these two overwrites, we are able to completely control the 4 bytes at x, and this becomes our where in the primitive. Now we just have to ignore more positions on the format string until we come to this point, and we have a write-what-where primitive.
```

We need to use technique describe on this article to exploit the  bug
Lucky when i debug the program, i can notice that when printf(user_input) is call, at stack 0ffset 6, we have a pointer to heap address (user_input) pointer  and at stack offset 7, we have a pointer to to stack offset 6. 

What we can do with that ?
the pointer at offset 6 is heap pointer but not point to flag location.
if we found some way to make it point to flag location and leak it with %6$s, we can read the flag.
To make it point to flag, we need to overwrite it least significant byte to 0xa0.

we have all information we need to exploit the pogram with our technique describe in the previous article.


# Construct the payload
the payload is to overwrite the lsb of address at offset 6 to 0xa0( make it equal to flag address) and after that, leak it contain.
the basic adeal is something like this : %160c%6$hhn%6$s

but it don't work. Remember format buffer is on heap not on stack. when we input payload like this, we will overwrite  the value of internal stack copy address not the real stack. because payload to overwrite lsb (%160c%6$hhn)  contain argument direct symbole "$".
our payload to overwrite lsb of pointer at offset 6 must not contain argument direct symbole. 
How to do that?
if you wanna leak value at offset 6, we can do it like this:
- %6$s  ---> here we use argument direct symbole "$"
or we can do it like that
- %x%x%x%x%x%s  --->  here we don't use argument direct symbole  every format specificator "%x" consome on offset of stack so %x%x%x%x%x consome five first offset and the last %s is for offset 6 wich eill be egal to %6$s

Our initial payload %160c%6$hhn%6$s become : f"%x%x%x%x%144c%hhn%6$s" we replace 160 to 140 because we already print 16 bytes with our %x%x%x%x

but this payload don't solve our problem.

In the  article it said that we must use other pointer to overwrite the pointer at offset 6. and thos other pointer here is at offset 7. so change our payload to tague offset 7 pointer.
it become: f"%x%x%x%x%x%140c%hhn%6$s" we pad it with one %x to reach to offset 7 and make subtruct 4 to padding char value.
we i input this payload, and make break point, i see that instead of overwrite lsb to 0xa0, i overwrite it to 0xa4 so i substruct padding value to and use 134 instead of 140.

our final payload is this:%x%x%x%x%x%134c%hhn.%6$p:%6$s 
