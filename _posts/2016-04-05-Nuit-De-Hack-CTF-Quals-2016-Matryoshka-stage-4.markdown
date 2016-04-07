---
layout: post
title:  "Nuit du Hack CTF Quals 2016 Matryoshka stage 4(category:reverse)"
subtitle: ""
categories: [ctf]
---

This task was worth 500 points and its category was **reverse**. You must solve all the stages to get stage4 since it is "Matryoshka". Get the file stage4.bin. Run command **file** 
{% highlight ruby %}
stage4.bin: x86 boot sector;...
{% endhighlight %}
This is an MBR bin file. So the code will start at *0x0* entry point. Load up in IDA PRO and pick **16-bit mode**. We see that IDA cant cope with this kind of file and shows us messed up assembly. We need to undefine that entry part(*0x0* address) and make it code part.
List of actions 

- Press *'G'* goto *0x0*
- Press *'U'* to undefine the IDA heuristic analysis
- Press *'C'* to define it as a code part

Since IDA is flow-driven dissassembler it will to lots of work for you just from defining the right entry point.
{% highlight ruby %}
seg000:0000                 jmp     short loc_29
{% endhighlight %}

Give a close look to the procedures that lie between *0x0* and *loc_29*(*0x29*)

{% highlight ruby %}
seg000:0002 sub_2           proc near               
seg000:0002                                         
seg000:0002                 push    ax
seg000:0003                 add     sp, 2
seg000:0006                 pop     ax
seg000:0007                 inc     ax
seg000:0008
seg000:0008 loc_8:                                 
seg000:0008                 push    ax
seg000:0009                 sub     sp, 2
seg000:000C                 pop     ax
seg000:000D                 retn
seg000:000D sub_2           endp
seg000:000D
seg000:000E
seg000:000E ; =============== S U B R O U T I N E =======================================
seg000:000E
seg000:000E
seg000:000E sub_E           proc near               
seg000:000E                                         
seg000:000E                 push    ax
seg000:000F                 add     sp, 2           
seg000:0012                 pop     ax
seg000:0013                 inc     ax
seg000:0014
seg000:0014 loc_14:                                
seg000:0014                 inc     ax
seg000:0015                 push    ax
seg000:0016                 sub     sp, 2
seg000:0019                 pop     ax
seg000:001A                 retn
seg000:001A sub_E           endp
seg000:001A
seg000:001B
seg000:001B ; =============== S U B R O U T I N E =======================================
seg000:001B
seg000:001B
seg000:001B sub_1B          proc near               
seg000:001B                                         
seg000:001B                 push    ax
seg000:001C
seg000:001C loc_1C:
seg000:001C                 add     sp, 2
seg000:001F                 pop     ax
seg000:0020                 add     ax, 4
seg000:0023                 push    ax
seg000:0024                 sub     sp, 2
seg000:0027                 pop     ax
seg000:0028                 retn
seg000:0028 sub_1B          endp
seg000:0028
{% endhighlight %}
We see that **sub_2**, **sub_E** and **sub_1B** they change the return address this is done to obfuscate the assembly. Imagine breaking into instruction somewhere from middle. If you want to reverse this you need to do this steps whenever you see a **call sub_2/sub_E/sub_1B**

- Press *'U'* on the instruction right after the call command
- Press *'C'* on the byte according on the adjusted return value(+1/+2/+4)
- The rest IDA does for you

So we delt with the obfusctation but what the program does?
It loads *0x7c0* into **DS** and **ES** registers. 

{% highlight ruby %}
seg000:0079 loc_79:                                 
seg000:0079                 mov     ax, 7C0h
seg000:007C
seg000:007C loc_7C:                                 
seg000:007C                                         
seg000:007C                 call    sub_1B
seg000:007C ; ---------------------------------------------------------------------------
seg000:007F                 db 0CDh ; -
seg000:0080                 db  10h
seg000:0081                 db  66h ; f
seg000:0082                 db  66h ; f
seg000:0083
seg000:0083 ; =============== S U B R O U T I N E =======================================
seg000:0083
seg000:0083
seg000:0083 sub_83          proc near
seg000:0083                 cli
seg000:0084                 mov     ds, ax
seg000:0086                 call    sub_2           
seg000:0089                 retn
seg000:0089 sub_83          endp
seg000:0089
{% endhighlight %}

