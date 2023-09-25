---
title: "DG'hAck 2020: The 'dharma.exe' challenge"
date: 2020-11-28T23:03:43+01:00
summary: "dharma binary analysis: Breaking binary protections"
tags:
  - dghack2020
---

# Introduction

*dharma* is a binary. It is required to reverse it, to understand it & retrieve the flag from it. No information is given outside the binary itself:

```sh
$ sha256sum dharma
99eebee06f1145bbc16e94afdf523c95035a1ca979a75f48d9d3f19a5cac1645  dharma
```

# Walkthrough

Launching it doesn't really help:

```sh
$ ./dharma 
Password: 
blah
Nope, try again...
```

No need to say the binary implements some anti-debugging techniques:

```sh
$ gdb ./dharma 
... snap ...
Thread 2 "dharma" received signal SIGTRAP, Trace/breakpoint trap.
[Switching to Thread 0x7ffff7d97700 (LWP 1308614)]
0x0000555555555894 in ?? ()
(gdb) bt
#0  0x0000555555555894 in ?? ()
#1  0x00007ffff7f6e432 in start_thread () from /lib64/libpthread.so.0
#2  0x00007ffff7e9c913 in clone () from /lib64/libc.so.6
```

The most common breakpoint trap is the int3/0xCC opcode trick. With objdump(1), it is quite easy to locate it. We'll replace it with a NOP:

```sh 
$ objdump -d dharma|grep -1 int3
    188f:       48 89 7d f8             mov    %rdi,-0x8(%rbp)
    1893:       cc                      int3   
    1894:       bf e8 03 00 00          mov    $0x3e8,%edi

$ cp dharma dharma.bak; printf '\x90' | dd of=dharma bs=1 seek=$((0x1893)) conv=notrunc
1+0 records in
1+0 records out
1 byte copied, 3.0021e-05 s, 33.3 kB/s

$ xxd dharma|grep 00001890
00001890: 897d f890 bfe8 0300 00e8 72fa ffff ebf3  .}........r.....
```

Next issue: Running the new binary will trigger another problem. The binary now receives `signal SIG32, Real-time event 32`:

```sh
(gdb) r
[New Thread 0x7ffff7d97700 (LWP 1310410)]
[New Thread 0x7ffff7596700 (LWP 1310411)]
Password: 

Thread 2 "dharma" received signal SIG32, Real-time event 32.
[Switching to Thread 0x7ffff7d97700 (LWP 1310410)]
0x00007ffff7e63801 in clock_nanosleep@GLIBC_2.2.5 () from /lib64/libc.so.6
(gdb) bt
#0  0x00007ffff7e63801 in clock_nanosleep@GLIBC_2.2.5 () from /lib64/libc.so.6
#1  0x00007ffff7e69157 in nanosleep () from /lib64/libc.so.6
#2  0x00007ffff7e94869 in usleep () from /lib64/libc.so.6
#3  0x000055555555589e in ?? ()
#4  0x00007ffff7f6e432 in start_thread () from /lib64/libpthread.so.0
#5  0x00007ffff7e9c913 in clone () from /lib64/libc.so.6
(gdb) info threads
  Id   Target Id                                    Frame 
  1    Thread 0x7ffff7d98740 (LWP 1310409) "dharma" 0x00007ffff7e8d4cc in read () from /lib64/libc.so.6
* 2    Thread 0x7ffff7d97700 (LWP 1310410) "dharma" 0x00007ffff7e63801 in clock_nanosleep@GLIBC_2.2.5 () from /lib64/libc.so.6
```

Okay, that's it. Checking with radare2:

```asm
            ; DATA XREF from main @ 0x18ad
            0x00001883      f30f1efa       endbr64
            0x00001887      55             push rbp
            0x00001888      4889e5         mov rbp, rsp
            0x0000188b      4883ec10       sub rsp, 0x10
            0x0000188f      48897df8       mov qword [rbp - 8], rdi
            ; CODE XREF from rip @ +0x29
        ┌─> 0x00001893      cc             int3
        ╎   0x00001894      bfe8030000     mov edi, 0x3e8
        ╎   0x00001899      e872faffff     call sym.imp.usleep         ; int usleep(int s)
        └─< 0x0000189e      ebf3           jmp 0x1893
            ; DATA XREF from entry0 @ 0x1341
┌ 86: int main (int argc, char **argv, char **envp);
│           0x000018a0      f30f1efa       endbr64
│           0x000018a4      55             push rbp
│           0x000018a5      4889e5         mov rbp, rsp
│           0x000018a8      b900000000     mov ecx, 0
│           0x000018ad      488d15cfffff.  lea rdx, [0x00001883]
│           0x000018b4      be00000000     mov esi, 0
│           0x000018b9      488d3db86900.  lea rdi, [0x00008278]
│           0x000018c0      e8ebf8ffff     call sym.imp.pthread_create
```

