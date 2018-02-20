## fd - 1 pt ##

Flaga: `mommy! I think I know what a file descriptor is!!`

Kod źródłowy **fd.c**:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){
		printf("pass argv[1] a number\n");
		return 0;
	}
	int fd = atoi( argv[1] ) - 0x1234;
	int len = 0;
	len = read(fd, buf, 32);
	if(!strcmp("LETMEWIN\n", buf)){
		printf("good job :)\n");
		system("/bin/cat flag");
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}
```
* Program prosi nas o podanie w argumencie wartości deskryptora pliku
* Po przekazaniu argumentu następuje odjęcie od niego wartości heksadecymalnie 0x1234 (dziesiętnie 4660), a następnie odczyt danych z pliku o danym deskryptorze
* W przypadku, kiedy odczytany bufor zawiera string `LETMEWIN` następuje wyświetlenie flagi
* Dla przypomnienia: standardowe wartości deskryptorów w Linux'ie: `1 - STDIN, 2 - STDOUT, 3 - STDERR`
* Aby uzyskać flagę należy w argumencie programu przekazać liczbę `4661 = 4660 + 1 (STDIN)` i wpisać `LETMEWIN`

```
fd@ubuntu:~$ ./fd 4661
LETMEWIN
good job :)
mommy! I think I know what a file descriptor is!!
```
