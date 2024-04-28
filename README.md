# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Сайт можно посмотреть [здесь](https://edu-focused-lamarr.sirius-k8s.dvmn.org).

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

Данная инструкция относится к ветке k8s-Lesson2. В репозитории также есть ешё ветка k8s-Lesson1, посвященная предыдущему уроку по minikube.



## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.



## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).



## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.


### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```


### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>



## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema). Если будете подключать БД PostgreSQL, находящуюся в том же кластере, что и Django Site, укажите вместо ip адреса `postgresql-k8s-test` или то название, которое выберете при разворачивании PostgreSQL.

При работе с django манифестом переменная `DATABASE_URL` соберётся из пяти следующих:
`username` -- имя пользователя PostgreSQL
`password` -- пароль
`host` -- хост БД
`port` -- порт
`name` -- название БД



## Как подготовить docker-образ сайта

Находясь в директории `backend_main_django`, экспортируйте переменную с хэшем последнего коммита:
```sh
export COMMIT_HASH=$(git rev-parse k8s-Lesson2)
```

Соберите образ, используя хэш последнего коммита в качестве тега:
```sh
docker build -t <DockerHub_username>/k8s-test_django_app:$COMMIT_HASH .
```

Затем загрузить образ в DockerHub:
```sh
docker push <DockerHub_username>/k8s-test_django_app:$COMMIT_HASH
```
Как варинат, можете воспользоваться уже имеющимся - `jmuriki/k8s-test_django_app`.



## Kubernetes


Должен быть установлен kubectl и запущен кластер Kubernetes.

Это [ссылка](https://sirius-env-registry.website.yandexcloud.net/edu-focused-lamarr.html) на описание выделенных ресурсов облачной инфраструктуры Yandex с инструкциями как до них добраться.


### Как безопасно доставить переменные в кластер

Закодируйте значение каждой переменной:
```sh
echo -n "<value>" | base64
```

Для настроек Django cоздайте файл `Secrets_django.yaml` и поместите в него переменные с закодированными значениями:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets-prod
  namespace: <namespace>
  labels:
    Lesson: kubernetes_2
    branch: k8s-Lesson2
    owner: <you>
    env: prod
type: Opaque
data:
  DEBUG: <base64 value>
  SECRET_KEY: <base64 value>
  ALLOWED_HOSTS: <base64 value>
```

Для настроек PostgreSQL DATABASE_URL cоздайте файл `Secrets_postgres.yaml` и поместите в него переменные с закодированными значениями (в выделенной инфраструктуре этот файл уже создан):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres
  namespace: <namespace>
  labels:
    Lesson: kubernetes_2
    branch: k8s-Lesson2
    owner: <you>
    env: prod
type: Opaque
data:
  username: <base64 value>
  password: <base64 value>
  host: <base64 value>
  port: <base64 value>
  name: <base64 value>
```

Передайте `Secrets_django.yaml` и `Secrets_postgres.yaml` внутрь кластера:
```sh
kubectl apply -f Secrets_django.yaml
kubectl apply -f Secrets_postgres.yaml
```
Добавьте параметр `-n <namespace>` при необходимости.


### Как подключить SSL сертификат с помощью Secret и Volume

Для доступа к облачной СУБД PostgreSQL может понадобится действующий SSL сертификат.

Создайте простой Secret-файл непосредственно в кластере из локального SSL-файла:
```sh
kubectl create secret generic secret-ssl-certificate --from-file=<ssl_filename>
```
Добавьте параметр `-n <namespace>` при необходимости.

Название `secret-ssl-certificate` используется в манифесте вашего pod, чтобы примонтировать `secret-ssl-volume` к директории `/root/.postgres/`, в которой будет содержаться сертификат. Этот ход нужен для того, чтобы сертификат можно было менять на лету, без необходимости пересобирать pod.


### Как применить миграции

Запустите команду:
```sh
kubectl apply -f django-migrate.yaml
```
Добавьте параметр `-n <namespace>` при необходимости.


### Как запустить сайт c помощью манифеста

Убедитесь, что все переменные загружены в кластер, и выполните команду:
```sh
kubectl apply -f django.yaml
```
Добавьте параметр `-n <namespace>` при необходимости.


### Как создать superuser

Зайдите в контейнер с сайтом:
```sh
kubectl exec -it <podname> -- sh -c "clear; (bash || ash || sh)"
```
Добавьте параметр `-n <namespace>` при необходимости.

Создайте superuser:
```sh
python manage.py createsuperuser
```


### Как настроить автоматическое удаление устаревших сессий Django

Запустите сервис `CronJob`:
```sh
kubectl create -f django-clearsessions-cronjob.yaml
```

Сервис `cronjob` настроен на срабатывание раз в сутки. Убедиться в том, что он работает, можно c помощью следующих команд:
```sh
kubectl get job
kubectl get job --watch
kubectl get cronjob
kubectl get cronjob --watch
```

Чтобы `job` сработал принудительно, вне расписания, используйте команду:
```sh
kubectl create job --from=cronjob/django-clearsessions-cronjob django-clearsessions-once
```

Добавьте параметр `-n <namespace>` при необходимости.



## Цели проекта

Код написан в учебных целях — это урок в курсе по Python и веб-разработке на сайте [Devman](https://dvmn.org).

Где используется репозиторий:

- Второй урок [учебного курса Kubernetes](https://dvmn.org/modules/k8s/)