Okay, in fact, the function called by `pthread_create` is basically useless. Let's just kill it. To do this, I could edit the binary again, but for some reason, I used an old shared library trick to overwrite `pthread_create` & `pthread_cancel` functions.

```c
#include <pthread.h>
#include <signal.h>
#define __USE_GNU
#include <dlfcn.h>
#include <stdio.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine) (void *), void *arg)
{
    printf("pthread_create: %p %p %p %p\n", thread, attr, start_routine, arg);
    return 0;
}

int pthread_cancel(pthread_t thread)
{
    return 0;
}
```

Build & use with:

```sh
$ gcc -shared -o fuck.so fuck.c -fPIC
$ export LD_PRELOAD=$(pwd)/fuck.so
```

Then, I got curious. What's happening when calling `strace(1)` ?

```c
... snip ...
memfd_create("2O3naSbh", MFD_CLOEXEC)   = 3
write(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0 \22\0\0\0\0\0\0"..., 16936) = 16936
... snip ...
openat(AT_FDCWD, "/proc/1314097/fd/3", O_RDONLY|O_CLOEXEC) = 4
read(4, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0 \22\0\0\0\0\0\0"..., 832) = 832
fstat(4, {st_mode=S_IFREG|0777, st_size=16936, ...}) = 0
mmap(NULL, 16544, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 4, 0) = 0x7f1b4803a000
mmap(0x7f1b4803b000, 4096, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 4, 0x1000) = 0x7f1b4803b000
mmap(0x7f1b4803c000, 4096, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 4, 0x2000) = 0x7f1b4803c000
mmap(0x7f1b4803d000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 4, 0x2000) = 0x7f1b4803d000
close(4)                                = 0
mprotect(0x7f1b4803d000, 4096, PROT_READ) = 0
close(3)                                = 0
write(1, "pthread_create: 0x561c5e63d278 ("..., 58pthread_create: 0x561c5e63d278 (nil) 0x7f1b4803bd64 (nil)
) = 58
write(1, "Password: \n", 11Password: 
)            = 11
fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x4), ...}) = 0

```

It is even better with `ltrace(1)`:

```c
$ ltrace ./dharma
pthread_create(0x55f1b1ac6278, 0, 0x55f1b1abf883, 0pthread_create: 0x55f1b1ac6278 (nil) 0x55f1b1abf883 (nil)
)                                             = 0
signal(SIGTRAP, 0x55f1b1abf875)                                                                  = nil
memfd_create(0x55f1b1ac6250, 1, 0x7ffcd1f76bb0, 0)                                               = 3
strlen("2O3naSbh")                                                                               = 8
write(3, "\177ELF\002\001\001", 16936)                                                           = 16936
uname(0x7ffcd1f767a0)                                                                            = 0
strtok("5.9.9-100.fc32.x86_64", ".")                                                             = "5"
atoi(0x7ffcd1f76822, 2, 2, 1)                                                                    = 5
atoi(0x7ffcd1f76822, 5, 0, 0x7ffcd1f76823)                                                       = 5
getpid()                                                                                         = 1314298
snprintf("/proc/1314298/fd/3", 1024, "/proc/%d/fd/%d", 1314298, 3)                               = 18
dlopen("/proc/1314298/fd/3", 1)                                                                  = 0x55f1b295a700
close(3)                                                                                         = 0
dlsym(0x55f1b295a700, "init")                                                                    = 0x7f5af81a5c80
dlsym(0x55f1b295a700, "_libc_start_main")                                                        = 0x7f5af81a5d64
pthread_cancel(0, 1, 1, 0)                                                                       = 0
pthread_create(0x55f1b1ac6278, 0, 0x7f5af81a5d64, 0pthread_create: 0x55f1b1ac6278 (nil) 0x7f5af81a5d64 (nil)
)                                             = 0
Password: 
```
The binary will create a new binary with `memfd_create`, and load it with `dlopen`! I didn't even know this function at the time. Well, let's "overwrite" it to use open instead in `fuck.c`:

