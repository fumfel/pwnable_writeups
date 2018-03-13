## lotto - 2 pt ##

Flaga: `sorry mom... I FORGOT to check duplicate numbers... :(`

Kod źródłowy **lotto.c**:


```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>

unsigned char submit[6];

void play(){
	
	int i;
	printf("Submit your 6 lotto bytes : ");
	fflush(stdout);

	int r;
	r = read(0, submit, 6);

	printf("Lotto Start!\n");
	//sleep(1);

	// generate lotto numbers
	int fd = open("/dev/urandom", O_RDONLY);
	if(fd==-1){
		printf("error. tell admin\n");
		exit(-1);
	}
	unsigned char lotto[6];
	if(read(fd, lotto, 6) != 6){
		printf("error2. tell admin\n");
		exit(-1);
	}
	for(i=0; i<6; i++){
		lotto[i] = (lotto[i] % 45) + 1;		// 1 ~ 45
	}
	close(fd);
	
	// calculate lotto score
	int match = 0, j = 0;
	for(i=0; i<6; i++){
		for(j=0; j<6; j++){
			if(lotto[i] == submit[j]){
				match++;
			}
		}
	}

	// win!
	if(match == 6){
		system("/bin/cat flag");
	}
	else{
		printf("bad luck...\n");
	}

}

void help(){
	printf("- nLotto Rule -\n");
	printf("nlotto is consisted with 6 random natural numbers less than 46\n");
	printf("your goal is to match lotto numbers as many as you can\n");
	printf("if you win lottery for *1st place*, you will get reward\n");
	printf("for more details, follow the link below\n");
	printf("http://www.nlotto.co.kr/counsel.do?method=playerGuide#buying_guide01\n\n");
	printf("mathematical chance to win this game is known to be 1/8145060.\n");
}

int main(int argc, char* argv[]){

	// menu
	unsigned int menu;

	while(1){

		printf("- Select Menu -\n");
		printf("1. Play Lotto\n");
		printf("2. Help\n");
		printf("3. Exit\n");

		scanf("%d", &menu);

		switch(menu){
			case 1:
				play();
				break;
			case 2:
				help();
				break;
			case 3:
				printf("bye\n");
				return 0;
			default:
				printf("invalid menu\n");
				break;
		}
	}
	return 0;
}
```
* Pierwszym krokiem jest zwrócenie uwagi na tablicę zawierającą input od usera: `unsigned char submit[6];` - zawiera znaki drukowalne z zakresu ASCII
* Czytając help do zadania dowiadujemy się, że rozwiązanie zawiera cyfry od 1 do 46: `lotto is consisted with 6 random natural numbers less than 46`
* Sprawdzając tablicę ASCII w zakresie rozwiązań lotto ze znaków drukowalnych na konsolę mamy: `!"#$%&'()*+,-`
* Clue zadania opiera się o subtelną podatność w pętli sprawdzającej rozwiązanie:
```c
// calculate lotto score
int match = 0, j = 0;
for(i=0; i<6; i++){
	for(j=0; j<6; j++){
		if(lotto[i] == submit[j]){
			match++;
		}
	}
}
```
* W przypadku "trafienia" jednego poprawnego bajtu i podaniu tego bajtu w inpucie sześć razy jesteśmy w stanie "manipulować" wartością zmiennej `match`, która determinuje wygraną.
* Jeżeli jest to, mówiąc kolokwialnie nieco "zamotane": pętla przechodzi po każdej wartości i nie sprawdza zduplikowanych bajtów
* W efekcie podanie sześciu takich samych znaków z zakresu `!"#$%&'()*+,-` i "trafienie" spowoduje wyświetlenie flagi:
```
Submit your 6 lotto bytes : ######
Lotto Start!
sorry mom... I FORGOT to check duplicate numbers... :(
- Select Menu -
1. Play Lotto
2. Help
3. Exit
```
