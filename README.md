# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Сайт в Docker

### Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

### Как запустить сайт для локальной разработки

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

### Как вести разработку

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


## Сайт в Minikube

### Установите minikube и kubectl себе на компьютер

- [Инструкция по установке](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/#установка-minikube)

- Создайте кластер:

  ```shell
  minikube start
  ```

### Соберите docker образ с django

Перейдите в директорию проекта и выполните следующую команду для сборки образа:

```shell
minikube image build ./backend_main_django/
```

### Установите helm и запустите в нем PostgreSQL

- [Инструкция по установке Helm](https://helm.sh/docs/intro/install/#from-script)
  
- Перейдите в директорию `k8s`, находящуюся в корне проекта, и создайте в ней файл `db-values.yaml` с параметрами DB:
  ```yaml
  global:
  postgresql:
    auth:
      username: "<db_username>"
      password: "<db_password>"
      database: "<db_name>"
  ```

- Выполните команды для развертывания БД:

  ```shell
  helm repo add bitnami https://charts.bitnami.com/bitnami
  ```

  ```shell
  helm install <name> bitnami/postgresql -f db-values.yaml
  ```

### Задайте секретные данные в манифест файле

- Перейдите в директорию `k8s`, находящуюся в корне проекта

- Создайте файл `Secrets.yaml` и наполните его следующим содержимым:

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: django-secrets
  type: Opaque
  stringData:
    SECRET_KEY: "<django secret_key>"
    DATABASE_URL: "postgres://<db_username>:<db_password>@<cluster_ip>/<db_name>"
  ```

- Запустите команду:

  ```shell
  kubectl apply -f secrets.yml
  ```

### Привяжите домен к локальной машине:

- Узнайте IP текущей ноды, выполнив следующую команду

  ```shell
  minikube ip
  ```

- Перейдите в файл `hosts`
  
  ```shell
  sudo nano /etc/hosts
  ```

- Поместите в конец файла следующую строчку:

  ```
  <minikube_ip> <domain_name>
  ```

### Запустите Django приложение

- Перейдите в директорию `k8s`, находящуюся в корне проекта

- Запустите приложение следующей командой:

  ```shell
  kubectl apply -f django-app.yml
  ```

- Запустите Cronjobs для удаления просроченных сессий и наката миграций:

  ```shell
  kubectl apply -f cronjobs.yml
  ```

### Результат

Перейдите по ссылке `<domain_name>`, указанном в файле `hosts`.
