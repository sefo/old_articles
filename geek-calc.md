# Easy reversing: GeekCalc.exe

*Published 2004-12-18*

Wayback machine: https://web.archive.org/web/20050223024505/http://www.osix.net/modules/article/?id=633

> How to play with the windows calculator. An easy and entertaining reversing session.
> We are going to hide inside calc.exe a password. Each time a button is pressed, the calculator will check the numbers entered and make some calculations.

## Part 1: Goal to achieve

- Get the value of each button pressed
- Calculate a password from it
- Compare it with our own password
- Display a message

You will need two tools:
OllyDbg and any Hexadecimal Editor

__Where to start?__

The first thing to do is to find where the program takes the value of each button pressed. For this, we will bet on the API `GetDlgItem`.

To find where is called this API, open calc.exe in OllyDbg. (Make sure to make a bakup of calc.exe before any manipulation)
Now you can press [CTRL]+N to find what API are used. In the list that appears find GetDlgItem, right-click on it and choose:
"Set Breakpoint on Every Reference".

By pressing [ALT]+B you can find the list of breakpoints you set and see that GetDlgItem is largely used.
If you run calc.exe now (F9) it will immediately break on one of our breakpoints. So before running the exe, open the list of breakpoints ([ALT]+B), right-click on any line and choose "Disable all".
Now run by using F9, in the breakpoints list, right-click and choose "enable all" and finally in the calculator press for example the digit "6" to break here:

```
01002604 PUSH / ESI ;begining of the function that contains GetDlgItem
01002605 PUSH | EDI
01002606 PUSH | 193
0100260B PUSH | DWORD PTR SS:[ESP+10]
0100260F CALL | DWORD PTR DS:[<&USER32.GetDlgItem>] ; breakpoint here
```

Now that we know which GetDlgItem is called, note the address 0100260F somewhere and delete all breakpoints ([ALT]+B and press delete) to keep our list clean.
We could start here but when we take a look at EAX in the registers window (top-right corner) we see:

`0006F824 "unicode 6, "`

*NOTE:*
You will always have a "," in decimal mode, but not in hexadecimal.

Some operation has been already done and a `<space>` has been appended to our digit.
We have to find where this space has been added and "plug" our own code there.

To find where GetDlgItem has been called from, right-click on the begining of the function (01002604) and choose "Find reference to selected command".
Now Set a breakpoint on every command. If you deleted the previous breakpoints, you should now have 3 calls:

```
0100405C CALL
01004908 CALL
01004A5F CALL
```

