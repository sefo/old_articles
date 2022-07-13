# The truth about the Gorgon trojan

*Published 2005-12-12*

Wayback machine: https://web.archive.org/web/20140214160456/http://osix.net/modules/article/?id=764

> It appears that this 'trojan' has one anti-debugging trick in its arsenal.
> Not having had any problem in my environment, I decided to give it another try.

First, I removed all my Olly pluggins. (especially the 2 hide-debugger ones)
Then I put a break point on TerminateProcess (Which is the one that is most likely used to close Olly)

Here is the function responsible for our problems:

```
004604A4 PUSH EBP
004604A5 MOV EBP,ESP
004604A7 ADD ESP,-148
004604AD PUSH EBX
004604AE PUSH ESI
004604AF PUSH EDI
004604B0 XOR EDX,EDX
004604B2 MOV DWORD PTR SS:[EBP-140],EDX
004604B8 MOV DWORD PTR SS:[EBP-148],EDX
004604BE MOV DWORD PTR SS:[EBP-144],EDX
004604C4 MOV DWORD PTR SS:[EBP-130],EDX
004604CA MOV DWORD PTR SS:[EBP-13C],EDX
004604D0 MOV DWORD PTR SS:[EBP-134],EDX
004604D6 MOV DWORD PTR SS:[EBP-138],EDX
004604DC MOV DWORD PTR SS:[EBP-4],EAX
004604DF MOV EAX,DWORD PTR SS:[EBP-4]
004604E2 CALL Server.00404B6C
004604E7 LEA EDI,DWORD PTR SS:[EBP-12C]
004604ED XOR EAX,EAX
004604EF PUSH EBP
004604F0 PUSH Server.00460612
004604F5 PUSH DWORD PTR FS:[EAX]
004604F8 MOV DWORD PTR FS:[EAX],ESP
004604FB XOR ESI,ESI
004604FD XOR EDX,EDX
004604FF MOV EAX,2
00460504 CALL Server.0045EAEC ;----CreateToolhelp32Snapshot----
00460509 MOV EBX,EAX
0046050B MOV DWORD PTR DS:[EDI],128
00460511 MOV EDX,EDI
00460513 MOV EAX,EBX
00460515 CALL Server.0045EB0C ;-----Process32First----
0046051A JMP Server.004605DE
0046051F /LEA EAX,DWORD PTR SS:[EBP-138]
00460525 |LEA EDX,DWORD PTR DS:[EDI+24]
00460528 |MOV ECX,104
0046052D |CALL Server.00404934
00460532 |MOV EAX,DWORD PTR SS:[EBP-138]
00460538 |LEA EDX,DWORD PTR SS:[EBP-134]
0046053E |CALL Server.00409154
00460543 |MOV EAX,DWORD PTR SS:[EBP-134]
00460549 |LEA EDX,DWORD PTR SS:[EBP-130]
0046054F |CALL Server.00408884
00460554 |MOV EAX,DWORD PTR SS:[EBP-130]
0046055A |PUSH EAX
0046055B |LEA EDX,DWORD PTR SS:[EBP-13C]
00460561 |MOV EAX,DWORD PTR SS:[EBP-4]
00460564 |CALL Server.00408884
00460569 |MOV EDX,DWORD PTR SS:[EBP-13C]
0046056F |POP EAX
00460570 |CALL Server.00404AC8 ;----!!!----
00460575 |JE SHORT Server.004605BE ;----JMP to ExitProcess----
00460577 |LEA EAX,DWORD PTR SS:[EBP-144]
0046057D |LEA EDX,DWORD PTR DS:[EDI+24]
00460580 |MOV ECX,104
00460585 |CALL Server.00404934
0046058A |MOV EAX,DWORD PTR SS:[EBP-144]
00460590 |LEA EDX,DWORD PTR SS:[EBP-140]
00460596 |CALL Server.00408884
0046059B |MOV EAX,DWORD PTR SS:[EBP-140]
004605A1 |PUSH EAX
004605A2 |LEA EDX,DWORD PTR SS:[EBP-148]
004605A8 |MOV EAX,DWORD PTR SS:[EBP-4]
004605AB |CALL Server.00408884
004605B0 |MOV EDX,DWORD PTR SS:[EBP-148]
004605B6 |POP EAX
004605B7 |CALL Server.00404AC8
004605BC |JNZ SHORT Server.004605D5
004605BE |PUSH 0 ; /ExitCode = 0
004605C0 |MOV EAX,DWORD PTR DS:[EDI+8] ;
004605C3 |PUSH EAX ; |/ProcessId
004605C4 |PUSH 0 ; ||Inheritable = FALSE
004605C6 |PUSH 1 ; ||Access = TERMINATE
004605C8 |CALL <JMP.&kernel32.OpenProcess> ; |OpenProcess
004605CD |PUSH EAX ; |hProcess
004605CE |CALL <JMP.&kernel32.TerminateProcess> ; TerminateProcess
004605D3 |MOV ESI,EAX
004605D5 |MOV EDX,EDI
004605D7 |MOV EAX,EBX
004605D9 |CALL Server.0045EB2C
004605DE |TEST EAX,EAX
004605E0 JNZ Server.0046051F
004605E6 PUSH EBX ; /hObject
004605E7 CALL <JMP.&kernel32.CloseHandle> ; CloseHandle
004605EC XOR EAX,EAX
004605EE POP EDX
004605EF POP ECX
004605F0 POP ECX
004605F1 MOV DWORD PTR FS:[EAX],EDX
004605F4 PUSH Server.00460619
004605F9 LEA EAX,DWORD PTR SS:[EBP-148]
004605FF MOV EDX,7
00460604 CALL Server.004046F0
00460609 LEA EAX,DWORD PTR SS:[EBP-4]
0046060C CALL Server.004046CC
00460611 RETN
```

