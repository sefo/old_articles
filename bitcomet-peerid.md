# Bitcomet Peer ID

*Published 2006-05-27*

Wayback machine: https://web.archive.org/web/20140214142846/http://osix.net/modules/article/?id=788

> This bittorrent client has been banned from the majority of private trackers.
> It is possible to bypass the ban by spoofing the peer_id it sends.

This article is by no means a way to protest against the measures taken by those private trackers or a way to promote the use of BitComet. This article is completely disinterested and tries to demonstrates a technique of spoofing by means of modifying a program.

Tools used:

- [OllyDbg](https://web.archive.org/web/20140214142846/http://www.ollydbg.de/)
- [WPE packet sniffer](https://web.archive.org/web/20140214142846/http://www.pimpsofpain.com/wpe.zip)

Note:
WPE is detected by some AVs as a trojan but this is not the case.

[http://www.symantec.com/avcenter/venc/data/hacktool.wpe.html](https://web.archive.org/web/20140214142846/http://www.symantec.com/avcenter/venc/data/hacktool.wpe.html)

When you start downloading a torrent, BitComet sends a packet identifying the request and the version of the client you use.
In the case of version 0.66, the packet looks like:

```
0010 70 3F 69 6E 66 6F 5F 68 61 73 68 3D 25 44 31 25 p?info_hash=%D1%
0020 38 46 4A 50 25 30 35 73 31 25 42 31 25 41 46 48 8FJP%05s1%B1%AFH
0030 6A 61 66 4F 25 32 36 25 39 34 25 43 36 25 37 45 jafO%26%94%C6%7E
0040 32 25 39 37 26 70 65 65 72 5F 69 64 3D 25 32 44 2%97&peer_id=%2D
0050 42 43 30 30 36 36 25 32 44 54 25 41 34 25 30 33 BC0066%2DT%A4%03
0060 25 46 34 25 42 45 7A 5A 25 31 31 6D 25 39 44 25 %F4%BEzZ%11m%9D%
0070 32 44 25 46 39 26 70 6F 72 74 3D 39 34 34 39 26 2D%F9&port=9449&
0080 75 70 6C 6F 61 64 65 64 3D 30 26 64 6F 77 6E 6C uploaded=0&downl
```

At offset **0050** we can find the identifier of the client _BC0066_ (BitComet v0.66)

In older versions, this peer_id was directly hardcoded in the executable and it was very trivial to modify it in an hexadecimal editor. (value excb if I remember)

If you take a look at the string references in version 0.66 using OllyDbg, you can still find this value, but it might be only a way to confuse the potential 'hacker'.

Being the only way to identify the client, we need to change this peer_id to a value like _UT1500_ which corresponds to the UTorrent client version 1.5

A tracker would then believe you are using a UTorrent client and let you in.

## Where do we start?

The hardest part in that kind of analysis is to find where to start; especially since this value is not hardcoded but generated on the fly.

In the packet we captured earlier, we had a string like `"&peer_id="`

Right-click anywhere on the disassembly windows and select `Search for -> All referenced textstrings`

In the list that pops up, right-click again and choose _Search for text_.
We find 2 occurences for the string _"&peer_id="_ at addresses 00537DA2 and 00537E54

This will be our starting point and we need to put a breakpoint on each of them.
(select the first line and press F2 then select the second line and press F2 again)

Let's start BitComet and see what happens.

## Ignoring exceptions

What happens when you run BitComet in OllyDbg is that the debugger gets stuck on 3 exceptions in the library msxml3.dll
Those exceptions are not anti-debugging technics as I first thought, but are actually real exceptions/errors occurring in the program. (mostly due to a badly formatted xml file or BitComet is maybe calling the dll too many times)

When BitComet runs on its own, it just ignores those exceptions as they are not critical, but the debugger lets you know that something is happening.

What we just need to do is to tell OllyDbg to ignore these exceptions.

Each time OllyDbg signals an exception, you can go to the menu `Option / Debugging options` and select the _Exceptions_ tab.
Check the checkbox `Ignore also custom exceptions` and click on the `Add last exception` (do this each time OllyDbg signals an exception - happens 3 times in our case)

The exceptions are:

```
C06D007F
E0000001
E06D7363
```

Now we can run BitComet in the debugger and start the analysis.

## Debugging

When you start downloading a torrent file, it immediately breaks on one of our breakpoints we set earlier at 00537E54
The function looks like:

```
00537E4A CALL BitComet.005558B0
00537E4F PUSH EAX
00537E50 LEA EDX,DWORD PTR SS:[ESP+20]
00537E54 PUSH BitComet.00634D34 ; ASCII "&peer_id="
00537E59 PUSH EDX
00537E5A MOV BYTE PTR SS:[ESP+B4],0C
00537E62 CALL BitComet.00412680
```

The reasoning we should have now is that we need to know what's happening before and after that _PUSH &peer_id=_
and especially what are those 2 CALLs we have at the begining and at the end.

The logic of the second function is that it pushes 2 parameters on the stack:

- The string "&peer_id="
- A value in EDX (actually a pointer to a value)

Let's take a look at the stack before the CALL.
You can click on the lower window (called the dump window, it looks like an hexadecimal editor) to activate it and then press Ctrl+G (goto address)

enter ESP and you will have the dump of the memory pointed by ESP.

There's nothing much to see for the moment except a value we saw earlier in our packet: 8FJP.
If we continue executing the program and trace over the CALL at 00537E62, you now see the value BC0066 we are looking for (see the memory dump of ESP)

What we could do is change this value now, directly by selecting each 6 bytes and press `<space>` to modify it, but the change will only be temporary and can't be apply to the executable as it is only a value in memory.

What this function actually does is to fill a buffer with the value of peer_id, but that has been already generated previously and this function is copying it from memory.

That's where the CALL at `00537E4A` comes into play.

This function is being called at different places in the code and it might look like:

`GeneratePacket(int part_id);`

Where part_id identifies what part of the packet is to be generated. (info_hash, peer_id...etc.)
In our case, since we are in the peer_id part, we will follow the peer_id generation code.

The first part of the function is just setting an Exception Handler and saves some registers before use:

```
005558B0 PUSH -1
005558B2 PUSH BitComet.0060E4B1
005558B7 MOV EAX,DWORD PTR FS:[0]
005558BD PUSH EAX
005558BE SUB ESP,30
005558C1 MOV EAX,DWORD PTR DS:[6B4D94]
005558C6 XOR EAX,ESP
005558C8 MOV DWORD PTR SS:[ESP+2C],EAX
005558CC PUSH EBX
005558CD PUSH EBP
005558CE PUSH ESI
005558CF PUSH EDI
```

Then it initialises some variables:

```
005558D0 MOV EAX,DWORD PTR DS:[6B4D94]
005558D5 XOR EAX,ESP
...
005558FE MOV EAX,DWORD PTR DS:[EDI+14]
00555901 CMP EAX,ESI
00555903 MOV DWORD PTR SS:[ESP+4C],ESI
```

And starts a loop with a number that will identify how many bytes of the peer_id it already generated:

```
00555907 MOV DWORD PTR SS:[ESP+1C],1
0055590F JBE BitComet.00555A9F ;start of the loop
00555915 LEA EBP,DWORD PTR DS:[EDI+4]
```

Then you have that very long and un-esthetic loop containing code that looks like a switch()/case with a series of comparisons and jumps:

Those comparisons actually check the loop index to see which part of the packet is being generated and where to put it.

```
0055592D CMP ESI,EAX
0055592F MOV DWORD PTR SS:[ESP+4C],1
00555937 JBE SHORT BitComet.0055593E
00555939 CALL BitComet.005A5261
0055593E CMP DWORD PTR DS:[EDI+18],10
00555942 JB SHORT BitComet.00555949
...
005559E7 CALL BitComet.005A5261
005559EC CMP DWORD PTR DS:[EDI+18],10
005559F0 JB SHORT BitComet.005559F7
005559F2 MOV EAX,DWORD PTR SS:[EBP]
005559F5 JMP SHORT BitComet.005559F9
...
```

Then, if we're still following the dump of ESP at the same time, we notice that after the CALL at 00555A09,
the first byte of the peer_id ("B") has been copied in memory. We continue the execution of the code, and on the second loop, the same CALL fills the memory with the second byte of the peer_id ("C").

Now on the third loop, we trace into the call (using F7 instead of F8) and see what's happening there.
The following steps are a bit indigest and very long. It consists in following what's happening in esp where each byte of the peer_id is copied

and finding what function called what function by following numerous CALLs and re-running the application and breaking a bit earlier in the code...etc.

So I'll give you directly the interesting part here:

```
004D2687 CALL BitComet.0040A900 ;
004D268C PUSH 30 ;
004D268E PUSH 1
004D2690 MOV ECX,ESI
004D2692 MOV DWORD PTR SS:[ESP+74],EBX
004D2696 MOV DWORD PTR SS:[ESP+1C],1
004D269E CALL BitComet.0041A4F0
004D26A3 PUSH 30 ;
004D26A5 PUSH 1
004D26A7 MOV ECX,ESI
004D26A9 CALL BitComet.0041A4F0
004D26AE PUSH 36 ;
004D26B0 PUSH 1
004D26B2 MOV ECX,ESI
004D26B4 CALL BitComet.0041A4F0
004D26B9 PUSH 36 ;
004D26BB PUSH 1
004D26BD MOV ECX,ESI
004D26BF CALL BitComet.0041A4F0
```

What catches the eye here is the series of _PUSH 30_ and _PUSH 36_
Knowing that BC0066 equals `42 43 30 30 36 36` we can now safely assume that the 0066 are generated by pushing the hexadecimal values directly on the stack.

For the moment we can change definitely the value `0066` to `1500` by selecting each of the 4 lines concerned and press `<space>`

You can now tell OllyDbg to assemble:

```
PUSH 31
PUSH 35
PUSH 30
PUSH 30
```

Now let's see how "BC" is generated from the function at 0040A900

The function is very long, but it actually only loads the 2 bytes "BC" hardcoded in the program at address 0062E538
Select the dump window and press Ctrl+G to go to this address and find the hardcoded value.

You can now also change definitely the "BC" part to "UT".

Right-click anywhere on the disassembly window and select `Save to File / All modifications` and you have you're new BitCometClient identified as a UTorrent client v1.5

Note for those who read only the end of the articles:

```
In the assembly window:
1/ Go to 004D268C and change PUSH 30 to PUSH 31
2/ Go to 004D26A3 and change PUSH 30 to PUSH 35
3/ Go to 004D26AE and change PUSH 30 to PUSH 30
4/ Go to 004D26B9 and change PUSH 30 to PUSH 30

In the dump window go to 0062E538:
5/ Change BC to UT
```

Save all modifications using right-click / save to file / all modifications.
