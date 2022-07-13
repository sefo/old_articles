# Create a loader for your reversing needs

*Published 2005-09-07*

Wayback machine: https://web.archive.org/web/20140214165758/http://osix.net/modules/article/?id=723

> There are several reasons why you would use a loader to modify a program.
> Maybe the program is packed or you need to bypass a CRC check, or you simply don't want to apply the changes definitely.
> The loader will do the modifications directly in the loaded process memory.

## The target:

Being out of good ideas I've decided to reverse the winXP version of `freecell.exe`

Remember that the aim of this first article is to introduce the use of a loader to modify a program.
The following part will try to explain briefly how to find the critical comparison that checks if a move is good or not (and eventually patch it)

For those who have win2k, they can download my version of the game so that they can follow with the right addresses.

## Gathering information:

In all your reversing sessions you will have to find a point where to start.
If you run freecell and try any illegal move you will see a message box.
The obvious first step is then to put a breakpoint on every MessageBox API.

Now you can run freecell, launch a new game an try an illegal move.
You should break here:

`01003C0A CALL DWORD PTR DS:[<&USER32.MessageBoxW>]`

The text of this message is actually set by LoadString just above.
(The first LoadString loads the caption of the MessageBox, the other the text.)

The obvious thing to do would be to jump over those LoadString and MessageBox.
But this is a common mistake:

```
01003B9C JMP freecell.01003CE1
01003BA1 CMP DWORD PTR DS:[1007440],EDI
01003BA7 JE freecell.01003CF4
01003BAD MOV ESI,DWORD PTR DS:[<&USER32.LoadString>
```

So we would want to take this JE, but the only thing that it will do is prevent the "Illegal move" message to popup. The program already decided that your move is illegal.

We need to find where this JE is called.
We have the following sequence:

```
............ a
01003B9C JMP b
01003BA1 CMP c
01003BA7 JE  d
01003BAD MOV e
............ f
```

From the JE we need to go up in the code. The previous instruction is a CMP and the instruction before this is a JMP (uncoditional jump).

It means that if we follow the code linearly from a to f, it will always jump over c, d and e.

So we are probably in an _if_ statement and _01003BA1 CMP_ has been called from somewhere.
To find it, we can select the line with CMP and right-click / Find Reference to / Selected Command.
It should display:

`01003B8C JA SHORT freecell.01003BA1`

Looking just above we have:

```
01003B73 CALL freecell.01004168
01003B78 MOV EBX,EAX
01003B7A CMP EBX,EDI
01003B7C JBE freecell.01003C2B
```

When you trace over this call, eax will be given a value.
If this value is <= 0, JBE will send you to the 'Illegal move' part.
This is the place to apply a patch.

The CALL will return 1 if the move is legal and 0 if it's illegal.
Patching by **JB** will let 0 and 1 pass and allow any move.

## The Loader:

To modify a program, we need:

- The address where to apply the patch
- The opcodes for the patch
- The size of the patch

In this case we have:

- The address of the instruction to modify is 01003B7C.
- The opcode for JBE is _0F86_ and if you modify it in OllyDebug to JB, you get _0F82_.
- The size of is then 1 bit, but let's modify 2 bytes for the sake of clarity (0F86 -> 0F82)

Then we need to know what the loader is going to do.

First it as to run freecell.exe
We have the choice between the API _CreateThread_ or _CreateProcess_.
We are going to choose `CreateProcess` as it gives us write permission in the created process.

The masm syntax for this API is:

`invoke CreateProcess, ADDR Process, NULL, NULL, NULL, NULL, CREATE_SUSPENDED, NULL, NULL, ADDR Startup, ADDR processinfo`

(use PUSH if you have another assembler)
Where **Process** is a string representing the path to freecell.exe

We use **CREATE_SUSPENDED** to suspend the process and safely modify it before it executes.

**Startup** and **processinfo** are both pointers to structures.
You can declare them in masm with:

```
Startup STARTUPINFO <>
processinfo PROCESS\_INFORMATION <>
```

Or if you have another assembler and not the include/windows.inc, you can use:

```
Startup db 48h dup (0)
processinfo dd 4 dup (0)
```

If CreateProcess succeeds, eax will not be equal to 0
I used **cmp eax, 0** for code readability, but for a comparison with 0, it should be either:

**test eax, eax**

or

**and eax, eax**

as the opcodes are smaller.

Now that our process has been created, we only have to modify it in memory.
The syntax for **WriteProcessMemory** is:

`invoke WriteProcessMemory, processinfo.hProcess, ToPatch, ADDR ReplaceBy, ReplaceSize, byteswritten`

The parameters here are quite obvious. We give WriteProcessMemory the address to look for and the bytes to modify along with the total size of the modification.

The only thing left is to run the newly created process and exit our loader.

Here is the complete source code:

```
.586
.model flat,stdcall
option casemap:none

include C:\\masm32\\include\\windows.inc
include C:\\masm32\\include\\user32.inc
include C:\\masm32\\include\\kernel32.inc
includelib C:\\masm32\\lib\\user32.lib
includelib C:\\masm32\\lib\\kernel32.lib

.data
  Process db "freecell.exe",0
  Error db "Error:",0
  ErrorMessage db "Process not loaded",0
  ReplaceBy db 0Fh,82h
  ReplaceSize dd 2
  AddressToPatch dd 01003B7Ch
  Startup STARTUPINFO <>
  processinfo PROCESS\_INFORMATION <>

.data?
  byteswritten dd ?

.code
  start:
    invoke CreateProcess, ADDR Process, NULL, NULL, NULL, NULL, CREATE\_SUSPENDED, NULL, NULL, ADDR Startup, ADDR processinfo
    cmp eax, 0
    jne ProcessCreated
    push 0
    push offset Error
    push offset ErrorMessage
    push 0
    call MessageBox
    push 0
    call ExitProcess

    ProcessCreated:
        invoke WriteProcessMemory, processinfo.hProcess, AddressToPatch, ADDR ReplaceBy, ReplaceSize, byteswritten
        invoke ResumeThread, processinfo.hThread
    push 0
    call ExitProcess
  end start
```

Equivalent in C:

```
#include <windows.h>

int main(int argc, char\* argv[]) {
  char\* process="freecell.exe";
  STARTUPINFO startup;
  PROCESS\_INFORMATION processInfo;

  memset(&startup, 0, sizeof(STARTUPINFO));
  startup.cb=sizeof(STARTUPINFO);

  if(!CreateProcess(process, NULL, NULL, NULL, NULL, CREATE\_SUSPENDED, NULL, NULL, &startup, &processInfo)) {
    MessageBox(0, "Process not loaded", "Error:", 0);
    exit(-1);
  }

  char\* replaceBy = "\\x0F\\x82";
  unsigned long bytesWritten;
  WriteProcessMemory(processInfo.hProcess, (void\*)0x01003B7C, replaceBy, 2, &bytesWritten);

  ResumeThread(processInfo.hThread);

  return 0;
}
```