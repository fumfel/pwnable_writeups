## coin1 - 6 pt ##

Flaga: `b1NaRy_S34rch1nG_1s_3asy_p3asy`

Opis zadania:

```
» nc pwnable.kr 9007

	---------------------------------------------------
	-              Shall we play a game?              -
	---------------------------------------------------
	
	You have given some gold coins in your hand
	however, there is one counterfeit coin among them
	counterfeit coin looks exactly same as real coin
	however, its weight is different from real one
	real coin weighs 10, counterfeit coin weighes 9
	help me to find the counterfeit coin with a scale
	if you find 100 counterfeit coins, you will get reward :)
	FYI, you have 30 seconds.
	
	- How to play - 
	1. you get a number of coins (N) and number of chances (C)
	2. then you specify a set of index numbers of coins to be weighed
	3. you get the weight information
	4. 2~3 repeats C time, then you give the answer
	
	- Example -
	[Server] N=4 C=2 	# find counterfeit among 4 coins with 2 trial
	[Client] 0 1 		# weigh first and second coin
	[Server] 20			# scale result : 20
	[Client] 3			# weigh fourth coin
	[Server] 10			# scale result : 10
	[Client] 2 			# counterfeit coin is third!
	[Server] Correct!

	- Ready? starting in 3 sec... -
	
N=677 C=10
```

* Od serwera otrzymujemy liczbę monet oznaczoną jako `N` i liczbę szans jako `C`
* Zadaniem jest wyszukanie podrobionych złotych monet w czasie 30 sekund za pomocą odpytywania serwer o wagę poszczególnych monet i ich porównanie
* Jest to klasyczny problem do wykorzystania wyszukiwania binarnego, w skrócie:
  * Dzielimy zbiór na dwie równe części
  * Sprawdzamy wybraną parę monet ze zbioru pod kątem wyszukiwanej cechy (mniejszej wagi)
  * Powtarzamy czynności do uzyskania wyniku
  * Bardzo obrazowy diagram można znaleźć w angielskim writeupie zadania: https://reboare.github.io/pwnablekr/random-coin1-bof.html
* Niestety, często serwer "nie wyrabia" i trzeba odpalać skrypty implementujące wyszukiwanie binarne z "wewnątrz" maszyny
