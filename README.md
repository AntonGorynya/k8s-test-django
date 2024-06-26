# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).
# Демо
- Под доступен по адрессу https://edu-goofy-allen.sirius-k8s.dvmn.org/
- Выделенные [ресурсы](https://sirius-env-registry.website.yandexcloud.net/edu-goofy-allen.html) 


## Как запустить local-версию

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


# Запуск в dev окружении

Установите helm
```commandline
choco install kubernetes-helm
```
Добавьте репозиторий
```commandline
helm repo add bitnami https://charts.bitnami.com/bitnami
```
Установите postgres
```commandline
helm install my-postgresql bitnami/postgresql --version 13.2.28
```
Выясняем пароль к БД
```commandline
$POSTGRES_PASSWORD = kubectl get secret --namespace default my-postgresql -o jsonpath="{.data.postgres-password}" 
$result = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($POSTGRES_PASSWORD))
```
Создадим секретный ключ вашего проекта на django Вы можете создать ключ выполнив команду
```commandline
$secret_key = python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'
```

Создаем `secret` из 2х переменных:
```commandline
kubectl create secret generic django-secret-config --from-literal=SECRET_KEY=$secret_key --from-literal=DATABASE_URL=postgres://postgres:$result@my-postgresql.default.svc.cluster.local:5432/postgres 
```

Заполните переменные окружения в файле `django-config.yml`
После чего примените манифесты
```commandline
kubectl apply -f django-config.yml
kubectl apply -f .\deployment.yaml
kubectl apply -f .\service.yml
kubectl.exe apply -f .\ingress.yml 
kubectl.exe apply -f .\migrate-job.yml 
kubectl.exe apply -f .\delete-sessions-cronjob.yml
```
Что бы узнать IP  назначенный на сервис выполните
```commandline
kubectl.exe get svc
```

Для доступа на сайт из локальной машины можете либо поднять туннель и получить доступ по IP

```commandline
minikube tunnel
```
либо проверьте назначенный IP на сервис и пропишите его в файле `hosts` задав необходимое для вас имя.
```commandline
10.97.40.183  star-burger.test www.star-burger.test
```

# Запуск в yandex-sirius облаке
Установите `kubernetes-cli`
```commandline
choco install kubernetes-cli
```

Установите и инициализируете консоль управления облаком
```commandline
iex (New-Object System.Net.WebClient).DownloadString('https://storage.yandexcloud.net/yandexcloud-yc/install.ps1')
yc init
```
Задайте `context` и `namespace`
```commandline
kubectl config use-context yc-sirius
kubectl config set-context --current --namespace=edu-goofy-allen
```
Создадим секретный ключ вашего проекта на django Вы можете создать ключ выполнив команду
```commandline
$secret_key = python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'
```
Закодируйте ключ в base64
```commandline
$encodedBytes = [System.Text.Encoding]::UTF8.GetBytes($secret_key)
$base64String = [System.Convert]::ToBase64String($encodedBytes)
```
Добавте ключ в `secret`
```commandline
kubectl edit secrets django-secret-config
```

Cкачайте сертификат 
```commandline
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem"
```
Создайте секрет
```commandline
kubectl create secret generic psql-cert --from-file=root.crt
```

Перейдите в папку `yc-sisrius` и примените конфигурационные файлы
```commandline
kubectl apply -f django-config.yml
kubectl apply -f yc-service.yml
kubectl.exe apply -f ingress.yml 
kubectl.exe apply -f yc-deployment.yml 
```


## Создание  superuser
При первом запуске так же необходимо создать superuser
Перейдите в нужный контейнер `django-container`
```commandline
kubectl exec -it app-deployment-7f6d7fb5c8-dsrjj  -c django-container -- /bin/sh
```
и создайте superuser
```commandline
python manage.py createsuperuser
```
# Создание Docker образа
Вы можете использовать готовый Docker образ, который залит на DockerHub. Так же есть возможность создать свой собственный.
Создайте репозиторий на [DockerHub](https://hub.docker.com/)
Для этого перейдите в папку `backend_main_django` и соберите образ
```commandline
docker build  -t yournaname/appname:tagname . 
```
Сделайте `push`
```commandline
docker login
docker push yournaname/appname:tagname
```