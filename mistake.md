## mistake - 1 pt ##

Flaga: `Mommy, the operator priority always confuses me :(`

Opis zadania (ma znaczenie w tym wypadku) : 
```We all make mistakes, let's move on.
(don't take this too seriously, no fancy hacking skill is required at all)

This task is based on real event
Thanks to dhmonkey

hint : operator priority`
```

Kod źródłowy **mistake.c**:

```c
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
	int i;
	for(i=0; i<len; i++){
		s[i] ^= XORKEY;
	}
}

int main(int argc, char* argv[]){
	
	int fd;
	if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
	}

	printf("do not bruteforce...\n");
	sleep(time(0)%20);

	char pw_buf[PW_LEN+1];
	int len;
	if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
		printf("read error\n");
		close(fd);
		return 0;		
	}

	char pw_buf2[PW_LEN+1];
	printf("input password : ");
	scanf("%10s", pw_buf2);

	// xor your input
	xor(pw_buf2, 10);

	if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
		printf("Password OK\n");
		system("/bin/cat flag\n");
	}
	else{
		printf("Wrong Password\n");
	}

	close(fd);
	return 0;
}

```
* Problem sygnalizowany w hincie zadania znajduje się w poniższym listingu kodu:
```c
if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
} 
```
* Operator `<` ma pierwszeństwo przed operatorem przypisania `=` -  http://en.cppreference.com/w/c/language/operator_precedence
* Powoduje to, że kod po skompilowaniu realizuje następującą logikę:
```c
if(fd = (open("/home/mistake/password",O_RDONLY,0400) < 0)){
		printf("can't open password %d\n", fd);
		return 0;
} 
```
* W/w kod zwróci wartość pierwszego wolnego deskryptora pliku, co jednocześnie spowoduje, że warunek w ifie nie będzie spełniony (0 wg. logiki Boole'a)
* Zmiennej fd zostanie przypisana wartość 0
* Przypomnienie: 0 jest deskryptorem STDIN, co spowoduje, że musimy sami wprowadzić hasło "z palca" a następnie poddać je operacji xor z 1, aby uzyskać flagę
* ` wartość xor 1 = wartość - 1`
* Wystarczy wprowadzić `10 x "1"` a później `10 x "0"`, aby uzyskać flagę:
```
mistake@ubuntu:~$ ./mistake 
do not bruteforce...
1111111111
input password : 0000000000
Password OK
Mommy, the operator priority always confuses me :(
```
