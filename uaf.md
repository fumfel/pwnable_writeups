## uaf - 8 pt ##

Flaga: `flag`

Kod źródłowy **uaf.c**:

```cpp
#include <fcntl.h>
#include <iostream> 
#include <cstring>
#include <cstdlib>
#include <unistd.h>
using namespace std;

class Human{
private:
	virtual void give_shell(){
		system("/bin/sh");
	}
protected:
	int age;
	string name;
public:
	virtual void introduce(){
		cout << "My name is " << name << endl;
		cout << "I am " << age << " years old" << endl;
	}
};

class Man: public Human{
public:
	Man(string name, int age){
		this->name = name;
		this->age = age;
        }
        virtual void introduce(){
		Human::introduce();
                cout << "I am a nice guy!" << endl;
        }
};

class Woman: public Human{
public:
        Woman(string name, int age){
                this->name = name;
                this->age = age;
        }
        virtual void introduce(){
                Human::introduce();
                cout << "I am a cute girl!" << endl;
        }
};

int main(int argc, char* argv[]){
	Human* m = new Man("Jack", 25);
	Human* w = new Woman("Jill", 21);

	size_t len;
	char* data;
	unsigned int op;
	while(1){
		cout << "1. use\n2. after\n3. free\n";
		cin >> op;

		switch(op){
			case 1:
				m->introduce();
				w->introduce();
				break;
			case 2:
				len = atoi(argv[1]);
				data = new char[len];
				read(open(argv[2], O_RDONLY), data, len);
				cout << "your data is allocated" << endl;
				break;
			case 3:
				delete m;
				delete w;
				break;
			default:
				break;
		}
	}

	return 0;	
}
```
* Output z checksec:
```
uaf@ubuntu:~$ checksec uaf
[*] '/home/uaf/uaf'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE
```
* Program na początku alokuje pamięć na (tworzy) dwa obiekty: `man` i `woman`
* Za pomocą opcji w menu, możemy wykonać operacje niezbędne do zmiany przebiegu wykonania kodu:
  * `1. use` - wypisanie na konsolę informacji o obiekcie
  * `2. after` - pobranie danych z `arg[1]` i `arg[2]` i wrzucenie ich do pamięci
  * `3. free` - zwolnienie obiektów utworzonych na początku pracy
* Ogólny schemat exploitacji podatności UAF zamyka się w tym: należy zwolnić prawidłowy obiekt i w jego miejsce "podsunąć" zmodyfikowany (o takim samym rozmiarze), tak aby uzyskać wykonanie kodu
* Warto zwrócić uwagę na rozmiar alokacji pamięci dla obiektów, który w obydwu przypadkach wynosi `0x18 (24)` - przyda się to w następnych krokach

```asm
   0x0000000000400ef7 <+51>:	lea    r12,[rbp-0x50]
   0x0000000000400efb <+55>:	mov    edi,0x18
   0x0000000000400f00 <+60>:	call   0x400d90 <_Znwm@plt>
```
* Warunkiem pomyślnego uzyskania flagi jest poznanie struktury pamięci obiektu, obsługiwanego przez opcję `1. use`. Do tego przyda się zdisasemblowany kod obsługujący tego brancha:
```asm
   0x0000000000400fcd <+265>:	mov    rax,QWORD PTR [rbp-0x38]
   0x0000000000400fd1 <+269>:	mov    rax,QWORD PTR [rax]
   0x0000000000400fd4 <+272>:	add    rax,0x8
   0x0000000000400fd8 <+276>:	mov    rdx,QWORD PTR [rax]
   0x0000000000400fdb <+279>:	mov    rax,QWORD PTR [rbp-0x38]
   0x0000000000400fdf <+283>:	mov    rdi,rax
   0x0000000000400fe2 <+286>:	call   rdx
   0x0000000000400fe4 <+288>:	mov    rax,QWORD PTR [rbp-0x30]
   0x0000000000400fe8 <+292>:	mov    rax,QWORD PTR [rax]
   0x0000000000400feb <+295>:	add    rax,0x8
   0x0000000000400fef <+299>:	mov    rdx,QWORD PTR [rax]
   0x0000000000400ff2 <+302>:	mov    rax,QWORD PTR [rbp-0x30]
   0x0000000000400ff6 <+306>:	mov    rdi,rax
   0x0000000000400ff9 <+309>:	call   rdx
   0x0000000000400ffb <+311>:	jmp    0x4010a9 <main+485>
```
