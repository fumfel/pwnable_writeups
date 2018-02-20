## collision - 3 pt ##

Flaga: `daddy! I just managed to create a hash collision :)`

Kod źródłowy **col.c**:

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}

int main(int argc, char* argv[]){
	if(argc<2){
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20){
		printf("passcode length should be 20 bytes\n");
		return 0;
	}

	if(hashcode == check_password( argv[1] )){
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;
}
```

* Program przyjmuje 20 bajtową tablicę znaków, która w rzeczywistości jest pięcioma liczbami typu int32
* Tytułowa kolizja to odnalezienie sumy takich pięciu liczb, która łącznie da wynik `0x21DD09EC (568134124)`
* Najprostszym sposobem jest podzielenie liczby docelowej na 5, co w wyniku daje `0x6C5CEC8 i 4 reszty`
* Liczbą do zdobycia flagi jest: `4 * 0x6C5CEC8 + 0x6C5CECC`
* Oczywiście parametry przekazywane są do programu "odwrotnie" w notacji little-endian:

```
col@ubuntu:~$ ./col `python -c 'print "\xc8\xce\xc5\x06"*4 + "\xcc\xce\xc5\x06"'`
daddy! I just managed to create a hash collision :)
```
