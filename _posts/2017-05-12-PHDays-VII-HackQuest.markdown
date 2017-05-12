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
1)логин-пароль фейковые
2)Бот из блога ходит по нашим ссылкам
на этот таск был опубликован хинт, который нам и будет тут полезен
{% highlight ruby %}
Hint: IAICA — StyleSheets and JavaScripts are you best friends here!
Hint: IAICA — Look at the response headers
{% endhighlight %}
Во первых замечаем что имена js и css файлов это md5('имя-файла-без-расширения') например можно заметить файл a1b01e734b573fca08eb1a65e6df9a38.css
Смотрим на разницу в хидерах при выдаче результата от php и при обращении к js/css ресурсам 
php:
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
js/css:
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
Основное различие здесь в хидерах кеширования, js/css -  кешируются, а остальные - нет причем.
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
Соответственно результат вывода будет кешироватся браузером на протяжении 3х минут - то есть столько же она пробудет в fastcgi_cache
Так как в основном id закешированного файла в fast_cgi должен создаваться на основе url-a:
{% highlight ruby %}
1.генерируем случайный md5
2.Идем на /blog/ и отправляем бота на /md5().css
3.Проверяем чтобы статус поста был DECLINED(значит, что бот прошел по ссылке)
4.Сразу же идем тоже по тому же адресу /md5().css
{% endhighlight %}
Внизу сайта замечаем:
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/iaica-admin.png)
Появилась Admin Panel, которая ведет на /a6805bde-30c8-4536-9137-39b32eb28218/ и там нам снова отказывают в доступе
![screenshot  of running program](http://{{ site.url }}/downloads/phdaysvii/iaica-panel.png)
Проделываем шаги описанные выше но уже отправляя бота на /a6805bde-30c8-4536-9137-39b32eb28218/md5().css
В конце концов видем Admin Panel
{% highlight ruby %}
IAICA secret phrase is "Hell yeah bro! You did it! Man, you blow up this fake anti-corruption organization."
{% endhighlight %}


## PWND!

## flag:<font color="red">md5("Hell yeah bro! You did it! Man, you blow up this fake anti-corruption organization.")</font>
