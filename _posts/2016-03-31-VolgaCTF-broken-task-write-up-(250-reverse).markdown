---
layout: post
title:  "VolgaCTF 2016 write-up on \"broken\" (category: reverse)"
subtitle: ""
categories: [ctf]
---
![screenshot  of task](http://{{ site.url }}/downloads/volga-broken/task.jpg)
So we can see a bit of legend on the task and a link to binary. After downloading it a quick examination of file showed that it is ELF64 binary.
Turn on IDA PRO and load the binary all the standard way. A quick checking on strings doesn't give us much
![screenshot  of strings](http://{{ site.url }}/downloads/volga-broken/strings.jpg)
Ok running program gives us almost nothing as well - it just waits 30 seconds and exits with:
{% highlight ruby %}
"The processing has taken too long, terminating.."
{% endhighlight %}
## Lets proceed to the main function
![screenshot  of main thread](http://{{ site.url }}/downloads/volga-broken/main.jpg)

As you can see I've already renamed some functions so you can understand what are they doing. The main function starts 4 threads and then waits for them to end (by calling <b>pthread_join</b>). After examining all threads it turned out that only 2 of them are interesting to us: the first 2 does some hashing and maths(supposedly for crypting the flag)
The fourth thread looks like this
 ![screenshot  of exit thread](http://{{ site.url }}/downloads/volga-broken/exit_thread.jpg)
As you can see thats the thread which kills us after sleeping for 30 sec, and I've already patched a call to <b>exit()</b> so it wont challenge us to debug it. Now look at the 3rd thread
 ![screenshot  of 3rd thread](http://{{ site.url }}/downloads/volga-broken/lock_graph.jpg)
This function is quite harsh to understand from the start but it worked out to be much easier. What is does: at first it initializes three semaphores: <b>stru_6034E0</b>, <b>stru_603500</b>, <b>sem</b>
![screenshot  of 3rd thread](http://{{ site.url }}/downloads/volga-broken/sem_init.jpg)
After that is spawns two threads

## Thread 1:
![screenshot  of 3rd thread](http://{{ site.url }}/downloads/volga-broken/thread1.jpg)

## Thread 2:
![screenshot  of 3rd thread](http://{{ site.url }}/downloads/volga-broken/thread2.jpg)
This threads firstly signal the semaphore <b>sem</b> then wait for a command to start  their "payload" through semaphore <b>stru_603500</b> (some of you may already understood what was causing the problem) after the "math" they do they fire semaphore <b>sem</b> again to inform their parent thread about finishing the task.
Here is the code of parent thread function after it inits the semaphore
![screenshot  of 3rd thread](http://{{ site.url }}/downloads/volga-broken/sem_run_waith.jpg)
Now we understand that the root of the problem is the deadlock that was caused by a mistake of the programmer, as one of the threads should wait on <b>stru_6034E0</b> instead of both waiting on <b>stru_603500</b>
Here is the diagram of semaphores if you need more explanation:
![screenshot  of 3rd thread](http://{{ site.url }}/downloads/volga-broken/diagram.svg)
So we go to the HEX-view in IDA press F2 and change the instruction to point to <b>stru_6034E0</b>
![screenshot  of 3rd thread](http://{{ site.url }}/downloads/volga-broken/patch1.jpg)
After we press F2 once again to save our patch and go to <b>Edit->Patch Program->Apply patches to input file..</b>
We run program once again... Now it dies. We load the progam into debugger and put a breakpoint in the <b>main</b>, it turns out it doesnt even enter <b>main</b> function so we restart the process but this time we trace till the moment it dies(it should be easier to watch the backtrace, but I just wanted to see the exact flow) and some moment we fall into this function
![screenshot  of 3rd thread](http://{{ site.url }}/downloads/volga-broken/hash.jpg)
So this function calculates hash for the code and checks it with the predefined value in binary, just patch this function to return always 0 in rax
Run the program for the last time
![screenshot  of 3rd thread](http://{{ site.url }}/downloads/volga-broken/flag.jpg)

## PWND!

## flag:<font color="red">VolgaCTF{avoid_de@dl0cks_they_br3ak_your_@pp}</font>