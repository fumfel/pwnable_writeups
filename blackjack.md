## blackjack - 1 pt ##

Flaga: `YaY_I_AM_A_MILLIONARE_LOL`

Kod źródłowy dostępny pod adresem: http://cboard.cprogramming.com/c-programming/114023-simple-blackjack-program.html

* Problemem jest subtelny błąd logiczny (dla mnie początkowo trudny do wyłapania) w funkcji `betting()`:
```c
int betting() //Asks user amount to bet
{
 printf("\n\nEnter Bet: $");
 scanf("%d", &bet);

 if (bet > cash) //If player tries to bet more money than player has
 {
		printf("\nYou cannot bet more money than you have.");
		printf("\nEnter Bet: ");
    scanf("%d", &bet);
    return bet;
 }
 else return bet;
} // End Function
```
* Przeznaczeniem tej funkcji jest pobieranie od gracza wartości zakładu do postawienia - problem powstaje w momencie kiedy użytkownik podaje ujemną wartość, na to niestety nie ma checka.
* Wartość przekazana w tej funkcji jest używana w funkcji `play()`:
```c
if(p > 21) //If player total is over 21, loss
{
     printf("\nWoah Buddy, You Went WAY over.\n");
     loss = loss + 1;
     cash = cash - bet;
     printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
     dealer_total=0;
     askover();
}
```
* Podstawowa znajomość matematyki wystarczy do tego, aby "wykombinować", że **dwa razy minus daje plus** i w konsekwencji zasila nasze konto u krupiera:
```c
Cash: $500
-------
|C    |
|  5  |
|    C|
-------

Your Total is 5

The Dealer Has a Total of 10

Enter Bet: $-100000000


Would You Like to Hit or Stay?
Please Enter H to Hit or S to Stay.
S

You Have Chosen to Stay at 5. Wise Decision!

The Dealer Has a Total of 13
The Dealer Has a Total of 16
The Dealer Has a Total of 19
Dealer Has the Better Hand. You Lose.

You have 0 Wins and 1 Losses. Awesome!

Would You Like To Play Again?
Please Enter Y for Yes or N for No
y

YaY_I_AM_A_MILLIONARE_LOL


Cash: $100000500
-------
|C    |
|  A  |
|    C|
-------

Your Total is 11

The Dealer Has a Total of 11
```
