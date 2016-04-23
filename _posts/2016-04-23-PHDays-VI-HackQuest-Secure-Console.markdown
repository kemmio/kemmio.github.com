---
layout: post
title:  "PHDays VI HackQuest Secure Console(category:reverse)"
subtitle: "YABRT: Yet Another BIOS Reversing Task"
categories: [ctf]
---
First you get a link to a web-site which emulates some BIOS and some basic MBR staged code. Open the source-code view and you will eventually find links to download the images, we need only **/linux.iso** one the other are just basic vgabios and seabios images, with no patching in them. As always dealing with this kind of tasks we need **QEMU** and the good-old **IDA**. Load image with **QEMU**
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvi-2/qemu.PNG)
Attach to **QEMU** with **remote GDB** from **IDA**. Suspend the process from **IDA** and press a couple of times *F7* to trace into and find where is code resided in the virtual memmory. You will end up with something like this
{% highlight ruby %}
MEMORY:00007C6B ; ---------------------------------------------------------------------------
MEMORY:00007C6B ror     dword_BCD98EC1[esi], 1
MEMORY:00007C71 add     byte_FFFFFFE5[ecx+ecx*4], bh
MEMORY:00007C75 jmp     short loc_7CB0
MEMORY:00007C77 ; ---------------------------------------------------------------------------
{% endhighlight %}
And this code will be followed with other non-null bytes, but not recognized as code. Use the power of **IDA** and create code from that bytes using key *'C'*. Evetually you will get something similar.
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvi-2/codepart.PNG)
We need to find the **int 16h** command now that provides the keyboard input and we have a couple of ways to do that. Either calc the offset from this **int 10h** instruction to the **int 16h** instruction by disassembling the ISO image(with which **IDA** copes with well) or use the same logic of "creating code from data" to find it, better use the first way if you don't have some knowledge about opcodes, because you will need to filter the trash bytes on your way. But there is a thing that we need to do first, we need to define that code segment as a 16-bit segment because the mode chages from 32 to 16 we can't use 16-bit at the start, so go to *Edit->Segments->Create Segment* put the start address as *0x7c00* and the 16-bit mode bulletpoint. So I found the address of **int 16h**
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvi-2/int16h.PNG)
Put breakpoint on it and run the process. Go to **QEMU** and enter something there, the breakpoint hits, it's time to trace now. First we will see that the pin needs to have length equal to 5.
{% highlight ruby %}
seg001:7CCD loc_7CCD:                               ; CODE XREF: seg001:7CDBj
seg001:7CCD call    loc_7DC8
seg001:7CD0 mov     si, 7D78h
seg001:7CD3 add     si, cx
seg001:7CD5 mov     [si], al
seg001:7CD7 inc     cx
seg001:7CD8 cmp     cx, 5							;compare the length with 5
seg001:7CDB jnz     short loc_7CCD
seg001:7CDD mov     si, 7C62h
seg001:7CE0 call    loc_7C77
seg001:7CE3 call    loc_7D84						;converts the string to a number e.g "12345" = 12345
seg001:7CE6 mov     si, 7CA0h						
seg001:7CE9 add     si, di
seg001:7CEB add     si, di
seg001:7CED mov     ax, ds:7D82h
seg001:7CF0 mov     [si], ax						;load the pins to the address 0x7CA0
seg001:7CF2 inc     di
seg001:7CF3 cmp     di, 8
seg001:7CF6 jnz     short loc_7CB3
seg001:7CF8 mov     cx, 0
seg001:7CFB mov     si, 7C90h
seg001:7CFE mov     di, 7CA0h
seg001:7D01
seg001:7D01 loc_7D01:                               ; CODE XREF: seg001:7D11j
seg001:7D01 mov     ax, [di]
seg001:7D03 cmp     ax, [si]						;compare 0x7CA0(our pins) with 0x7c90
seg001:7D05 jnz     short loc_7D2D
seg001:7D07 add     si, 2
seg001:7D0A add     di, 2
seg001:7D0D inc     cx
seg001:7D0E cmp     cx, 8
seg001:7D11 jnz     short loc_7D01
seg001:7D13 mov     ax, ds:7CA0h
seg001:7D16 inc     ax
seg001:7D17 cmp     ax, 0C0DFh
seg001:7D1A jnz     short loc_7D22
seg001:7D1C mov     si, 7C02h
seg001:7D1F call    loc_7C77
{% endhighlight %}
The *0x7C90* seems very interesting and it points to the following bytes
{% highlight ruby %}
seg001:7C90 db 0DEh ; ¦
seg001:7C91 db 0C0h ; +
seg001:7C92 db 0ADh ; ¡
seg001:7C93 db 0DEh ; ¦
seg001:7C94 db 0B0h ; ¦
seg001:7C95 db 0B0h ; ¦
seg001:7C96 db 0D0h ; -
seg001:7C97 db 0BAh ; ¦
seg001:7C98 db    1
seg001:7C99 db    0
seg001:7C9A db    3
seg001:7C9B db    0
seg001:7C9C db    3
seg001:7C9D db    0
seg001:7C9E db    7
seg001:7C9F db    0
{% endhighlight %}
So it won't be too hard to calculate the 8 pins(also have in mind the little-endianness for pins). Go and type pins into **QEMU** 
![screenshot  of QEMU pins](http://{{ site.url }}/downloads/phdaysvi-2/pins.PNG)
Really, well done, lets try it in js version. And it fails. Time to look into the virtualization script that you can find in the source-code, if you search for a differences from the original file you will get this code
{% highlight ruby %}
if (49374 == d || 57005 == d || 45232 == d || 47824 == d) d = 65518;
{% endhighlight %}
Ok, **65518** = **ffee**.


## PWND!

## flag:<font color="red">ffeeffeeffeeffee00001000300030007</font>