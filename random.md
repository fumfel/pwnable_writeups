## random - 1 pt ##

Flaga: `Mommy, I thought libc random is unpredictable...`

Kod źródłowy **random.c** :

```c
#include <stdio.h>

int main(){
	unsigned int random;
	random = rand();	// random value!

	unsigned int key=0;
	scanf("%d", &key);

	if( (key ^ random) == 0xdeadbeef ){
		printf("Good!\n");
		system("/bin/cat flag");
		return 0;
	}

	printf("Wrong, maybe you should try 2^32 cases.\n");
	return 0;
}
```

* Wymagane jest wprowadzenie takiej wartości aby ` wartość ^ rand() = 0xdeadbeef`
* Funkcja `rand()` uruchamiana jest bez uprzedniej inicjalizacji seedem - otwiera to znaną furtkę podczas generowania liczb pseudolosowych (w takim wypadku seed zawsze jest równy 1; resztę wyników da się przewidzieć)
* Potrzebny jest testowy program do wygenerowania pierwszej pseudolosowej liczby o seedzie 1 na serwerze:

```c
#include <stdio.h>

int main()
{
        printf("RAND VALUE: %d", rand());
        return 0;
}
```
* Po uruchomieniu wyświetli "losową" liczbę z seedu 1: `RAND VALUE: 1804289383`
* Do uzyskania flagi wystarczy wrzucić tą liczbę w binarkę z zadania:
```
random@ubuntu:~$ ./random 
3039230856
Good!
Mommy, I thought libc random is unpredictable...
```
