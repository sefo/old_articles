# Image File Execution Options

*Published 2006-02-13*

Wayback machine: https://web.archive.org/web/20140214160456/http://osix.net/modules/article/?id=764

> Located at `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options`,
> there exists a registry key which allows the redirection of the excution of one application to another.

## ::Theory::

The key you create in `HKEY\_LOCAL\_MACHINE\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Image File Execution Options`
must have the name of the application you want to _take over_.

For example you can create a new key called **notepad.exe** and then create a new string with the name **Debugger**
and value **C:\\WINDOWS\\system32\\calc.exe**

Now if you try to run notepad, the calculator will be launched instead.

Moreover, if you have for example .txt files associated with notepad,
double-clicking on any text file will also run calc.exe

## ::Details::

Now we're going to replace the string value by `C:\\WINDOWS\\system32\\write.exe`

If you run notepad or double-click on a .txt file, you will see that wordpad runs instead,
but this time it displays the contents of notepad.exe in its editor.

The reason is that write.exe (like notepad) opens directly the file specified in the second argument of its commandline.
(first argument being the name of the program itself)

So it means that when you redirect the execution of an application using this `Image File Execution Options` key,
the program executed instead of notepad will have the name and path of the file that has been overtaken.

As an exercise, write a program _CmdLine.exe_ that displays it's arguments.
Now if you change the value of the string to `c:\\CmdLine.exe` and run notepad, you will see:

- arg 0 = c:\\CmdLine.exe
- arg 1 = C:\\WINDOWS\\system32\\notepad.exe

The interesting part is that if you try to double-click on a .txt file, _CmdLine.exe_ will display an additional argument
containing the path to the textfile!

- arg 2 = C:\\testfile.txt

Knowing the path to the text file, it is easy to open it and check for a value or modify something and give control back to
notepad with ShellExecute or CreateProcess providing the text file as argument.

## ::Test Program::

How to use the [program included]

- The executable **must be** placed in `c:\\ExecOption.exe`
- Run the program by double-clicking on it. It will create the registry key.
(open Regedit and go to `Image File Execution Options` to see exactly how a valid key looks like)
- Then you can click any .txt file (size < 512)
- When you click on a .txt, the program `ExecOption.exe` is launched
- It first deletes the registry key. (which is necessary)
- Then it hooks the program associated (of course notepad)

_Note that your firewall should warn you when ExecOption.exe runs notepad_

- And finally opens the file.
- The registry will be cleaned when you try to open a text file
- To set the key again, you have to run the .exe one more time

See source code in the zip file for more details.

Basically, here's how it works:

```
GetCommandLine()

if NumberOfParam < 2
    Create(RegistryKey)

else if NumberOfParam > 2
    Delete(RegistryKey)
    Execute(Param2)
    WithCommandLine(Param3)
end if
```

## ::Conclusion::

This registry key is pretty evil.

I haven't found a virus or malware using this technique yet but no doubt it will be exploited one day.
However one might found a positive way to use this feature and I'd be curious to see what utility you can create.
So please post a comment if you have a nice idea.

A problem I encountered was that the .txt you open can be read, but is locked for write access by the system.
So if anyone has an idea on how to unlock it easily please post a comment.

_Monday Morning note_:
It works fine on my XP sp1 but not on XP sp2.

The time I wanted to give to this topic being over, I will not fix it. It comes I believe from the poor implementation of the commandline parsing procedure :)

You can still test it of course, the program is harmless and it gives you a good idea of how things work.
