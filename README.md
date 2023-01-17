# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Как запустить prod-версию (на примере Minikube)
* Minikube и kubectl должны быть установлены
* Запустите minikube:
```commandline
minikube start --driver=virtualbox
```
* Переопределите переменные окружения в файле `django-config-map.yaml`.
* Создайте images:
```commandline
docker-compose up
```
* Загрузите image `django_app` в кластер:
```commandline
minikube image load django_app 
```
* Загрузите конфигурационный файл в кластер:
```commandline
kubectl apply -f kubernetes/django-config-map.yaml
```
* Разверните сайт:
```commandline
kubectl apply -f kubernetes/django-deployment.yaml
```
* Включите надстройку minikube ingress (в случае возникновения проблем, попробуйте minikube версии v1.23.2):
```commandline
minikube addons enable ingress
```
* Создайте объект Ingress:
```commandline
kubectl apply -f kubernetes/django-ingress.yaml
```
* Узнайте ip minikube:
```commandline
minicube ip
```
* Чтобы сайт [star-burger.test](http://star-burger.test) был доступен, добавьте в файл `/etc/hosts`:
```
<ваш ip minikube> star-burger.test
```
* Запустите регулярное удаление сессий раз в месяц (1-го числа в полночь):
```commandline
kubectl apply -f kubernetes/django-clearsessions.yaml
```
* Сделайте миграции базы данных:
```commandline
kubectl apply -f kubernetes/django-migrate.yaml
```