Then it loads *0x8000* into **SS** and *0xf000* into **SP**. This later part is not that much interesting as the data and extended registers. Basicly this part is pretty much strandard stage 1 mbr loader. After correcting all this he makes a long jmp but configures **DS** and **ES** as *0x100*.
{% highlight nasm %}
seg000:004A                 jmp     large far ptr 100h:0
{% endhighlight %}
So in memory this code will reside in *0x1000* but in the file it it not hard to see that this is  *0x200*, just after the large nopsled. The second stage of bootloader has same obfuscation techniques hence **loc_255** **loc_261** and **loc_26E**. This part of code prints of the screen useing **int 10h**
{% highlight ruby %}
Whats the secret word?
>
{% endhighlight %}
And then enters the loop for analyzing inputted keys
{% highlight ruby %}
seg000:0340                 int     16h             ; KEYBOARD - READ CHAR FROM BUFFER, WAIT IF EMPTY
seg000:0340                                         ; Return: AH = scan code, AL = character
seg000:0342                 cmp     al, 8
seg000:0344                 jz      short loc_37E
seg000:0346                 call    loc_255
seg000:0349                 db      66h
seg000:0349                 cmp     al, 20h ; ' '
seg000:034C                 jl      short loc_393
seg000:034E                 call    loc_255
seg000:0351                 retn
seg000:0352 ; ---------------------------------------------------------------------------
seg000:0352                 cmp     al, 7Eh ; '~'
seg000:0354                 jg      short loc_393
seg000:0356                 call    loc_261
seg000:0359                 nop
seg000:035A                 nop
seg000:035B                 cmp     bx, 0
seg000:035E                 jz      short loc_393
seg000:0360                 jmp     short loc_363
{% endhighlight %}
So the symbols must be between *0x20* and *0x7E* in the ASCII table the input is written in *0x1003*(*0x100**16+**SI**) . Then it moves *0x1003* into **EAX** and doea a long jmp
{% highlight ruby %}
seg000:04FE                 jmp     large far ptr 8:1400h
{% endhighlight %}
The code resides in *0x1400* in memory but in file it will be after another nopsled at the address *0x600*. This part is used to validate user input thus being the most interesting part for us. Unfortunately this is the most hash part to understand because the code is more randomly splited and using *undefine*+*define as code* technique be not that resultive. But still we could find out quite interesting code part
{% highlight ruby %}
seg000:06AB                 cmp     cl, 47h ; 'G'
seg000:06AE                 jnz     short loc_6C4
seg000:06B0                 inc     bx
seg000:06B1                 mov     cl, [bp+di]
seg000:06B3                 cmp     cl, 6Fh ; 'o'
seg000:06B6                 jnz     short loc_6C4
seg000:06B8                 inc     bx
seg000:06B9                 mov     cl, [bp+di]
seg000:06BB                 cmp     cl, 6Fh ; 'o'
seg000:06BE                 jnz     short loc_6C4
...
...
seg000:06E4                 mov     cl, [bp+di]
seg000:06E6                 cmp     cl, 64h ; 'd'
seg000:06E9                 jnz     short loc_6FD
seg000:06EB                 inc     bx
seg000:06EC                 mov     cl, [bp+di]
seg000:06EE                 cmp     cl, 5Fh ; '_'
seg000:06F1                 jnz     short loc_6FD
seg000:06F3                 inc     bx
seg000:06F4                 mov     cl, [bp+di]
seg000:06F6                 cmp     cl, 47h ; 'G'
seg000:06F9                 jnz     short loc_6FD
...
...
seg000:070C                 mov     cl, [bp+di]
seg000:070E                 cmp     cl, 61h ; 'a'
seg000:0711                 jnz     short loc_727
seg000:0713                 inc     bx
seg000:0714                 mov     cl, [bp+di]
seg000:0716                 cmp     cl, 6Dh ; 'm'
seg000:0719                 jnz     short loc_727
seg000:071B                 inc     bx
seg000:071C                 mov     cl, [bp+di]
seg000:071E                 cmp     cl, 65h ; 'e'
seg000:0721                 jnz     short loc_727
{% endhighlight %}
I started cheering up at this moment I went straightforward to validate the flag Good_Game, of course it's too short for a flag so it was missleading. At this point it is obvious that we need to debug somehow this MBR bootlader. And it did took some time to set up a comfortable environment for debugging it and I stopped at installing **QEMU**
![screenshot  of cmd running QEMU](http://{{ site.url }}/downloads/nuitdehack-matryoshka/cmd.PNG)
Unpause **QEMU** and attaching **IDA** with a remote **GDB** debugger. 
![screenshot  of cmd remote GBD](http://{{ site.url }}/downloads/nuitdehack-matryoshka/gbd.PNG)
Press *F9* in **IDA** and write in **Good_Game** as the input
![screenshot  of QEMU window](http://{{ site.url }}/downloads/nuitdehack-matryoshka/qemu.PNG)
We see that Good_Game was not that useless at least we have the "Magic word". We have a good point to start the debugging. Goto **IDA** and press *'G'* write there *0x1400*(already discussed why not *0x600*) and put a breakpoint there(*F2* button). Write anything that starts with **Good_Game** e.g **Good_Gameblablabla** since it just looks for the input to contain it and not necessarily be equal. Breakpoint hits. Now trace up the code with *F8*(over) or *F7*(into). At first time you may be confused since from some moment code becomes obscure and uses lots of *int* instructions but no calls to any procedures that seems to have payloads. This is just a misthinking and if you watch the code once more you will get the answer. Keep a close look at this line of code
{% highlight ruby %}
MEMORY:00001945 lidt    fword ptr byte_17FE
{% endhighlight %}
It loads the new **IDT** from the address *0x17FE*. **byte_17FE** should point at fword like this:
![screenshot  of IDT](http://{{ site.url }}/downloads/nuitdehack-matryoshka/lidt.jpg)
Goto hex view and take a look at the bytes pointed by the variable
{% highlight ruby %}
000017FE  40 01 04 18 00 00
{% endhighlight %}
Keep in mind that this is in little endian so we get **IDT base address:0x1800** and **IDT limit:0x140**. Now goto address *0x1800* and start creating code parts from there. Put breakpoints on each of them and continue. At some point we will come across this code while tracing.
{% highlight ruby %}
MEMORY:000017EC mov     eax, 169Ah
MEMORY:000017F1 mov     word_1904, ax
MEMORY:000017F7 call    loc_1944
{% endhighlight %}
There would be a couple of them and each with a different value in **EAX**, not hard to check that **EAX** points at some procedure and that this address(**word_1904**) actually resides in **IDT**. Examin **loc_1944**
{% highlight ruby %}
MEMORY:00001944 loc_1944:                               ; CODE XREF: MEMORY:000014D3p
MEMORY:00001944                                         ; MEMORY:00001536p ...
MEMORY:00001944 cli
MEMORY:00001945 lidt    fword ptr byte_17FE
MEMORY:0000194C mov     al, 11h
MEMORY:0000194E out     20h, al                         ; Interrupt controller, 8259A.
MEMORY:00001950 jmp     short $+2
MEMORY:00001952 ; ---------------------------------------------------------------------------
MEMORY:00001952
MEMORY:00001952 loc_1952:                               ; CODE XREF: MEMORY:00001950j
MEMORY:00001952 out     0A0h, al                        ; PIC 2  same as 0020 for PIC 1
MEMORY:00001954 jmp     short $+2
MEMORY:00001956 ; ---------------------------------------------------------------------------
MEMORY:00001956
MEMORY:00001956 loc_1956:                               ; CODE XREF: MEMORY:00001954j
MEMORY:00001956 mov     al, 20h ; ' '
MEMORY:00001958 out     21h, al                         ; Interrupt controller, 8259A.
MEMORY:0000195A jmp     short $+2
MEMORY:0000195C ; ---------------------------------------------------------------------------
MEMORY:0000195C
MEMORY:0000195C loc_195C:                               ; CODE XREF: MEMORY:0000195Aj
MEMORY:0000195C mov     al, 70h ; 'p'
MEMORY:0000195E out     0A1h, al                        ; Interrupt Controller #2, 8259A
MEMORY:00001960 jmp     short $+2
MEMORY:00001962 ; ---------------------------------------------------------------------------
MEMORY:00001962
MEMORY:00001962 loc_1962:                               ; CODE XREF: MEMORY:00001960j
MEMORY:00001962 mov     al, 4
MEMORY:00001964 out     21h, al                         ; Interrupt controller, 8259A.
MEMORY:00001966 jmp     short $+2
MEMORY:00001968 ; ---------------------------------------------------------------------------
MEMORY:00001968
MEMORY:00001968 loc_1968:                               ; CODE XREF: MEMORY:00001966j
MEMORY:00001968 mov     al, 2
MEMORY:0000196A out     0A1h, al                        ; Interrupt Controller #2, 8259A
MEMORY:0000196C jmp     short $+2
MEMORY:0000196E ; ---------------------------------------------------------------------------
MEMORY:0000196E
MEMORY:0000196E loc_196E:                               ; CODE XREF: MEMORY:0000196Cj
MEMORY:0000196E mov     al, 1
MEMORY:00001970 out     21h, al                         ; Interrupt controller, 8259A.
MEMORY:00001972 jmp     short $+2
MEMORY:00001974 ; ---------------------------------------------------------------------------
MEMORY:00001974
MEMORY:00001974 loc_1974:                               ; CODE XREF: MEMORY:00001972j
MEMORY:00001974 out     0A1h, al                        ; Interrupt Controller #2, 8259A
MEMORY:00001976 jmp     short $+2
MEMORY:00001978 ; ---------------------------------------------------------------------------
MEMORY:00001978
MEMORY:00001978 loc_1978:                               ; CODE XREF: MEMORY:00001976j
MEMORY:00001978 in      al, 21h                         ; Interrupt controller, 8259A.
MEMORY:0000197A and     al, 0EFh
MEMORY:0000197C jmp     short $+2
MEMORY:0000197E ; ---------------------------------------------------------------------------
MEMORY:0000197E
MEMORY:0000197E loc_197E:                               ; CODE XREF: MEMORY:0000197Cj
MEMORY:0000197E out     21h, al                         ; Interrupt controller, 8259A.
MEMORY:00001980 jmp     short $+2
MEMORY:00001982 ; ---------------------------------------------------------------------------
MEMORY:00001982
MEMORY:00001982 loc_1982:                               ; CODE XREF: MEMORY:00001980j
MEMORY:00001982 mov     al, 20h ; ' '
MEMORY:00001984 out     20h, al                         ; Interrupt controller, 8259A.
MEMORY:00001986 jmp     short $+2
MEMORY:00001988 ; ---------------------------------------------------------------------------
MEMORY:00001988
MEMORY:00001988 loc_1988:                               ; CODE XREF: MEMORY:00001986j
MEMORY:00001988 sti
MEMORY:00001989 retn
{% endhighlight %}
Seems confusing, right? Not that much if you take a look at I/O ports *0x20, 0x21, 0xA0, 0xA1* - these are the **PIC** port so he just dynamically changes the **ISR** for *0x20* interrupt and invokes it by masking **PIC** ports. All we need to do is to trace all the way in to this **ISR**'s. So lets start from the start, put breakpoints on **ISR**'s and hit **F9**. Goto **QEMU** and enter **Good_Gameblablabla** and start tracing.
Eventually we will come up with very interesting procedure
{% highlight ruby %}
MEMORY:00001467 sub_1467 proc near                      ; CODE XREF: MEMORY:000015AFp
MEMORY:00001467                                         ; MEMORY:000015EAp ...
MEMORY:00001467 push    edi
MEMORY:00001468 push    eax
MEMORY:00001469 push    esi
MEMORY:0000146A push    ebx
MEMORY:0000146B
MEMORY:0000146B loc_146B:                               ; CODE XREF: sub_1467+18j
MEMORY:0000146B                                         ; sub_1467+1Ej
MEMORY:0000146B cmp     eax, 0
MEMORY:0000146E jz      short loc_1489
MEMORY:00001470 mov     cl, [edi]
MEMORY:00001472 mov     dl, [esi]
MEMORY:00001474 xor     cl, dl
MEMORY:00001476 mov     [edi], cl
MEMORY:00001478 dec     eax
MEMORY:00001479 dec     ebx
MEMORY:0000147A inc     edi
MEMORY:0000147B inc     esi
MEMORY:0000147C cmp     ebx, 0
MEMORY:0000147F jg      short loc_146B
MEMORY:00001481 pop     ebx
MEMORY:00001482 push    ebx
MEMORY:00001483 sub     esi, ebx
MEMORY:00001485 jmp     short loc_146B
{% endhighlight %}
Here the **EDI** points to our input string and **ESI** points to some sequence of byte, it is obvious that it just XOR's them and puts back in **EDI**.
 **EDI** = **EDI** ^ **ESI**
 After that we will come up with this kind of functions
 {% highlight ruby %}
MEMORY:00001565 cli
MEMORY:00001566 mov     ebx, dword_19C7
MEMORY:0000156C mov     cl, [ebx]
MEMORY:0000156E cmp     cl, 28h ; '('
MEMORY:00001571 jnz     short loc_158F
MEMORY:00001573 add     ebx, 7
MEMORY:00001576 mov     cl, [ebx]
MEMORY:00001578 cmp     cl, 68h ; 'h'
MEMORY:0000157B jnz     short loc_158F
MEMORY:0000157D inc     ebx
MEMORY:0000157E mov     cl, [ebx]
MEMORY:00001580 cmp     cl, 0DFh ; '¯'
MEMORY:00001583 jnz     short loc_158F
MEMORY:00001585 inc     ebx
MEMORY:00001586 mov     cl, [ebx]
MEMORY:00001588 cmp     cl, 2Ch ; ','
MEMORY:0000158B jnz     short loc_158F
MEMORY:0000158D sti
MEMORY:0000158E iret
{% endhighlight %}
Now we undertstand why the compareble bytes are not alphanumeric(well, almost). having the index and the testing byte we can recover what the input string should be like.
Write a script or do it manually and you will get the flag.

## PWND!

## flag:<font color="red">Ddr1ml/frf</font>