## memcpy - 10 pt ##

Flaga: `flag`

Readme: `the compiled binary of "memcpy.c" source code (with real flag) will be executed under memcpy_pwn privilege if you connect to port 9022.
execute the binary by connecting to daemon(nc 0 9022).`

Kod źródłowy **memcpy.c** (http://pwnable.kr/bin/memcpy.c):
```c
// compiled with : gcc -o memcpy memcpy.c -m32 -lm
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <sys/mman.h>
#include <math.h>

unsigned long long rdtsc(){
        asm("rdtsc");
}

char* slow_memcpy(char* dest, const char* src, size_t len){
	int i;
	for (i=0; i<len; i++) {
		dest[i] = src[i];
	}
	return dest;
}

char* fast_memcpy(char* dest, const char* src, size_t len){
	size_t i;
	// 64-byte block fast copy
	if(len >= 64){
		i = len / 64;
		len &= (64-1);
		while(i-- > 0){
			__asm__ __volatile__ (
			"movdqa (%0), %%xmm0\n"
			"movdqa 16(%0), %%xmm1\n"
			"movdqa 32(%0), %%xmm2\n"
			"movdqa 48(%0), %%xmm3\n"
			"movntps %%xmm0, (%1)\n"
			"movntps %%xmm1, 16(%1)\n"
			"movntps %%xmm2, 32(%1)\n"
			"movntps %%xmm3, 48(%1)\n"
			::"r"(src),"r"(dest):"memory");
			dest += 64;
			src += 64;
		}
	}

	// byte-to-byte slow copy
	if(len) slow_memcpy(dest, src, len);
	return dest;
}

int main(void){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Hey, I have a boring assignment for CS class.. :(\n");
	printf("The assignment is simple.\n");

	printf("-----------------------------------------------------\n");
	printf("- What is the best implementation of memcpy?        -\n");
	printf("- 1. implement your own slow/fast version of memcpy -\n");
	printf("- 2. compare them with various size of data         -\n");
	printf("- 3. conclude your experiment and submit report     -\n");
	printf("-----------------------------------------------------\n");

	printf("This time, just help me out with my experiment and get flag\n");
	printf("No fancy hacking, I promise :D\n");

	unsigned long long t1, t2;
	int e;
	char* src;
	char* dest;
	unsigned int low, high;
	unsigned int size;
	// allocate memory
	char* cache1 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	char* cache2 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	src = mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

	size_t sizes[10];
	int i=0;

	// setup experiment parameters
	for(e=4; e<14; e++){	// 2^13 = 8K
		low = pow(2,e-1);
		high = pow(2,e);
		printf("specify the memcpy amount between %d ~ %d : ", low, high);
		scanf("%d", &size);
		if( size < low || size > high ){
			printf("don't mess with the experiment.\n");
			exit(0);
		}
		sizes[i++] = size;
	}

	sleep(1);
	printf("ok, lets run the experiment with your configuration\n");
	sleep(1);

	// run experiment
	for(i=0; i<10; i++){
		size = sizes[i];
		printf("experiment %d : memcpy with buffer size %d\n", i+1, size);
		dest = malloc( size );

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		slow_memcpy(dest, src, size);		// byte-to-byte memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for slow_memcpy : %llu\n", t2-t1);

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		fast_memcpy(dest, src, size);		// block-to-block memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for fast_memcpy : %llu\n", t2-t1);
		printf("\n");
	}

	printf("thanks for helping my experiment!\n");
	printf("flag : ----- erased in this source code -----\n");
	return 0;
}

```
* Problemem zadania jest to, że przebieg programu kończy się mniej więcej w jego środku (`experiment 4`), uniemożliwiając odczytanie flagi:
```
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 12
specify the memcpy amount between 16 ~ 32 : 30
specify the memcpy amount between 32 ~ 64 : 55
specify the memcpy amount between 64 ~ 128 : 120
specify the memcpy amount between 128 ~ 256 : 145
specify the memcpy amount between 256 ~ 512 : 266
specify the memcpy amount between 512 ~ 1024 : 999
specify the memcpy amount between 1024 ~ 2048 : 2000
specify the memcpy amount between 2048 ~ 4096 : 4001
specify the memcpy amount between 4096 ~ 8192 : 8101
ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 12
ellapsed CPU cycles for slow_memcpy : 1512
ellapsed CPU cycles for fast_memcpy : 582

experiment 2 : memcpy with buffer size 30
ellapsed CPU cycles for slow_memcpy : 528
ellapsed CPU cycles for fast_memcpy : 735

experiment 3 : memcpy with buffer size 55
ellapsed CPU cycles for slow_memcpy : 861
ellapsed CPU cycles for fast_memcpy : 1050

experiment 4 : memcpy with buffer size 120
ellapsed CPU cycles for slow_memcpy : 1818
```
* Niestety poprzez netcata nie da się zdebugować problemu - stderr nie jest przekierowany do socketu obsługującego zadanie
* Mając kod źródłowy możemy skompilować zadanie u siebie i zobaczyć jaka jest przyczyna problemu (w przypadku korzystania z platformy `amd64` i Ubuntu, przed kompilacją należy zainstalować paczkę `libc6-dev-i386` jeżeli kompilujemy poleceniem podanym w kodzie).
* Program po uruchomieniu i wczytaniu parametrów kończy się SIGSEGV:
```
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 15
specify the memcpy amount between 16 ~ 32 : 30
specify the memcpy amount between 32 ~ 64 : 60
specify the memcpy amount between 64 ~ 128 : 100
specify the memcpy amount between 128 ~ 256 : 200
specify the memcpy amount between 256 ~ 512 : 400
specify the memcpy amount between 512 ~ 1024 : 1000
specify the memcpy amount between 1024 ~ 2048 : 2000
specify the memcpy amount between 2048 ~ 4096 : 4000
specify the memcpy amount between 4096 ~ 8192 : 8000
ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 15
ellapsed CPU cycles for slow_memcpy : 9579
ellapsed CPU cycles for fast_memcpy : 1238

experiment 2 : memcpy with buffer size 30
ellapsed CPU cycles for slow_memcpy : 871
ellapsed CPU cycles for fast_memcpy : 1033

experiment 3 : memcpy with buffer size 60
ellapsed CPU cycles for slow_memcpy : 1565
ellapsed CPU cycles for fast_memcpy : 1521

experiment 4 : memcpy with buffer size 100
ellapsed CPU cycles for slow_memcpy : 2275
ellapsed CPU cycles for fast_memcpy : 1735

experiment 5 : memcpy with buffer size 200
ellapsed CPU cycles for slow_memcpy : 4263
Segmentation fault (core dumped)
```
* Przy pomocy `valgrind` można szybko sprawdzić co dokładnie się stało:
```
==19287== Memcheck, a memory error detector
==19287== Copyright (C) 2002-2015, and GNU GPL'd, by Julian Seward et al.
==19287== Using Valgrind-3.11.0 and LibVEX; rerun with -h for copyright info
==19287== Command: ./memcpy
==19287== 
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 15
specify the memcpy amount between 16 ~ 32 : 30
specify the memcpy amount between 32 ~ 64 : 60
specify the memcpy amount between 64 ~ 128 : 120
specify the memcpy amount between 128 ~ 256 : 240
specify the memcpy amount between 256 ~ 512 : 480
specify the memcpy amount between 512 ~ 1024 : 960
specify the memcpy amount between 1024 ~ 2048 : 1920
specify the memcpy amount between 2048 ~ 4096 : 3840
specify the memcpy amount between 4096 ~ 8192 : 7680
ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 15
ellapsed CPU cycles for slow_memcpy : 1934385
ellapsed CPU cycles for fast_memcpy : 1598857

experiment 2 : memcpy with buffer size 30
ellapsed CPU cycles for slow_memcpy : 13243
ellapsed CPU cycles for fast_memcpy : 6961

experiment 3 : memcpy with buffer size 60
ellapsed CPU cycles for slow_memcpy : 7202
ellapsed CPU cycles for fast_memcpy : 8040

experiment 4 : memcpy with buffer size 120
ellapsed CPU cycles for slow_memcpy : 11582
==19287== 
==19287== Process terminating with default action of signal 11 (SIGSEGV)
==19287==  General Protection Fault
==19287==    at 0x80487CC: fast_memcpy (memcpy.c:29)
==19287==    by 0x8048B8A: main (memcpy.c:115)
==19287== 
==19287== HEAP SUMMARY:
==19287==     in use at exit: 225 bytes in 4 blocks
==19287==   total heap usage: 5 allocs, 1 frees, 1,249 bytes allocated
==19287== 
==19287== LEAK SUMMARY:
==19287==    definitely lost: 105 bytes in 3 blocks
==19287==    indirectly lost: 0 bytes in 0 blocks
==19287==      possibly lost: 0 bytes in 0 blocks
==19287==    still reachable: 120 bytes in 1 blocks
==19287==         suppressed: 0 bytes in 0 blocks
==19287== Rerun with --leak-check=full to see details of leaked memory
==19287== 
==19287== For counts of detected and suppressed errors, rerun with: -v
==19287== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```
* Problem znajduje się w 29 linijce kodu czyli we wstawce assemblera - funkcja `fast_memcpy()`:
```c
char* fast_memcpy(char* dest, const char* src, size_t len){
	size_t i;
	// 64-byte block fast copy
	if(len >= 64){
		i = len / 64;
		len &= (64-1);
		while(i-- > 0){
			__asm__ __volatile__ (
			"movdqa (%0), %%xmm0\n"
			"movdqa 16(%0), %%xmm1\n"
			"movdqa 32(%0), %%xmm2\n"
			"movdqa 48(%0), %%xmm3\n"
			"movntps %%xmm0, (%1)\n"
			"movntps %%xmm1, 16(%1)\n"
			"movntps %%xmm2, 32(%1)\n"
			"movntps %%xmm3, 48(%1)\n"
			::"r"(src),"r"(dest):"memory");
			dest += 64;
			src += 64;
		}
	}

	// byte-to-byte slow copy
	if(len) slow_memcpy(dest, src, len);
	return dest;
}
```

