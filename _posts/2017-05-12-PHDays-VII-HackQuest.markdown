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
Пробуем брутить и там и там, нигде не добиваемся успехов. Порыскав еще немного идет в wp-admin и пытаемся сделать сброс пароля, оказывается на сервере стоит конфигурация вывода ошибок (скорее всего WP_DEBUG,WP_DEBUG_DISPLAY включены), кстати эту же ошибку можно увидеть, если попытаться оставить комментарий 
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/beta-error.png)
Судя по ошибке, wordpress умирает от того что в поле Form, отправки мейла стоит значение "wordpress@_". Бежим на github вордпресса смотреть исходники функции wp_mail()
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/beta-wp-mail.png)
Судя по коду $_SERVER["SERVER_NAME"]="_". Во время просмотра сайта, в хидерах или же в выводе кода 404 можно было заметить что на сервере работает nginx. 
<br>Прочитав документацию узнаем, что в конфигурации nginx символ  "_" имеет специальное значени - им обозначается хостнейм по умолчанию (default), то есть, это хостнейм к которому быдет обращаться nginx если не будет других совпадений. 
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
<br>Конектимся у mysql
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
