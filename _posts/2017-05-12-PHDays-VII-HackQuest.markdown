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

<br>Так как код там большой, чтобы показать в скриншоте, покажу лишь основную часть функции и то уже в рипнутом виде на язык C(пришлось использовать Hex-rays и потом еще его проправлять долго)
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/ford-cipher.png)
Понимаем, что тут самописный block cipher, с 129 раундами и двумя Sbox-ами(это byte_4031A0[] и byte_4032A0[]). причем первые 128 раундов проделываются при помощи byte_4031A0(далее будем называть его Sbox1), и лишь последний с byte_4032A0(Sbox2  соответственно), уже понимаем, что загруженный в esi массив - плейнтекст, в ebx - шифротекст, а нам серийник является ключем шифрования. В первых 128 раундах, после подстановки Sbox1, вывод также xor-ится с ключем(плюс еще делается xor с (i%8)+1, где i - номер раунда), а также делается некий шифр. во время которого для каждого байта делается следующее - оставляются нижние 4 бита, а верхние 4 бита заменяются верхними следующего бита, если речь идет о последнем байте result+7, то верхние для него беруть и первого байта result+0.
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
Или же, иными словами, что 2й бит всех байтов остается неизменным при подстновке Sbox1, вспомнив как работает шифт в этом шифре получаем, что 2й бит неизменен вплоть до самого последнего раунда(не включительно, так как для Sbox2 соотношение не выполняется), а как же xor c (i%8)+1, где i - номер раунда
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
Тут дело в том, что номер раунда берется по модулю 8, а значит xor у нас будет с этим битом, только в 1 из 8 случаев, получаем 128/8=16, 16 - четное, а xor с одним и тем же четное количество раз ничего не изменяет.
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
