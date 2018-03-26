## asm - 6 pt ##

Flaga: `mock`

Kod źródłowy **asm.c**:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <seccomp.h>
#include <sys/prctl.h>
#include <fcntl.h>
#include <unistd.h>

#define LENGTH 128

void sandbox(){
	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
	if (ctx == NULL) {
		printf("seccomp error\n");
		exit(0);
	}

	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

	if (seccomp_load(ctx) < 0){
		seccomp_release(ctx);
		printf("seccomp error\n");
		exit(0);
	}
	seccomp_release(ctx);
}

char stub[] = "\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff";
unsigned char filter[256];
int main(int argc, char* argv[]){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Welcome to shellcoding practice challenge.\n");
	printf("In this challenge, you can run your x64 shellcode under SECCOMP sandbox.\n");
	printf("Try to make shellcode that spits flag using open()/read()/write() systemcalls only.\n");
	printf("If this does not challenge you. you should play 'asg' challenge :)\n");

	char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
	memset(sh, 0x90, 0x1000);
	memcpy(sh, stub, strlen(stub));
	
	int offset = sizeof(stub);
	printf("give me your x64 shellcode: ");
	read(0, sh+offset, 1000);

	alarm(10);
	chroot("/home/asm_pwn");	// you are in chroot jail. so you can't use symlink in /tmp
	sandbox();
	((void (*)(void))sh)();
	return 0;
}
```
* Output z checksec:
```
asm@ubuntu:~$ checksec asm
[*] '/home/asm/asm'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```
* Binarka posiada "sandbox'a" zbudowanego na SECCOMP - piaskownica filtruje syscalle i dopuszcza dla wątku jedynie operacje:
  * `SCMP_SYS(open)`
  * `SCMP_SYS(read)`
  * `SCMP_SYS(write)`
  * `SCMP_SYS(exit)`
  * `SCMP_SYS(exit_group)`
* Każda inna operacja zabija wątek, bez możliwości przechwycenia sygnału
* Bajty "shellcode" ze zmiennej `stub` na platformie x64 odpowiadają następującym instrukcjom:
```asm
   0:   48 31 c0                xor    rax,rax
   3:   48 31 db                xor    rbx,rbx
   6:   48 31 c9                xor    rcx,rcx
   9:   48 31 d2                xor    rdx,rdx
   c:   48 31 f6                xor    rsi,rsi
   f:   48 31 ff                xor    rdi,rdi
  12:   48 31 ed                xor    rbp,rbp
  15:   4d 31 c0                xor    r8,r8
  18:   4d 31 c9                xor    r9,r9
  1b:   4d 31 d2                xor    r10,r10
  1e:   4d 31 db                xor    r11,r11
  21:   4d 31 e4                xor    r12,r12
  24:   4d 31 ed                xor    r13,r13
  27:   4d 31 f6                xor    r14,r14
  2a:   4d 31 ff                xor    r15,r15
  2d:   20                      .byte 0x20
```
