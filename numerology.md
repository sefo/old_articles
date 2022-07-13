# Numerology

*Published 2004-10-16*

Wayback machine: https://web.archive.org/web/20050222145608/http://www.osix.net/modules/article/?id=593

> Ever wondered how numerologists could say you are an intelligent, impatient or a strong familly person?
> You will find the secrets of numerology in this article.entered and make some calculations.

##  Introduction:

I found an interesting shareware on \[Download.com\] called optName.
One of its features is to calculate a number from any name you enter.

If you look at \[optName's site\], you will understand that from each name, you can calculate a number from 1 to 9 or two additional (special) numbers 11 and 22. Each number has its unique definition. (1 is a masculine number...etc.)

## Reversing the process

The source code of this packed Delphi 3 shareware is of course not available, and I'm sure you don't want to read through loads of numerology web sites.
Hopefully you have your computer and your favorite debugger. (I did it with Olly)

The shareware is packed but you can find the original EOP with the option:

`debugging options/sfx/trace real entry bytewise`

Now that we have the real EOP (00474F18) we can start our research.

**Note:**
I work on the original shareware unpacked in memory.
It would have taken too long to dump it and rebuild the import table.

The first thing to do is to look at the string references. (View/references)
We easily find the descriptions of every numbers (1-9 and 11 and 22) at address 0043B519.
Double clicking on this line leads you to this address in the debugger.
There we find directly where the program displays the results of the calculation.
Following is a sample of the code when your number is 9.

```
0043B5F7 MOV EAX,DWORD PTR SS:\[EBP-4\] ;Case 9 of switch 0043B49E
0043B5FA MOV EAX,DWORD PTR DS:\[EAX+218\]
0043B600 MOV EAX,DWORD PTR DS:\[EAX+130\]
0043B606 MOV EDX,optName.0043C5FC ;ASCII "This number is..."
0043B60B MOV ECX,DWORD PTR DS:\[EAX\]
0043B60D CALL DWORD PTR DS:\[ECX+2C\]
0043B610 JMP SHORT optName.0043B666
```

We will have to go up in the listing to find the calculation function.
It is obviously above `0043B49` (switch).

At one point (around 0043B362) you find a list of Char and their hexadecimal values.
We are on the right track...
We finally find the switch() at address 0043B2DE.

**Note:**
The shareware automatically puts the first letter upper case and the rest lower case.

```
0043B2DE MOV EAX,optName.00490D75 ;EAX='Sefo'
0043B2E3 /XOR ECX,ECX ;initialize ECX (0)
0043B2E5 |MOV CL,BYTE PTR DS:\[EAX\] ;CL='S' (53)
0043B2E7 |ADD ECX,-41 ;Switch(cases 41-7A)
0043B2EA |CMP ECX,39 ;is it a letter?
0043B2ED |JA optName.0043B38B ;jump if invalid
0043B2F3 |MOV CL,BYTE PTR DS:\[ECX+43B300\];index of the current letter of the name
0043B2F9 |JMP DWORD PTR DS:\[ECX\*4+43B33A\];jump if valid letter
```

It is the begining of the calculation process.
EAX contains our name and this function will process each letter.
If one character in the name doesn't correspond to a letter from `41-7A (A-Za-z)`, you go to the default case 'Invalid name' at `0043B38B`. If everything went well you go to `0043B362`.

```
0043B362 |INC EBX ;Cases 41,4A,53,61,6A,73
0043B363 |JMP SHORT optName.0043B38B
0043B365 |ADD BL,2 ;Cases 42,4B,54,62,6B,74
0043B368 |JMP SHORT optName.0043B38B
0043B36A |ADD BL,3 ;Cases 43,4C,55,63,6C,75
0043B36D |JMP SHORT optName.0043B38B
0043B36F |ADD BL,4 ;Cases 44,4D,56,64,6D,76
0043B372 |JMP SHORT optName.0043B38B
0043B374 |ADD BL,5 ;Cases 45,4E,57,65,6E,77
0043B377 |JMP SHORT optName.0043B38B
0043B379 |ADD BL,6 ;Cases 46,4F,58,66,6F,78
0043B37C |JMP SHORT optName.0043B38B
0043B37E |ADD BL,7 ;Cases 47,50,59,67,70,79
0043B381 |JMP SHORT optName.0043B38B
0043B383 |ADD BL,8 ;Cases 48,51,5A,68,71,7A
0043B386 |JMP SHORT optName.0043B38B
0043B388 |ADD BL,9 ;Cases 49,52,69,72
0043B38B |INC EAX ;Default case
0043B38C |DEC EDX
0043B38D JNZ optName.0043B2E3
0043B393 JMP SHORT OPTNAME.0043B3B4 ;when processed all the letters, jump
```

Very well! We are deep inside the function.
We can identify a sort of matrix with letters and the action to do in each case:

```
A J S a j s -> EBX+=1
B K T b k t -> EBX+=2
C L U c l u -> EBX+=3
D M V d m v -> EBX+=4
E N W e n w -> EBX+=5
F O X f o x -> EBX+=6
G P Y g p y -> EBX+=7
H Q Z h q z -> EBX+=8
I R i r -> EBX+=9
```

For example, if one letter of our name is 'X', we will have to add 6 to our 'magic number' (EBX).
Thus, 'Sefo' will give the magic number:

```
53/65/66/6F
EBX=1+5+6+6=12 (18 in decimal)
```

Then we jump to `0043B3B4`:

```
0043B3B4 CMP BL,9
0043B3B7 |JBE SHORT OPTNAME.0043B3C3;if EBX-9<=0
0043B3B9 |CMP BL,0B
0043B3BC |JE SHORT OPTNAME.0043B3C3 ;if EBX-0B=0
0043B3BE |CMP BL,16
0043B3C1 JNZ SHORT OPTNAME.0043B395;if EBX-16!=0
```

In this case we will go to `0043B395`.

```
0043B395 /XOR EAX,EAX
0043B397 |MOV AL,BL ;put our number in EAX
0043B399 |MOV ECX,0A ;put 10 in ECX
0043B39E |CDQ
0043B39F |IDIV ECX ;EAX/=ECX -> EAX=12/0A=1
0043B3A1 |MOV ECX,EAX ;ECX=1
0043B3A3 |MOV EAX,ECX ;EAX=1
0043B3A5 |ADD EAX,EAX ;EAX=2
0043B3A7 |LEA EAX,DWORD PTR DS:\[EAX+EAX\*4\] ;EAX=0A
0043B3AA |MOV EDX,EBX ;EDX=12
0043B3AC |SUB DL,AL ;EDX=12-12=0
0043B3AE |PUSH EDX
0043B3AF |POP EAX
0043B3B0 |ADD AL,CL ;EAX=8+1=9
0043B3B2 |MOV EBX,EAX ;EBX=EAX
```

Let's analyze this step by step:

- The program puts 12 (the previous calculated number) in EAX.
- Initializes another variable ECX to 0A.
- Divides EAX and ECX and returns only the Integer in EAX. (here 1)
- Initializes ECX and EAX with this integer.
- Adds EAX and ECX
- Puts `(EAX+EAX\*4)` in EAX.
- Put our number 12 in EDX.
- Substracts EDX-EAX
- Adds EAX and ECX
- Puts EAX in EBX
- EBX is our final magic number.

_Hence:_
```
x=(magicnum/0A)+((magicnum/0A)\*4)
Final Number = (magic number/0A)+(magic number-x)
```

Numerology only uses numbers 1-9, 11 and 22

- If your final number is 10, it's considered as 9
- If it is between 11 and 22, it's considered as 11
- If it is > 22 then it's considered as 22

From all of this, we can deduce that a child whose name, once calculated, gives the number 9
will be blessed with 'high intellectual and spiritual development'.
Since there's no feature like this in optName (should we request it?), you can code a simple
'elite-name' brute forcer to get all names with number=9

Here's a sample list of the names you should give to your future intelligent child: (=9)
- sefo :joy:
- Paris
- Daniel
- Devnull
- user

Even better is the number 22.
You can Obtain it with 'Osix' :joy:

## conclusion

Now that you know the secret way numerologists calculate your future, you can do your own
FREEware with lots of nice features. But make it Open Source!
