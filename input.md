## input - 4 pt ##

Flaga: `Mommy! I learned how to pass various input in Linux :)`

Kod źródłowy **input.c**:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
	printf("Welcome to pwnable.kr\n");
	printf("Let's see if you know how to give input to program\n");
	printf("Just give me correct inputs then you will get the flag :)\n");

	// argv
	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");	

	// stdio
	char buf[4];
	read(0, buf, 4);
	if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
	read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
	printf("Stage 2 clear!\n");
	
	// env
	if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
	printf("Stage 3 clear!\n");

	// file
	FILE* fp = fopen("\x0a", "r");
	if(!fp) return 0;
	if( fread(buf, 4, 1, fp)!=1 ) return 0;
	if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
	fclose(fp);
	printf("Stage 4 clear!\n");	

	// network
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

	// here's your flag
	system("/bin/cat flag");	
	return 0;
}
```
* Program wymaga od nas przekazania inputu (zgodnie z nazwą zadania ;-) ) za pomocą różnych mechanizmów: argumenty, standardowe wejście i stderr, zmienne środowiskowe, pliki oraz sieć
* Pierwszy etap - przekazanie inputu za pomocą `argv`
  * Binarka wymaga od nas podania 100 argumentów
  * Dwa argumenty muszą zawierać odpowiednie dane, konkretnie 65 i 66 (kod ASCII dla liter "A" oraz "B"):
    * Argument 65 - `\x00`
    * Argument 66 - `\x20\x0a\x0d`
* Drugi etap - standardowe wejście i stderr
  * STDIN - wartość `\x00\x0a\x00\xff`
  * STDERR - wartość `\x00\x0a\x02\xff`
  * Pisząc exploita musimy pamiętać aby przekazać te wartości za pomocą pipe'ów - forkujemy proces i wrzucamy jednym pipe'm do stdin i jednym do stderr
* Trzeci etap - zmienne środowiskowe
  * Przekazanie zmiennej o nazwie w postaci szesnastkowej `\xde\xad\xbe\xef` i wartości `\xca\xfe\xba\xbe`
  * Tutaj oczywistym wyborem jest parametr env w funkcji `execve()`
* Czwarty etap - plik
  * IMHO najłatwiejszy stage - wystarczy dowolną metodą utworzyć plik binarny zawierający cztery bajty NULL: `\x00\x00\x00\x00`
* Etap piąty - sieć
  * Wymagane jest stworzenie serwera nasłuchującego lokalnie i wysłanie tam czterech bajtów: `\xde\xad\xbe\xef`
* Lekko przerobiony pod kątem czytelności exploit z https://gist.github.com/cubarco/cd96eca5e3940c7a3fe4 : 

```python
#!/usr/bin/env python
# coding=utf8

import os
import socket
import time 

port = 13337
r_stdin, w_stdin = os.pipe()
r_stderr, w_stderr = os.pipe()

os.system('ln -s /home/input/flag ./')
with open('\n', 'w') as f:
    f.write('\x00\x00\x00\x00')

pid = os.fork()

if pid == 0:
    os.close(w_stdin)
    os.close(w_stderr)
    
    os.dup2(r_stdin, 0)
    os.dup2(r_stderr, 2)

    os.putenv('\xde\xad\xbe\xef', '\xca\xfe\xba\xbe')
    os.execv('/home/input/input', ['input'] + ['A'] * 64 + [''] + ['\x20\x0a\x0d'] + [str(port)] + ['A']* 32)
else:
    os.close(r_stdin)
    os.close(r_stderr)

    wf0 = os.fdopen(w_stdin, 'w')
    wf2 = os.fdopen(w_stderr, 'w')
    
    wf0.write('\x00\x0a\x00\xff')
    wf2.write('\x00\x0a\x02\xff')

    wf0.close()
    wf2.close()
    
    client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    time.sleep(1)

    client.connect(('localhost', port))
    client.send('\xde\xad\xbe\xef')
    client.close()
```