```c
int memfd_create(const char *name, unsigned int flags)
{
    return open("/tmp/blah", flags | O_CREAT, 0755);
}
```

This will allow to dump this new binary of 16936 bytes, in `/tmp/blah`. A quick look with radare2 didn't help much, so this time I used [ghidra](https://ghidra-sre.org):

```c
  undefined8 in_RSI;
  long in_FS_OFFSET;
  ulong local_20;
  int *local_18;
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  local_18 = (int *)(*(code *)ctx)(ctx,in_RSI,ctx);
  puts("Password: ");
  __isoc99_scanf(&DAT_00102105,&local_20);
  if (((long)((long)(int)(uint)in_RSI * (local_20 ^ (uint)in_RSI & 0x42)) / (long)local_18[1] ==
       (long)*local_18) && ((local_20 & 0xffff) == 0xfb4c)) {
    printf("Yes !\nHere is your flag: ");
    print_flag(local_20);
  }
  else {
    puts("Nope, try again...");
  }
```

We're near it. Unfortunately, breaking this code won't help you getting the flag, as it will be computed with the given input. The only way is to get the expected value. We only need to find out the local_18 & in_RSI values, as local_20 is the input (which must be an integer), and make sure it matches the condition. Let's use gdb & find out the values. ghidra already told us where those values were in memory, so it will be mostly easy:

```c
(gdb) set env LD_PRELOAD=/home/mycroft/dev/dghack/dharma/fuck.so
(gdb) b init
Function "init" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y
.. snip ...
0x00007ffff7fbfc80 in init () from /proc/1316873/fd/3
(gdb) disas
Dump of assembler code for function init:
=> 0x00007ffff7fbfc80 <+0>:     endbr64
   0x00007ffff7fbfc84 <+4>:     push   rbp
   0x00007ffff7fbfc85 <+5>:     mov    rbp,rsp
   0x00007ffff7fbfc88 <+8>:     sub    rsp,0x30
   0x00007ffff7fbfc8c <+12>:    mov    QWORD PTR [rbp-0x28],rdi
   0x00007ffff7fbfc90 <+16>:    mov    DWORD PTR [rbp-0x2c],esi
   0x00007ffff7fbfc93 <+19>:    mov    rax,QWORD PTR fs:0x28
   0x00007ffff7fbfc9c <+28>:    mov    QWORD PTR [rbp-0x8],rax
   0x00007ffff7fbfca0 <+32>:    xor    eax,eax
   0x00007ffff7fbfca2 <+34>:    mov    rdx,QWORD PTR [rbp-0x28]
   0x00007ffff7fbfca6 <+38>:    mov    eax,0x0
   0x00007ffff7fbfcab <+43>:    call   rdx
   0x00007ffff7fbfcad <+45>:    mov    QWORD PTR [rbp-0x10],rax
   0x00007ffff7fbfcb1 <+49>:    lea    rdi,[rip+0x451]        # 0x7ffff7fc0109
   0x00007ffff7fbfcb8 <+56>:    call   0x7ffff7fbf140 <puts@plt>
(gdb) b puts
Breakpoint 2 at 0x7ffff7e0e4d0
(gdb) c
Continuing.

Breakpoint 2, 0x00007ffff7e0e4d0 in puts () from /lib64/libc.so.6

(gdb) x/2wx *(long *)($rbp-0x10)
0x55555555c270: 0x6e334f32      0x68625361

(gdb) x/wx $rbp-0x2c
0x7fffffffcd24: 0x00000c80
```

We have the values. Still need to find the `local_20` value.  It is just a simple equation: local_18[0] * local_18[1] / in_RS1 = local_20 ^ (uint)in_RSI & 0x42, and local_20 must finished by 0xfb4c:

```python
>>> l = int((0x68625361 * 0x6e334f32) / 0x0c80)
>>> l + (0xfb4c - l & 0xffff)
1011829597993804
```

and, finally, in the shell:

```sh
$ ./dharma
Password: 
1011829597993804
Yes !
Here is your flag: b2d4837830f4853656e993e4561670d0e461471acbb0e2f7183b514b04440d69
```

# What did I learn?

I already knew the 0xCC & the binary encryption technics. I didn't know about ghidra or radare2 tools. Very impressive what you can do with those today.
