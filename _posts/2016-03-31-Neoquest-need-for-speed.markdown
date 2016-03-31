---
layout: post
title:  "Neoquest 2016 CTF \"Need for Speed:Catch-up\" (category: reverse)"
subtitle: "cheating the task"
categories: [ctf]
---

![screenshot  of task](http://{{ site.url }}/downloads/neoquest-nfs/task.jpg)

> Прилетев в Швейцарию, мы с ребятами не теряли времени зря. И вот мы уже так близки к цели! Преследуем машину политика, прорвавшись через охрану, в окне с бешеной скоростью мелькают люди, дома… За нами уже слышны полицейские сирены, поэтому водитель жмет на газ изо всех сил, а мы, как можем, помогаем ему разогнать тачку!

For those of you who dont know russian: we have a "car" and need to speed it up. Downloading the attached binary file and examining it shows **PE32** file format, since this is windows reverse task, it should be packed, right?
Open up the *RDG Packer Detector* or *Protection ID* the other packers seems not to work for this file. And this step is mandatory since binary doesn't seem to work in packed state.

## Click *Detect*

![screenshot  of task](http://{{ site.url }}/downloads/neoquest-nfs/rdg.jpg)
Now we know that its packed with **Themida + WinLicense 2.x.** 

To unpack it we need a bit instrumentary:

1. OllyDbg
2. ODBGScript
3. StrongOD
4. PhantOm
5. ARImpRec.dll
6. Themida - Winlicense Ultra Unpacker 1.4
7. and possibly a Win32 virual machine(this one not mandatory)

Not to provide you all the dirty procees through just google **"LCF-AT Themida"** and right the article on how to unpack **Themida** as this will cut the length of the article about twice. Now when we have the unpacked binary I loaded it up into **IDA PRO**. But soon we are going to fail because the IAT is corrupted as well as the symbol table. And it wouldn't be that much of a problem if the binary wasn't complied Golang. After a couple of tries to reverse Golang run-time I decided to run the program and watch what it does with **Promon.exe** from **Sysinternals tools**, turned out it openes a connection on TCP:8080.
![screenshot  of task](http://{{ site.url }}/downloads/neoquest-nfs/procexp.jpg)
Open-up browser and goto **localhost:8080**
![screenshot  of task](http://{{ site.url }}/downloads/neoquest-nfs/browser.jpg)
We can see that the "car" stops on 3rd gear and doesn't want to "speed-up".

## Examin the source code of the front-end
![screenshot  of task](http://{{ site.url }}/downloads/neoquest-nfs/ws.jpg)
It uses WebSockets to communitate to **localhost:8080/ws** and gets the commands from the back-end "car engine(binary)".

The commads consist of:

- Change Gear:{g:"somevalue"}
- Change tachometer:{t:"somevalue"}
- Break:{b:"somevalue"}
- Change Speed:{s:"somevalue"}

All this research continues with very lang and tedious and painfull try to reverse engineer the Golang run-time, recreate IAT and symbol tables, find the possible DWARF2 segments(lol wut?!), having no reslut I decided to take a break and to come back to this task when some new idea comes to me. And hopefully it came: *Lets cheat!*.

## Download CheatEngine

CheatEngine is really great tool for finding bytes of memories that hold the value that you want(even better, the one you dont even know!).

Here are the steps:

1. Load the binary in a debugger
2. Choose the car process in CheatEngine
3. Scan Type->Unknown initial value
4. First Scan
5. Open the second tab in CheatEngine and repeat the steps 3-4 (so one tab will be or tachometer and the other one for "speed" arrow)
6. Run the process and as soon as the arrows start to change
7. Pause the process in debugger and change do this in every tab: Scan Type->Increase value or Decrease value according the direction of arrow(if you are not shure use the Websocket tab in browser to see the exact values for this arrows)
8. Repeat this process untill you have one or two memory addresses for each tab
9. Change the values in this addresses for a bigger value
![screenshot  of task](http://{{ site.url }}/downloads/neoquest-nfs/flag.jpg)

## PWND!

## flag:<font color="red">bBVtgK6L4TjChonuP4vdSLG+6f7tmNTJnRXqmr+tJaI=</font>