I put a break point at `004604A4` which is the begining of this function and traced from there entering every CALL.

In the 2nd CALL (00460504) gorgon hooks `CreateToolhelp32Snapshot` and in the next one it hooks `ProcessFirst` then `ProcessNext`.

With these APIs, gorgon can now scan all the running processes.

Now we right-click on `004605BE` which is the first parameter of ExitProcess, and we select "Find Reference to selected command". It leads us to:

```
00460570 CALL Server.00404AC8
00460575 JE SHORT Server.004605BE
```

So this is the call that will check if we have OllyDbg running.
when tracing into this call, we can see clearly that it scans every process and compares them to OLLYDBG.EXE

```
00404AF1 /MOV ECX,DWORD PTR DS:[ESI] ;name of current process
00404AF3 |MOV EBX,DWORD PTR DS:[EDI] ;name to compare with
00404AF5 |CMP ECX,EBX
00404AF7 |JNZ SHORT Server.00404B51
00404AF9 |DEC EDX
00404AFA |JE SHORT Server.00404B11
00404AFC |MOV ECX,DWORD PTR DS:[ESI+4] ;process name minus .EXE
00404AFF |MOV EBX,DWORD PTR DS:[EDI+4]
00404B02 |CMP ECX,EBX
00404B04 |JNZ SHORT Server.00404B51
00404B06 |ADD ESI,8
00404B09 |ADD EDI,8
00404B0C |DEC EDX
00404B0D JNZ SHORT Server.00404AF1
```

Once it finds it, it returns 0 and then jumps to `TerminateProcess`.

Solutions:

- Use the HideDebugger pluggin (the one that changes the class name)
- Change the class name of Olly by hand
- Look for the string "OLLYDBG.EXE" in gorgon (appears twice) and replace it
- Replace the JE (7474) that jumps to TerminateProcess by 2 NOPs (9090)

I used the 3rd option. To modify the strings, activate the dump window, press CTRL+G (goto)
and put `00464210` then `0046A410` (strings OLLYDBG.EXE hardcoded in gorgon)

Select the string and press `<SPACE>` to modify it.

## Conclusion:

It confirms what I initially knew about this 'trojan':

It IS a malware, but it is not going to spread anywhere in its current state.
I wish all virus writers used delphi with pointless anti-debug tricks. Internet would be safer.