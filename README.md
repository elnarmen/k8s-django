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
## Разертывание с помощью Minikube
Перед началом работы убедитесь что у вас установлены следующие инструменты:
* Инструмент для управления кластерами Kubernetes: [Kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/). 
* [Minikube](https://minikube.sigs.k8s.io/docs/)<br>

Если используете Linux, то этого достаточно. Можете запускать minikube:
```
minikube start --driver=docker
```
Для Windows необходимо установить гипервизор. Например, [virtualbox](https://www.virtualbox.org/wiki/Downloads).
В таком случае, команда для запуска minikube будет выглядеть следующим образом:
```
minikube start --driver=virtualbox
```
### Настройка и запуск базы данных
* Создайте файл `values.yaml` в директории проекта и заполните его по шаблону:
```
auth:
  enablePostgresUser: true
  postgresPassword: {your database password}
  username: {your username}
  password: {your user`s password}
  database: {your database`s name}
```
* Установите пакетный менеджер для Kubernetes - [Helm](https://helm.sh/)
Выполните следующие команды:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgres -f values.yaml bitnami/postgresql
```
Helm установит и запустит postgresql, а также создаст пользователя по параметрам, указанным в `values.yaml`.

### Настройка и запуск Django
* Создайте образ Django-приложения в кластере:
```
minikube image build -t ks8-django backend_main_django/
```
* Задайте нужные значения переменных окружения в файле конфигурации `k8s/django-secret.yaml`:
```
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
stringData:
  ALLOWED_HOSTS: ['*']
  DATABASE_URL: 'postgres://user_name:password@db_host:5432/db_name'
  DEBUG: 'False'
  SECRET_KEY: 'your-secret-key'

```
В DATABASE_URL укажите данные в формате `postgres://username:password@postgres:5432/database_name`, где username, password, database_name - данные из файла `values.yaml`
* Запустите `ConfigMap` командой:
```
kubectl apply -f k8s/django-secret.yaml
```
Значения переменных будут автоматически закодированы в формат `base64` при запуске конфига `django-secret`


* Запустите деплой:
```
kubectl apply -f k8s/deployment.yaml
```
* Запустите миграции:
```
kubectl apply -f k8s/django-migrate-job.yaml
```
* Создайте kron службу для очистки устаревших сессий:
```
kubectl apply -f k8s/django-clearsessions-cronjob.yaml
```
### Настройка Ingress
* В папке `/etc/hosts` добавьте строку:
```
minikube-ip-address star-burger.test
```
* Чтобы узнать IP-адрес minikube запустите команду:
```
minikube ip
```
* Включите Ingress Controller:
```
minikube addons enable ingress
```
* Запустите Ingress манифест:
```
kubectl apply -f k8s/ingress.yaml
```
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