Disable (but don't delete) these breakpoints, restart the program ([CTRL]+F2) and run (F9). Enable all breakpoints and finally press the digit "6" one more time to break here:

```
010048EF PUSH g5.01001364 ; StringToAdd = " "
010048F4 PUSH EAX
010048F5 CALL DWORD PTR DS:[<&KERNEL32.lstrcatW>]
010048FB LEA EAX,DWORD PTR SS:[EBP-200]
01004901 PUSH EAX
01004902 PUSH DWORD PTR DS:[1014D6C]
01004908 CALL g5.01002604
```

Good, we saw how to properly "reverse" an application, that is which function does what and from where it is called.
We also found where the space character is concatenated.
Now go up in the listing to find:

```
01004835 PUSH DWORD PTR DS:[1014DB0] ; String = "6,"
0100483B CALL DWORD PTR DS:[<&KERNEL32.lstrlenW>]
```

Here the space character has not been concatenated yet and we could plug our code here.
But we will keep looking for the place where our "6," appears for the very first time.
Go up in the listing again until you find the following call with 3 parameters.

```
010047A0 PUSH EAX ; /Arg3 => 0000000A
010047A1 PUSH 01014DC0 ; |Arg2 = 01014DC0
010047A6 PUSH 01014DB0 ; |Arg1 = 01014DB0
010047AB CALL 010022F9 ; call 010022F9
010047B0 JMP 0100485C
```

Put a breakpoint on this call, run the program and press the digit "6".
When you break here, press F8 to trace to the next instruction.
Now EAX will contain the address 0009AE40 which points to the string "6,".
This is where `everything starts` and we will plug our code there at `010047B0 JMP 0100485C`

*NOTE:*
If you trace into the call 010022F9 (using F7), you will find the place where only the character "6" is copyied into EAX.
We won't choose this location for our patch because it is in the middle of a call, inside a function. We will prefer the exit of the CALL with the returned value "6,".

```
_into the call_:
0100233E PUSH EAX ; /String2 = "6"
0100233F LEA EAX,DWORD PTR SS:[EBP+EDI\*2-108]
01002346 PUSH EAX
01002347 CALL EBX ; lstrcpyW
```

Then, after getting the content of the editbox (6,)
it copies the string in another variable here:

```
0100486B PUSH 0FE
01004870 PUSH ECX
01004871 LEA EAX,DWORD PTR SS:[EBP-200]
01004877 PUSH EAX
01004878 CALL DWORD PTR DS:[<&KERNEL32.lstrcpynW>]
```

(6, is stored in 0006F784)

Then is added 00 with the function we saw earlier (lstrcatW).
When done, it returns to the main program:

```
010027DB CALL 010045C4
010027E0 JMP 010042EB
```

We will add our code before it concatenates the space character, that is "where everything starts":
010047AB CALL 010022F9
010047B0 JMP 0100485C

## Part 2: Getting prepared for code injection

*Adding strings*

First of all, we will have to display a message when the correct code is entered.
we check in the API list if there's something that could help and we find `MessageBoxW`.

Then we have to add the string for the message box and the string that contains our password (explained later).
For this, we will simply use a string in calc.exe that will never be used. I choose "An unknown error has occured...".
In your hexadecimal editor, find 41006E002000 (begining of the string in hexadecimal) delete the error message and replace it by our strings (each char separated by 00 and each string terminated by at least 00 00)
you should get something like:

```
3000000028003D000000000059006F00 | 0...(.=.....Y.o.
7500200066006F0075006E0064002000 | u. .f.o.u.n.d. .
74006800650020007000610073007300 | t.h.e. .p.a.s.s.
77006F00720064000000000000000000 | w.o.r.d.........
38003700360039003400000000000000 | 8.7.6.9.4.......
```

Now back to the debugger (don't forget to save your changes), right-click and choose `"Seach for" > "All referenced text strings"`. In the list you should see the strings you just added at address:

```
0100131C "You found the password",0
01001350 "87694",0
```

*The password*

When we enter a digit in the calculator, it is stocked as a string.
For exemple, we entered the number 6 but it is the **character** "6" that is taken by the program, what results in **hexadecimal** 36. That's why each operation will be done on hexadecimal values.

Here is how we will calculate a checksum on the digits entered:

- Take each character
- SUB 5
- XOR 9

If the checksum that results from the number entered in the calculator equals "87694" it means the right password has been entered.

> entered: 6CD5B (36-43-44-35-42)
> checksum: 87694 (38-37-36-39-34)

6CD5B will be the password to enter in the calculator (hexadecimal mode).

## Part 3: Code Injection

In the debugger, if you select a line and double-click on it (or press space) you can change its code.
In most cases the code that follows the line you changed will be deleted and you will have to write it again at the end of your own code. It is also recomended to use pushad at the begining of the code you inject and popad at the end to save the registers.

But here we are lucky! We will plug our code on `010047B0 JMP 0100485C`, no code will be deleted.

To add our code we need some free space. Then we will look for some padding at the end of the program. The padding starts at `010136AF`.

We will have to change the `JMP 0100485C` to `JMP 010136AF`.
0100485C is the place to return when we finished everything. It returns to the original program.

Description of the code:

- Get the length of the entered digit.
- Compare it to 5 (length of the password)
- If < 5 go back to the program (0100485C)
- if = 5 calculate checksum
- compare the two checksums
- if the 2 checksums are equal, call MessageBoxW and exit
- else exit

Here it is:

```
010136AF PUSH EBX
010136B0 PUSH ECX
010136B1 PUSH EDI
010136B2 PUSH ESI
010136B3 MOV EBX,01001350 ;UNICODE "87694"
010136B8 MOV ECX,EAX ;ecx is gonna be modified after the call
010136BA MOV EDI,EAX ;so i save EAX twice in ECX and EDX
010136BC PUSH ECX ;push the entered number as parameter
010136BD CALL kernel32.lstrlenW
010136C2 CMP EAX,5
010136C5 JNZ 0100485C ;if length != 5 go back to calc.exe
010136CB MOV ECX,EDI ;save length in EDI
010136CD MOV EDI,0A ;strings look like 8.7.6.9.4 so size=10
010136D2 MOV ESI,0 ;initialize counter for loop
010136D7 CMP ESI,EDI ;is the loop finished?
010136D9 JE SHORT 010136FA ;if yes than password = OK
010136DB XOR EAX,EAX
010136DD SUB BYTE PTR DS:[ECX+ESI],5 ;SUB 5 to each char
010136E1 XOR BYTE PTR DS:[ECX+ESI],9 ;XOR it with 9
010136E5 MOV AL,BYTE PTR DS:[ECX+ESI]
010136E8 CMP BYTE PTR DS:[EBX+ESI],AL ;char = char of checksum?
010136EB JNZ SHORT 010136F2 ; if no ExitProcess
010136ED ADD ESI,2 ;loop 2 by 2 because of the '.'
010136F0 JMP SHORT 010136D7 ;next char
010136F2 POPAD
010136F3 PUSH 0
010136F5 CALL kernel32.ExitProcess
010136FA PUSH 0
010136FC PUSH 0100131C ;UNICODE "You found the password"
01013701 PUSH 0100131C ;UNICODE "You found the password"
01013706 PUSH 0
01013708 CALL USER32.MessageBoxW
0101370D POPAD
0101370E PUSH 0 ;/ExitCode = 0
01013710 CALL kernel32.ExitProcess ;ExitProcess
```

## Part 4: Source Codes

Here is the source that simulates the calculations I used in the code above:

```
.386
.model flat, stdcall
option casemap:none

include \\masm32\\include\\kernel32.inc
include \\masm32\\include\\user32.inc
includelib \\masm32\\lib\\kernel32.lib
includelib \\masm32\\lib\\user32.lib

.data
MsgOK db "OK!",0
MsgNO db "NO!",0
Edit1 db "87694",0 ;checksum to obtain
Edit2 db "6CD5B",0 ;digits entered

.code
start:
pushad
lea ebx,offset Edit1
lea ecx,offset Edit2
mov edi,5
mov esi,0

;masm syntax makes it easy to debug
;but you can use jmps if you want

.WHILE esi < edi
  xor eax,eax
  sub byte ptr[ecx+esi],5h
  xor byte ptr[ecx+esi],9h
  mov al,[ecx+esi]
    .IF byte ptr[ebx+esi] != al
      push 0
      push offset MsgNO
      push offset MsgNO
      push 0
      call MessageBox
      popad
      push 0
      call ExitProcess
    .ENDIF
  inc esi
.ENDW
push 0
push offset MsgOK
push offset MsgOK
push 0
call MessageBox
popad
end start
```

And finaly the code to bruteforce the password:

```
#include <stdio.h>

int main(void) {

  int x1,x2,x3,x4,x5,res;

  for (x1=48;x1<=70;x1++) {
    res=(x1-5)^9;
    if (res==56) printf("%c",x1);
  }
  for (x2=48;x2<=70;x2++) {
    res=(x2-5)^9;
    if (res==55) printf("%c",x2);
  }
  for (x3=48;x3<=70;x3++) {
    res=(x3-5)^9;
    if (res==54) printf("%c",x3);
  }
  for (x4=48;x4<=70;x4++) {
    res=(x1-5)^9;
    if (res==57) printf("%c",x4);
  }
  for (x5=48;x5<=70;x5++) {
    res=(x5-5)^9;
    if (res==52) printf("%c",x5);
  }
}
```