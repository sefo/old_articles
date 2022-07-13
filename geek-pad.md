# Geek pad

*Published 2005-11-24*

Wayback machine: https://web.archive.org/web/20140214173506/http://osix.net/modules/article/?id=758

> We are going to transform notepad.exe into a wonderful IDE for your favourite commandline based compiler.
> Today's menu: API hook, advanced re-engineering and other various modifications.

## INTRODUCTION:

The goal we want to achieve is to add a menu and tell notepad to run our compiler.
The path to the compiler and the compilation parameters will be hardcoded into notepad.exe for simplicity sake but we could have used a file containing the path to the compiler, parameters...etc.

Now get your free tools ready:

- OllyDbg
- Resource Hacker
- Win32.hlp
- HexEditor

## PART 1: Adding the Menu

We don't want to waste our time on this task so we are going to use a resource editor.
There not much to say in this section, you can put that menu item wherever you want. Just be sure to give it an ID that is not already used and don't forget it.

I put my item in the "File" menu and it looks like `MENUITEM "&Compile",8` (8 = ID of the item)

Finally click on 'Compile Script' then File/Save As np.exe

## PART 2: Calling the compiler

Here we just need to call _ShellExecuteA_ which is located in `SHELL32.DLL`.
Open np.exe in Ollydbg and look at the imported functions (Ctrl+N).

Here we encounter our first difficulty, ShellExecute is not imported...
No problem, we can use _GetModuleHandleA_ and _GetProcAddressA_ to hook this function.

Look in your API reference to see how each function works:

- GetModuleHandleA only needs one string as parameter (for us it will be "SHELL32.DLL") and returns the handle to the DLL.
- GetProcAddressA requires 2 parameters: the handle to the DLL and a string defining the name of the function. (we will need "ShellExecuteA"). for each call to ShellExecuteA you will have a code like this:

```
PUSH DLL_NAME
CALL GetModuleHandleA ;handle is put in eax
PUSH FUNCTION_NAME
PUSH EAX ;handle to Shell32.dll
CALL GetProcAddress
PUSH [parameters]
CALL EAX ; call ShellExecute
```

The next thing to do now is to add the strings we need in the program.

In the string references (View / References) you can see that the strings in notepad are stored between 010010E0 and 01007D66 (the last string is USER32.dll).

This is the limit for the _.text_ section. You could check in a PE editor to see if we have enough padding to add a few strings but just open notepad in your Hexadecimal Editor and look for USER32.dll

We don't have much space here but it will be enough.

We need:
- The function: ShellExecuteA
- The path to the compiler: D:BC++bin
- The parameters: -I..include -L..lib Hello.c
- The files name: bcc32.exe
- The operation to execute: open

I wanted to keep everything in the text section so it's a bit messy.
After modification, it looks like:

```
offset 00007160
6578 7457 0000 5553 4552 3332 2E64 6C6C | extW..USER32.dll
0000 5368 656C 6C45 7865 6375 7465 4100 | ..ShellExecuteA.
443A 5C42 432B 2B5C 6269 6E00 740E 83FE | D:BC++bin.t...
some random bytes here (actually opcodes)...
FF00 0062 6363 3332 2E65 7865 006F 7065 | ...bcc32.exe.ope
6E00 2D49 2E2E 5C69 6E63 6C75 6465 202D | n.-I..include -
4C2E 2E5C 6C69 6220 4865 6C6C 6F2E 6300 | L..lib Hello.c.
```

Check in the reference strings in Olly to verify that your strings are there.

## PART 3: Targetting the menu item

When you code a win32 application in ASM, you use the following code to retrieve the WM_COMMAND:

```
CMP EAX,111h ;WM_COMMAND
MOV EAX,wParam
SHR EAX,10h
CMP AX,Resource_ID ;ID of a button...etc.
JE MyAddress
CMP AX,Resource_ID2
JE MyAddress2
```

So we look for a `SHR EAX,10` in OllyDbg (Ctrl+F). you can use Ctr+L to find the next instruction.
You will find WM_COMMAND here:

```
010035D0 CMP EDI,DWORD PTR DS:[1008818] ; Case 111 (WM_COMMAND)
010035D6 JNZ SHORT notepad.01003624
010035D8 MOV EAX,DWORD PTR SS:[EBP+10]
010035DB SHR EAX,10
010035DE CMP AX,500
010035E2 JE SHORT notepad.010035EA
010035E4 CMP AX,501
```

Usually we would have to add our condition (`cmp eax,ItemID`) somewhere after the `SHR EAX,10`.
But we are not in presence of a nice and clean ASM program and I can't see any comparison to other menu items (1 to 7 for the File menu, see in resource editor).

After having looked at how notepad handles the events, I can tell you that this is a real mess!

So let's put some breakpoints on `PostMessageW`. Now disable all the breakpoints and run notepad.
When notepad is running, enable every breakpoint and click on menu File/Exit.
You will break here:

