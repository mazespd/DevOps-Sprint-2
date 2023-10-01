Cпринт 2

ЗАДАЧА

```
1. Клонируем репозиторий, собираем его на сервере srv.
Исходники простого приложения можно взять здесь. Это простое приложение на Django с уже написанным Dockerfile. 
Приложение работает с PostgreSQL, в самом репозитории уже есть реализация docker-compose — её можно брать за 
референс при написании Helm-чарта. Необходимо склонировать репозиторий выше к себе в Git и настроить пайплайн 
с этапом сборки образа и отправки его в любой docker registry. Для пайплайнов можно использовать GitLab, 
Jenkins или GitHub Actions — кому что нравится. Рекомендуем GitLab.

2. Описываем приложение в Helm-чарт.
Описываем приложение в виде конфигов в Helm-чарте. По сути, там только два контейнера — с базой и приложением, 
так что ничего сложного в этом нет. Стоит хранить данные в БД с помощью PVC в Kubernetes.

3. Описываем стадию деплоя в Helm.
Настраиваем деплой стадию пайплайна. Применяем Helm-чарт в наш кластер. Нужно сделать так, чтобы наше приложение 
разворачивалось после сборки в Kubernetes и было доступно по бесплатному домену или на IP-адресе с выбранным портом.
Для деплоя должен использоваться свежесобранный образ. По возможности нужно реализовать сборку из тегов в Git, где 
тег репозитория в Git будет равен тегу собираемого образа. Чтобы создание такого тега запускало пайплайн на сборку 
образа c таким именем hub.docker.com/skillfactory/testapp:2.0.3.
```

РЕШЕНИЕ

1) Клонируем репозиторий, собираем его на сервере srv.
```
git clone https://github.com/vinhlee95/django-pg-docker-tutorial.git
```
Собираем приложение:
```
docker compose up -d --project-name dilpom-app
```
Можно посмотреть, что контейнеры успешно запущены:

И по URL открывается сайт:


Удаляем контейнеры:
```
docker compose down
```

2)  

```
Необходимо склонировать репозиторий выше к себе в Git и настроить пайплайн с этапом сборки образа и отправки его в любой docker registry. Для пайплайнов можно использовать GitLab, Jenkins или GitHub Actions — кому что нравится. Рекомендуем GitLab.
```

Для хранения Docker Image используется Docker Hub.
Сначала нужно залогиниться в Docker Hub.
Команда: 
```
docker login -u mazespd
```

Для того, чтобы доставить приложение на сервер испольуется Gitlab CI/CD.
Ссылка на созданный репозиторий: https://gitlab.com/devops1411739/ci-cd
Для запуска pipeline нужно к проекту прикрепить Runner.
Для этого есть раздел 
```
Settings -> CI/CD -> Runners -> New Project Runner.
```

После того, как Runner был создан, нужно на ВМ srv его настроить и запустить:
```
gitlab-runner register --url https://gitlab.com --token {{ токен полученный при создании runner }}
gitlab-runner run
```
Для запуска pipeline нужно создать переменные в разделе:
```
Settings -> CI/CD -> Variables
```
Список переменных:
```
NAMESPACE - имя namespace в kubernetes
DB_HOST - хост базы данных
DB_NAME - имя базы данных
DB_PASS - пароль от базы данных
DB_USER - пользователь базы данных
POSTGRES_DB - postgres база данных
POSTGRES_PASSWORD - postgres пароль
POSTGRES_USER - postgres пользователь
REG_PASSWORD - пароль от docker hub
REG_USER - пользователь от docker hub
```
В разделе 
```
Build -> Pipeline Editor
```
Нужно описать шаги, которые должен выполнить runner.
Так же можно это сделать в файле .gitlab-ci.yml.
Сначала описываем сборку приложения и отправку его в Docker Hub.
```
variables:
  TAG: "v1.1"

stages:
  - build
  - deploy

docker_build:
  stage: build
  rules:
    - if: $TAG =~ /^v\d+.\d+/
      exists:
        - Dockerfile
    - when: on_success
    - when: manual

  before_script:
     - sudo docker login -u "$REG_USER" -p "$REG_PASSWORD"

  script:
    - export
    - tag='$TAG'
    - echo "Running build with tag $TAG "
    - sudo docker build -t mazespd/sprint2:$TAG .
    - sudo docker push mazespd/sprint2:$TAG
```
Если job выполнился успешно, тогда вывод будет следующий:

Результат можно посмотреть в Docker Hub:

3) В каталоге kubernetes описываем все нужные манифесты:
```
https://github.com/mazespd/DevOps-Sprint-2/tree/main/kubernetes
```

Чтобы развернуть приложение на нашем ранее созданном кластере, используется следующая команда:
```
kubectl apply -f . -n diplom
 ```
 
 Успешно развёрнутое приложение выглядит так:
 
 
 Соответственно приложение доступно по двум адресам на порту 30773
```
http://51.250.104.179:30773
http://130.193.54.175:30773
```


4) Описываем приложение в Helm-чарт.
В каталоге, где находятся манифесты kubernetes используем следующие команды:
```
helm create app-dpdt
tree app-dpdt
```

Разорачиваем chart:
```
helm upgrade --install -n diplom --values templates/credentials.yaml --set service.type=NodePort app-dpdt .
```
Архивируем chart и заливаем в gitlab.
Дорабатываем pipeline:
```
---
variables:
  TAG: "v1.1"
  CI_NAMESPACE: "diplom"

stages:
  - build
  - deploy

docker_build:
  stage: build
  rules:
    - if: $TAG =~ /^v\d+.\d+/
      exists:
        - Dockerfile
    - when: on_success
    - when: manual
  before_script:
     - sudo docker login -u "$REG_USER" -p "$REG_PASSWORD"
  script:
    - export
    - tag='$TAG'
    - echo "Running build with tag $TAG "
    - sudo docker build -t mazespd/sprint2:$TAG .
    - sudo docker push mazespd/sprint2:$TAG

app_deploy: 
  stage: deploy
  environment: production
  script:
    - tag=$TAG
    - echo "Deploy app django-pg-docker-tutorial build version $TAG"
    - export KUBECONFIG=/opt/.kube/config
    - cd app-dpdt/ && helm upgrade --install -n diplom --values templates/credentials.yaml --set service.type=NodePort app-dpdt .
    - kubectl get pods -n $CI_NAMESPACE
```

Настраиваем запуск по созданию тега.
Успешно выполненный job выглядит так:






