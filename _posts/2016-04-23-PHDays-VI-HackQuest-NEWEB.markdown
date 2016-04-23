---
layout: post
title:  "PHDays VI HackQuest NEWEB(category:reverse)"
subtitle: ""
categories: [ctf]
---
Download the exe file. Run any packer detector it says it's **UPX packer**, so it won't be a big deal. If you run the program without debugger and don't unpack it you will se something like this.
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvi/wallarm.PNG)
But if you do unpack or run with debugger you get redirected to youtube with **ShellExecute API**. So instead run **procmon** and watch the programs behaviour. You will evetually see that it create **w.exe** in temp directory, so it should be what we need. Copy it to somewhere more reliable and open it up in **IDA**. Take a glance on functions subview
![screenshot  of funcs](http://{{ site.url }}/downloads/phdaysvi/funcs.PNG)
There is a **DialogFunc** but the main logic of serial checking goes in **sub_401000**. It first checkes for serial to have 32 characters. 
{% highlight ruby %}
.text:00401010                 inc     eax
.text:00401011                 cmp     byte ptr [eax+ecx], 0
.text:00401015                 jnz     short loc_401010
.text:00401017                 cmp     eax, 20h
.text:0040101A                 jz      short loc_401025
{% endhighlight %}
After that it changes the serial a bit and adds a weird(for the first time) rule for the serial
{% highlight ruby %}
movsx   eax, byte ptr [ecx]
movsx   esi, byte ptr [ecx+1]
lea     eax, [eax+eax*4]
lea     eax, [esi+eax*2-210h]
add     ecx, 2
mov     [ebp+edx*4+var_44], eax ;save the result into stack
cmp     eax, 40h
jge     short loc_40101C
{% endhighlight %}
This code converts the 2 symbols into a byte and then checks a formula 
{% highlight ruby %}
ord(serial_as_byte[i+1])+ord(serial_as_byte[i])*10 < 0x250
{% endhighlight %}
And at the end it runs this code
{% highlight ruby %}
mov     ebx, [ebp+var_4] ;read the first result from the stack
call    $+5
pop     eax
add     eax, [ebx]
add     eax, [ebx]
add     eax, 0Ah
call    eax              ;use it as an offset to run code
{% endhighlight %}
The logic is being explained in my comments. So what can we do with this? Lets open up strings subview in **IDA**
![screenshot  of funcs](http://{{ site.url }}/downloads/phdaysvi/strings.PNG)
*"You do it!"* that seems interesting. Double click and go to **DATA XREFS** of the string.
{% highlight ruby %}
.text:00401983 loc_401983:                             ; CODE XREF: .text:00401969j
.text:00401983                 pop     eax
.text:00401984                 add     ebx, 4
.text:00401987                 popa
.text:00401988                 mov     ecx, [ebp+8]
.text:0040198B                 pop     edi
.text:0040198C                 pop     esi
.text:0040198D                 mov     dword ptr [ecx], offset aYouDoIt ; "You do it!"
.text:00401993                 mov     al, 1
.text:00401995                 pop     ebx
.text:00401996                 mov     esp, ebp
.text:00401998                 pop     ebp
.text:00401999                 retn
{% endhighlight %}
This part is referenced as code in **.text:00401969**, here it is
{% highlight ruby %}
.text:00401967 loc_401967:                             ; CODE XREF: .text:loc_40191Dj
.text:00401967                 jmp     short loc_401959
.text:00401969 ; ---------------------------------------------------------------------------
.text:00401969                 jmp     short loc_401983 ;this one is the one!
.text:0040196B ; ---------------------------------------------------------------------------
{% endhighlight %}
But as we see this jmp is not referenced by anyone, at it's far too ahead from the **call eax** instruction in **sub_401000**, so what we do? Examin this jmp "ocean" if you are careful you will sometimes come up with a code like this.
{% highlight ruby %}
.text:004018EE loc_4018EE:                             ; CODE XREF: .text:loc_401870j
.text:004018EE                 jmp     short loc_4018A2
.text:004018F0 ; ---------------------------------------------------------------------------
.text:004018F0
.text:004018F0 loc_4018F0:                             ; CODE XREF: .text:004018C2j
.text:004018F0                 pop     eax
.text:004018F1                 add     ebx, 4
.text:004018F4                 call    $+5
.text:004018F9                 pop     eax
.text:004018FA                 add     eax, [ebx]
.text:004018FC                 add     eax, [ebx]
.text:004018FE                 add     eax, 0Ah
.text:00401901                 call    eax
.text:00401903
.text:00401903 loc_401903:                             ; CODE XREF: .text:loc_40192Fj
.text:00401903                 jmp     short loc_401957
.text:00401905 ; ---------------------------------------------------------------------------
{% endhighlight %}
So there are hidden **call** parts in this jmp section. Search for the "call eax" command in **IDA** and you will get a list of 16 call commands, just ideal for our 32 character serial. All is needed to do, is to calculate the right offsets to make a chain of **call eax** code commands

## PWND!

## flag:<font color="red">numerical variant of the serial key you found</font>