```
01002C11 PUSH EBX ; /lParam; Case 7 of switch 01002929
01002C12 PUSH EBX ; |wParam
01002C13 PUSH 10h ; |Message = WM_CLOSE
01002C15 PUSH DWO ; |hWnd
01002C18 CALL DWO ; PostMessageW
```

Great, we found something that can be useful. We have the ID of the Exit item (check in a resource editor ID=7)
After the PostMessageW, you will land here:

```
010028CA PUSH EAX ; /pMsg
010028CB CALL DWO ; TranslateMessage
010028D1 LEA EAX,DWORD PTR SS:[EBP-20]
010028D4 PUSH EAX ; /pMsg
010028D5 CALL DWO ; DispatchMessageW
```

It passes WM_CLOSE to DispatchMessageW and closes the app.

But we're not going to use this. Instead select the line `01002C11 PUSH EBX`, right-click and find reference to selected command.

(I choose this line because the line just above is an unconditional JMP. This means that PUSH EBX has been called from somewhere)

You find the reference at **01002BE7**. Go up in the code again and find a reference to `01002BE1 SUB ESI,6` (the line above is a JMP). you find:

```
01002961 CMP ESI,5
01002964 JG notepad.01002BE1
0100296A JE notepad.01002B7E
```

Excellent. That's exactly what we need. 5 is the ID of the 5th item in the File menu.
We just need to find a place where to add our _CMP ESI,8_

Here, if the ItemID is greater than 5, it goes to 01002BE1.
The original code is:

```
01002961 CMP ESI,5
01002964 JG notepad.01002BE1 ;if ID > 5, jump
0100296A JE notepad.01002B7E
01002970 DEC ESI
01002971 JE notepad.01002B72
01002977 DEC ESI
01002978 JE notepad.01002A89
```

Here is how it looks after modification:

```
01002961 CMP ESI,8
01002964 JMP notepad.01007D8C ;start of padding at the end of np.exe
01002969 NOP
0100296A JE notepad.01002B7E
01002970 DEC ESI
01002971 JE notepad.01002B72
01002977 DEC ESI
01002978 JE notepad.01002A89
```

The lines:

```
CMP ESI,5
JG notepad.01002BE1
```

have been replaced by:

```
CMP ESI,8
JMP notepad.01007D8C
```

so we have to re-write them in our patch.

_Note:_
The opcode for JG and JMP are different, that's why you have a NOP in the middle. JMP is bigger and JG has been overwritten.
For the moment, we can write a temporary code at 01007D8C:

```
01007D8C JE SHORT np.01007D9C ;if esi=8, our menu item
01007D8E CMP ESI,5 ;deleted lines
01007D91 JG np.01002BE1
01007D97 JMP np.0100296A ;continue original flow of program
01007D9C MOV EAX,666 ;our temporary code
01007DA1 JMP np.01002984 ;give control back to np.exe
```

Now we can test if our menu works by putting a bp on 01007D9C and click on File/Compile.

_Note:_
To save your patch: right-clik / Copy to executable / all modifications.
On the popup choose "Copy all" and on the 'Patch window' that appears, right-click and choose "Save File".

## PART 4: Let's get serious

Now that we correctly plugged our patch, we just have to replace the temporary code by the call to the compiler.
We just have to hook ShellExecuteA in SHELL32.DLL.

```
GetModuleHandleA (01007D72) ;"SHELL32.dll"
GetProcAddressA (eax,01007D7F) ;eax = handle of shell32, "ShellExecuteA"
push [some parameters]
call eax
```

Final code

```
01007D8C JE SHORT np.01007D9C ;if menu ID = 8
01007D8E CMP ESI,5
01007D91 JG np.01002BE1 ;if ID = 5
01007D97 JMP np.0100296A ;else return to normal flow

;---Hooking
01007D9C PUSH np.010071F2 ; /pModule = "SHELL32.dll"
01007DA1 CALL kernel32.GetModuleHandleA
01007DA6 PUSH np.01007D72 ; /ProcNameOrOrdinal = "ShellExecuteA"
01007DAB PUSH EAX ; |hModule
01007DAC CALL kernel32.GetProcAddress ; GetProcAddress

;---ShellExecute
01007DB1 PUSH 5
01007DB3 PUSH np.01007D80 ; ASCII "D:BC++bin"
01007DB8 PUSH np.01007DE2 ; ASCII "-I..include -L..lib Hello.c"
01007DBD PUSH np.01007DD3 ; ASCII "bcc32.exe"
01007DC2 PUSH np.01007DDD ; ASCII "open"
01007DC7 PUSH 0
01007DC9 CALL EAX

;---give control back to notepad
01007DCB POP ESI
01007DCC JMP np.01002984
```

## Conclusion

The ideal would have been to get the compiler information from a file and the current file from notepad itself.
As it is now it is pretty static and can only be used with BC++ and Hello.c :P
But you've hopefully learnt a little more about notepad, re-engineering and easy 'API' hooking.