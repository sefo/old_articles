# uTorrent 2 banned

*Published 2010-03-24*

Wayback machine: https://web.archive.org/web/20140208050406/http://osix.net/modules/article/?id=958

> Not sure if that's recent news or not, but I've had some request about changing uTorrent 2's peer_id
> because it's been banned from some trackers.
> UPDATE: The peer_id I used in this article is not correct. It should work using -UT160B- instead (see article for link)

Edit:
[http://forum.utorrent.com/viewtopic.php?pid=468992](https://web.archive.org/web/20140208050406/http://forum.utorrent.com/viewtopic.php?pid=468992)

So following my BitComet hack:
[http://osix.net/modules/article/?id=788](https://web.archive.org/web/20140208050406/http://osix.net/modules/article/?id=788)

I apply it to uTorrent 2 (the process is easier than BitComet!)

- Tools: OllyDbg, uTorrent2

## STRINGS:

_Note:_
`uTorrent2` is upx'ed to make the exe smaller, upx -d will do the trick.

First thing we do is check if the peer_id we want is hardcoded:
`right-click->search for->all reference text strings`

In the list, right click->search text = "peer_id"
The first occurrence is the string it sends to identify itself to the tracker.
It looks like this: `...&peer_id=%.20U...`

uTorrent will replace %.20U by -UT2000- (and we want -UT1600-)
If you double click on this line you go to the code:

`0040A28D PUSH utorrent.0046D0A8 ;ASCII "...&peer_id=%.20U...`


At this point in code, the string has already been generated so we go up a little in code (randomly a few lines above) and by debugging from there we find:

_Note:_
After setting a BP, it takes a few seconds before breaking. Do not think your BP didn't work!

`0040A281 |. 68 303D4900 PUSH utorrent.00493D30`

`00493D30 = -UT2000-`
We can find this by a right-click on line above->follow in dump->immediate constant

We now need to find where 1 or both address are accessed (written to).

In the disasm window, `r-click->go to->expression = 00493d30`
r-clk on the new line that appears and ->breakpoint->on memory write

_Note:_
Before you set this kind of BP, remove all previous BP (atlt+B) and restart olly.
Then go to address above, set BP and run.

Run (F9) and it breaks there:

```
0040D877 PUSH ESI
0040D878 MOV ESI,ECX
0040D87A MOV DWORD PTR DS:[ESI],EAX ------->BP
0040D87C MOV EAX,DWORD PTR DS:[46D978]
0040D881 PUSH EDI
0040D882 MOV DWORD PTR DS:[ESI+4],EAX
0040D885 MOV EAX,48BC
```

So we set a BP on memory, write, again at 0040d87a and run again to stop here:

`0040D86F MOV EAX,DWORD PTR DS:[46D974]`

Now this address contains UT2000... (and will transfer it to 00493D30)

This looks harcoded (although i didn't see it in the strings listing)

So i follow it in dump, select UT2000 and press space, replace with UT1600 and copy to exe/save to file as usual. (see bitcomet article for details on how to patch an exe)

To test it, run the new exe and break on the following address:

`0040A2F2 PUSH DWORD PTR SS:[EBP+6C]`

You will see that the generated (the one submitted to trackers) has been modified.

`http://www.blaahtorrent.com:3394/announce?info_hash=Pz%e4%a3%c2B%d4%cc%27%02%ee%bfqk%93%d3%03Y%fe%0e&peer_id=-UT1600-%bcHq%18%7f%3b%1c%ae%83%e8%d9%90&port=11145&uploaded=0&downloaded=0&left=722337&corrupt=0&key=0B54A196&event=started`

For the patched exe:
[uTorrent160B.exe - uTorrent2 patched peer_id updated](https://web.archive.org/web/20140208050406/http://osix.net/modules/folder/index.php?tid=47362&action=df)