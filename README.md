# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

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

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Kubernetes

Должен быть установлен kubectl и запущен кластер Kubernetes (далее представлены команды на примере кластера minikube). Для наглядности можете запустить панель Kubernetes:
```sh
minikube dashboard --url
``` 

### Как подготовить docker-образ сайта

Находясь в папке `backend_main_django` выполните команду:
```sh
docker build -t k8s-test_django_app
```

Затем доставьте образ внутрь кластера и убедитесь, что операция прошла успешно:
```sh
minikube image load k8s-test_django_app
minikube image ls
```

### Как безопасно доставить переменные в кластер

Закодируйте значение каждой переменной:
```sh
echo -n "<value>" | base64
```

В директории `manifests` создайте файл `Secrets.yaml` и поместите в него переменные с закодированными значениями:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: k8s-test-django-secrets-prod
  labels:
    Lesson: minikube_1
type: Opaque
data:
  # DEBUG: <base64 value>
  SECRET_KEY: <base64 value>
  DATABASE_URL: <base64 value>
  ALLOWED_HOSTS: <base64 value>
```

Передайте `Secrets.yaml` внутрь кластера следующей командой:
```sh
kubectl apply -f Secrets.yaml
```

### Как запустить сайт c помощью `deployment`

Находясь в директории `manifests`, выполните команду:
```sh
kubectl apply -f k8s-test-django-deployment.yaml
```

### Как обеспечить доступ к сайту c помошью LoadBalancer

Находясь в директории `manifests`, запустите сервис `LoadBalancer`:
```sh
kubectl apply -f LoadBalancer.yaml
```

Откройте доступ к сервису извне:
```sh
minikube service k8s-test-loadbalancer-service --url
```

### Как обеспечить доступ к сайту c помошью ClusterIP и Ingress

Включите minikube `Ingress`:
```sh
minikube addons enable ingress
```

Находясь в директории `manifests`, запустите сервис `ClusterIP`:
```sh
kubectl apply -f ClusterIP.yaml
```

И затем запустите сервис `Ingress`:
```sh
kubectl apply -f Ingress.yaml
```

На локальной машине создайте туннель:
```sh
minikube tunnel
```

### Как настроить автоматическое удаление устаревших сессий Django

Находясь в директории `manifests`, запустите сервис `CronJob`:
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
kubectl create job --from=cronjob/k8s-test-django-clearsessions-cronjob django-clearsessions-once
```

Добавьте параметр `-n <namespace>` при необходимости.

### Как применить миграции

Находясь в директории `manifests`, запустите команду:
```sh
kubectl apply -f django-migrate.yaml
```

## Цели проекта

Код написан в учебных целях — это урок в курсе по Python и веб-разработке на сайте [Devman](https://dvmn.org).

Где используется репозиторий:

- Первый урок [учебного курса Kubernetes](https://dvmn.org/modules/k8s/)
