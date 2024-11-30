# Развертывание на Nginx + uWSGI
(основано на https://uwsgi-docs.readthedocs.io/en/latest/tutorials/Django_and_nginx.html)

Схема взаимодействия:

the web client <-> the web server <-> the socket <-> uwsgi <-> Django

---

## Оглавление
  [1. Getting Started:](#title1)
  + [1.1 Прежде чем приступить к настройке uWSGI](#title1.1)
  + [1.2 Django](#title1.2)
    
  [2. Базовая установка и настройка uWSGI](#title2)
   + [2.1 Установите uWSGI в virtualenv](#title2.1)
   + [2.2 Протестируйте свой проект Django](#title2.2)
     
  [3. Основы работы с nginx](#title3)
   + [3.1 Установите nginx](#title3.1)
   +  [3.2 Настройте nginx для вашего сайта](#title3.2)
   +  [3.3 Развертывание статических файлов](#title3.3)
   + [3.4 Базовый тест nginx](#title3.4)
     
  [4. Nginx, uWSGI и test.py](#title4)
   + [4.1 Запуск приложения Django с помощью uwsgi и nginx](#title4.1)
   + [4.2 Настройка uWSGI на запуск с помощью .ini файла](#title4.2)
   +  [4.3 Установите uWSGI на всю систему](#title4.3)
   + [4.4 Режим императора](#title4.4)
     
  [5. Создание файла юнитов systemd для uWSGI](#title5)


   
   
## <a id="title1">Getting Started:</a> 


### <a id="title1.1"> 1.1. Прежде чем приступить к настройке uWSGI</a> 

Убедитесь, что вы находитесь в виртуальной среде (virtualenv) для программного обеспечения, которое нам нужно установить (позже мы опишем, как установить общесистемный uwsgi):

virtualenv uwsgi-tutorial
cd uwsgi-tutorial
source bin/activate

---


### <a id="title1.2">1.2 Django</a>

Установите Django в virtualenv, создайте новый проект и перейдите в него:

pip install Django
django-admin.py startproject mysite
cd mysite

---

## <a id="title2">Базовая установка и настройка uWSGI</a>
### <a id="title1.2">2.1 Установите uWSGI в virtualenv</a>

pip install uwsgi
Помните, что у вас должны быть установлены пакеты разработки Python. В случае Debian или производных от Debian систем, таких как Ubuntu, вам нужно установить pythonX.Y-dev, где X.Y - это ваша версия Python.

Запустите для проверки uWSGI:
uwsgi --http :8000 --wsgi-file test.py

Опции означают:

http :8000: использовать протокол http, порт 8000

wsgi-file test.py: загрузить указанный файл, test.py

Это должно передать сообщение 'hello world' непосредственно в браузер на порт 8000. Посетите:

http://localhost:8000

---
### <a id="title2.2">2.2 Протестируйте свой проект Django</a>
 
Теперь мы хотим, чтобы uWSGI делал то же самое, но запускал сайт Django вместо модуля test.py.

Если вы еще не сделали этого, убедитесь, что ваш проект mysite действительно работает:

python manage.py runserver 0.0.0.0:8000
И если это работает, запустите его с помощью uWSGI:

uwsgi --http :8000 --module mysite.wsgi

module mysite.wsgi: загрузка указанного модуля wsgi

Наведите браузер на сервер; если сайт появился, значит, uWSGI может обслуживать ваше Django-приложение из виртуальной среды, и этот стек работает корректно:

веб-клиент <-> uWSGI <-> Django

Обычно браузер не обращается напрямую к uWSGI. Это работа для веб-сервера, который будет выступать в качестве промежуточного звена.

---
## <a id="title3">Основы работы с nginx </a>
 
### <a id="title3.1">3.1 Установите nginx </a>



sudo apt-get install nginx

sudo /etc/init.d/nginx start # запустите nginx


А теперь проверьте, что nginx работает, зайдя на него через веб-браузер по порту 80 - вы должны получить сообщение от nginx: «Welcome to nginx!». Это означает, что эти компоненты полного стека работают вместе:

веб-клиент <-> веб-сервер


Если на 80-м порту уже работает что-то другое, а вы хотите использовать nginx для этого, вам придется перенастроить nginx на работу с другим портом. В этом учебнике мы будем использовать порт 8000.

---

### <a id="title3.2">3.2 Настройте nginx для сайта</a>



Вам понадобится файл uwsgi_params, который можно найти в директории nginx в дистрибутиве uWSGI или на сайте https://github.com/nginx/nginx/blob/master/conf/uwsgi_params.

Скопируйте его в каталог вашего проекта. Через некоторое время мы укажем nginx ссылаться на него.

Теперь создайте файл mysite_nginx.conf в каталоге /etc/nginx/sites-available/ и поместите в него следующее:

Этот conf-файл указывает nginx обслуживать медиа- и статические файлы из файловой системы, а также обрабатывать запросы, требующие вмешательства Django. При большом развертывании считается хорошей практикой, чтобы один сервер обслуживал статические/медийные файлы, а другой - приложения Django, но на данный момент и этого будет достаточно.

Сделайте ссылку на этот файл из /etc/nginx/sites-enabled, чтобы nginx мог его видеть:

sudo ln -s /etc/nginx/sites-available/mysite_nginx.conf /etc/nginx/sites-enabled/

---
### <a id="title3.3">3.3 Развертывание статических файлов</a>

Перед запуском nginx необходимо собрать все статические файлы Django в папке static. Для этого сначала нужно отредактировать файл mysite/settings.py, добавив в него:

STATIC_ROOT = os.path.join(BASE_DIR, «static/»)
а затем запустить

python manage.py collectstatic

---

### <a id="title3.4">3.4 Базовый тест nginx </a>


Перезапустите nginx:
sudo /etc/init.d/nginx restart

Чтобы проверить, правильно ли обслуживаются медиафайлы, добавьте изображение с именем media.png в каталог /path/to/your/project/project/media, а затем посетите http://example.com:8000/media/media.png - если это сработает, то вы будете знать, что nginx обслуживает файлы правильно.

Стоит не просто перезапустить nginx, а фактически остановить и затем запустить его снова, что позволит узнать, есть ли проблема и где она находится.

---

## <a id="title4">Nginx, uWSGI и test.py </a>
 
Давайте заставим nginx обратиться к приложению test.py «hello world».

uwsgi --socket :8001 --wsgi-file test.py
Это почти то же самое, что и раньше, за исключением того, что в этот раз одна из опций отличается:

socket :8001: использовать протокол uwsgi, порт 8001

Тем временем nginx был настроен на взаимодействие с uWSGI через этот порт, а с внешним миром - через порт 8000. Посетите:

http://localhost:8000/

чтобы проверить. И это наш стек:

веб-клиент <-> веб-сервер <-> сокет <-> uWSGI <-> Python


Тем временем вы можете попробовать посмотреть на вывод uwsgi по адресу http://localhost:8001 - но, скорее всего, это не сработает, потому что ваш браузер говорит на http, а не на uWSGI, хотя вы должны увидеть вывод uWSGI в терминале.

Использование сокетов Unix вместо портов
До сих пор мы использовали сокеты TCP-портов, потому что это проще, но на самом деле лучше использовать сокеты Unix, чем порты - меньше накладных расходов.

Отредактируйте файл mysite_nginx.conf, изменив его на соответствующий:

server unix:///path/to/your/mysite/mysite.sock; # для файлового сокета

#server 127.0.0.1:8001; # для сокета веб-порта (мы будем использовать его в первую очередь)

и перезапустите nginx.

Снова запустите uWSGI:

uwsgi --socket mysite.sock --wsgi-file test.py


На этот раз опция socket указывает uWSGI, какой файл использовать.

Попробуйте зайти на http://localhost:8000/ в браузере.

Если это не сработает
Проверьте журнал ошибок nginx (/var/log/nginx/error.log). Если вы видите что-то вроде:

connect() to unix:///path/to/your/mysite/mysite.sock failed (13: Permission
отказано)
то, вероятно, вам нужно изменить права на сокет, чтобы nginx мог его использовать.

Попробуйте:

uwsgi --socket mysite.sock --wsgi-file test.py --chmod-socket=666 # (очень разрешительно)
или:

uwsgi --socket mysite.sock --wsgi-file test.py --chmod-socket=664 # (более разумно).


Возможно, вам также придется добавить своего пользователя в группу nginx (которая, вероятно, является www-data) или наоборот, чтобы nginx мог правильно читать и писать в ваш сокет.

Стоит сохранить вывод журнала nginx в окне терминала, чтобы вы могли легко обратиться к нему при устранении неполадок.

### <a id="title4.1">4.1 Запуск приложения Django с помощью uwsgi и nginx </a>


Давайте запустим наше Django-приложение:

uwsgi --socket mysite.sock --module mysite.wsgi --chmod-socket=664

Теперь uWSGI и nginx должны обслуживать не просто модуль «Hello World», а ваш проект Django.

### <a id="title4.2">4.2 Настройка uWSGI на запуск с помощью .ini файла  </a>

Мы можем поместить те же опции, которые мы использовали в uWSGI, в файл, а затем попросить uWSGI работать с этим файлом. Это упрощает управление конфигурациями.

Создайте файл с именем `mysite_uwsgi.ini`:

И запустите uwsgi, используя этот файл:

uwsgi --ini mysite_uwsgi.ini # опция --ini используется для указания файла
Еще раз проверьте, что сайт Django работает так, как ожидалось.

### <a id="title4.3">4.3 Установите uWSGI на всю систему  </a>

Пока что uWSGI установлен только в нашей виртуальной среде, но для развертывания нам понадобится установить его во всей системе.

Деактивируйте свой virtualenv:

deactivate
и установите uWSGI на всю систему:

sudo pip install uwsgi

Или установите LTS (долгосрочную поддержку).
pip install https://projects.unbit.it/downloads/uwsgi-lts.tar.gz


На вики uWSGI описано несколько процедур установки. Перед установкой uWSGI на всю систему стоит подумать, какую версию выбрать и наиболее подходящий способ установки.

Еще раз проверьте, что вы можете запускать uWSGI так же, как и раньше:

uwsgi --ini mysite_uwsgi.ini # опция --ini используется для указания файла

### <a id="title4.4">4.4 Режим императора  </a>

uWSGI может работать в режиме «императора». В этом режиме он следит за каталогом конфигурационных файлов uWSGI и порождает экземпляры («вассалы») для каждого из них.

При изменении конфигурационного файла император будет автоматически перезапускать вассала.

создайте каталог для вассалов
sudo mkdir -p /etc/uwsgi/vassals
симлинк из каталога конфигурации по умолчанию в ваш файл конфигурации
sudo ln -s /path/to/your/mysite/mysite_uwsgi.ini /etc/uwsgi/vassals/
запустите императора
uwsgi --emperor /etc/uwsgi/vassals --uid www-data --gid www-data
Вам может потребоваться запустить uWSGI с правами sudo:

sudo uwsgi --emperor /etc/uwsgi/vassals --uid www-data --gid www-data
Опции означают:

emperor: где искать вассалов (файлы конфигурации)

uid: идентификатор пользователя процесса после его запуска

gid: групповой идентификатор процесса после его запуска

Проверьте сайт; он должен быть запущен.

## <a id="title5">Создание файла юнитов systemd для uWSGI </a>

Теперь у нас есть конфигурационные файлы, необходимые для обслуживания наших проектов Django, но мы все еще не автоматизировали этот процесс. Далее мы создадим юнит-файл systemd для автоматического запуска uWSGI при загрузке.

Мы создадим юнит-файл в каталоге /etc/systemd/system, где хранятся созданные пользователем юнит-файлы. Мы назовем наш файл uwsgi.service:

sudo nano /etc/systemd/system/uwsgi.service

Начните с раздела [Unit], который используется для указания метаданных и информации о заказе. Мы просто поместим сюда описание нашей услуги:
/etc/systemd/system/uwsgi.service
[Unit]
Description=uWSGI Emperor service
Далее мы откроем раздел [Service]. С помощью директивы ExecStartPre мы установим части, необходимые для запуска нашего сервера. Убедимся, что каталог /run/uwsgi создан и что наш обычный пользователь владеет им, а группа www-data является владельцем группы. И mkdir с флагом -p, и команда chown возвращаются успешно, даже если в их работе нет необходимости. Это то, что нам нужно.

Для фактической команды запуска, указанной директивой ExecStart, мы укажем на исполняемый файл uwsgi. Мы укажем ему запускаться в «режиме императора», что позволит ему управлять несколькими приложениями с помощью файлов, которые он найдет в /etc/uwsgi/sites. Мы также добавим элементы, необходимые systemd для корректного управления процессом. Они взяты из документации uWSGI здесь.

/etc/systemd/system/uwsgi.service
[Unit]
Description=uWSGI Emperor service

[Service]
ExecStartPre=/bin/bash -c 'mkdir -p /run/uwsgi; chown sammy:www-data /run/uwsgi'
ExecStart=/usr/local/bin/uwsgi --emperor /etc/uwsgi/sites
Restart=always
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all

